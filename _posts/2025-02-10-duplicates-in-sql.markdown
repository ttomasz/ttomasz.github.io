---
layout: post
title: "Duplicates in a dataset and how to handle them in SQL"
date: 2025-02-10 22:42:36 +0100
author: tomasz
tags: data SQL
---

When working with data you are quite likely to encounter duplicates. There are many ways to handle them, but it all depends on what you are trying to accomplish. In this post I'll try to present some ways you can try to approach it in SQL.

# What is a duplicate?
This may sound like a dumb question - duplicate is a repeated item, duh - but it can actually be quite complex.

In the simplest case it can be simple to spot and remove them. Imagine you have a list of names like this:

| name |
|------|
| Adam |
| Eve  |
| John |
| Anna |
| Anna |

We see that value "Anna" is there exactly twice. In SQL it would be as simple as `SELECT DISTINCT name FROM <table_name>` to get only unique strings.

But what if the values are not quite the same:

| name |
|------|
| Adam |
| Eve  |
| John |
| Anna |
| anna |

Human brain can easily see that "Anna" and "anna" refers to the same name, it's just that the letter case is different. In the most simple of cases you can try to standardize the string by e.g. casting all letter to lowercase ( `SELECT lower(name) as name_standardized FROM <table_name>` ), trimming the strings to strip them of whitespace characters at the beginning and end or replacing some characters like slashes, dashes, spaces etc.

