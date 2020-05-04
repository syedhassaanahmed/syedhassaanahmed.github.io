---
layout:	post
title:	"Load Testing Analytical Workloads using Kubernetes"
date:	2019-06-08
---

Like any Distributed System, designing a Big Data Analytics solution requires careful planning and experimenation. Depending on the choice of technologies, the planning phase often involves evaluating the schema, indices, sharding and replication strategies, and examine them against factors such as;
* type of data ingestion (*batch, streaming*)
* frequency of data ingestion
* amount of data that should be kept in various storage tiers (*hot, warm, cold*)
* personas that will concurrently query the system (*Data Analyst, Data Scientist, C-Level Executive*)
* types of queries, aggregations and their accepted response times
* TCO of the analytics solution

![](https://www.guru99.com/images/L1.png)

Having a load testing framework in place to simulate the above, allows us to validate the architecture before releasing it for broader consumption. Based on a real-life project we did with an energy company, we'll focus on concurrent load testing of queries against an analytical data store in this post. Even though our use-case was Time Series Analysis against [Azure Data Explorer](https://docs.microsoft.com/en-us/azure/data-explorer/data-explorer-overview) (*aka Kusto*), the pattern we've implemented can easily be adapted to test other types of analytical stores and queries. The test framework we implemented had the following characteristics;

## Programming Language
In order to ensure we aren't always getting in-memory cached results from the analytics engine, we wanted the queries that are executed during load test to not be static. Let's say for a given set of IoT sensors we'd like to fetch the average sensor value within a time range; we should be able to customize Sensor IDs and the time range for each query in the test. We also wanted to keep the test framework generic enough and decouple it from the actual query construction. The interpreted and dynamically-typed nature of Python along with the availability of client SDK for a wide variety of analytical stores, made Python an obvious choice.

![](https://i.pinimg.com/474x/19/89/1b/19891b1eb9c47b70b739e06b20ba83cd--computer-humor-python.jpg)

Here is an example code `query.py` that returns a [KQL](https://github.com/microsoft/Kusto-Query-Language) query;
```py
from random import randint

def get_query():
    return """
LargeTimeseries
| where Sensor == "{}"
| where Timestamp > ago({})
| summarize avg(Value) by bin(Timestamp, 15m)
""".format(randint(0, 4999), "30d")
```
Then in the test framework code `test.py`;
```py
from query import get_query

raw_query = get_query()
execute(raw_query)
```
As it can be seen, the Sensor ID has been randomized. The `query.py` is just a regular Python file, so more complex "randomizations" such as choosing from a set of sensors stored in a dimension table, can also be performed if needed.

## Compute
Since the load tests were to be executed several times against various configuration changes in the analytics engine, we wanted something we can provision and de-commission fast - [immutable infrastructure](https://www.hashicorp.com/resources/what-is-mutable-vs-immutable-infrastructure/). And what more fast and immutable can it be than Docker containers? It also provided the added benefit of a self-contained environment. 

![](https://external-preview.redd.it/aR6WdUcsrEgld5xUlglgKX_0sC_NlryCPTXIHk5qdu8.jpg?auto=webp&s=5fe64dd318eec71711d87805d43def2765dd83cd)

Here is the `Dockerfile` we created to package the test framework;
```Dockerfile
FROM python:3-slim

WORKDIR /app

COPY requirements.txt ./
RUN apt-get update && apt-get install -y curl && \
    pip install --no-cache-dir -r requirements.txt

COPY test.py .

CMD ["sh", "-c", "curl -sS ${QUERY_SCRIPT_URL} > query.py && python -u ./test.py"]
```
Notice how `query.py` is being download from a web url, injected by environment variable `QUERY_SCRIPT_URL`. If your queries do not contain sensitive content, you can use [secret gists](https://help.github.com/en/github/writing-on-github/creating-gists#creating-a-gist). A more secure approach is to host them on Azure Blob Storage and generate a short-lived [SAS token](https://docs.microsoft.com/en-us/azure/storage/common/storage-sas-overview), scoped just to the specific query files. Additionally we can also [peer](https://docs.microsoft.com/en-us/azure/virtual-network/virtual-network-peering-overview) the container host with Blob Storage in the same [Virtual Network](https://docs.microsoft.com/en-us/azure/storage/common/storage-network-security?toc=/azure/virtual-network/toc.json#grant-access-from-a-virtual-network) and enforce strict firewall rules.

The above Docker image has been [published to my DockerHub](https://hub.docker.com/r/syedhassaanahmed/azure-kusto-load-test). To start executing the tests;
```sh
docker run -it --rm --env-file .env syedhassaanahmed/azure-kusto-load-test
```
Where `.env` file contains environment variables for authenticating and connecting to the Kusto cluster, such as;
```
CLUSTER_QUERY_URL=https://<ADX_CLUSTER>.<REGION>.kusto.windows.net
CLIENT_ID=<AAD_CLIENT_ID>
CLIENT_SECRET=<AAD_SECRET>
TENANT_ID=<AAD_TENANT>
DATABASE_NAME=adx_db
QUERY_SCRIPT_URL=https://.../query.py
```

## Orchestration
Containers turned out to be great. And if each container instance represents a querying user, we can just spin up multiple instances in order to simulate concurrent users right? What if we have to simulate 100+ concurrent users? Answer: ~~Container Orchestrator~~ Kubernetes (*aka k8s*).

![](https://i.redd.it/iv0oiaz7aqe41.jpg)

Here is a k8s `deployment.yaml` that we created in order to have several replicas of the above container;
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: adx-load-test
  labels:
    app: adx-load-test
spec:
  replicas: 3
  selector:
    matchLabels:
      app: adx-load-test
  template:
    metadata:
      labels:
        app: adx-load-test
    spec:
      containers:
      - name: adx-load-test
        image: syedhassaanahmed/azure-kusto-load-test
        env:
        - name: CLUSTER_QUERY_URL
          value: "https://<ADX_CLUSTER>.<REGION>.kusto.windows.net"
        ...
```
To start executing the tests, just apply the above yaml on an [AKS cluster](https://docs.microsoft.com/en-us/azure/aks/);
```sh
kubectl apply -f deployment.yaml
```

## Telemetry
We used the [Application Insights](https://docs.microsoft.com/en-us/azure/azure-monitor/app/app-insights-overview) feature of Azure Monitor to record and analyze query execution times for each test session. We modified the test framework to include Application Insights SDK and also enhanced it with the notion of a test session by introducing `test_id`;
```py
test_id = os.environ.get("TEST_ID", str(uuid.uuid4()))
telemetry_client.track_metric("query_time", query_time, properties={"test_id": test_id})
```

After a test session is completed, `test_id` allows us to query Application Insights and project the test execution results using the following KQL query;
```sql
customMetrics
| where name == "query_time" and customDimensions.test_id == "my_stressful_test"
| summarize percentiles(value, 5, 50, 95) by bin(timestamp, 1m)
| render timechart
```

![](/img/ai_query_render.png)

## Resources
All the code referenced above has been published in this [GitHub repo](https://github.com/syedhassaanahmed/azure-kusto-load-test). And remember when we said this load testing pattern can easily be adapted to other types of analytical stores? Well we weren't kidding! we've performed similar testing on Azure Synapse Analytics SQL pool and have published the [repo here](https://github.com/syedhassaanahmed/azure-sql-load-test).

## Future improvements
- [Terraform template](https://www.terraform.io/docs/providers/azurerm/index.html) to provision all the required Azure resources.
- Connection strings and other authentication parameters stored in [Azure Key Vault](https://github.com/Azure/secrets-store-csi-driver-provider-azure) instead of environment variables.
- VNET Peering between AKS Cluster and the analytics store, so that all traffic flows through Azure backbone.
