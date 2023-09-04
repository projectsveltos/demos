## Display Kubernetes add-ons

Create ConfigMap with Kyverno ClusterPolicy:

```bash
wget https://raw.githubusercontent.com/projectsveltos/demos/main/visibility/kyverno_disallow_latest.yaml
kubectl create configmap kyverno-latest --from-file kyverno_disallow_latest.yaml
```

Post ClusterProfile that deploys Kyverno Helm chart and Kyverno ClusterPolicy __disallow latest tag__ to managed clusters matching __clusterSelector: env=production__

```bash
kubectl apply -f https://raw.githubusercontent.com/projectsveltos/demos/main/visibility/clusterprofile_kyverno.yaml
```

Post ClusterProfile that deploys Prometheus and Grafana Helm charts to managed clusters matching __clusterSelector: team=devops__

```bash
kubectl apply -f https://raw.githubusercontent.com/projectsveltos/demos/main/visibility/clusterprofile_prometheus_grafana.yaml
```

Post ClusterProfile that deploys Nginx Helm chart to managed clusters matching __clusterSelector: tier=frontend__

```bash
kubectl apply -f https://raw.githubusercontent.com/projectsveltos/demos/main/visibility/clusterprofile_nginx.yaml
```

```bash
kubectl exec -it -n projectsveltos sveltosctl-0 -- ./sveltosctl show addons                    
+----------------------------+--------------------------+------------+-----------------------------+---------+-------------------------------+----------------------+
|          CLUSTER           |      RESOURCE TYPE       | NAMESPACE  |            NAME             | VERSION |             TIME              |   CLUSTER PROFILES   |
+----------------------------+--------------------------+------------+-----------------------------+---------+-------------------------------+----------------------+
| docker/clusterapi-workload | helm chart               | kyverno    | kyverno-latest              | 3.0.1   | 2023-09-03 06:57:02 -0700 PDT | kyverno              |
| docker/clusterapi-workload | helm chart               | prometheus | prometheus                  | 23.4.0  | 2023-09-03 07:06:44 -0700 PDT | prometheus-grafana   |
| docker/clusterapi-workload | helm chart               | grafana    | grafana                     | 6.58.9  | 2023-09-03 07:06:53 -0700 PDT | prometheus-grafana              |
| docker/clusterapi-workload | kyverno.io:ClusterPolicy |            | disallow-latest-tag         | N/A     | 2023-09-03 07:21:36 -0700 PDT | kyverno              |
| gke/cluster1               | helm chart               | kyverno    | kyverno-latest              | 3.0.1   | 2023-09-03 06:57:03 -0700 PDT | kyverno              |
| gke/cluster1               | kyverno.io:ClusterPolicy |            | disallow-latest-tag         | N/A     | 2023-09-03 07:21:38 -0700 PDT | kyverno              |
| hetzner/hetzner            | helm chart               | kyverno    | kyverno-latest              | 3.0.1   | 2023-09-03 06:57:02 -0700 PDT | kyverno              |
| hetzner/hetzner            | helm chart               | nginx      | nginx-latest                | 0.17.1  | 2023-09-03 07:11:19 -0700 PDT | clusterprofile-nginx |
| hetzner/hetzner            | kyverno.io:ClusterPolicy |            | disallow-latest-tag         | N/A     | 2023-09-03 07:21:36 -0700 PDT | kyverno              |
+----------------------------+--------------------------+------------+-----------------------------+---------+-------------------------------+----------------------+
```

## Preview the effects of a change

Update the ClusterProfile clusterprofile-nginx by:

1. changing SyncMode to DryRun
2. changing Nginx Helm chart to 0_18_1
 
```bash
kubectl apply -f https://raw.githubusercontent.com/projectsveltos/demos/main/visibility/clusterprofile_nginx_dryrun.yaml
```

