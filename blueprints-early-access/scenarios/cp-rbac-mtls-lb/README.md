# Deploy CFK with RBAC, mTLS, and Load Balancer using Blueprint

This scenario uses the Control Plane and Data Plane you deploy in [Quick Start](../quickstart-deploy/single-site-deployment.rst) and creates a new Blueprint with the following features:

- mTLS authentication
- Confluent Platform RBAC 
- External access using Load Balancer
- Custom Kafka listeners and its credential coming from deployment API

## Prerequisite
- Set the home directory for this tutorial:

  ```bash
  export SCENARIO_BASEPATH=<CFK examples directory>/confluent-kubernetes-examples/blueprints-early-access/scenario/cp-rbac-mtls-lb
  ```

  ```bash
  export MY_NAMESPACE=<your org namespace>
  ``` 

- [Deploy the Control Plane with the Orchestrator](../quickstart-deploy/single-site-deployment.rst#deploy-control-plane).

- [Deploy the Data Plane with the Agent](../quickstart-deploy/single-site-deployment.rst#deploy-local-data-plane).

- The above setup creates the namespace for the Blueprint system resources, `cpc-system`.

### Deploy OpenLDAP
This repo includes a Helm chart for OpenLdap. The chart `values.yaml` includes the set of principal definitions that Confluent Platform needs for Confluent Platform RBAC.

1. Deploy OpenLdap:

   ```bash
   helm upgrade --install -f $SCENARIO_BASEPATH/../../../assets/openldap/ldaps-rbac.yaml test-ldap    $SCENARIO_BASEPATH/../../../assets/openldap --namespace cpc-system
   ```

1. Validate that OpenLDAP is running:

   ```bash
   kubectl get pods --namespace cpc-system
   ```

## Install Blueprint

### Install Blueprint Certificate Authority (CA)

The Control Plane uses CA keypair to generate the certificates for all the Confluent Platform
components. 

To create the required CA, run the following commands:

1. Create a secret `ca-key-pair-sce-2`:

   ```bash
   kubectl -n cpc-system create secret tls  ca-key-pair-sce-2 --cert=/tmp/cpc-ca.pem --key=/tmp/cpc-ca-key.pem
   ```

2. Create the CertificateStoreConfig to inject the CA keypair generated as above:

   ```bash
   kubectl apply -f $SCENARIO_BASEPATH/blueprint/certificatestoreconfig.yaml --namespace cpc-system
   ```

### Install Blueprint Credentials

- Create a secret object, `cp-rbac-mtls-lb-bp-credentials`, that contains all the required credential for the Blueprint:

  ```bash
  kubectl -n cpc-system create secret generic cp-rbac-mtls-lb-bp-credentials \
  --from-file=mds-token-key.txt=$SCENARIO_BASEPATH/blueprint/credentials/mds-token-key.txt \
  --from-file=mds-public-key.txt=$SCENARIO_BASEPATH/blueprint/credentials/mds-public-key.txt \
  --from-file=$SCENARIO_BASEPATH/blueprint/credentials/client-bearer.txt \
  --from-file=idp-simple.txt=$SCENARIO_BASEPATH/blueprint/credentials/idp-simple.txt  \
  --from-file=kafka-pwd-encoder.txt=$SCENARIO_BASEPATH/blueprint/credentials/kafka-pwd-encoder.txt  \
  --from-file=sr-pwd-encoder.txt=$SCENARIO_BASEPATH/blueprint/credentials/sr-pwd-encoder.txt  \
  --save-config --dry-run=client -oyaml | kubectl apply -f -
  ```

- Create the credential store used by this Blueprint. The credential store is only used by this Blueprint and can't be share with other resource or Blueprints:

  ```bash
  kubectl apply -f $SCENARIO_BASEPATH/blueprint/credentialstoreconfig.yaml --namespace cpc-system
  ```

- Install Blueprint:

  ```bash
  kubectl apply -f $SCENARIO_BASEPATH/blueprint/blueprint.yaml --namespace cpc-system
  ```
## Deploy Confluent Platform in Single Site Deployment

1. Create the namespace for Confluent Platform:

   ```bash 
   kubectl create namespace $MY_NAMESPACE
   ```
2. Install Confluent Platform:

   ```bash
     kubectl apply -f $SCENARIO_BASEPATH/cp-clusters/deployment_ss.yaml -n ${MY_NAMESPACE}
   ```

## Install Confluent Applications

### Topic
 
- Create a topic:

  ```bash 
  kubectl -n ${MY_NAMESPACE} apply -f $SCENARIO_BASEPATH/cp-apps/topics/topic_ss.yaml
  ```
  
- Validate:

  ```bash 
  kubectl -n ${MY_NAMESPACE} get kafkatopics.apps topic-foo-ss
  ```
  The `STATE` should be set to `Created`.

### Rolebindings

- Make sure to find the ids for Schema Registry, Connect, and ksqlDB clusters before creating role bindings:

  ```bash 
  kubectl -n ${MY_NAMESPACE} get schemaregistrycluster -oyaml | grep schemaRegistryClusterId
  ```
  
  ```bash 
  kubectl -n ${MY_NAMESPACE} get connectcluster -oyaml | grep connectClusterId
  ``` 
  
  ```bash 
  kubectl -n ${MY_NAMESPACE} get ksqldbcluster -oyaml | grep ksqlClusterId
  ``` 
- Create role bindings: 

  ```bash 
  cat $SCENARIO_BASEPATH/cp-apps/rolebinding/rolebiding_ss.yaml | sed 's/__NAMESPACE__/'"$MY_NAMESPACE"'/g' | kubectl apply -f -
  ```

- Validate:

  ```bash
  kubectl -n ${MY_NAMESPACE} get confluentrolebindings.apps
  ```

  The `STATE` should be set to `Created`.

### Schema

- Create a schema: 

  ```bash
  kubectl -n ${MY_NAMESPACE} apply -f $SCENARIO_BASEPATH/cp-apps/schema/schema_ss.yaml
  ``` 
  
- Validate:

  ```bash
  kubectl -n ${MY_NAMESPACE} get schemas.app schema-foo-ss
  ``` 
  
  The `STATE` should be set to `Created`.

### Connector

- Create a connector:

  ```bash 
  cat $SCENARIO_BASEPATH/cp-apps/connectors/connector_ss.yaml | sed 's/__NAMESPACE__/'"$MY_NAMESPACE"'/g' | kubectl apply -f -
  ```

- Validate:
  
  ```bash 
  kubectl -n ${MY_NAMESPACE} get connectors.apps
  ```
  
  The `STATE` should be set to `Created`.

## Validate the Deployment

1. Check when the Confluent components are up and running:
   
   ```bash 
   kubectl get pods --namespace $MY_NAMESPACE -w
   ```

1. Navigate to Control Center in a browser and check the Confluent cluster:

   ```bash       
   kubectl confluent dashboard controlcenter --namespace $MY_NAMESPACE
   ```
   
   Log in as the `kafka` user with the `kafka-secret` password.

1. In Control Center, check if the `topic-foo-ss` topic exists.