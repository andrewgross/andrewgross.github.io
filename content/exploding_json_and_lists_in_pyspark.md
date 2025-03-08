Title: Exploding JSON and Lists in Pyspark
Date: 2025-02-28 00:00

JSON can kind of suck in PySpark sometimes. It is often that I end up with a dataframe where the response from an API call or other request is stuffed into a column as a JSON string. It will have a consistent structure but it was either not known or consistent enough ahead of time to be worth expanding before adding it to the dataframe.  The problem I run in to is: *How do I dynamically make that JSON into columns and rows?*

Usually the process involves manually looking at the JSON, figuring out what columns I care about, and then hacking some code to parse the JSON and extract what I need. Its pretty boring and repetitive. It is especially painful when the JSON has a large number of fields you want to extract. Too much typing and too easy to make mistakes. The final straw is when you have lists in the JSON that you want to expose as separate rows.  

To (hopefully) deal with this once more for all time, I have made some generic helper functions to parse JSON in a few formats. For background, this code was developed with the help of, but not exclusively by GPT-4o.

## Finding our schema

Pyspark has some more niche functions you can use, and some weirdness around how it likes things defined. Our first trick is to combine `F.schema_of_json` with filtering for the first non null value to identify our JSON schema. Obviously, this means that if our JSON schema is not consistent between rows, things will go sideways.  Once we have the schema, we can use it to populate a new column with the parsed JSON.

```
schema_df = df.withColumn(
    "struct_schema", F.schema_of_json(F.col(json_string_column))
).select("struct_schema")

schema = schema_df.filter(F.col("struct_schema").isNotNull()).first()[
    "struct_schema"
]
df = df.withColumn(column, F.from_json(F.col(json_string_column), schema=schema))
```

## Exploding JSON

Next up, we need to take all of our top level JSON keys and make them in to columns.  This is pretty simple, with just needing to know that certain functions and classes exist to be called. The full code has additional error handling and deals with the map type, but since that uses RDDs I suspect it wouldn't work in production anyways.

```
schema = df.schema[parsed_json_column].dataType

for field in schema.fields:
    df = df.withColumn(field.name, F.col(f"{parsed_json_column}.{field.name}"))
```

## Pesky List Items

The real kicker in a lot of these situations is nested data. It is nice to explode the JSON as columns but things get a bit more painful when we have variable length lists of data that can be inside JSON keys. These can be accessed directly via `foo.bar[1]` but since they have variable length it is really painful to do aggregations across them without hacks like exploding exactly 10 rows and filling gaps with nulls.  To get around this, we can explode the lists into individual rows.  We can do this for multiple columns, although it definitely gets a bit messy if there are lots of relevant columns. In addition, we want to make sure we retain ordering indexes to make it simpler to select only a single row from each record when aggregating outside of the exploded items.

There are two functions below to handle these cases. One for when there are multiple list columns that have corresponding data, so we can zip them together and provide a unified index. Another for when we have a list column where each element is a map, and we want to expand those items out as additional columns with a special prefix. Neither approach is perfect, and this code won't handle all edge cases, but it is a starting point.

# Full Code

