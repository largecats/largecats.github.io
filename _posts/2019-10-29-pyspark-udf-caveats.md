---
layout: post
title:  "Pyspark UDF Caveats"
date:   2019-10-29
categories: life-saver work
tags: pyspark spark
---

* content
{:toc}

User defined function (udf) is a feature in (Py)spark that allows user to define customized functions with column arguments. E.g.,
```python
# dataframe of channelids
n = 100
orderids = list(set(sum([[i] * 4 for i in range(n//4)], [])))
channelids = np.random.choice(list(range(5)),len(orderids))
channel_df_pd = pd.DataFrame(
    {'orderid': orderids,
    'channelid': channelids}
)
channel_df = spark.createDataFrame(channel_df_pd).select('orderid', 'channelid')
channel_df.printSchema()
channel_df.show()
channel_df.registerTempTable('channel_df')

# get channelid from orderid
def get_channelid(orderid):
    channelid = channel_df.where(channel_df.orderid == orderid).select('channelid').collect()[0]['channelid']
    return str(channelid)

# register as udf
get_channelid_udf = udf(get_channelid, StringType())

# apply udf
df_channelid = df.withColumn('channelid', get_channelid_udf(df['orderid']))
```
I encountered the following pitfalls when using udfs. 


# Blackbox

