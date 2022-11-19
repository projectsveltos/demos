# Sveltos

<img src="https://raw.githubusercontent.com/projectsveltos/sveltos-manager/main/logos/logo.png" width="200">

Sveltos provides declarative APIs allowing you to deploy Kubernetes addons across multiple Kubernetes clusters.

The idea is simple:
1. from the management cluster, selects one or more `clusters` with a Kubernetes [label selector](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/#label-selectors);
2. lists which `features` need to be deployed on such clusters.

where term:
1. `clusters` represents [CAPI cluster](https://github.com/kubernetes-sigs/cluster-api/blob/main/api/v1beta1/cluster_types.go);
2. `features` represents either an [helm release](https://helm.sh) or a Kubernetes resource.

## Addons deployment

![Sveltos in action](../../docs/sveltos.png)

Values can be passed to helm charts. Sveltos can also fetch needed values from Kubernetes instances present in the management cluster.
For instance following ClusterProfile is reading pod CIDRs from CAPI Cluster instance.

<img src="https://raw.githubusercontent.com/projectsveltos/demos/main/docs/sveltos_calico.png" width="500">

## Cluster classification

Sveltos Classifier is an optional component of the Sveltos project and it is used to dynamically classify a cluster based on its runtime configuration (Kubernetes version, deployed resources and more).

Classifier currently supports the following classification criterias:
1. Kubernetes version
2. Kubernetes resources

![Sveltos Classifier in action](https://github.com/projectsveltos/demos/blob/main/classifier/classifier.gif)

## Display outcome of ClusterProfile's in DryRun mode

A ClusterProfile can be set in DryRun mode. While in DryRun mode, nothing gets deployed/withdrawn to/from matching CAPI clusters. A report is instead generated listing what would happen if ClusterProfile sync mode would be changed from DryRun to Continuous.

Here is an example of outcome

```
./bin/sveltosctl show dryrun
+-------------------------------------+--------------------------+-----------+----------------+-----------+--------------------------------+------------------+
|               CLUSTER               |      RESOURCE TYPE       | NAMESPACE |      NAME      |  ACTION   |            MESSAGE             | CLUSTER PROFILE |
+-------------------------------------+--------------------------+-----------+----------------+-----------+--------------------------------+------------------+
| default/sveltos-management-workload | helm release             | kyverno   | kyverno-latest | Install   |                                | dryrun           |
| default/sveltos-management-workload | helm release             | nginx     | nginx-latest   | Install   |                                | dryrun           |
| default/sveltos-management-workload | :Pod                     | default   | nginx          | No Action | Object already deployed.       | dryrun           |
|                                     |                          |           |                |           | And policy referenced by       |                  |
|                                     |                          |           |                |           | ClusterProfile has not changed |                  |
|                                     |                          |           |                |           | since last deployment.         |                  |
| default/sveltos-management-workload | kyverno.io:ClusterPolicy |           | no-gateway     | Create    |                                | dryrun           |
+-------------------------------------+--------------------------+-----------+----------------+-----------+--------------------------------+------------------+
```


**show dryrun** command has some argurments which allow filtering by:
1. clusters' namespace
2. clusters' name
3. ClusterProfile 

```
./bin/sveltosctl show dryrun --help  
Usage:
  sveltosctl show dryrun [options] [--namespace=<name>] [--cluster=<name>] [--clusterprofile=<name>] [--verbose]

     --namespace=<name>      Show which features would change in clusters in this namespace. If not specified all namespaces are considered.
     --cluster=<name>        Show which features would change in cluster with name. If not specified all cluster names are considered.
     --clusterprofile=<name> Show which features would change because of this clusterprofile. If not specified all clusterprofile names are considered.
```