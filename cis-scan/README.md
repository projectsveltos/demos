# Kubernetes CIS Scan

Sveltos can leverage [kube-bench](https://github.com/aquasecurity/kube-bench) to run a security scan on all managed clusters.


## Lab Setup

A Kind cluster is used as management cluster. Then two extra Civo clusters and a GKE cluster all with label `env=production`.

```
+------------------------+-------------+-------------------------------------+
|    Cluster Name        |   Version   |             Comments                |
+------------------------+-------------+-------------------------------------+
|    civo/cluster1       | v1.28.7+k3s1| Civo 3 Node - Medium Standard       |
|    civo/cluster2       | v1.29.2+k3s1| Civo 3 Node - Medium Standard       |
|    gke/cluster         | v1.30.3-gke | GKE 3 Node                          |
+------------------------+-------------+-------------------------------------+
```

## Step 1: Install Sveltos on Managament Cluster

For this demonstration, we will install Sveltos in the management cluster. Sveltos installation details can be found here.

```
kubectl apply -f https://raw.githubusercontent.com/projectsveltos/sveltos/v0.38.2/manifest/manifest.yaml
kubectl apply -f https://raw.githubusercontent.com/projectsveltos/sveltos/v0.38.2/manifest/default-classifier.yaml
```

## Step 2: Register Clusters with Sveltos

Using Civo UI, download the Kubeconfigs, then:

```
kubectl create ns civo
sveltosctl register cluster --namespace=civo --cluster=cluster1 --kubeconfig=civo-cluster1-kubeconfig --labels=env=production
sveltosctl register cluster --namespace=civo --cluster=cluster2 --kubeconfig=civo-cluster2-kubeconfig --labels=env=production
```

Using sveltosctl, pointing to the GKE cluster:

```
sveltosctl generate kubeconfig --create --expirationSeconds=86400 > gke_kubeconfig
sveltosctl register cluster --namespace=gke --cluster=cluster --kubeconfig=gke_kubeconfig --labels=env=production
```

Verify your Civo and GKE clusters were successfully registered:

```
kubectl get sveltoscluster -A --show-labels                                                               
NAMESPACE   NAME       READY   VERSION               LABELS
civo        cluster1   true    v1.28.7+k3s1          env=production,projectsveltos.io/k8s-version=v1.28.7,sveltos-agent=present
civo        cluster2   true    v1.29.2+k3s1          env=production,projectsveltos.io/k8s-version=v1.29.2,sveltos-agent=present
gke         cluster    true    v1.30.3-gke.1969001   env=production,projectsveltos.io/k8s-version=v1.30.3,sveltos-agent=present
mgmt        mgmt       true    v1.31.0               projectsveltos.io/k8s-version=v1.31.0,sveltos-agent=present
```

## Step 3: Deploy kube-bench

![Sveltos: Deploy Kube-bench](sveltos-deploy-kube-bench.png)

`Sveltos` is used to deploy _kube-bench_ in all production clusters:

```
kubectl apply -f https://raw.githubusercontent.com/projectsveltos/demos/main/cis-scan/kube-bench.yaml
kubectl apply -f https://raw.githubusercontent.com/projectsveltos/demos/main/cis-scan/clusterprofile-deploy-kube-bench.yaml
```

Above creates a _ConfigMap_ in the management cluster containing kube-bench in its data section. And a ClusterProfile resource 
that targets all production clusters. The ClusterProfile deploys kube-bench.

Verify all resources are deployed:

```
sveltosctl show addons
+---------------+---------------+------------+------------+---------+--------------------------------+----------------------------------+
|    CLUSTER    | RESOURCE TYPE | NAMESPACE  |    NAME    | VERSION |              TIME              |             PROFILES             |
+---------------+---------------+------------+------------+---------+--------------------------------+----------------------------------+
| civo/cluster1 | batch:Job     | kube-bench | kube-bench | N/A     | 2024-09-24 14:33:27 +0200 CEST | ClusterProfile/deploy-kube-bench |
| civo/cluster2 | batch:Job     | kube-bench | kube-bench | N/A     | 2024-09-24 14:33:49 +0200 CEST | ClusterProfile/deploy-kube-bench |
| gke/cluster   | batch:Job     | kube-bench | kube-bench | N/A     | 2024-09-24 14:33:37 +0200 CEST | ClusterProfile/deploy-kube-bench |
+---------------+---------------+------------+------------+---------+--------------------------------+----------------------------------+
```

## Step 4: Use Event Frameworks to Collect Kube-bench Results

```
kubectl apply -f https://raw.githubusercontent.com/projectsveltos/demos/main/cis-scan/clusterprofile-deploy-kube-bench.yaml
kubectl apply -f https://raw.githubusercontent.com/projectsveltos/demos/main/cis-scan/events.yaml
```

Kube-bench results are initially stored in the kube-bench logs. To facilitate analysis and reporting, this process deploys 
a Job in each managed cluster. This Job post-processes the kube-bench logs, extracting and storing all failure information 
in a ConfigMap. These ConfigMaps are then automatically collected by Sveltos and sent to the management cluster for centralized review and action.

![Sveltos: Post process kube-bench results](post-process-collect-kube-bench-failures.png)

## Step 5: Use sveltosctl to display all failures

sveltosctl provides a comprehensive overview of issues identified by kube-bench across our managed production clusters.

```
sveltosctl show resources --kind=configmap 
+---------------+---------------------+------------+---------------------+--------------------------------+
|    CLUSTER    |         GVK         | NAMESPACE  |        NAME         |            MESSAGE             |
+---------------+---------------------+------------+---------------------+--------------------------------+
| civo/cluster1 | /v1, Kind=ConfigMap | kube-bench | kube-bench-failures | [FAIL] 4.1.1 Ensure that       |
|               |                     |            |                     | the kubelet service file       |
|               |                     |            |                     | permissions are set to 600 or  |
|               |                     |            |                     | more restrictive (Automated)   |
|               |                     |            |                     | [FAIL] 4.1.5 Ensure that the   |
|               |                     |            |                     | --kubeconfig kubelet.conf      |
|               |                     |            |                     | file permissions are set       |
|               |                     |            |                     | to 600 or more restrictive     |
|               |                     |            |                     | (Automated) [FAIL] 4.1.6       |
|               |                     |            |                     | Ensure that the --kubeconfig   |
|               |                     |            |                     | kubelet.conf file ownership is |
|               |                     |            |                     | set to root:root (Automated)   |
|               |                     |            |                     | [FAIL] 4.1.9 If the kubelet    |
|               |                     |            |                     | config.yaml configuration      |
|               |                     |            |                     | file is being used validate    |
|               |                     |            |                     | permissions set to 600 or      |
|               |                     |            |                     | more restrictive (Automated)   |
|               |                     |            |                     | [FAIL] 4.1.10 If the kubelet   |
|               |                     |            |                     | config.yaml configuration      |
|               |                     |            |                     | file is being used validate    |
|               |                     |            |                     | file ownership is set          |
|               |                     |            |                     | to root:root (Automated)       |
|               |                     |            |                     | [FAIL] 4.2.1 Ensure that the   |
|               |                     |            |                     | --anonymous-auth argument      |
|               |                     |            |                     | is set to false (Automated)    |
|               |                     |            |                     | [FAIL] 4.2.2 Ensure that       |
|               |                     |            |                     | the --authorization-mode       |
|               |                     |            |                     | argument is not set to         |
|               |                     |            |                     | AlwaysAllow (Automated)        |
|               |                     |            |                     | [FAIL] 4.2.3 Ensure that the   |
|               |                     |            |                     | --client-ca-file argument is   |
|               |                     |            |                     | set as appropriate (Automated) |
|               |                     |            |                     | [FAIL] 4.2.6 Ensure that the   |
|               |                     |            |                     | --make-iptables-util-chains    |
|               |                     |            |                     | argument is set to             |
|               |                     |            |                     | true (Automated) [FAIL]        |
|               |                     |            |                     | 4.2.10 Ensure that the         |
|               |                     |            |                     | --rotate-certificates          |
|               |                     |            |                     | argument is not set to false   |
|               |                     |            |                     | (Automated) [FAIL] 4.3.1       |
|               |                     |            |                     | Ensure that the kube-proxy     |
|               |                     |            |                     | metrics service is bound       |
|               |                     |            |                     | to localhost (Automated)       |
|               |                     |            |                     | [FAIL] 5.1.1 Ensure that       |
|               |                     |            |                     | the cluster-admin role is      |
|               |                     |            |                     | only used where required       |
|               |                     |            |                     | (Automated) [FAIL] 5.1.2       |
|               |                     |            |                     | Minimize access to secrets     |
|               |                     |            |                     | (Automated) [FAIL] 5.1.3       |
|               |                     |            |                     | Minimize wildcard use in Roles |
|               |                     |            |                     | and ClusterRoles (Automated)   |
|               |                     |            |                     | [FAIL] 5.1.4 Minimize access   |
|               |                     |            |                     | to create pods (Automated)     |
|               |                     |            |                     | [FAIL] 5.1.5 Ensure that       |
|               |                     |            |                     | default service accounts are   |
|               |                     |            |                     | not actively used. (Automated) |
|               |                     |            |                     | [FAIL] 5.1.6 Ensure that       |
|               |                     |            |                     | Service Account Tokens are     |
|               |                     |            |                     | only mounted where necessary   |
|               |                     |            |                     | (Automated)                    |
| civo/cluster2 |                     | kube-bench | kube-bench-failures | [FAIL] 4.1.1 Ensure that       |
|               |                     |            |                     | the kubelet service file       |
|               |                     |            |                     | permissions are set to 600 or  |
|               |                     |            |                     | more restrictive (Automated)   |
|               |                     |            |                     | [FAIL] 4.1.5 Ensure that the   |
|               |                     |            |                     | --kubeconfig kubelet.conf      |
|               |                     |            |                     | file permissions are set       |
|               |                     |            |                     | to 600 or more restrictive     |
|               |                     |            |                     | (Automated) [FAIL] 4.1.6       |
|               |                     |            |                     | Ensure that the --kubeconfig   |
|               |                     |            |                     | kubelet.conf file ownership is |
|               |                     |            |                     | set to root:root (Automated)   |
|               |                     |            |                     | [FAIL] 4.1.9 If the kubelet    |
|               |                     |            |                     | config.yaml configuration      |
|               |                     |            |                     | file is being used validate    |
|               |                     |            |                     | permissions set to 600 or      |
|               |                     |            |                     | more restrictive (Automated)   |
|               |                     |            |                     | [FAIL] 4.1.10 If the kubelet   |
|               |                     |            |                     | config.yaml configuration      |
|               |                     |            |                     | file is being used validate    |
|               |                     |            |                     | file ownership is set          |
|               |                     |            |                     | to root:root (Automated)       |
|               |                     |            |                     | [FAIL] 4.2.1 Ensure that the   |
|               |                     |            |                     | --anonymous-auth argument      |
|               |                     |            |                     | is set to false (Automated)    |
|               |                     |            |                     | [FAIL] 4.2.2 Ensure that       |
|               |                     |            |                     | the --authorization-mode       |
|               |                     |            |                     | argument is not set to         |
|               |                     |            |                     | AlwaysAllow (Automated)        |
|               |                     |            |                     | [FAIL] 4.2.3 Ensure that the   |
|               |                     |            |                     | --client-ca-file argument is   |
|               |                     |            |                     | set as appropriate (Automated) |
|               |                     |            |                     | [FAIL] 4.2.6 Ensure that the   |
|               |                     |            |                     | --make-iptables-util-chains    |
|               |                     |            |                     | argument is set to             |
|               |                     |            |                     | true (Automated) [FAIL]        |
|               |                     |            |                     | 4.2.10 Ensure that the         |
|               |                     |            |                     | --rotate-certificates          |
|               |                     |            |                     | argument is not set to false   |
|               |                     |            |                     | (Automated) [FAIL] 4.3.1       |
|               |                     |            |                     | Ensure that the kube-proxy     |
|               |                     |            |                     | metrics service is bound       |
|               |                     |            |                     | to localhost (Automated)       |
|               |                     |            |                     | [FAIL] 5.1.1 Ensure that       |
|               |                     |            |                     | the cluster-admin role is      |
|               |                     |            |                     | only used where required       |
|               |                     |            |                     | (Automated) [FAIL] 5.1.2       |
|               |                     |            |                     | Minimize access to secrets     |
|               |                     |            |                     | (Automated) [FAIL] 5.1.3       |
|               |                     |            |                     | Minimize wildcard use in Roles |
|               |                     |            |                     | and ClusterRoles (Automated)   |
|               |                     |            |                     | [FAIL] 5.1.4 Minimize access   |
|               |                     |            |                     | to create pods (Automated)     |
|               |                     |            |                     | [FAIL] 5.1.5 Ensure that       |
|               |                     |            |                     | default service accounts are   |
|               |                     |            |                     | not actively used. (Automated) |
|               |                     |            |                     | [FAIL] 5.1.6 Ensure that       |
|               |                     |            |                     | Service Account Tokens are     |
|               |                     |            |                     | only mounted where necessary   |
|               |                     |            |                     | (Automated)                    |
| gke/cluster   |                     | kube-bench | kube-bench-failures | [FAIL] 3.2.1 Ensure that the   |
|               |                     |            |                     | --anonymous-auth argument      |
|               |                     |            |                     | is set to false (Automated)    |
|               |                     |            |                     | [FAIL] 3.2.2 Ensure that       |
|               |                     |            |                     | the --authorization-mode       |
|               |                     |            |                     | argument is not set to         |
|               |                     |            |                     | AlwaysAllow (Automated)        |
|               |                     |            |                     | [FAIL] 3.2.3 Ensure that the   |
|               |                     |            |                     | --client-ca-file argument is   |
|               |                     |            |                     | set as appropriate (Automated) |
|               |                     |            |                     | [FAIL] 3.2.6 Ensure that the   |
|               |                     |            |                     | --protect-kernel-defaults      |
|               |                     |            |                     | argument is set to true        |
|               |                     |            |                     | (Manual) [FAIL] 3.2.9 Ensure   |
|               |                     |            |                     | that the --event-qps argument  |
|               |                     |            |                     | is set to 0 or a level which   |
|               |                     |            |                     | ensures appropriate event      |
|               |                     |            |                     | capture (Automated) [FAIL]     |
|               |                     |            |                     | 3.2.12 Ensure that the         |
|               |                     |            |                     | RotateKubeletServerCertificate |
|               |                     |            |                     | argument is set to true        |
|               |                     |            |                     | (Automated)                    |
+---------------+---------------------+------------+---------------------+--------------------------------+
```

## Clean 

```
kubectl delete -f https://raw.githubusercontent.com/projectsveltos/demos/main/cis-scan/clusterprofile-deploy-kube-bench.yaml
kubectl delete -f https://raw.githubusercontent.com/projectsveltos/demos/main/cis-scan/events.yaml
```