⚠️ Note:
If you're working with English alphabet only - which fits into ASCII - it may seem easy to change letter case since all letters have direct mappings: a -> A, B -> b, etc. But if you work with full unicode character set and can encounter accented letter from other alphabets then it can get more complex (the same goes for sorting, I recommend [talk by Dylan Beattie: There's No Such Thing As Plain Text](https://www.youtube.com/watch?v=ajfb5LSbQVM) to get a better perspective). Also not every tool works with unicode case transformations out of the box, infamous example is SQLite which by default is not compiled with full utf-8 support and if you try to run query like: `SELECT lower('Ślęża')` instead of expected string "ślęża" you [will get "Ślęża"](https://dbfiddle.uk/Zph0RMDU).

But getting back to the example of names... what if you happen to have misspellings?

|  name  |
|--------|
|  Adam  |
|  Eve   |
|  John  |
|  Joohn |
|  Jhon  |

You can try to compare the strings using fuzzy matching techniques like checking their [Edit distance](https://en.wikipedia.org/wiki/Edit_distance) using one of the algorithms like [Levenshtein distance](https://en.wikipedia.org/wiki/Levenshtein_distance) but that can get complicated so I will not consider such cases here. If you are interested you can try reading more about [Record linkage](https://en.wikipedia.org/wiki/Record_linkage).

Another common scenario is when we have multiple versions of a records capturing how it changed over time.

Let's make an example with a list of products with their attributes:

| product_name               | created_at          | modified_at         | amount | units | requires_cold_storage | promoted |
|----------------------------|---------------------|---------------------|--------|-------|-----------------------|----------|
| Bottled Mineral Water 1,5l | 2025-01-02T06:53:11 | 2025-01-02T06:53:11 | 1,5    | l     |                       | false    |
| Frozen Vegetable Mix       | 2025-01-02T06:55:40 | 2025-01-02T06:55:40 | 450    | g     | true                  | true     |
| Toothbrush                 | 2025-01-02T07:12:55 | 2025-01-02T07:12:55 | 1      | pcs   | false                 | false    |
| Bottled Mineral Water 1,5l | 2025-01-02T06:53:11 | 2025-01-03T15:01:32 | 1,5    | l     | false                 | true     |
| Frozen Vegetable Mix       | 2025-01-02T06:55:40 | 2025-01-04T00:00:00 | 450    | g     | true                  | false    |

In this example let's assume that product's name is stable and together with "created_at" it uniquely identifies our product entity.

As we can see we have multiple entries for "Bottled Mineral Water 1,5l" and "Frozen Vegetable Mix". However beside "created_at" other attributes may differ so a simple `SELECT DISTINCT * FROM <table_name>` won't get rid of the duplicates if we want to see only one row per product.

If the dataset is large enough that we can't just look at the values - which would usually be the case - but we know that some combination of columns should be unique then we may first want to see how many cases we have. This simple SQL query will show us all of the product entities and how many times did they appear:

```sql
SELECT
  product_name,
  created_at,
  count(*) as cnt -- number of time product shows up in this dataset
FROM <table_name>
GROUP BY
  product_name,
  created_at
```

If we want to filter only those that appear more than once we can add a filter on the aggregate:

```sql
SELECT
  product_name,
  created_at,
  count(*) as cnt -- number of time product shows up in this dataset
FROM <table_name>
GROUP BY
  product_name,
  created_at
HAVING count(*) > 1 -- only show products that appear more than once
```

The result would look like this:

| product_name               | created_at          | cnt |
|----------------------------|---------------------|-----|
| Bottled Mineral Water 1,5l | 2025-01-02T06:53:11 | 2   |
| Frozen Vegetable Mix       | 2025-01-02T06:55:40 | 2   |

Now let's try two scenarios. First let's try to show all rows that have duplicates.

# See all duplicated records (self-join)
To do that we'll use the result of the previous query and join in back to the same table. Inner join filters only matching rows so we'll get a result that is filtered only to the matching records but showing them in their entirety unlike the previous query which just counts them.

```sql
WITH
duplicates as (
    SELECT
        product_name,
        created_at
    FROM <table_name>
    GROUP BY
        product_name,
        created_at
    HAVING count(*) > 1
)
SELECT t.*
FROM <table_name> as t
JOIN duplicates USING(product_name, created_at)
```

The same could be achieved using a subquery instead of CTE but CTEs are more readable. If you're unfamiliar with `USING` keyword it's the same as `ON t.product_name=duplicates.product_name and t.created_at=duplicates.created_at` (with an added bonus that these columns won't considered ambiguous in SELECT) and it's part of SQL standard. You can try [DBFiddle](https://dbfiddle.uk/PFTLoH6H) as an interactive example. The result will be:

| product_name               | created_at             | modified_at            | amount | units | requires_cold_storage | promoted |
|----------------------------|------------------------|------------------------|--------|-------|-----------------------|----------|
| Bottled Mineral Water 1,5l | 2025-01-02 06:53:11 | 2025-01-02 06:53:11 | 1,5    | l     |                       | false    |
| Frozen Vegetable Mix       | 2025-01-02 06:55:40 | 2025-01-02 06:55:40 | 450    | g     | true                  | true     |
| Bottled Mineral Water 1,5l | 2025-01-02 06:53:11 | 2025-01-03 15:01:32 | 1,5    | l     | false                 | true     |
| Frozen Vegetable Mix       | 2025-01-02 06:55:40 | 2025-01-04 00:00:00 | 450    | g     | true                  | false    |

This can be useful when exploring dataset and trying to figure out why do we have the duplicates.

Now let's try to select only the newest row.

# Select one record (rank and filter)

We could use previously shown method except instead of joining only on key attributes we could also select `max(modified_at) as modified_at` and add that to the join condition to get only row matching record with newest timestamp.

However there is another approach using analytical functions (a.k.a. window functions) to rank rows. Newest row would get rank one, second newest would get rank two, etc. Then we would select only the rows where the rank would equal one. Here's the query:

```sql
WITH
ranked as (
    SELECT *, row_number() over(partition by product_name, created_at order by modified_at desc) as rn
    FROM products
)
SELECT
    product_name,
    created_at,
    modified_at,
    amount,
    units,
    requires_cold_storage,
    promoted
FROM ranked
WHERE rn = 1
```

Result:

| product_name               | created_at          | modified_at         | amount | units | requires_cold_storage | promoted |
|----------------------------|---------------------|---------------------|--------|-------|-----------------------|----------|
| Toothbrush                 | 2025-01-02T07:12:55 | 2025-01-02T07:12:55 | 1      | pcs   | false                 | false    |
| Bottled Mineral Water 1,5l | 2025-01-02T06:53:11 | 2025-01-03T15:01:32 | 1,5    | l     | false                 | true     |
| Frozen Vegetable Mix       | 2025-01-02T06:55:40 | 2025-01-04T00:00:00 | 450    | g     | true                  | false    |

If you want to play with interactive version try [DBFiddle](https://dbfiddle.uk/AC63PGso).

Some databases support `QUALIFY` keyword which allows filtering on result of window function. Similarly how `HAVING` allows filtering on result of aggregate function. If the query engine that we use supports that then the query could be as simple as:

```sql
SELECT *
FROM products
QUALIFY row_number() over(partition by product_name, created_at order by modified_at desc) = 1
```

# What about DISTINCT ON ?
Some databases extend `DISTINCT` keyword to allow specifying columns to use for duplicate check and selecting another set of columns.

To quote from [PostgreSQL's documentation](https://www.postgresql.org/docs/current/sql-select.html#SQL-DISTINCT):
> SELECT DISTINCT ON ( expression [, ...] ) keeps only the first row of each set of rows where the given expressions evaluate to equal. The DISTINCT ON expressions are interpreted using the same rules as for ORDER BY (...). Note that the “first row” of each set is unpredictable unless ORDER BY is used to ensure that the desired row appears first.

It can shorten the query but should be used with care since omitting `ORDER BY` will lead to **indeterministic results**. This is important because if you put random data in you analysis then you will get garbage results. Example query:

```sql
SELECT DISTINCT ON (product_name, created_at) *
FROM products
ORDER BY product_name, created_at, modified_at desc
```

# Epilogue
There are many approaches to dealing with duplicates in your data but the important thing is to understand what classifies as a duplicate in your case and where do they come from. You should be sure that the method you chose will lead to correct and deterministic results. As they say: [Garbage in, Garbage out](https://en.wikipedia.org/wiki/Garbage_in,_garbage_out).
