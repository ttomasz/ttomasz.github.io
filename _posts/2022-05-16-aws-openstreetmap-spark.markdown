---
layout: post
title:  "Analyse OpenStreetMap data published by AWS open data programme locally with Spark"
date:   2022-05-16 23:01:07 +0200
author: tomasz
tags: OpenStreetMap Spark
---

AWS [recently announced](https://aws.amazon.com/de/blogs/big-data/querying-openstreetmap-with-amazon-athena/) that OpenStreetMap data will be available among their open datasets. If you don't want to use it in the cloud with Athena and pay per query like they described in their blog post you can download the data for free and analyse it locally with Spark.

# Download the data

URLs from the blog post are in `s3://` format. You can use AWS CLI with a free account to download the file using command like: `aws s3 cp s3://osm-pds/planet/planet-latest.orc ./planet-latest.orc` or just download it via http using URL: `https://osm-pds.s3.amazonaws.com/planet/planet-latest.orc` (this is url for file with newest data - no history).

# Prepare tools

We'll use PySpark with Jupyter Lab and Sedona. Sedona (formerly GeoSpark) is an extension to Spark - like PostGIS is to PostgreSQL - that supports spatial data and operations. It contains function for working with geometries and allows creating spatial index that can make some operations much faster.

Here are commands that you need to prepare the environment (on linux):
```bash
# create python virtual env
python3 -m venv venv_pyspark
# activate it
source venv_pyspark/bin/activate
# install pyspark, sedona, and jupyter lab
pip install --upgrade pip
pip install pyspark pyspark[sql] shapely attrs apache-sedona jupyterlab
```

Then just launch jupyter by executing:
```bash
jupyter lab
```

On the first run we'll need to download additional libraries for Spark.

Create a notebook and run the following:
```python
from pyspark.sql import SparkSession

spark = SparkSession. \
    builder. \
    config('spark.jars.packages',
           'org.apache.sedona:sedona-python-adapter-3.0_2.12:1.2.0-incubating,'
           'org.datasyslab:geotools-wrapper:1.1.0-25.2'). \
    getOrCreate()
# on the first run maven will download necessary jars
```

On subsequent runs you'll use this to initialize session:
```python
from multiprocessing import cpu_count

from pyspark.sql import SparkSession
from sedona.utils import SedonaKryoRegistrator, KryoSerializer
from sedona.register import SedonaRegistrator

spark = SparkSession.\
    builder.\
    master(f"local[{cpu_count() - 1}]").\
    appName("Sedona App").\
    config("spark.executor.memory", "3g").\
    config("spark.driver.memory", "5g").\
    config("spark.memory.offHeap.enabled", "true").\
    config("spark.memory.offHeap.size", "3g").\
    config("spark.serializer", KryoSerializer.getName).\
    config("spark.kryo.registrator", SedonaKryoRegistrator.getName).\
    config("spark.jars.packages", "org.apache.sedona:sedona-python-adapter-3.0_2.12:1.2.0-incubating,org.datasyslab:geotools-wrapper:1.1.0-25.2").\
    getOrCreate()
```

You can adjust memory settings and available threads.

Then activate Sedona extension by running:
```python
# Register function is essential for Apache Sedona Core and Apache Sedona SQL. 

SedonaRegistrator.registerAll(spark)
```

You can run these commands to check if Spark session was initialized correctly:
```python
spark.version
```
```python
from sedona import version as sedona_version
sedona_version
```

I also recommend installing ipython extension that will automatically measure how long commands run:
```python
try:
    %load_ext autotime
except:
    !pip install ipython-autotime
    %load_ext autotime
```

# Query the data

First create the DataFrame. Spark loads data lazily which means that it won't try to load all the data to RAM as e.g. Pandas would.

```python
planet = spark.read.orc("planet-latest.orc")
```

You can see columns and their datatypes with this command:
```python
planet.dtypes
```

Let's count how many rows there are:
```python
print('{:,d}'.format(planet.count()))
```

This file does not include data that was marked in OpenStreetMap as deleted. We can verify that by running:
```python
planet.groupby("visible").count().show()
```

We can print a few rows to see how the data looks.
This custom function will format it a little nicer than what `planet.head()` or `planet.show()` would:
```python
rows = planet.head(2)
columns = planet.columns


def pretty_print_row(row, columns):
    col_max_len = max(map(len, list(columns)))

    def separator(key: str, max_len: int = col_max_len) -> str:
        return " " * (max_len - len(key)) + "="

    def mask(user: str) -> str:
        return user[:1] + "..."

    for key in columns:
        if key == "user":
            print(key, separator(key), mask(row["user"]))
        elif key == "tags":
            tags = row["tags"]
            print(key, separator(key), "{")
            tags_obj = dict(tags)
            key_max_len = max(map(len, tags_obj.keys()))
            for k, v in tags_obj.items():
                print("   ", k, separator(k, key_max_len), v)
            print("}")
        else:
            print(key, separator(key), row[key])

for row in rows:
    pretty_print_row(row, columns)
    print("-" * 25)
```

Some basic DataFrames which would be used in further queries:
```python
from pyspark.sql import functions as F

# create dataframes with basic types
raw_nodes = (
    planet
    .drop("visible", "members", "nds")
    .filter(F.col("type") == "node")
    .withColumnRenamed("id", "node_id")
)
raw_ways = (
    planet
    .drop("visible", "members", "lat", "lon")
    .filter(F.col("type") == "way")
    .withColumnRenamed("id", "way_id")
)
raw_relations = (
    planet
    .drop("visible", "nds", "lat", "lon")
    .filter(F.col("type") == "relation")
    .withColumnRenamed("id", "relation_id")
)

# register them as views in case we want to query them using SQL
raw_nodes.createOrReplaceTempView("raw_nodes")
raw_ways.createOrReplaceTempView("raw_ways")
raw_relations.createOrReplaceTempView("raw_relations")


# create dataframes utilizing geospatial types
nodes = spark.sql("""
    SELECT *, ST_SetSRID(ST_Point(lon, lat), 4326) as geom
    FROM raw_nodes
""")
nodes.createOrReplaceTempView("nodes")
```

# Recommendations for dealing with ways

While `nodes` are pretty easy to handle `ways` and `relation` are more tricky as they require joins.

I did not test these queries but they should provide a starting point to building queries that handle `ways`.

```python

# since Sedona doesn't support ST_MakeLine function yet
# we will have to create text representations of lines
# which will be parsed by ST_LineStringFromText
sep = ","
temp_nodes = (
    raw_nodes
    .withColumn("lonlat_txt", F.concat_ws(sep, "lon", "lat"))
    .select("node_id", "lon", "lat", "lonlat_txt")
)


temp_node_way_bridge = (
    raw_ways
    .withColumn("nds", F.transform("nds", lambda x: x["ref"]))
    .select("way_id", F.posexplode("nds"))
    .withColumnRenamed("col", "node_id")
)

way_geom = (
    temp_node_way_bridge
    .join(temp_nodes, on="node_id")
    .groupby("way_id")
    .agg(
        F.collect_list("lonlat_txt").alias("coords"),
        F.avg("lon").alias("centroid_lon"),
        F.avg("lat").alias("centroid_lat"),
        F.min("lon").alias("min_lon"),
        F.min("lat").alias("min_lat"),
        F.max("lon").alias("max_lon"),
        F.max("lat").alias("max_lat"),
    )
    .withColumn("coordinates", F.array_join("coords", sep))
    .selectExpr(
        "way_id",
        "ST_LineStringFromText(coordinates, ',') as geom",
        "min_lon",
        "min_lat",
        "max_lon",
        "max_lat",
    )
)
```

Unfortunately joining `nodes` back to `ways` is slow and inconvenient. Shame that AWS did not do it while creating the dataset.

One potential optimization would be to create a Hive table with nodes (just id and coordinates) that is already bucketed and sorted by nodes' ids.
This could potentially make the join much faster however I was not able to achieve that in my short test. Queries I used were:
```python
# write nodes locations to separate table
# .repartition(F.expr("pmod(hash(node_id), 200)")) is to create only one file per bucket
# otherwise number of files would explode
(
  temp_nodes.repartition(F.expr("pmod(hash(node_id), 200)"))
  .write
  .mode("overwrite")
  .bucketBy(200, "node_id")
  .sortBy("node_id")
  .option("path", "/home/tomasz/PycharmProjects/osm_tests/spark_data/nodes_cache/")
  .saveAsTable("nodes_cache")
)
# this took about 240GB it would be probably much less if I didn't write coordinates converted to string to the files

nodes_cache = spark.sql("select * from nodes_cache")

spark.conf.set("spark.sql.legacy.bucketedTableScan.outputOrdering", "true")

way_geom2 = (
    temp_node_way_bridge
    .join(nodes_cache, on="node_id")
    .groupby("way_id")
    .agg(
        F.collect_list("lonlat_txt").alias("coords"),
        F.avg("lon").alias("centroid_lon"),
        F.avg("lat").alias("centroid_lat"),
        F.min("lon").alias("min_lon"),
        F.min("lat").alias("min_lat"),
        F.max("lon").alias("max_lon"),
        F.max("lat").alias("max_lat"),
    )
    .withColumn("coordinates", F.array_join("coords", sep))
    .selectExpr(
        "way_id",
        "ST_LineStringFromText(coordinates, ',') as geom",
        "min_lon",
        "min_lat",
        "max_lon",
        "max_lat",
    )
)
```