```bash
kubectl exec -it -n projectsveltos sveltosctl-0 -- ./sveltosctl show dryrun 
+-----------------+---------------+-----------+--------------+---------+--------------------------------+----------------------+
|     CLUSTER     | RESOURCE TYPE | NAMESPACE |     NAME     | ACTION  |            MESSAGE             |   CLUSTER PROFILE    |
+-----------------+---------------+-----------+--------------+---------+--------------------------------+----------------------+
| hetzner/hetzner | helm release  | nginx     | nginx-latest | Upgrade | Current version: "0.17.1".     | clusterprofile-nginx |
|                 |               |           |              |         | Would move to version:         |                      |
|                 |               |           |              |         | "0.18.1"                       |                      |
+-----------------+---------------+-----------+--------------+---------+--------------------------------+----------------------+
```

## Display Configuration Changes Over Time

Take a snapshot of the Sveltos configuration.

Create ConfigMap with another Kyverno ClusterPolicy:

```bash
wget https://raw.githubusercontent.com/projectsveltos/demos/main/visibility/kyverno_disallow_empty_ingress.yaml
kubectl create configmap kyverno-empty-ingress --from-file kyverno_disallow_empty_ingress.yaml
```

Update content of the kyverno-latest ConfigMap with this [content](https://raw.githubusercontent.com/projectsveltos/demos/main/visibility/kyverno_disallow_latest_modified.yaml)

Take a new snapshot and compare what has changed:

```bash
kubectl exec -it -n projectsveltos sveltosctl-0 -- ./sveltosctl snapshot diff --snapshot=hourly --from-sample=2023-09-03:07:18:00 --to-sample=2023-09-03:07:23:00
+----------------------------------+--------------------------+-----------+-----------------------------+----------+--------------------------------+
|             CLUSTER              |      RESOURCE TYPE       | NAMESPACE |            NAME             |  ACTION  |            MESSAGE             |
+----------------------------------+--------------------------+-----------+-----------------------------+----------+--------------------------------+
| docker/capi--clusterapi-workload | kyverno.io/ClusterPolicy |           | disallow-empty-ingress-host | added    |                                |
| docker/capi--clusterapi-workload | kyverno.io/ClusterPolicy |           | disallow-latest-tag         | modified | use --raw-diff option to see   |
|                                  |                          |           |                             |          | diff                           |
| gke/sveltos--cluster1            | kyverno.io/ClusterPolicy |           | disallow-empty-ingress-host | added    |                                |
| gke/sveltos--cluster1            | kyverno.io/ClusterPolicy |           | disallow-latest-tag         | modified | use --raw-diff option to see   |
|                                  |                          |           |                             |          | diff                           |
| hetzner/capi--hetzner            | kyverno.io/ClusterPolicy |           | disallow-empty-ingress-host | added    |                                |
| hetzner/capi--hetzner            | kyverno.io/ClusterPolicy |           | disallow-latest-tag         | modified | use --raw-diff option to see   |
|                                  |                          |           |                             |          | diff                           |
+----------------------------------+--------------------------+-----------+-----------------------------+----------+--------------------------------+
```

```bash
kubectl exec -it -n projectsveltos sveltosctl-0 -- ./sveltosctl snapshot diff --snapshot=hourly --from-sample=2023-09-03:07:18:00 --to-sample=2023-09-03:07:23:00 --raw-diff --namespace=docker
--- kyverno.io/ClusterPolicy disallow-latest-tag from /collection/snapshot/hourly/2023-09-03:07:18:00
+++ kyverno.io/ClusterPolicy disallow-latest-tag from /collection/snapshot/hourly/2023-09-03:07:23:00
@@ -8,11 +8,6 @@
     policies.kyverno.io/minversion: 1.6.0
     policies.kyverno.io/severity: medium
     policies.kyverno.io/subject: Pod
-    policies.kyverno.io/description: >-
-      The ':latest' tag is mutable and can lead to unexpected errors if the
-      image changes. A best practice is to use an immutable tag that maps to
-      a specific version of an application Pod. This policy validates that the image
-      specifies a tag and that it is not called `latest`.
 spec:
   validationFailureAction: audit
   background: true
```

## Create management cluster
Sveltos [addon-controller](https://github.com/projectsveltos/addon-controller) has a Makefile target __make quickstart__ that creates:

1. management cluster with Sveltos and ClusterAPI
2. managed cluster using docker as infrastructure provider.

To deploy Hetnzer as yet another infrastructure provider, follow instructions present [here](https://github.com/syself/cluster-api-provider-hetzner/blob/main/docs/topics/preparation.md).
