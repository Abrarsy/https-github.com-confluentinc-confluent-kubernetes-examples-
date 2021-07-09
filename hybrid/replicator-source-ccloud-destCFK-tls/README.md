# Replicator

## Set up Pre-requisites

Set the tutorial directory for this tutorial under the directory you downloaded
the tutorial files:

```
export TUTORIAL_HOME=<Tutorial directory>/hybrid/replicator-source-ccloud-destCFK-tls
```

Create namespace,  for the destination cluster.

```
kubectl create ns destination
```


```
helm upgrade --install confluent-operator \
  confluentinc/confluent-for-kubernetes \
  --namespace destination
```


## Prep the Confluent Cloud user password and endpoint

Search and replace the following: 

```
<ccloud-key>
  hybrid/replicator-source-ccloud-destCFK-tls/creds-client-kafka-sasl-user.txt:
  hybrid/replicator-source-ccloud-destCFK-tls/README.md

<ccloud-pass>
  hybrid/replicator-source-ccloud-destCFK-tls/creds-client-kafka-sasl-user.txt
  hybrid/replicator-source-ccloud-destCFK-tls/README.md

<ccloud-endpoint:9092>
  hybrid/replicator-source-ccloud-destCFK-tls/README.md
  hybrid/replicator-source-ccloud-destCFK-tls/components-destination.yaml
```

## Deploy source and destination clusters, including Replicator


Deploy destination cluster:  

```
kubectl create secret generic kafka-tls \
--from-file=fullchain.pem=$TUTORIAL_HOME/certs/server.pem \
--from-file=cacerts.pem=$TUTORIAL_HOME/certs/ca.pem \
--from-file=privkey.pem=$TUTORIAL_HOME/certs/server-key.pem \
--namespace destination

kubectl create secret generic cloud-plain \
--from-file=plain.txt=$TUTORIAL_HOME/creds-client-kafka-sasl-user.txt \
--namespace destination

kubectl apply -f $TUTORIAL_HOME/components-destination.yaml

```

In `$TUTORIAL_HOME/components-destination.yaml`, note that the `Connect` CRD is used to define a 
custom resource for Confluent Replicator.


## Create topic in source cluster

```
 kafka-topics --bootstrap-server <ccloud-endpoint:9092> \
--command-config  ~/kafkaexample/ccloud-team/client.properties \
--create \
--partitions 3 \
--replication-factor 3 \
--topic moshe-topic-in-source
```


## Produce messages to source topic in source cluster

```
seq 1000  | kafka-console-producer --broker-list  <ccloud-endpoint:9092> \
--producer.config  ~/kafkaexample/ccloud-team/client.properties \
--topic moshe-topic-in-source
```

## Configure Replicator in destination cluster

Confluent Replicator requires the configuration to be provided as a file in the running Docker container.
You'll then interact with it through the REST API, to set the configuration.

### SSH into the `replicator-0` pod

```
kubectl -n destination exec -it replicator-0 -- bash
```

##### Define the configuration as a file in the pod

```
cat <<EOF > replicator.json
 {
 "name": "replicator",
 "config": {
     "connector.class":  "io.confluent.connect.replicator.ReplicatorSourceConnector",
     "src.kafka.sasl.jaas.config": "org.apache.kafka.common.security.plain.PlainLoginModule required username=\"<ccloud-key>\" password=\"<ccloud-pass>\";",
     "confluent.license": "",
     "confluent.topic.replication.factor": "3",
     "confluent.topic.security.protocol": "SSL",
     "confluent.topic.ssl.truststore.location":"/mnt/sslcerts/kafka-tls/truststore.p12",
     "confluent.topic.ssl.truststore.password":"mystorepassword",
     "confluent.topic.ssl.truststore.type": "PKCS12",
     "dest.kafka.bootstrap.servers": "kafka.destination.svc.cluster.local:9071",
     "dest.kafka.security.protocol": "SSL",
     "dest.kafka.ssl.keystore.type": "PKCS12",
     "dest.kafka.ssl.truststore.location": "/mnt/sslcerts/kafka-tls/truststore.p12",
     "dest.kafka.ssl.truststore.password": "mystorepassword",
     "dest.kafka.ssl.truststore.type": "PKCS12",
     "key.converter": "io.confluent.connect.replicator.util.ByteArrayConverter",
     "src.kafka.bootstrap.servers": "<ccloud-endpoint:9092>",
     "src.kafka.sasl.mechanism": "PLAIN",
     "src.kafka.security.protocol": "SASL_SSL",
     "src.kafka.ssl.keystore.type": "PKCS12",
     "src.kafka.ssl.truststore.location": "/mnt/sslcerts/kafka-tls/truststore.p12",
     "src.kafka.ssl.truststore.password": "mystorepassword",
     "src.kafka.ssl.truststore.type": "PKCS12",
     "tasks.max": "4",
     "topic.whitelist": "moshe-topic-in-source",
      "topic.rename.format": "\${topic}_replica",
     "value.converter": "io.confluent.connect.replicator.util.ByteArrayConverter"
   }
 }
EOF
``` 

