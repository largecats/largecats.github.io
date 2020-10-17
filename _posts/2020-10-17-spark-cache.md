---
layout: post
title:  "Caching in Spark"
date:   2020-10-17
categories: work
tags: spark YARN
---
<head>
    <script src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML" type="text/javascript"></script>
    <script type="text/x-mathjax-config">
        MathJax.Hub.Config({
            tex2jax: {
            skipTags: ['script', 'noscript', 'style', 'textarea', 'pre'],
            inlineMath: [['$','$']]
            }
        });
    </script>
</head>
* content
{:toc}

# Motivation   

Spark's caching mechanism can be leveraged to optimize performance. Here are some basic facts and caveats about caching.



# Basics
## Ways to cache
Dataframes or tables may be cached in the following ways.

* `df.cache()` - lazy, `df` is only evaluated after an action is called.
* `spark.catalog.cacheTable('t0')` - also lazy.
* `spark.sql('cache table t0')` - eager, `t0` is evaluated immediately.

## Ways to "uncache"
* `df.unpersist()` - convenient when there is a variable readily referencing the dataframe.
* `spark.catalog.clearCache()` - will clear all dataframes/tables cached via any of the above 3 ways.

```sh
>>> l = [('Alice', 1)]
>>> df = spark.createDataFrame(l) # create sample temp tables
>>> df
DataFrame[_1: string, _2: bigint]
>>> df1 = spark.createDataFrame(l)
>>> df2 = spark.createDataFrame(l)
>>> df.cache() # cache df lazily
DataFrame[_1: string, _2: bigint]
>>> df1.registerTempTable('df1')
>>> spark.catalog.cacheTable('df1') # cache df1 lazily
>>> df2.registerTempTable('df2')
>>> spark.sql('cache table df2') # cache df2 eagerly
DataFrame[]
>>> for (id, rdd) in spark.sparkContext._jsc.getPersistentRDDs().items(): print id # all 3 dfs are cached
...
28
18
16
>>> spark.catalog.clearCache()
>>> for (id, rdd) in spark.sparkContext._jsc.getPersistentRDDs().items(): print id # all cached dfs are cleared
...
>>>
```

# Caveats

## Temporary tables in Spark SQL are not automatically cached

Information online (e.g., [here](https://www.essentialsql.com/get-ready-to-learn-sql-server-20-using-subqueries-in-the-select-statement/) and [here](https://stackoverflow.com/questions/54737872/how-to-cache-subquery-result-in-with-clause-in-spark-sql)) says that temporary tables created in SQL are computed only once and can be reused. But the physical plans of SparkSQL shows that the temp tables are evaluated each time they are used. Consider the following query:

```python
query = '''
WITH t0 AS (
    SELECT
        *
    FROM
        selected_companies
    WHERE
        region = 'TH'
),
 
t1 AS (
    SELECT
        *
    FROM
        t0
    WHERE
        business = 1000
),
 
t2 AS (
    SELECT
        *
    FROM
        t0
    WHERE
        business = 2000
)
 
SELECT
    *
FROM
    (SELECT * FROM t1)
    UNION
    (SELECT * FROM t2)
'''
print query
df = spark.sql(query)
df.show() # triggers evaluation
```
Execution plan of this query shows that `t0` is effectively evaluated twice.

<div style="text-align: center"><img src="/images/t0_without_cache_DAG_visualization.jpg" width="800px" /></div>
<div align="center">
<sup>In particular, the region = 'TH' filter in t0 is evaluated twice, once to compute t1, the other time to compute t2.</sup>
</div>

If `t0` is explicitly cached, e.g.,:

```python
query = '''
SELECT
    *
FROM
    selected_companies
WHERE
    region = 'TH'
'''
print query
t0 = spark.sql(query)
t0.cache() # lazy
t0.registerTempTable('t0') 
# spark.catalog.cacheTable('t0') # lazy, same effect as t0.cache() in terms of physical plan
# spark.sql("cache table t0") # eager, t0 is evaluated immediately
 
query = '''
WITH t1 AS (
    SELECT
        *
    FROM
        t0
    WHERE
        business = 1000
),
 
t2 AS (
    SELECT
        *
    FROM
        t0
    WHERE
        business = 2000
)
 
SELECT
    *
FROM
    (SELECT * FROM t1)
    UNION
    (SELECT * FROM t2)
'''
print query
df = spark.sql(query)
df.show()
```
Execution plan of the second query shows that `t0` is stored in cache memory and reused by `t1` and `t2`.
<div style="text-align: center"><img src="/images/t0_cache_DAG_visualization.jpg" width="800px" /></div>
<div align="center">
<sup>Because t0 is cached, it is read from InMemoryTableScan, and the filter on region in t0 is not re-evaluated when computing t1 and t2.</sup>
</div>

## Caching may take more time than not caching

If the time it takes to compute a table * the times it is used > the time it takes to cache the table, then caching may save time. Otherwise, not caching would be faster.

In other words, if the query is simple but the dataframe is huge, it may be faster to not cache and just re-evaluate the dataframe as needed. If the query is complex and the resulting dataframe is small, then caching may improve performance if the dataframe needs to be reused.

In the above example, the query is simple but the underlying dataframe is quite huge. As a result, caching `t0` takes more time (~14min) than not caching (~9min).

<div style="text-align: center"><img src="/images/t0_without_cache_start-end_time.jpg" width="800px" /></div>
<div align="center">
</div>

<div style="text-align: center"><img src="/images/t0_cache_start-end_time.jpg" width="800px" /></div>
<div align="center">
</div>

## Caching eagerly can improve readability when tracking progress in YARN UI

For Spark jobs that use complex SQL queries, the `SQL` page in YARN UI is a good way to track the progress of each query. However due to Spark's lazy evaluation, if the intermeidate tables are not cached eagerly or don't have any actions called upon them (e.g., `df.show()`), all the queries will be lumped together into one huge execution plan to be evaluated at the last step, e.g.:

<div style="text-align: center"><img src="/images/yarn_ui_sql_page.jpg" width="800px" /></div>
<div align="center">
</div>

So if a dataframe needs to be cached at all, can consider caching eagerly using `spark.sql('cache table xxx')`, so that the query execution can be broken down into more trackable pieces. Moreover, when optimizing queries, it is recommended to cache each intermediate table eagerly, so as to make identifying bottlenecks easier.