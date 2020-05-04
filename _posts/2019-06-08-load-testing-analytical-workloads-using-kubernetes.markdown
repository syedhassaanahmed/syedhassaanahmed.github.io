---
layout:	post
title:	"Load Testing Analytical Workloads using Kubernetes"
date:	2019-06-08
---

Like any Distributed System, designing a Big Data Analytics solution requires careful planning and experimenation. Depending on the choice of technologies, the planning phase often involves evaluating the schema, indices, sharding and replication strategies, and examine them against factors such as;
* rate of data ingestion
* amount of data that should be kept in various storage tiers (*hot, warm, cold*)
* personas that will concurrently query the system (*Data Analyst, Data Scientist, C-Level Executive*)
* types of queries, aggregations and their accepted response times
* cost of the analytics solution

![](https://www.guru99.com/images/L1.png)

Having a load testing framework in place to simulate the above, allows us to validate the architecture before releasing it to a broader audience. In this post we'll focus on concurrent load testing of queries against an analytical data store. Based on a real-life project we did with an energy company, we'll be performing Time Series Analysis against [Azure Data Explorer](https://docs.microsoft.com/en-us/azure/data-explorer/data-explorer-overview) (*aka Kusto*). Having said that, the pattern we've implemented can easily be adapted to test other types of analytical stores and queries. The test framework we implemented had the following characteristics;

## Programming Language
In order to ensure we aren't always getting in-memory cached results from the analytics engine, we wanted the queries that are executed during load test to not be static. Let's say for a given set of IoT sensors we'd like to fetch the average sensor value within a time range. We should be able to customize Sensor IDs and the time range for each query in the load test. We also wanted to keep the test framework generic enough and decouple it from the actual query construction. The interpreted and dynamically-typed nature of Python along with the availability of client SDK for a wide variety of analytical stores, made Python an obvious choice.

![](https://i.pinimg.com/474x/19/89/1b/19891b1eb9c47b70b739e06b20ba83cd--computer-humor-python.jpg)

Here is an example file `query.py` which returns a query to a Kusto table;
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
Docker containers are great. If each executing container represents a querying user, we can just spin up multiple containers in order to simulate concurrent users right? What if we have to simulate 100+ concurrent users? Answer: ~~Container Orchestrator~~ Kubernetes.

![](https://i.redd.it/iv0oiaz7aqe41.jpg)

Here is a k8s deployment that we created in order to have several replicas of the above container;
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

## Telemetry
We used the [Application Insights](https://docs.microsoft.com/en-us/azure/azure-monitor/app/app-insights-overview) feature of Azure Monitor to record and analyze query execution times for each test session. We modified the test framework to include Application Insights SDK and also enhanced it with the notion of a `test_id`;
```py
test_id = os.environ.get("TEST_ID", str(uuid.uuid4()))
telemetry_client.track_metric("query_time", query_time, properties={"test_id": test_id})
```

Aftet the test session has ended, `test_id` allowed us to query Application Insights and project the test execution results;
```sql
customMetrics
| where name == "query_time" and customDimensions.test_id == "my_stressful_test"
| summarize percentiles(value, 5, 50, 95) by bin(timestamp, 1m)
| render timechart
```

![](/img/ai_query_render.png)

## Future improvements
- ARM/Terraform template to provision the infra
- Key Vault for connection string instead of env vars
- AAD auth to Storage
- VNet peering

## Bonus