##### Instantiate the Replicator Connector instance through the REST interface

```
curl -XPOST -H "Content-Type: application/json" --data @replicator.json https://localhost:8083/connectors -k
```
##### Check the status of the Replicator Connector instance
```
curl -XGET -H "Content-Type: application/json" https://localhost:8083/connectors -k

curl -XGET -H "Content-Type: application/json" https://localhost:8083/connectors/replicator/status -k
```

##### To delete the connector: 

```
curl -XDELETE -H "Content-Type: application/json" https://localhost:8083/connectors/replicator -k
```

### View in Control Center
```
  kubectl port-forward controlcenter-0 9021:9021
```
Open Confluent Control Center.


## Validate that it works

Open Control center, select destination cluster, topic `${topic}_replica` where $topic is the name of the approved topic (whitelist). 

```
seq 1000  | kafka-console-producer --broker-list  <ccloud-endpoint:9092> \
--producer.config  ~/kafkaexample/ccloud-team/client.properties \
--topic moshe-topic-in-source
```

You should start seeing messages flowing into the destination topic. 

####  Tear down 

```
kubectl --namespace destination delete -f $TUTORIAL_HOME/components-destination.yaml           
kubectl --namespace destination delete secrets cloud-plain kafka-tls 
```


#### Appendix: Create your own certificates


When testing, it's often helpful to generate your own certificates to validate the architecture and deployment.

You'll want both these to be represented in the certificate SAN:

- external domain names
- internal Kubernetes domain names

The internal Kubernetes domain name depends on the namespace you deploy to. If you deploy to `confluent` namespace, 
then the internal domain names will be: 

- *.kafka.destination.svc.cluster.local
- *.zookeeper.destination.svc.cluster.local
- *.replicator.destination.svc.cluster.local
- *.destination.svc.cluster.local


::

####  Install libraries on Mac OS
```
  brew install cfssl
```
#### Create Certificate Authority
  
```
  mkdir $TUTORIAL_HOME/../../assets/certs/generated && cfssl gencert -initca $TUTORIAL_HOME/../../assets/certs/ca-csr.json | cfssljson -bare $TUTORIAL_HOME/../../assets/certs/generated/ca -
```

#### Validate Certificate Authority

```
  openssl x509 -in $TUTORIAL_HOME/../../assets/certs/generated/ca.pem -text -noout
```
####  Create server certificates with the appropriate SANs (SANs listed in server-domain.json)

```
  cfssl gencert -ca=$TUTORIAL_HOME/../../assets/certs/generated/ca.pem \
  -ca-key=$TUTORIAL_HOME/../../assets/certs/generated/ca-key.pem \
  -config=$TUTORIAL_HOME/../../assets/certs/ca-config.json \
  -profile=server $TUTORIAL_HOME/../../assets/certs/server-domain.json | cfssljson -bare $TUTORIAL_HOME/../../assets/certs/generated/server
```

#### Validate server certificate and SANs

```  
  openssl x509 -in $TUTORIAL_HOME/../../assets/certs/generated/server.pem -text -noout
```



At this point you need to include the letsencrypt root certificates in the CA and server pem files.

The block to copy paste is located in the hybrid/replicator-source-ccloud-destCFK-tls/certs/cloudchain.pem file. 
All you need is to combine the files: 


```
cat $TUTORIAL_HOME/../../assets/certs/generated/ca.pem $TUTORIAL_HOME/certs/cloudchain.pem > ca.pem
cat $TUTORIAL_HOME/../../assets/certs/generated/server.pem $TUTORIAL_HOME/certs/cloudchain.pem > server.pem
```

Use the above files when creating the secret. 


Return to `step 1 <#provide-component-tls-certificates>`_ now you've created your certificates  
















##################################### 
```
CONSIDER FOR LATER
```
#####################################


##### Produce data to topic in source cluster

Create the kafka.properties file in $TUTORIAL_HOME. Add the above endpoint and the credentials as follows:

```
bootstrap.servers=kafka.source.svc.cluster.local:9071
sasl.jaas.config= org.apache.kafka.common.security.plain.PlainLoginModule required username="<ccloud-key>" password="<ccloud-pass>";
sasl.mechanism=PLAIN
security.protocol=SASL_SSL
ssl.truststore.location=/mnt/sslcerts/kafka-tls/truststore.p12
ssl.truststore.password=mystorepassword
```

##### Create a configuration secret for client applications to use
kubectl create secret generic kafka-client-config-secure \
  --from-file=$TUTORIAL_HOME/kafka.properties -n destination
```

Deploy a producer application that produces messages to the topic `topic-in-source`:

```
kubectl apply -f $TUTORIAL_HOME/secure-producer-app-data.yaml
```
