Title: Accidentally Leaking Private Registries via UV
Date: 2025-06-28 7:35

I was messing around with Cursor Background agents to see what the fuss was about and I managed to (nearly) leak some private registry credentials.  In my work environment, we use a managed private Python package registry. This works similar to PyPI but we can also keep private packages.  It allows use to have a read through cache to PyPI, while getting a degree of auditability and security in addition to having private packages. 

As part of our dev setup, to ensure that we are always using this registry, we update our `.pypirc` or other equivalent configuration to make sure that any `pip install` or `uv add` will automatically fetch the appropriate endpoint.  This also handles authentication, API keys etc.  This is rather conventient and one of those things you setup once and forget even exists until there is a problem.

In my experiments with Cursor Background agents, I focused on a small open source [project](https://github.com/andrewgross/pyspark_transform_registry) I was working on to keep a tight scope and minimize the security scope.  One key thing for the agent setup is of course the environment. I leaned on `uv` to perform [setup](https://github.com/andrewgross/pyspark_transform_registry/blob/main/.cursor/environment.json) due to speed and consistency.  This allowed me to have a pretty minimal installation by leveraging my UV lock file to repliace my local environment.

Sounds great. UV doing a good job as usual. That was until I tried to actually run the agent. I received an error that `uv` couldn't install my packages as it was getting an `HTTP 404` from the endpoint.   Pretty usual behavior for PyPI.  Further digging revealed that the cause of the issue was my `uv.lock` file had been created using links to my private package registry, even though all packages being used were public.  As the Background Agent container did not have any credentials to access this registry, it failed to install the packages.

The fix was rather simple, just update `pyproject.toml` to force UV to use PyPI by default for this project and regenerate the lock file:

```
[[tool.uv.index]]
name = "pyspark"
url = "https://pypi.org/simple"
default = true
```

Thankfully, no credentials were exposed, it would have just been paths to packages, though it seems like people ran into this [issue](https://github.com/astral-sh/uv/issues/8296) with UV in the past.  Thankfully Astral quickly fixed the issue to minimize the chance of leaking credentials.

A good reminder of how global dev environment setups can still be dangerous, though convenient.

**Bonus:** Cursor Backround agent docks are pretty lacking at the moment. Feel free to steal the linked env file above, or this [Dockerfile](https://github.com/andrewgross/pyspark_transform_registry/blob/main/.cursor/Dockerfile) to get started with installs for UV and system packages.