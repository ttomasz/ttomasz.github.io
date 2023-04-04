---
layout: post
title: "Run SQL queries with DuckDB in Jupyter over Parquet files stored in Azure ADLS storage"
date: 2023-04-04 21:33:01 +0200
author: tomasz
tags: DuckDB Jupyter
---

You can use [DuckDB](https://duckdb.org/) to directly query Parquet files on S3 thanks to [HTTPFS extension](https://duckdb.org/docs/extensions/httpfs). Unfortunately the same is not so easy for Azure's ADLS Gen. 2 Storage (S3-like service but on Azure). However thanks to the recent addition of support for Python's [Filespec](https://filesystem-spec.readthedocs.io/en/latest/) compatible libraries we can do it and here's an instruction how to configure Jupyter to do it.

First let's install all required libraries:
```bash
pip install duckdb jupyterlab jupysql duckdb-engine pandas adlfs ipython-autotime
```

Then you can start Jupyter Lab server by running:
```bash
jupyter lab
```

Create a notebook and execute these commands to configure the environment:
```python
%load_ext autotime
```
This is not strictly necessary but it will provide a nice timer under each cell.

Note: if you need more verbose logging you can enable it by running:
```python
import logging

logging.basicConfig(level=logging.DEBUG)
```

First let's load [JupySQL](https://jupysql.ploomber.io/en/latest/quick-start.html) extension that adds `%sql` magic commands:
```python
%load_ext sql
```

Useful settings that make the output more readable:
```python
import pandas as pd

pd.set_option("display.max_rows", None)
pd.set_option("display.max_columns", None)
pd.set_option("display.width", None)
pd.set_option("display.max_colwidth", None)

# configure JupySQL to return data as a Pandas DataFrame and have less verbose output
%config SqlMagic.autopandas = True
%config SqlMagic.feedback = False
%config SqlMagic.displaycon = False
```

[ADLFS](https://github.com/fsspec/adlfs) library allows authenticating to Azure in multiple ways. One simple way is to generate a SAS token with read-only permissions on a container.

Let's finish configuration:
```python
from adlfs import AzureBlobFileSystem
from sqlalchemy import create_engine

# in case you need to be connected to corporate proxy to access the data you can do it by running:
# import os
# os.environ["http_proxy"] = "<some_url>"
# os.environ["https_proxy"] = "<some_url>"

fs = AzureBlobFileSystem(
    account_name="my_storage_account",
    sas_token="?xxx",
)

engine = create_engine("duckdb:///:memory:")
engine.raw_connection().register_filesystem(fs)

# register SQLAlchemy engine with JupySQL
%sql engine
```

Check DuckDB version or settings:
```python
%sql select version()
```
```python
%sql select current_setting('threads');
```

You can use either `%sql` for single line queries or `%%sql` for multiline queries.

** Important note: Protocol needs to be `abfs://` not `abfss://` like in Apache Spark. **

You can run queries like:
```python
%%sql
SELECT *
FROM parquet_scan('abfs://container@my_storage_account.dfs.core.windows.net/directory/dataset/*/*.parquet', hive_partitioning=1)
LIMIT 5 
```

You can also save the result to a DataFrame:
```python
%%sql df <<
SELECT *
FROM parquet_scan('xxx')
```
```python
for col in df.columns:
    print(col)
```
