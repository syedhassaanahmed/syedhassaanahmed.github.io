---
layout:	post
title:	"Bringing public Neo4j graph datasets to Azure"
date:	2018-01-08
---

  They say "Necessity is the mother of invention". My colleague [Brian Sherwin](https://channel9.msdn.com/Niners/bsherwin) wanted to do an apples-to-apples comparison of the popular graph database Neo4j with the relatively fresh [Azure Cosmos DB Graph API](https://docs.microsoft.com/en-us/azure/cosmos-db/graph-introduction). Due to lack of a migration tool at that time, in order to load data from Neo4j to Cosmos DB, I joined Brian to build a simple tool called … wait-for-it … [**neo-to-cosmos**](https://github.com/syedhassaanahmed/neo-to-cosmos) (more on this in later posts). One thing led to another and we ended up needing substantially sized real-life datasets to test our migrations.

![sandbox meme](https://i.chzbgr.com/full/6704383488/hF7B9C21B/what-happens-in-the-sandbox-stays-in-the-sandbox)

Enter [Neo4j Sandbox](https://neo4j.com/sandbox-v2/)! Folks at Neo4j are kind enough to let you get started with free time-limited, internet-facing Sandbox environments, pre-loaded with public datasets such as [ICIJ's Paradise papers and Panama Papers](https://offshoreleaks.icij.org/), [US Congressional data](http://www.lyonwj.com/2015/09/20/legis-graph-congressional-data-using-neo4j/) from [GovTrack](https://www.govtrack.us/), [BuzzFeed's published](https://www.buzzfeed.com/johntemplon/help-us-map-trumpworld?utm_term=.ooL6kkqPEe#.lh7Nee3DJ9) network of corporate connections associated with Trump Administration. They even let you sign in using Twitter, and get a Sandbox with Graph data of people and tweets from your own Twitter network which is really awesome!!!

We used the Sandbox for a while until it expired. Thankfully, Neo4j allows you to extend the expiration or do a simple renewal, however as we were trying to [**scale out neo-to-cosmos**](https://github.com/syedhassaanahmed/neo-to-cosmos#scale-out), it put the Sandbox under tremendous read load — something it wasn't intended for.

![](https://docs.microsoft.com/en-us/azure/container-instances/media/container-instances-quickstart/view-an-application-running-in-an-azure-container-instance.png)

This enticed me to host Neo4j datasets on Azure and since some of them such as [Panama](https://hub.docker.com/r/ryguyrg/neo4j-panama-papers/) and [Paradise](https://hub.docker.com/r/ryguyrg/neo4j-paradise-papers/) papers are already available as docker images — Thanks to [ryan boyd](https://medium.com/u/5639edce041f); I thought what simpler way to bring it to Azure than with [Azure Container Instances](https://docs.microsoft.com/en-us/azure/container-instances/container-instances-overview)? (Of course if you need production-level workloads, you should [deploy Neo4j from Azure marketplace](https://neo4j.com/blog/deploy-neo4j-microsoft-azure-part-2/) or enter [the Helm-charted territory](https://github.com/kubernetes/charts/tree/master/stable/neo4j) of managed Kubernetes — [AKS](https://docs.microsoft.com/en-us/azure/aks/tutorial-kubernetes-deploy-cluster)). ACI allows you to start a container in Azure with public IP connectivity in seconds. It lets you request exactly what you want in terms of CPU cores and memory, without the need to provision and manage VMs. Oh and you pay based on what you request, billed by the second!

Benefiting from the above-mentioned docker images, I stumbled across a whole new world of [community-driven public Neo4j datasets](https://neo4j.com/developer/example-data/) and decided to bring some of them as containers on Azure as well. They're available in [**my Docker Hub**](https://hub.docker.com/u/syedhassaanahmed/) under the prefix "*neo4j-*".

![](/img/neo4j_arm_deploy.png)

To keep the above docker images all in one place, I've created a [**GitHub repository**](https://github.com/syedhassaanahmed/neo4j-datasets), which lets you [Deploy to Azure](https://azure.microsoft.com/en-us/blog/deploy-to-azure-button-for-azure-websites-2/) by selecting any of these images (including stock Neo4j Community image which is the default, as well as ryan's ICIJ images). The template also let's you configure container resources (CPU cores and memory).

Based on [official performance tuning guidelines](https://neo4j.com/developer/guide-performance-tuning/), Neo4j server is exposed with the following value for `dbms.memory.pagecache.size` and `dbms.memory.heap.maxSize` — `(CONTAINER_MEMORY_IN_GB - 1GB) / 2` (1GB reserved for other activities on server) — i.e for a 7GB container, both page cache size and heap size will have a value of 3GB each. Additionally, you also need to provide a Password for Neo4j user (Template sets the environment variable `NEO4J_AUTH` with value `neo4j/<your_password>`). And last but not least, once the container is up and running, [Neo4j Browser](https://neo4j.com/developer/guide-neo4j-browser/) can be publicly accessed on port 7473 (HTTPS).

I also wanted to share the lessons I learned from building these docker images.

## Dockerfile
Each Dockerfile uses [multi-stage build](https://docs.docker.com/develop/develop-images/multistage-build/) and follows more or less the following blueprint in the build phase;

* Use [Neo4j](https://hub.docker.com/_/neo4j/) as base image (surprise surprise!).
* Add a common [***neo4j-start.sh***](https://github.com/syedhassaanahmed/neo4j-datasets/blob/master/neo4j-start.sh) bash script which is responsible for starting the Neo4j server and wait for it to kick in (more on this below).
* Copy `import.cypher` script if needed. I said "if needed" because instead of writing my own, I've tried to leverage as much unmodified [Cypher](https://neo4j.com/developer/cypher-query-language/) scripts from the community as possible (e.g. [the one for GoT](https://github.com/neo4j-examples/game-of-thrones/blob/master/got-import.cypher)). `import.cypher` usually contains a [LoadCSV](https://neo4j.com/developer/guide-import-csv/) statement, specifying how to import graph nodes and relationships*.*
* Some Cypher scripts need additional plugins such as [Awesome procedures — "apoc"](https://github.com/neo4j-contrib/neo4j-apoc-procedures) and [Graph Algorithms](https://github.com/neo4j-contrib/neo4j-graph-algorithms). Copy their `jar` files to `$NEO4J_HOME/plugins`.
**Note:** Base Neo4j docker image [simplifies non-root execution](https://github.com/neo4j/docker-neo4j/commit/d2dc9b27156a2a09ac6d1dc9b087893f773d7db4#diff-70a07072a11e01e9fcd2cc69c2eae4da) by running under `neo4j:neo4j` group user, hence above bash scripts and plugins `jar` must be added with `--chown=neo4j:neo4j`.
* Some datasets (such as [Inside Airbnb](http://insideairbnb.com/get-the-data.html) and [Stack Exchange](https://archive.org/details/stackexchange)) are quite large and trying to import them all at once will result in enormously long container boot time, hence we declare an environment variable (with a default value) to dictate how much data we want to import (e.g. [Stack Exchange](https://github.com/syedhassaanahmed/neo4j-datasets/tree/master/stackexchange) image only allows to import a specific community, while [Inside Airbnb](https://github.com/syedhassaanahmed/neo4j-datasets/tree/master/inside-airbnb) let's you restrict based on *data_size_in_MB/city*).
* Since Neo4j image is based on `Debian`, we use [apt-get](https://linux.die.net/man/8/apt-get) to install/update necessary packages if needed (e.g. *curl, git, python, grep, p7zip*).
* Git clone dataset repository if needed. Some datasets exist as multiple CSVs, others already contain useful import Cypher scripts like [this one](https://github.com/johnymontana/neo4j-datasets) from [William Lyon](https://medium.com/u/1b83fdec4e46).
* Grant our `neo4j-start.sh` *execute* permission with `chmod` and run the script.
* Use [Cypher Shell](https://neo4j.com/docs/operations-manual/current/tools/cypher-shell/) to execute `import.cypher`.
* Copy `$NEO4J_HOME/data/databases/graph.db` directory to root / so that it can be put to the final container during build phase. We have to do this because the `$NEO4J_HOME/data/databases` directory is declared as a [volume](https://docs.docker.com/storage/volumes/) in Neo4j base image and hence can't be copied to the final container.

## Start script
`neo4j-start.sh` also follows a common blueprint;

* Neo4j comes with default credentials `neo4j/neo4j` but `Cypher Shell` won't allow us to run queries with it, hence we need to use [Neo4j Admin to set an initial temp password](https://neo4j.com/docs/operations-manual/current/configuration/set-initial-password/).
* Neo4j 3.2 has increased security for procedures and functions. Procedures that use internal APIs have to be allowed in `$NEO4J_HOME/conf/neo4j.conf` with, e.g. `dbms.security.procedures.unrestricted=apoc.*,algo.*`
* Start the Neo4j server and wait for it to be up and running.

## One more thing!
Remember the `neo-to-cosmos` tool I mentioned in the intro? Yeah, the ARM template also allows you to optionally **migrate all Neo4j data to Cosmos DB** using this tool! *(Additional resources such as Cosmos DB and an Azure Container Instance of [neo-to-cosmos docker image](https://hub.docker.com/r/syedhassaanahmed/neo-to-cosmos/) will be deployed to your Azure subscription)*.

## Summary
I'd like to thank the awesome Neo4j community, without their help this project wouldn't have been possible.

In case you've missed it among the plethora of links above, here is the link to my [github repo](https://github.com/syedhassaanahmed/neo4j-datasets) again. Happy graphing on Azure!
  