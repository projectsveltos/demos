# Claudie and Sveltos integration

[Claudie](https://github.com/berops/claudie) simplifies the process of programmatically establishing Kubernetes clusters from a management
cluster across multiple cloud vendors and on-prem datacenters.

[Sveltos](https://github.com/projectsveltos) is a Kubernetes add-on controller that simplifies the deployment and management of add-ons and
applications across multiple clusters. It runs in the management cluster and can programmatically deploy and manage add-ons and applications
on any cluster in the fleet, including the management cluster itself. 
Sveltos supports a variety of add-on formats, including Helm charts, raw YAML, Kustomize, Carvel ytt, and Jsonnet.

## Install Claudie in the management cluster

To deploy Claudie in the management cluster, simply follow Claudie's concise [instructions](kubectl apply -f https://github.com/berops/claudie/releases/latest/download/claudie.yaml)

At the time of this demo, this command is the only step required for deployment:

```
kubectl apply -f https://github.com/berops/claudie/releases/latest/download/claudie.yaml
```

## Install Sveltos in the management cluster 

To deploy Sveltos in the management cluster, follow Sveltos's [instructions](https://projectsveltos.github.io/sveltos/install/install/):

```
kubectl apply -f https://raw.githubusercontent.com/projectsveltos/sveltos/main/manifest/manifest.yaml
kubectl apply -f https://raw.githubusercontent.com/projectsveltos/sveltos/main/manifest/default-classifier.yaml
```

## Install claudie-sveltos

Claudie seamlessly provisions Kubernetes clusters and manages their lifecycle. Upon cluster provisioning, Claudie creates a secret in the
management cluster containing the kubeconfig to access the newly created cluster.

Sveltos can effectively manage add-ons and applications in any cluster, including those powered by Claudie.
It simply requires the kubeconfig information stored in the management cluster secret.

To seamlessly integrate Claudie-powered clusters with Sveltos, deploy an additional controller in the management cluster:

```
kubectl apply -f https://raw.githubusercontent.com/gianlucam76/claudie-sveltos-integration/main/manifest/manifest.yaml
```

This controller continuously monitors Claudie's secrets and automatically creates corresponding SveltosCluster objects.
Conversely, when a secret is deleted, the controller ensures the corresponding SveltosCluster object is also removed.

![Claudie and Sveltos in the management cluster](https://github.com/gianlucam76/claudie-sveltos-integration/blob/main/docs/claudie-sveltos.png)

## Provision a Kubernetes cluster on GCP

Full instructions can be found on Claudie [website](https://docs.claudie.io/v0.6.3/input-manifest/providers/gcp/).

If you have a GCP project, recover the service-account key

```
gcloud iam service-accounts keys create claudie-credentials.json --iam-account=<YOUR SERVICE ACCOUNT>"
```

Create a secret in the management cluster using above information

```
kubectl create secret generic gcp-secret-1 --namespace=<YOUR NAMESPACE> --from-literal=gcpproject='<YOUR PROJECT>' --from-file=credentials=./claudie-credentials.json
```

Post the [InputManifest](gcp.yaml)

## Provision a Kubernetes cluster on Hetzner

Full instructions can be found on Claudie [website](https://docs.claudie.io/v0.6.3/input-manifest/providers/hetzner/).

Get Hetzner [API token](https://docs.hetzner.com/cloud/api/getting-started/generating-api-token/) and create a secret in the management cluster

```
kubectl create secret generic hetzner-secret-1 --namespace=mynamespace --from-literal=credentials='<YOUR API TOKEN>'
```

Post the [InputManifest](hetzner.yaml)


## Verify clusters are created

Run the following command to verify Claudie has successfully provisioned the clusters:

```
kubectl get inputmanifest                                    
NAME               STATUS
gcp-manifest       DONE
hetzner-manifest   DONE
```

## Verify SveltosClusters are created

At this point, two SveltosClusters have been automatically created

```
kubectl get sveltoscluster -A  
NAMESPACE   NAME              READY   VERSION
claudie     gcp-cluster       true    v1.24.0
claudie     hetzner-cluster   true    v1.24.0
```

## Deploy add-ons and applications with Sveltos

Sveltos can be used to deploy [add-ons and applications](https://projectsveltos.github.io/sveltos/addons/addons/)

In this tutorial we added following label _env: fv_ to both SveltosCluster instances.

Then we deployed two ClusterProfile instances, instructing Sveltos to deploy __Kyverno__ and __Nginx__ helm charts.

1. Kyverno [ClusterProfile](kyverno.yaml)
2. Nginx [ClusterProfile](nginx.yaml)

Sveltos will deploy add-ons and applications. You can use [Sveltosctl](https://github.com/projectsveltos/sveltosctl) to verify
helm charts have been deployed:

```
sveltosctl show addons 
+-------------------------+---------------+-----------+----------------+---------+-------------------------------+------------------+
|         CLUSTER         | RESOURCE TYPE | NAMESPACE |      NAME      | VERSION |             TIME              | CLUSTER PROFILES |
+-------------------------+---------------+-----------+----------------+---------+-------------------------------+------------------+
| claudie/gcp-cluster     | helm chart    | kyverno   | kyverno-latest | 3.0.1   | 2023-12-02 17:13:59 +0100 CET | kyverno          |
| claudie/gcp-cluster     | helm chart    | nginx     | nginx-latest   | 0.14.0  | 2023-12-02 17:14:19 +0100 CET | nginx            |
| claudie/hetzner-cluster | helm chart    | kyverno   | kyverno-latest | 3.0.1   | 2023-12-02 17:14:08 +0100 CET | kyverno          |
| claudie/hetzner-cluster | helm chart    | nginx     | nginx-latest   | 0.14.0  | 2023-12-02 17:14:17 +0100 CET | nginx            |
+-------------------------+---------------+-----------+----------------+---------+-------------------------------+------------------+
```

![Add-ons deployment](https://github.com/projectsveltos/sveltos/blob/main/docs/assets/claudie-sveltos.gif)

