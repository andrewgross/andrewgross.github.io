Title: Simple lists to dataframes for PySpark
Date: 2023-11-30 2:38

Here’s a simple helper function I can’t believe I didn’t write sooner:

----------

```
import pandas as pd
import pyspark

from yipit_databricks_utils.helpers.pyspark_utils import get_spark_session

def list_to_df(items: list, column: str, unique=False) -> pyspark.sql.DataFrame:
    """
    Converts a python list to a dataframe with the given column name

    Pass unique=True to only get distinct values. Order is not guaranteed to be preserved.
    """
    if unique:
        items = list(set(items))
    df = pd.DataFrame(items, columns=[column])
    spark = get_spark_session()
    return spark.createDataFrame(df)

def df_to_list(df: pyspark.sql.DataFrame, column: str, unique=False) -> list:
    """
    Converts a column of a dataframe to a python list.

    Pass unique=True to only get distinct values. Order is not guaranteed to be preserved.
    """
    values = df.select(column)
    if unique:
        values = values.distinct()
    return list(map(lambda row: row[0], values.collect()))
```

---------

**NOTE:** `get_spark_session` is just a helper function to ensure we have spark defined. It is often already defined in many environments.

These functions help bridge the gap between python code and dataframes. Something we have noticed in the past with Databricks is that is can be difficult to get small amounts of data into the system. There is plenty of support in general but the ergonomics of adding it can be less that ideal. These functions are another option to smooth that process.

The existing workflow for this usually involves googling the question and getting an out of date response with RDDs or one that requires setting explicit types for PySpark. I find it to be a lot simpler to use Pandas to do the type inference and let Spark convert the pandas dataframe.

*This post was originally published on my medium blog.*