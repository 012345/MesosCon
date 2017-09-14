Welcome to the MesoCon Example!
===================


In this session, you'll learn how to get started with DataStax on DC/OS. 

First you can watch the videos on deploying DSE on DC/OS
[Install DSE](https://youtu.be/IgUBKw1EOFU)

[Install Opscenter](https://youtu.be/8jxRCiDulp0)

[Use the CLI](https://youtu.be/BUAVHDwAvHE)

[Add a Node](https://youtu.be/DNOtr-YlgQQ)

#### LETS GET STARTED

First youu'll need to install the CLI as shown in the video. In the DC/OS Gui go to the top left and click the down arrow and choose "Install CLI" and follow the directions.

If you prefer to install by command line you can as shown below but if you've already installed using the GUI just skip these steps. 

```
dcos package install datastax-dse
dcos package install datastax-ops
```

Next you'll need to install our specific CLI's for DC/OS
```
dcos package install --cli datastax-dse
dcos package install --cli datastax-ops
```

Oh yah.. Setup the display
```
export TERM=xterm-256color
export HOME=/mnt/mesos/sandbox
```

You can then look at the endpoints deployed in your DC/OS Cluster

```
dcos datastax-dse endpoints

[
  "spark-master-webui",
  "spark-worker-webui",
  "native-client",
  "solr-admin",
  "cassandra-thrift"
]
```
And then you can look specifically at the cluster nodes by viewing the cassandra endpoints

```
dcos datastax-dse endpoints native-client
{
  "address": [
    "10.200.177.76:9042",
    "10.200.177.80:9042",
    "10.200.177.8:9042"
  ],
  "dns": [
    "dse-0-node.datastax-dse.autoip.dcos.thisdcos.directory:9042",
    "dse-1-node.datastax-dse.autoip.dcos.thisdcos.directory:9042",
    "dse-2-node.datastax-dse.autoip.dcos.thisdcos.directory:9042"
  ]
}
```

Now lets look at the OpsCenter endpoints to find the URL for the GUI

```
dcos datastax-ops endpoints
[
  "opscenter"
]
````

And then dig into that endpoint further

```
dcos datastax-ops endpoints opscenter
{
  "address": ["10.200.177.79:8888"],
  "dns": ["opscenter-0-node.datastax-ops.autoip.dcos.thisdcos.directory:8888"]
}

```

As long as you have direct connectivity, VPN or a DC/OS TUnnel you can open that URL 


Awesome!  You can see the cluster. Now lets go back to the CLI and add some data.

Look back at that endpoint data from the dse endpoints command to find the name of one of your DSE nodes. Reference that node in the task exec command and DC/OS to get you a dang shell. we want to load data! 

```
dcos task exec -it dse-0-node /bin/bash
```

you in?  you're in.

Lets CLQSH. Grab an IP address of one of the native-client endpoints. for me it was 10.0.3.22. for you it's something else

```
cqlsh 10.0.3.22
```

Now lets create a simple yet impractical keyspace. Remember a keyspace is equivalent to a database

```
CREATE KEYSPACE customer WITH replication = {'class': 'SimpleStrategy', 'replication_factor': 3 };
```

Now lets add a table to that keyspace

```
CREATE TABLE customer.sales (
	name text,
	time int,
	item text,
	price double,
	PRIMARY KEY (name, time)
) WITH CLUSTERING ORDER BY ( time DESC );
```

And then how about we add some data to that table in that keyspace


```
INSERT INTO customer.sales (name, time, item, price) VALUES ('marc', 20150205, 'Apple Watch', 299.00);
INSERT INTO customer.sales (name, time, item, price) VALUES ('marc', 20150204, 'Apple iPad', 999.00);
INSERT INTO customer.sales (name, time, item, price) VALUES ('rich', 20150206, 'Music Man Stingray Bass', 1499.00);
INSERT INTO customer.sales (name, time, item, price) VALUES ('marc', 20150207, 'Jimi Hendrix Stratocaster', 899.00);
INSERT INTO customer.sales (name, time, item, price) VALUES ('rich', 20150208, 'Santa Cruz Tallboy 29er', 4599.00);
```

Sooooooo Closeeeeeeee.. Lets query that data!

```
SELECT * FROM customer.sales where name='marc' AND time >=20150205 ;
```

Yah.. That's it. You've deployed a stateful service. You've connected it to another stateful service (the management gui). You've inserted data in the database. 

Reach out to us with questions at techpartner@datastax.com


OHHH. YOU WANT TO KEEP GOING?

#### The secret sauce of the Cassandra data model: Primary Key

There are just a few key concepts you need to know when beginning to data model in Cassandra. But if you want to know the real secret sauce to solving your use cases and getting great performance, then you need to understand how Primary Keys work in Cassandra. 

Let's dive in! Check out [this exercise for understanding how primary keys work](https://github.com/robotoil/Cassandra-Primary-Key-Exercise/blob/master/README.md) and the types of queries enabled by different primary keys.

----------


Hands On Cassandra Consistency 
-------------------

#### Let's play with consistency!


Consistency in Cassandra refers to the number of acknowledgements replica nodes need to send to the coordinator for an operation to be successful while also providing good data (avoiding dirty reads). 

We recommend a ** default replication factor of 3 and consistency level of LOCAL_QUORUM as a starting point**. You will almost always get the performance you need with these default settings.

In some cases, developers find Cassandra's replication fast enough to warrant lower consistency for even better latency SLA's. For cases where very strong global consistency is required, possibly across data centers in real time, a developer can trade latency for a higher consistency level. 

Let's give it a shot. 


**In CQLSH**:

```tracing on```

```consistency all```

>Any query will now be traced. **Consistency** of all means all 3 replicas need to respond to a given request (read OR write) to be successful. Let's do a **SELECT** statement to see the effects.

```
SELECT * FROM customer.sales where name='<enter name>';
```

How did we do? 

**Let's compare a lower consistency level:**
```consistency local_quorum```
>Quorum means majority: RF/2 + 1. In our case, 3/2 = 1 + 1 = 2. At least 2 nodes need to acknowledge the request. 

Let's try the **SELECT** statement again. Any changes in latency? 
>Keep in mind that our dataset is so small, it's sitting in memory on all nodes. With larger datasets that spill to disk, the latency cost become much more drastic. 

**Let's try this again** but this time, let's pay attention to what's happening in the trace
```
consistency local_one
```
```
SELECT * FROM <yourkeyspace>.sales where name='<enter name>';
```

Take a look at the trace output. Look at all queries and contact points. What you're witnessing is both the beauty and challenge of distributed systems. 

```
consistency local_quorum
```
```
SELECT * FROM customer.sales where name='<enter name>';
```

>This looks much better now doesn't it? **LOCAL_QUORUM** is the most commonly used consistency level among developers. It provides a good level of performance and a moderate amount of consistency. That being said, many use cases can warrant  **CL=LOCAL_ONE**. 

For more detailed classed on data modeling, consistency, and Cassandra 101, check out the free classes at the [DataStax Academy] https://academy.datastax.com website. 

----------


Hands On DSE Search.
-------------
DSE Search is awesome. You can configure which columns of which Cassandra tables you'd like indexed in **lucene** format to make extended searches more efficient while enabling features such as text search and geospatial search. 

Let's start off by indexing the tables we've already made. Here's where the dsetool really comes in handy:

```
	dsetool create_core customer.sales generateResources=true reindex=true
```

Grab a quick coffee.. This will take a minute to propogate.

```
cqlsh 10.0.0.X
``` 

>If you've ever created your own Solr cluster, you know you need to create the core and upload a schema and config.xml. That **generateResources** tag does that for you. For production use, you'll want to take the resources and edit them to your needs but it does save you a few steps. 

This by default will map Cassandra types to Solr types for you. Anyone familiar with Solr knows that there's a REST API for querying data. In DSE Search, we embed that into CQL so you can take advantage of all the goodness CQL brings. Let's give it a shot. 

```
SELECT * FROM customer.sales WHERE solr_query='{"q":"name:*"}';

SELECT * FROM customer.sales WHERE solr_query='{"q":"name:marc", "fq":"item:*pple*"}'; 
```
> For your reference, [here's the doc](http://docs.datastax.com/en/datastax_enterprise/4.8/datastax_enterprise/srch/srchCql.html?scroll=srchCQL__srchSolrTokenExp) that shows some of things you can do




OKAY.. REALLY.. BACK TO WORK. REACH OUT WITH QUESTIONS. techpartner@datastax.com
