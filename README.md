# Debezium OpenShift Edge-To-Cloud Demo

The purpose of this demo is to illustrate how simple it is to create data pipelines between the edge and the cloud on Kubernetes.
This project includes the following open-source projects:

* [OpenShift](https://github.com/openshift), an industry-leading hybrid cloud application platform powered by Kubernetes.
* [Strimzi](https://github.com/strimzi), provides the ability to run Apache Kafka and Kafka Connect on Kubernetes.
* [Debezium](https://github.com/debezium/debezium/), a CDC platform for capturing changes from database transaction logs.
* [Apicurio Registry](https://github.com/apicurio/apicurio-registry), a high performance, runtime registry for API designs and schemas.
* [UI for Apache Kafka](https://github.com/provectus/kafka-ui), an open-source web-based user interface for managing Apache Kafka.
* [Elasticvue](https://github.com/cars10/elasticvue), an open-source GUI for Elasticsearch as a browser extension.
* [PostgreSQL](https://github.com/postgres/postgres), a high performance open-source relational database.
* [Elasticsearch](https://github.com/elastic/elasticsearch), an open-source, distributed REST-based search engine.
* [Elasticsearch Sink Connector](https://github.com/confluentinc/kafka-connect-elasticsearch), an open-source Elasticsearch Kafka sink connector.


## Prerequisites

1. This demo assumes that you have already provisioned an OpenShift cluster.
If you have not provisioned a cluster, please do so now as the remainder of this document assumes that OpenShift is available and that you have authenticated using `oc login`.
2. Create a project namespace of your choice.  Throughout this document, it will be referred to as `your-namespace`.


## Installation

The following section will cover the installation of the various components used in this demo.


### Docker Secret

This demo requires access to `docker.io` and `quay.io` container registries.
There are several configuration files that require a secret called `docker-secret` that contains credentials to these two registries.

Assuming that you have your Docker configuration at `~/.docker/config.json`, execute the following command:
```bash
oc create secret generic docker-secret \
  --from-file=.dockerconfigjson=<path>/.docker/config.json \
  --type=kubernetes.io/dockerconfigjson \
  -n <your-namespace>
```


### Strimzi

[Strimzi](https://github.com/strizmi) provides the ability to run [Apache Kafka](https://kafka.apache.org) and its Connect runtime on [Kubernetes](https://kubernetes.io/).
To be able to use the Strimzi operator to coordinate the deployment of Kafka and Kafka Connect clusters, the operator must first be installed.

1. Open your OpenShift's web console and navigate to **Operators > OperatorHub**.
2. Search for **Strimzi**, click the operator, follow the prompts, clicking the **Install** button.
Accept all the default configuration settings when presented.

Once the installation completes, click **View Operator** to be taken to the **Strimzi** operator page.
To come back to this page later, you navigate to **Operators > Installed Operators**, and then click on **Strimzi**.

It's important to note that if you want to see deployments of your Kafka clusters, Connect clusters, or connectors, you can come to this screen and click **Strimzi**.
The following screen will show a top-level menu with links to **Kafka**, **Kafka Connect**, and **Kafka Connector** tabs that each list the deployed components.
These screens allow you to quickly see the deployment status should a connector fail to deploy.

We won't be using the operator page to install **Strimzi** components for this demo; however, you certainly could.
Instead, these will be installed later by applying YAML configurations.


### Apache Kafka

To install the [Apache Kafka](https://kafka.apache.org) broker, we're going to use the [Strimzi](https://github.com/strimzi/) operator that was installed above.
This installation uses ephemeral storage by default, so if you need persistent storage, you'll need to adjust the `kafka/kafka-cluster.yaml` file accordingly.

In a terminal window, execute the following:

```bash
oc apply -f kafka/kafka-cluster.yaml -n <your-namespace>
```

The above will create a Kafka cluster named `kafka-cluster` in your namespace.
When looking at the **Workloads > Pods** view in OpenShift, you will eventually see several pods started, and there will be 3 that are prefixed with `kafka-cluster-zookeep-<#>` and another 3 prefixed with `kafka-cluster-kafka-<#>`.

You can also check the deployment status of the broker by navigating to **Operators > Installed Operators**, click **Strimzi** followed by clicking the **Kafka** tab.
When the broker is successfully deployed and running, the status column should read "Condition: Ready".
If the broker does not deploy successfully, you can click on **kafka-cluster** and look at the conditions near the end of the view to see the failure reason in the "Message" column.


### Kafka UI

This demo uses the [UI for Apache Kafka](https://github.com/provectus/kafka-ui) to have an easy way to view Kafka topics and the messages that are written to those topics.
To use the Kafka UI, we are going to deploy a pre-built container image `provectuslabs/kafka-ui:latest` and expose this as a service on OpenShift on port 8080.
However, the Kafka UI requires prior knowledge of the Apache Kafka broker's endpoint, which we'll need to set in the `kafka/kafka-ui.yaml` file before we apply the configuration.

In a terminal window, execute the following:
```bash
oc apply -f kafka/kafka-ui.yaml -n <your-namespace>
```

Once the Kafka UI is deployed, you should be able to access the Kafka UI by navigating to `http://kafka-ui-service.<your-cluster-hostname>`.
If you're not entirely sure what the URL may be, you can also navigate to the **Topology** screen in the developer view of the OpenShift console, locate the `kafka-ui` pod, and click the arrow to open the application's URL.


### Apicurio Registry

[Apicurio Registry](https://github.com/apicurio/apicurio-registry) is a high performance, runtime registry where the deployed Debezium connectors will write the event schemas.
A schema registry provides multiple benefits including data validation, compatibility checking, versioning, and evolution.
It simplifies the development and maintenance of data pipelines and reduces the risk of data compatibility issues, data corruption, and data loss.

This demo requires the Apicurio Registry Operator. 

```bash
export NAMESPACE="<your-namespace>"
cat schema-registry/apicurio-2.4.3-install.yaml | sed "s/apicurio-registry-operator-namespace/$NAMESPACE/g" | oc apply -f -
```

This will install all the necessary components for Apicurio's Schema Registry so that it can be used by the source and sink Kafka connectors.

Next, storage needs to be configured for Apicurio.
This demo will use a Kafka topic as the persistence layer for the schema registry.

In a terminal window, execute the following:
```bash
oc apply -f schema-registry/kafkasql.yaml -n <your-namespace>
```


### Elasticsearch

Before the Elasticsearch search engine can be installed, several pre-requisite steps must be performed.
In a terminal window, execute the following:

```bash
oc apply -f https://download.elastic.co/downloads/eck/2.9.0/crds.yaml -n <your-namespace>
oc apply -f https://download.elastic.co/downloads/eck/2.9.0/operator.yaml -n <your-namespace>
```

At this point, the Elasticsearch custom resource definitions and the operator have been added, so you can safely deploy the Elasticsearch search engine.
The configuration for Elasticsearch can be found in `elasticsearch/es.yaml`.
This configuration will add a subdomain `elastic-search` to your OpenShift cluster's host.

In a terminal window, execute the following:
```bash
oc apply -f elasticsearch/es.yaml -n <your-namespace>
```

You may have noticed that this configuration also specifies a secret called `elastic-search-es-elastic-user`.
This must be defined prior to deploying the Elasticsearch engine so that it does not generate a random password for the `elastic` user account.
This demo uses the username `elastic` with a password of `elastic`.

Once this is deployed, you should be able to open a browser and go to `http://elastic-search.<your-cluster-hostname>`.
If it prompts you to log into the application, provide the username and password of `elastic`.
This should show an output from Elasticsearch that looks similar to the following:

```json
{
  "name" : "elastic-search-es-default-0",
  "cluster_name" : "elastic-search",
  "cluster_uuid" : "TRNKemjfRoq7Loo-rIoksg",
  "version" : {
    "number" : "8.10.0",
    "build_flavor" : "default",
    "build_type" : "docker",
    "build_hash" : "e338da74c79465dfdc204971e600342b0aa87b6b",
    "build_date" : "2023-09-07T08:16:21.960703010Z",
    "build_snapshot" : false,
    "lucene_version" : "9.7.0",
    "minimum_wire_compatibility_version" : "7.17.0",
    "minimum_index_compatibility_version" : "7.0.0"
  },
  "tagline" : "You Know, for Search"
}
```

If you don't see this, you may want to examine the logs to see what may have failed during the deployment and start-up of Elasticsearch.


### PostgreSQL (source database)

This demo will be illustrating capturing changes from a source [PostgreSQL](https://github.com/postgres/postgres) database using [Debezium](https://github.com/debezium/debezium).
For simplicity, we're going to make use of a Debezium pre-configured PostgreSQL container, `quay.io/debezium/example-postgres:2.3`.
The database will use `postgres` as the user, password, and main database names.

To apply the PostgreSQL configuration, execute the following in a terminal:

```bash
oc apply -f databases/postgres.yaml -n <your-namespace>
```


### Kafka Connect

To run source and sink connectors in the data pipeline requires a Kafka Connect cluster.
This demo defines the configuration for the cluster in the `kafka/connect-cluster.yaml` file.
The configuration file defines several plug-ins that will be added to the base Kafka Connect cluster as a build step.

To install the connect cluster in a terminal window, execute the following:
```bash
export NAMESPACE="<your-namespace>"
cat kafka/connect-cluster.yaml | sed "s/connect-cluster-namespace/$NAMESPACE/g" | oc apply -f -
```


### Connectors

At this point, all the infrastructure components have been installed, and now it's time to deploy the data pipeline connectors.
In this demo, we're going to use Debezium's PostgreSQL connector to capture changes from a relational database and the Elasticsearch connector to publish changes from Kafka into Elasticsearch.
There will not be any type of aggregation or transformation done to the data, and it will simply reflect an identical structure in the search index as is in the relational database.


#### Source Connector

The Debezium PostgreSQL connector's details are defined in `connectors/source-postgres-connector.yaml`.

In a terminal window, apply the PostgreSQL connector by executing the following:
```bash
oc apply -f connectors/source-postgres-connector.yaml -n <your-namespace>
```

You can navigate to **Operators > Installed Operators** in the OpenShift web console and click on **Strimzi** followed by **Kafka Connectors**.
In this view, you should eventually see the status change from "-" to "Condition: Ready" status; however, if you notice anything, you can click on the connector name and see what the failure is near the bottom.

If the connector was deployed successfully, you can now navigate to the Kafka UI's web interface and click on **Current Demo > Topics**.
You should eventually see the following topics created:

* server1.inventory.customers
* server1.inventory.geom
* server1.inventory.orders
* server1.inventory.products 
* server1.inventory.products_on_hand

You can click on the topic name followed by clicking the *Messages* tab near the top and see all messages that were published to the topic.
In the Filters tab, be sure to use the option *SchemaRegistry* for both the value and the key. This will allow the Kafka UI to deserialize the data and display using the schema associated with that message from the Schema Registry.

Under the scene, for each topic, a new schema has been created for both, the message value and the key value in the Schema Registry which enables powerful features like schema evolution.

This is possible using Apicurio Registry thanks to the Schema Registry compatibility API that enables using tooling like Kafka UI.

#### Sink connector

The elastic search sink connector's configuration is defined in `connectors/sink-es-connectors.yaml`.
This will define a sink connector for each of the topics defined in Kafka and write the data to a unique index per topic.

In a terminal window, apply the Elasticsearch connectors by executing the following:
```bash
oc apply -f connectors/sink-es-connectors.yaml -n <your-namespace>
```

You can navigate to **Operators > Installed Operators** in the OpenShift web console and click on **Strimzi** followed by **Kafka Connectors**.
In this view, you should eventually see the statuses change from "-" to "Condition: Ready" status; however, if you notice anything, you can click on the connector name and see what the failure is near the bottom.

If the connectors were deployed successfully, let's take a look and see if the data is in Elasticsearch.
You can either manually navigate to the Elasticsearch engine and query the indices yourself, but we're going to utilize a small Chrome extension that provides an easy-to-use interface for Elasticsearch.

In Chrome, install this extension: https://chrome.google.com/webstore/search/elasticvue

You will need to provide the following details for the extension:

* Username: `elastic`
* Password: `elastic`
* URI: `http://elastic-search.<your-cluster-hostname>/`

Once configured, navigate to the **Indices** tab, and you should notice 5 indices have been created:

* server1.inventory.customers
* server1.inventory.geom
* server1.inventory.orders
* server1.inventory.products
* server1.inventory.products_on_hand

By clicking on an index, such as `server1.inventory.customers`, you'll be directed to a view that shows several columns for each document in the index.

```text
_index                          _type   _id     _sc id      first_name  last_name   email                       __deleted
------------------------------- ------- ------- --- ------- ----------- ----------- --------------------------- -----------
server1.inventory.customers     _doc    1001    1   1001    Sally       Thomas      sally.thomas@acme.com       false
server1.inventory.customers     _doc    1002    1   1002    George      Bailey      gbailey@foobar.com          false
server1.inventory.customers     _doc    1003    1   1003    Edward      Walker      ed@walker.com               false
server1.inventory.customers     _doc    1004    1   1004    Anne        Kretchmar   annek@noanswer.org          false
```


## Using the demo

The data that is currently shown in the Kafka UI and Elasticsearch is the existing example data that Debezium captured as part of the source connector's snapshot.
In this section, we're going to manipulate the data and see how the data flows through the pipeline at lightning speeds.

Open a terminal connection to the `postgres` pod by executing the following:
```bash
oc rsh postgres
```

To login to the PostgreSQL database, run `psql --user postgres --password postgres`.
The pasword is `postgres`.

At the SQL terminal, you can execute `select * from inventory.customers;`, and you should get the following:

```text
  id  | first_name | last_name |         email         
------+------------+-----------+-----------------------
 1001 | Sally      | Thomas    | sally.thomas@acme.com
 1002 | George     | Bailey    | gbailey@foobar.com
 1003 | Edward     | Walker    | ed@walker.com
 1004 | Anne       | Kretchmar | annek@noanswer.org
(4 rows)
```

Now insert a new row by executing:
```sql
insert into inventory.customers values (1005, 'John', 'Doe', 'john@doe.com');
```

In the web browser, navigate to the Kafka UI endpoint and view the messages for the topic `server1.inventory.customers`.
You'll notice that there is now a new message that was appended, and the value preview should show that it is related to the new customer we added named `John Doe.`

Also, in the web browser, navigate to the Elasticvue UI and reload the documents in the `server1.inventory.customers` index.
You'll notice that there is now a new document that was added as follows:

```text
_index                          _type   _id     _sc id      first_name  last_name   email                       __deleted
------------------------------- ------- ------- --- ------- ----------- ----------- --------------------------- -----------
server1.inventory.customers     _doc    1001    1   1001    Sally       Thomas      sally.thomas@acme.com       false
server1.inventory.customers     _doc    1002    1   1002    George      Bailey      gbailey@foobar.com          false
server1.inventory.customers     _doc    1003    1   1003    Edward      Walker      ed@walker.com               false
server1.inventory.customers     _doc    1004    1   1004    Anne        Kretchmar   annek@noanswer.org          false
server1.inventory.customers     _doc    1005    1   1005    John        Doe         john@doe.com                false
```

Now, let's update John's record by changing his email address to see how that flows through the pipeline.
Back in the SQL terminal, execute the following SQL:

```sql
UPDATE inventory.customers SET email = 'john.doe@noanswer.org' where id = 1005;
```

In the Kafka UI, you'll notice a new message was added to the topic that looks slightly different from the former events.
This event is an update and includes details about the before and after state of the record.
You can see a reference to the old email `john@doe.com` and the new email `john.doe@noanswer.org`.

Finally, when checking the Elasticvue UI for the `server1.inventory.customers` index, we now see the following:

```text
_index                          _type   _id     _sc id      first_name  last_name   email                       __deleted
------------------------------- ------- ------- --- ------- ----------- ----------- --------------------------- -----------
server1.inventory.customers     _doc    1001    1   1001    Sally       Thomas      sally.thomas@acme.com       false
server1.inventory.customers     _doc    1002    1   1002    George      Bailey      gbailey@foobar.com          false
server1.inventory.customers     _doc    1003    1   1003    Edward      Walker      ed@walker.com               false
server1.inventory.customers     _doc    1004    1   1004    Anne        Kretchmar   annek@noanswer.org          false
server1.inventory.customers     _doc    1005    1   1005    John        Doe         john.doe@noanswer.org       false
```

John's email address has now changed from `john@doe.com` to `john.doe@noanswer.org`.

Lastly, let's test what happens when deleting this customer from the source database.
In the SQL terminal, execute the following:

```sql
DELETE FROM inventory.customers WHERE id = 1005;
```

In the Kafka UI, you'll notice that 2 new messages were added to the topic.
The one with data in the value portion of the event is the delete operation, which provides the previous record state.
The second with no value is a tombstone, which signals Kafka that it is safe to remove all messages with the key (ID=1005) when compaction runs.

In the Elasticvue UI, you'll see that the document for John Doe has been removed and that all that remains is:

```text
_index                          _type   _id     _sc id      first_name  last_name   email                       __deleted
------------------------------- ------- ------- --- ------- ----------- ----------- --------------------------- -----------
server1.inventory.customers     _doc    1001    1   1001    Sally       Thomas      sally.thomas@acme.com       false
server1.inventory.customers     _doc    1002    1   1002    George      Bailey      gbailey@foobar.com          false
server1.inventory.customers     _doc    1003    1   1003    Edward      Walker      ed@walker.com               false
server1.inventory.customers     _doc    1004    1   1004    Anne        Kretchmar   annek@noanswer.org          false
```


## Conclusion

These open-source projects provide a very flexible and reliable way to move data between the edge and the cloud or vice versa.
However, this demo only scratched the surface of the capabilities within these projects.

Out of the box, they also provide you with the ability to transform data event-by-event using single message transformations.
We utilized a few transformations in the connector YAML files to convert the data into desired formats; however, there are dozens of out-of-the-box transformations available to you, and you can create your own to fit your use case.
Additionally, Kafka provides stream processing, where you can aggregate data from multiple topics and events and much more.

Finally, all of these projects seamlessly integrate on Kubernetes to take advantage of its automation, scaling, reliability, and management features.