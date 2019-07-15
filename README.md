# KafkaConnectMongoDB

This is an example of how to dump data from Kafka to MongoDB.

Overview of the components:
- Kafka on Kubernetes
- Kafka Client
- Sample Kafka topics
- Sample Kafka messages
- MongoDB cluster

## Requirements
For this example you'll need a Kubernetes Cluster and a MongoDB cluster. 
#### Kubernetes Cluster
This example has been tested on a Kubernetes Cluster on Azure (AKS). Follow instructions on how to create an AKS cluster [here]([https://docs.microsoft.com/en-us/azure/aks/kubernetes-walkthrough](https://docs.microsoft.com/en-us/azure/aks/kubernetes-walkthrough)).

You can also setup a cluster on other cloud providers like [AWS]([https://docs.microsoft.com/en-us/azure/aks/kubernetes-walkthrough](https://docs.microsoft.com/en-us/azure/aks/kubernetes-walkthrough)) or [Google Cloud]([https://cloud.google.com/kubernetes-engine/docs/how-to/creating-a-cluster](https://cloud.google.com/kubernetes-engine/docs/how-to/creating-a-cluster)).

#### MongoDB
Create a database on MongoDB by following the instructions [here]([https://docs.mongodb.com/manual/tutorial/atlas-free-tier-setup/#create-free-tier-manual](https://docs.mongodb.com/manual/tutorial/atlas-free-tier-setup/#create-free-tier-manual)). You can use the free tier and choose your preferred cloud provider.

Once you have created a cluster, you'll need to create:
- A database
- A collection

For the above, follow the [G]etting Started](https://docs.atlas.mongodb.com/getting-started/) instructions.

##### MongoDB Network Access
Enable all network access by going to. The **Network Access** Tab, click the Add IP Address button and select all. This should populate `0.0.0.0` in the allowed IP addresses table.

<img src="images/mongodb-network.png?sanitize=true">

## Kafka on Kubernetes with Strimzi
There are several ways to deploy Kafka on Kubernetes. One is by using a Kuberentes Operator. In this example we use the Strimzi Cluster Operator.

Familiarize yourself with the Strimzi Operator by reading the [Overview of Strimzi](https://strimzi.io/docs/latest/#overview-str). There are several ways of installing Strimzin. For this example we used the yaml files. 

Follow the instructions to [download Strimzi](https://strimzi.io/docs/latest/#downloads-str) in the cluster.


### Install the Strimzi Cluster Operator
The Strimzi Cluster Operator is what we will use to deploy and manage Kafka.
After you downloaded Strimzi, deploy the Strimzi Cluster Operator to your cluster. We recommend deploying the operator under the `kafka` namespace. 

Create the kafka namespace:
```
kubectl create namespace kafka
```

Follow these [instructions](https://strimzi.io/docs/latest/#deploying-cluster-operator-kubernetes-str) to deploy the cluster operator.


### Deploy Kafka
For Kafka and its related resources we are using the setup from the [cnadolny/azure-kafka-kubernetes](https://github.com/cnadolny/azure-kafka-kubernetes/tree/master/kafka-operator-strimzi) repository.

From that repository download these files:
- simple-kafka.yaml: deploys a Kafka cluster that listens on both the plaintext and tls listeners
- kafkaclient.yaml: kafka client will be used to create sample data
- kafka-topics.yaml: sample kafka topics `test` and `test-one-rep`
- kafka-users.yaml: sample kafka user

All the above files are a starting point. You can change the configuration based on your needs.

Deploy Kafka:
`kubectl create -n kafka -f simple-kafka.yaml`

Wait for the resource to appear by checking the pod is running: `kubectl get pods -n kafka`.

Deploy the remaining resources:
```
kubectl create -n kafka -f kafka-topics.yaml
kubectl create -n kafka -f kafka-users.yaml
kubectl create -n kafka -f kafkaclient.yaml
```

## MongoDB Connector
For this example, the [edsa14/mongo-kafka-connect](edsa14/mongo-kafka-connect:v2) docker image containing the MongoDB Connector was created. For reference, the Dockerfile used to creat this image is included in the `examples` directory of this repository.

This examples uses the [mongodb/mongo-kafka](https://www.confluent.io/hub/mongodb/kafka-connect-mongodb) connector.


## Deploy Kafka Connect
Kafka Connect is deployed using the Strimzi Cluster Operator.

1. `cd` to the directory where you downloaded strimzi.
2. Open the `examples/kafka-connect/kafka-connect.yaml`
3. Add the name of the docker image for the MongoDB Connector under the `spec` section. See the image below:
```
apiVersion: kafka.strimzi.io/v1beta1
kind: KafkaConnect
metadata:
  name: my-connect-cluster
spec:
  version: 2.2.1
  replicas: 1
  bootstrapServers: my-cluster-kafka-bootstrap:9093
  image: edsa14/mongo-kafka-connect:v2
  tls:
    trustedCertificates:
      - secretName: my-cluster-cluster-ca-cert
        certificate: ca.crt
```

## Create the MongoDB Connector
Kafka Connect comes with a REST API that can be used to create and update connectors.
You can test the API by doing a port-forward:
`kubectl -n kafka port-forward my-connect-cluster-connect-679d68f9fb-4lrz8 8083:8083`

Go to `localhost:8083`. You should see a JSON response from the API.

Under `examples/create-mongodb-connector.json` you'll find the following:
```
{"name": "my-mongodb-connector",
    "config": {
        "topics": "test",
        "connector.class": "com.mongodb.kafka.connect.MongoSinkConnector",
        "tasks.max": "1",
        "key.converter": "org.apache.kafka.connect.json.JsonConverter",
"key.converter.schema.registry.url": "http://localhost:8081",
"value.converter": "org.apache.kafka.connect.json.JsonConverter",
"value.converter.schema.registry.url": "http://localhost:8081",
"connection.uri": "mongodb+srv://<username>:<password>@<cluster>/<database>",
"database": "<my-database>",
"collection": "<my-collection>",
"max.num.retries": "3",
"retries.defer.timeout": "5000",
"key.projection.type": "none",
"key.projection.list": "",
"value.projection.type": "none",
"value.projection.list": "",
"field.renamer.mapping": "[]",
"field.renamer.regex": "[]",
"document.id.strategy": "com.mongodb.kafka.connect.sink.processor.id.strategy.BsonOidStrategy",
"post.processor.chain": "com.mongodb.kafka.connect.sink.processor.DocumentIdAdder",
"delete.on.null.values": "false",
"writemodel.strategy": "com.mongodb.kafka.connect.sink.writemodel.strategy.ReplaceOneDefaultStrategy",
"max.batch.size ": "0",
"rate.limiting.timeout": "0",
"rate.limiting.every.n": "0",
"topic.override.sourceB.collection": "sourceB",
"topic.override.sourceB.document.id.strategy": "com.mongodb.kafka.connect.sink.processor.id.strategy.ProvidedInValueStrategy"
}} 
```

You **must** change at least the following variables:

| Variable  | Description | Example |
| ------------- | ------------- | ------------- |
| name | the name of the connector | `my-mongodb-connector` |
| connection.uri | the connection string for mongodb | `mongodb+srv://<username>:<password>@<cluster>/<database>`|
| database | the name of your database on MongoDB | `kafka-db` |
| collection | the name of your collection in MongoDB | `kafka` |

More information on the settings can be found on the [MongoDB Kafka sink connector guide](https://github.com/mongodb/mongo-kafka/blob/master/docs/sink.md).


After updating the settings, create the connector:
```
curl -H 'Content-Type: application/json' -X POST -d @/home/KafkaConnectMongoDB/create-mongodb-connector.json http://localhost:8083/connectors
```

Verify the connector is created on `localhost:8083/connectors`.

##### Update an existing connector

In `examples/update-mongodb-connector.json` you can find sample settings to update the connector. In this case we changed the `topic` from `test` to `blob-test`. Make sure other variables like: `connection.uri`, `database`, `collection`, and others from the `examples/create-mongodb-connector.json` are also in this file.

To update the connector:
```
curl -H 'Content-Type: application/json' -X PUT -d @/home/KafkaConnectMongoDB/update-mongodb-connector.json http://localhost:8083/connectors/my-mongodb-connector/config
```

You can swap `my-mongodb-connector` in the URL with the corresponding name of the connector you created.

To verify the changes were applied:
```
localhost:8083/connectors/my-mongodb-connector/config
```

To check the status of the connector:
```
localhost:8083/connectors/my-mongodb-connector/status
```

## Add sample data to Kafka
Sample data can be added to Kafka via the Kafka Client.

Sample Message:
```
{"propertyA": "A", "propertyB": "B"}
```

To add a message in the `test` topic:
```
kubectl -n kafka exec -ti kafkaclient-0 -- bin/kafka-console-producer.sh --broker-list my-cluster-kafka-brokers.kafka:9092 --topic test
```


This will launch an interative command prompt:
1. Paste the Sample Message from above
2. Type Ctrl+C to exit

##### Troubleshooting the Connector
Check the status of the connector to see if there were any errors:
```
localhost:8083/connectors/my-mongodb-connector/status
```

If there were no errors, you can check on MongoDB that the data is present. 
