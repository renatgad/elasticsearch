# Picnic Elasticsearch Home Assignment
## Design Decisions
For identical role on each node requirement;
- added pod affinity to node sets to deploy at least one pod on every kubernetes node. If one node down; other nodes can continue to work.
- Created 2 separate node sets for cluster. One is for master role other nodeset is for data and ingest. Because master and other roles can have different affinity requirements (Like 2 data pods on every kubernetes nodes and 1 master pod on every node). Also node sets can have different resource limits, volume and storageclass requirements.
For ensuring availability requirement;
- Identical role requirement supplies availability on kubernetes node maintance and downtimes.
- Also update strategy and pod disruption budget settings for operator manages pod maintance limits (Like cluster can add aditional 3 pods for maintance) and disruption limit to cluster (Like cluster must have minimum 2 pods alive)
> SSL, exposing service with LB omited. Tried to implement assignment's requirements only.

> After implementation uninstalled elasticsearch cluster from given kubernetes cluster. 

## Before Begin
`elasticsearch/cluster.yml` file contains definitions and configuration settings of cluster. Settings described below.
Before starting installation please change desired values.
### Elasticsearch Cluster Config 
- Cluster version can be determined with version attribute. Default is 8.2.0
- Cluster name is `elasticsearch`. If you want to change it please change name attribute.
- Cluster will be installed under `elasticsearch` namespace. If you want to change namespace please change namespace attribute under metadata.
- Cluster has 2 node sets for master nodes and data, ingest nodes. For  nodesets you can configure following; If required additional nodesets can be added by role or rolegroups.
#### NodeSet Count
Node count must be greater than kubernetes worker node count.
#### NodeSet Config
Defined 2 types of nodes. Master and data, ingest. Roles for nodesets can be determined under this section.
For further info:
> https://www.elastic.co/guide/en/cloud-on-k8s/2.2/k8s-node-configuration.html
#### podTemplate
Pod template section manages pods affinity.
Default configuration has affinity to schedule 1 pod for every worker node on kubernetes cluster. For this working correctly nodesets must have count greater than kubernetes worker node count.
For further info:
> https://www.elastic.co/guide/en/cloud-on-k8s/2.2/k8s-advanced-node-scheduling.html
#### volumeClaimTemplates
Storage size and class can be set under this attribute. Storage class is set to `gp2` please change for different storage classes.
For further info: 
>  https://www.elastic.co/guide/en/cloud-on-k8s/2.2/k8s-volume-claim-templates.html
#### updateStrategy
Specifies limit the number of changes on cluster when pod has new configuration or update.
> maxsurge: Maximum extra pod count while doing operation
> maxUnavailable: Maximum number of pods can be while doing operation.

For further info:
> https://www.elastic.co/guide/en/cloud-on-k8s/2.2/k8s-update-strategy.html
#### podDisruptionBudget
Allows you to limit the disruption to your application when its pods need to be rescheduled for some reason such as upgrades or routine maintenance work on the Kubernetes nodes.
For further info: 
> https://www.elastic.co/guide/en/cloud-on-k8s/2.2/k8s-pod-disruption-budget.html
## Installation
For elasticsearch installation using official Elasticsearch (ECK) Operator.
For operator installation please follow steps below.

### ECK Operator Install
> Please read `Before Begin` section carefully.
- Create required namespace
```shell
 kubectl create ns elasticsearch
```
- Install required custom resource definitions.
```shell
kubectl apply -f operator/crds.yaml 
```
- After CRD creation we can install operator with RBAC rules.
```shell
kubectl apply -f operator/operator.yaml
```
> Note: You can track operator logs and status with `kubectl -n elastic-system logs -f statefulset.apps/elastic-operator
`
 
>Elasticsearch operator will be installed under `elastic-system` namespace

## Elasticsearch Cluster Install
For installing elasticsearch apply steps below.
```shell
kubectl apply -f elasticsearch/cluster.yml
```
If you want to see pods use
```shell
kubectl get pods --selector='elasticsearch.k8s.elastic.co/cluster-name=elasticsearch'

```
Operator creates a ClusterIp and Service.You can get info about service with
```shell
kubectl get service [CLUSTERNAME]-es-http -n elasticsearch
```

For testing if elasticsearch work use following steps
Get elasticsearch password
```shell
PASSWORD=$(kubectl get secret quickstart-es-elastic-user -o go-template='{{.data.elastic | base64decode}}')
```
Port forward
```shell
kubectl port-forward service/[CLUSTERNAME]-es-http 9200
```
Test via curl
```shell
curl -u "elastic:$PASSWORD" -k "https://localhost:9200"
```
Output needs to be like following
```
{
  "name" : "elasticsearch-es-data-0",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "52pKbrhtRnGGghTZzqO3xw",
  "version" : {
    "number" : "8.2.0",
    "build_flavor" : "default",
    "build_type" : "docker",
    "build_hash" : "b174af62e8dd9f4ac4d25875e9381ffe2b9282c5",
    "build_date" : "2022-04-20T10:35:10.180408517Z",
    "build_snapshot" : false,
    "lucene_version" : "9.1.0",
    "minimum_wire_compatibility_version" : "7.17.0",
    "minimum_index_compatibility_version" : "7.0.0"
  },
  "tagline" : "You Know, for Search"
}
```

## Uninstalling
Delete pods
```shell
kubectl delete -f elasticsearch/cluster.yml
```
Delete Namespace
```shell
kubectl delete ns elasticsearch
```
Delete Operator
```shell
kubectl delete -f operator/operator.yaml 
```
Delete CRD's
```shell
 kubectl delete -f operator/crds.yaml
 ```
