---
layout: post
title:  "Read XML using Spark - example for parsing OSM changeset data"
date:   2023-01-30 21:56:53 +0100
author: tomasz
tags: OpenStreetMap Spark
---

__Edit: fixed xml schema which had an error__

Databricks [created a library](https://github.com/databricks/spark-xml) that allows Spark to parse XML files.

Here I'll convert OpenStreetMap dump of [changesets](https://wiki.openstreetmap.org/wiki/Changeset) that can be downloaded from [planet.osm.org](https://planet.osm.org).
XML file is compressed using `BZ2` codec but thankfully Spark handles that out of the box.

Example will use Python but Scala code would look very similar.

### Create session

```python
from multiprocessing import cpu_count

from pyspark.sql import SparkSession

spark = (
    SparkSession.
    builder
    .master(f"local[{cpu_count() - 1}]")
    .appName("Spark App")
    .config('spark.driver.extraJavaOptions', '-Duser.timezone=UTC')  # important so timestamps won't be auto converted to different TZ
    .config('spark.executor.extraJavaOptions', '-Duser.timezone=UTC')
    .config('spark.sql.session.timeZone', 'UTC')
    .config("spark.executor.memory", "3g")  # some params that theoretically should let spark use more memory, facultative
    .config("spark.driver.memory", "10g")
    .config("spark.memory.offHeap.enabled", "true")
    .config("spark.memory.offHeap.size", "3g")
    .config('spark.jars.packages', 'com.databricks:spark-xml_2.12:0.16.0')  # use Databricks XML library
    .getOrCreate()
)
```

### Prepare schema

While spark can autodetect schema it's very slow as it would need to decompress and parse the entire file first.
We'll define schema so it's faster.

Fragment of XML file:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<osm license="http://opendatacommons.org/licenses/odbl/1-0/" copyright="OpenStreetMap and contributors" version="0.6" generator="planet-dump-ng 1.2.4" attribution="http://www.openstreetmap.org/copyright" timestamp="2023-01-16T01:00:02Z">
 <bound box="-90,-180,90,180" origin="http://www.openstreetmap.org/api/0.6"/>
 <changeset id="1" created_at="2005-04-09T19:54:13Z" closed_at="2005-04-09T20:54:39Z" open="false" user="Steve" uid="1" min_lat="51.5288506" min_lon="-0.1465242" max_lat="51.5288620" max_lon="-0.1464925" num_changes="2" comments_count="33"/>
 <changeset id="909337" created_at="2009-04-23T08:06:48Z" closed_at="2009-04-23T08:06:51Z" open="false" user="Alberto58" uid="91650" min_lat="44.4486102" min_lon="11.6230873" max_lat="44.4623357" max_lon="11.6425428" num_changes="40" comments_count="0">
  <tag k="comment" v="Via Graffio"/>
  <tag k="created_by" v="JOSM"/>
 </changeset>
 </osm>
```

Spark schema:
```python
from pyspark.sql.types import StructType, StructField, StringType, IntegerType, TimestampType, DoubleType, ArrayType
from pyspark.sql import functions as f

schema = StructType([
    StructField("id", IntegerType(), nullable=False),
    StructField("created_at", TimestampType()),
    StructField("closed_at", TimestampType()),
    StructField("open", StringType()),
    StructField("user", StringType()),
    StructField("uid", IntegerType()),
    StructField("min_lat", DoubleType()),
    StructField("min_lon", DoubleType()),
    StructField("max_lat", DoubleType()),
    StructField("max_lon", DoubleType()),
    StructField("num_changes", IntegerType()),
    StructField("comments_count", IntegerType()),
    StructField("tag", ArrayType(StructType([
        StructField("k", StringType()),
        StructField("v", StringType()),
    ]))),
])
```

### Create DataFrame

```python
df = (
    spark.read
    .format("com.databricks.spark.xml")
    .schema(schema)
    .option("mode", "FAILFAST")  # throw an error if any row can't be parsed
    .option("rootTag", "osm")
    .option("rowTag", "changeset")
    .option("inferSchema", "false")  # we specify schema manually above
    .option("ignoreNamespace", "true")  # no namespaces but why not be explicit
    .option("attributePrefix", "")  # don't add any prefix to attributes as we know there won't be any conflict in names
    .load("/home/tomasz/Downloads/changesets-230116.osm.bz2")
)
```

### Adjust DataFrame format

Possible there is a way to specify schema for MapType straight away but I was too lazy to try so we are reading tags into StructType and then convert to MapType.
```python
df = (
    df
    # change struct to map - it will be easier to query later
    .withColumn("tags", f.when(f.col("tag").isNotNull(), f.create_map("tag.k", "tag.v")).otherwise(None)).drop("tag")
    # add column with year which we will use to partition dataset
    .withColumn("opened_year", f.year(f.col("created_at")))
    # standardize "open" column to boolean
    .withColumn("open", f.when(f.col("open") == f.lit("true"), True).when(f.col("open") == f.lit("false"), False).otherwise(None))
    # drop user name
    .drop("user")
)
```
```python
# pull out some commonly used values from map into separate columns to make querying faster
df = (
    df
    .withColumn("created_by", f.col("tags")["created_by"])
    .withColumn("source", f.col("tags")["source"])
    .withColumn("locale", f.col("tags")["locale"])
    .withColumn("bot", f.col("tags")["bot"])
    .withColumn("review_requested", f.col("tags")["review_requested"])
    .withColumn("hashtags", f.col("tags")["hashtags"])
)
```

### Save DataFrame to Parquet

Normally you wouldn't repartition the dataset but for my use case I wanted to have one file per partition.
```python
(
    df
    .repartition("opened_year")
    .write
    .partitionBy("opened_year")
    .mode("overwrite")
    .format("parquet")
    .option("spark.sql.parquet.compression.codec", "zstd")
    .save("/home/tomasz/PycharmProjects/sedona_xml_bz2/parquet_partitioned_1_file_per_partition/")
)
```
