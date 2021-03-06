---
layout:	post
title:	"Writing Apache Spark GraphFrames to Azure Cosmos DB"
date:	2018-10-28
---

[GraphFrames](https://graphframes.github.io/graphframes/docs/_site/user-guide.html) is an Apache Spark package which extends [DataFrames](https://spark.apache.org/docs/latest/sql-programming-guide.html#datasets-and-dataframes) to provide graph analytics capabilities. Azure Cosmos DB is Microsoft's multi-model database which supports the [Gremlin query language](https://tinkerpop.apache.org/gremlin.html) to store and operate on graph data.

[Cosmos DB Spark connector](https://github.com/Azure/azure-cosmosdb-spark/tree/master) contains [samples to ***read*** graph data into GraphFrames](https://github.com/Azure/azure-cosmosdb-spark/blob/master/samples/notebooks/On-Time%20Flight%20Performance%20with%20Spark%20and%20Cosmos%20DB%20-%20Seattle.ipynb). In this post we'll demonstrate how to build upon this connector to ***write*** GraphFrames to Cosmos DB using an [Azure Databricks](https://docs.azuredatabricks.net/) `PySpark` notebook.

## Prerequisites
These are the libraries required and must be attached to the Databricks cluster, **prior** to the notebook execution.

* [GraphFrames](https://spark-packages.org/package/graphframes/graphframes)
* [azure-cosmosdb-spark](https://github.com/Azure/azure-cosmosdb-spark#using-databricks-notebooks) (*make sure to upload the latest **uber** jar*)

## Example Data
We'll use the [friends and followers example graph data](https://graphframes.github.io/graphframes/docs/_site/user-guide.html#tab_python_0) specified in the GraphFrames documentation.

![](https://databricks.com/wp-content/uploads/2016/03/social-network-graph-diagram.png)

## Getting started
Based on the example, here is how we can add a vertices DataFrame;
```py
v = sqlContext.createDataFrame([
  ("a", "Alice", 34),
  ("b", "Bob", 36),
  ("c", "Charlie", 30),
  ("d", "David", 29),
  ("e", "Esther", 32),
  ("f", "Fanny", 36),
  ("g", "Gabby", 60)
], ["id", "name", "age"]) \
.withColumn("entity", lit("person"))
```

And the edges DataFrame;
```py
e = sqlContext.createDataFrame([
  ("a", "b", "friend"),
  ("b", "c", "follow"),
  ("c", "b", "follow"),
  ("f", "c", "follow"),
  ("e", "f", "follow"),
  ("e", "d", "friend"),
  ("d", "a", "friend"),
  ("a", "e", "friend")
], ["src", "dst", "relationship"])
```

And finally create a GraphFrame out of the above regular DataFrames;
```py
from graphframes import GraphFrame
g = GraphFrame(v, e)
```

If all is well, we should see the following outputs when displaying the `g.vertices` and `g.edges`;

![](/img/display_vertices.png)

![](/img/display_edges.png)

## Cosmos DB Graph backend format
A Cosmos DB graph collection stores data in JSON format in the backend. This data can also be accessed using the SQL API. A detailed description of this representation of vertices and edges is [described here](https://github.com/LuisBosquez/azure-cosmos-db-graph-working-guides/blob/master/graph-backend-json.md).

### Preparing Vertices
From the GraphFrame vertices, we'll create a new DataFrame containing Cosmos DB vertex rows. Each such row contains the following columns;

* `id`: Unique ID of the vertex — **Note:** `id` in Cosmos DB is [part of the resource URI](https://github.com/Azure/azure-cosmosdb-dotnet/issues/35#issuecomment-121009258) and hence must be URL encoded. So we've created a [Spark UDF](https://docs.azuredatabricks.net/spark/latest/spark-sql/udf-python.html) for that;

```py
from pyspark.sql.types import StringType
from urllib.parse import quote

def urlencode(value):
  return quote(value, safe="")

udf_urlencode = udf(urlencode, StringType())
```

* `label`(*optional*): The *type* of vertex entity. In our example we've used the value `person` stored in the `entity` column of vertices DataFrame.
* Partition key: If the Cosmos DB graph is provisioned as a [partitioned collection](https://docs.microsoft.com/en-us/azure/cosmos-db/graph-partitioning), there must be a column with that partition key name.
* Property bag: Gremlin vertex properties are stored as Arrays of Struct, because multiple values are allowed in a single property. e.g. For vertex `a` the property bag `name` would look like below. (**Note**: partition key property is stored as a regular column and NOT a property bag)
```json
[
  {
    "id":"2effca75-a334-4964-a2d6-27c534ecbc5d", 
    "_value":"Alice"  
  }
]
```

Armed with the above knowledge, let's put this into a function;
```py
def to_cosmosdb_vertices(dfVertices, labelColumn, partitionKey = ""):
  dfVertices = dfVertices.withColumn("id", udf_urlencode("id"))
  
  columns = ["id", labelColumn]
  
  if partitionKey:
    columns.append(partitionKey)
  
  columns.extend(['nvl2({x}, array(named_struct("id", uuid(), "_value", {x})), NULL) AS {x}'.format(x=x) \
                for x in dfVertices.columns if x not in columns])
 
  return dfVertices.selectExpr(*columns).withColumnRenamed(labelColumn, "label")
```

Applying the function on `g.vertices` this is what we should get;

![](/img/to_cosmosdb_vertices.png)

If the graph was partitioned by `age` property, the same function would yield;

![](/img/to_cosmosdb_vertices_age.png)

Notice how `age` is not a property bag like `name` anymore

### Preparing Edges
Now its time to transform GraphFrame edges into a Cosmos DB DataFrame. Each row inside that DataFrame will contain the following columns;

* `id`: Similar to the `id` column in a vertex row, a unique ID of the edge. We've decided to generate it based on the convention `<src>_<relationship>_<dst>` e.g. `a_friend_b`.
* `label`: The name of edge relationship. In our example we've used the values `friend` and `follow` stored in the `relationship` column of edges DataFrame.
* Gremlin edge properties are stored as regular columns, since multi-valued properties are not supported in Gremlin edges.
* `_isEdge`: Hardcoded boolean column with value true.
* `_vertexId`: ID of the source vertex. e.g. In an edge from vertex `a` to vertex `b` the `_vertexId` should be `a`.
* `_sink`: ID of the destination vertex. e.g. In an edge from vertex `a` to vertex `b` the `_sink` should be `b`.

Similar to vertices, if the Cosmos DB graph is provisioned as a partitioned collection, the following additional columns must also be provided;

* Partition key: Column with the source vertex partition key name.
* `_sinkPartition`: Value of partition key of destination vertex.
Let's put the above knowledge into a function as well;

```py
from pyspark.sql.functions import concat_ws, col

def to_cosmosdb_edges(g, labelColumn, partitionKey = ""): 
  dfEdges = g.edges
  
  if partitionKey:
    dfEdges = dfEdges.alias("e") \
      .join(g.vertices.alias("sv"), col("e.src") == col("sv.id")) \
      .join(g.vertices.alias("dv"), col("e.dst") == col("dv.id")) \
      .selectExpr("e.*", "sv." + partitionKey, "dv." + partitionKey + " AS _sinkPartition")

  dfEdges = dfEdges \
    .withColumn("id", udf_urlencode(concat_ws("_", col("src"), col(labelColumn), col("dst")))) \
    .withColumn("_isEdge", lit(True)) \
    .withColumn("_vertexId", udf_urlencode("src")) \
    .withColumn("_sink", udf_urlencode("dst")) \
    .withColumnRenamed(labelColumn, "label") \
    .drop("src", "dst")
  
  return dfEdges
```

Notice that the function expects `g` instead of `g.edges`. Applying the function on our graph `g` we should get;

![](/img/to_cosmosdb_edges.png)

If the graph was partitioned by `age` property, the same function would yield;

![](/img/to_cosmosdb_edges_age.png)

Notice the additional columns `age` and `_sinkPartition`

## Just insert to Cosmos DB already!!!
Enough said! here is all that's left for insertion to Cosmos DB
```py
cosmosDbConfig = {
  "Endpoint" : "https://<COSMOSDB_ENDPOINT>.documents.azure.com:443/",
  "Masterkey" : "<COSMOSDB_PRIMARYKEY>",
  "Database" : "<DATABASE>",
  "Collection" : "<COLLECTION>",
  "Upsert" : "true"
}

cosmosDbFormat = "com.microsoft.azure.cosmosdb.spark"

cosmosDbVertices.write.format(cosmosDbFormat).mode("append").options(**cosmosDbConfig).save()
cosmosDbEdges.write.format(cosmosDbFormat).mode("append").options(**cosmosDbConfig).save()
```
Make sure to use the [Cosmos DB https endpoint](https://docs.microsoft.com/en-us/azure/cosmos-db/how-to-use-regional-gremlin#portal-endpoint-discovery) and **NOT** the `wss://` endpoint.

## Resources
The entire notebook is [**available here**](https://github.com/syedhassaanahmed/databricks-notebooks/blob/master/graph_write_cosmosdb.py). Oh and if you're into Scala instead of Python, here is the [**Scala version**](https://github.com/syedhassaanahmed/databricks-notebooks/blob/master/graphWriteCosmosDB.scala).

Happy analyzing graph data on Azure!
  