```python
import pyspark
import pyspark.sql.functions as F
from pyspark.sql.types import ArrayType

## Shared Functions

def map_json_column(
    df: pyspark.sql.DataFrame, column: str, drop=True
) -> pyspark.sql.DataFrame:
    """
    Takes a column of JSON strings and remaps them to a Map type by
    inferring the schema from the first row.
    """
    raw_column = f"{column}_raw"
    df = df.withColumnRenamed(column, raw_column)
    schema_df = df.withColumn(
        "struct_schema", F.schema_of_json(F.col(raw_column))
    ).select("struct_schema")

    schema = schema_df.filter(F.col("struct_schema").isNotNull()).first()[
        "struct_schema"
    ]
    df = df.withColumn(column, F.from_json(F.col(raw_column), schema=schema))
    if drop:
        df = df.drop(raw_column)
    return df


def extract_json_keys_as_columns(
    df: pyspark.sql.DataFrame, json_column: str
) -> pyspark.sql.DataFrame:
    """
    Extracts each top-level key from a parsed JSON column and creates separate columns for each key.

    :param df: Input PySpark DataFrame
    :param json_column: Name of the column containing parsed JSON (as MapType or StructType)
    :return: DataFrame with top-level keys extracted as individual columns
    """
    # Get schema of the parsed JSON column
    schema = df.schema[json_column].dataType

    if not isinstance(
        schema, (pyspark.sql.types.StructType, pyspark.sql.types.MapType)
    ):
        raise ValueError(f"Column '{json_column}' must be of StructType or MapType.")

    # Extract top-level keys as new columns
    if isinstance(schema, pyspark.sql.types.StructType):
        for field in schema.fields:
            df = df.withColumn(field.name, F.col(f"{json_column}.{field.name}"))
    elif isinstance(schema, pyspark.sql.types.MapType):
        keys = (
            df.select(F.map_keys(F.col(json_column)))
            .rdd.flatMap(lambda x: x[0])
            .distinct()
            .collect()
        )
        for key in keys:
            df = df.withColumn(key, F.col(f"{json_column}[{key}]"))
    return df


## Multi Line Item Column Functions

def explode_all_list_columns(df: pyspark.sql.DataFrame) -> pyspark.sql.DataFrame:
    """
    Explodes all columns in the DataFrame that are lists (ArrayType) into rows,
    ensuring they are exploded together. Adds an 'index' column representing the
    array index of the exploded value.

    ...
    "items": [1, 2, 3],
    "cost": [a, b, c],
    ...

    :param df: Input PySpark DataFrame
    :return: DataFrame with exploded rows and an 'index' column
    """
    list_columns = [
        field.name
        for field in df.schema.fields
        if isinstance(field.dataType, ArrayType)
    ]

    if not list_columns:
        raise ValueError("No columns with ArrayType found in the DataFrame.")

    temp_col = "temp_struct"
    df = df.withColumn(temp_col, F.arrays_zip(*[F.col(col) for col in list_columns]))

    # Use select with posexplode to extract both index and values
    exploded_df = df.select(
        *[
            col for col in df.columns if col != temp_col
        ],
        F.posexplode_outer(F.col(temp_col)).alias("index", "temp_struct_exploded"),
    )

    for col in list_columns:
        exploded_df = exploded_df.withColumn(col, F.col(f"temp_struct_exploded.{col}"))

    return exploded_df.drop("temp_struct", "temp_struct_exploded")


def clean_dataframe_with_separate_line_item_lists(
    df: pyspark.sql.DataFrame,
    raw_response_col: str = "raw_response",
) -> pyspark.sql.DataFrame:
    """
    Cleans up the invoice DataFrame by:
    1. Parsing the JSON column
    2. Extracting top-level keys as separate columns
    3. Exploding list columns into rows

    :param df: Input PySpark DataFrame
    :return: Cleaned up DataFrame
    """
    df = map_json_column(df, raw_response_col)
    df = extract_json_keys_as_columns(df, raw_response_col)
    df = explode_all_list_columns(df)
    return df


## Single Line Item Functions

def explode_array_of_maps(
    df: pyspark.sql.DataFrame, array_col: str
) -> pyspark.sql.DataFrame:
    """
    Explodes an array column containing maps into separate columns.
    
    Takes a DataFrame with an array column containing maps and explodes it into separate rows,
    with each map's keys becoming new columns. The original array column is replaced with individual 
    columns for each key in the maps, suffixed with the original column name.
    
    ...
    items = [
        {"cost": 1.0, "other_cost": 0.12},
        {"cost": 2.0, "other_cost": 0.13},
    ]
    ...

    Args:
        df (pyspark.sql.DataFrame): Input DataFrame containing the array column
        array_col (str): Name of the array column containing maps to explode
        
    Returns:
        pyspark.sql.DataFrame: DataFrame with array column exploded into separate columns and rows,
            with an additional 'index' column indicating the position in the original array
    """
    df_exploded = df.select("*", F.posexplode_outer(F.col(array_col)).alias("index", "map"))

    # Extract the keys dynamically from the schema
    map_schema = (
        df.select(F.explode(F.col(array_col)).alias("map")).schema["map"].dataType
    )
    keys = [field.name for field in map_schema.fields]

    # Select original columns, map keys, and index as the last field
    original_cols = [F.col(col) for col in df.columns if col != array_col]
    return df_exploded.select(
        *original_cols,
        *[F.col(f"map.{key}").alias(f"{array_col}_{key}") for key in keys],
        F.col("index"),
    )


def clean_dataframe_with_single_line_item_list(
    df: pyspark.sql.DataFrame,
    line_item_col: str = "line_items",
    raw_response_col: str = "raw_response",
) -> pyspark.sql.DataFrame:
    """
    Cleans up the invoice DataFrame by:
    1. Parsing the JSON column
    2. Extracting top-level keys as separate columns
    3. Exploding line_item column into rows

    :param df: Input PySpark DataFrame
    :return: Cleaned up DataFrame
    """
    df = map_json_column(df, raw_response_col)
    df = extract_json_keys_as_columns(df, raw_response_col)
    df = explode_array_of_maps(df, line_item_col)
    return df
```