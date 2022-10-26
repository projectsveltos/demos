# Kubernetes API Gateway: Sveltos with Contour and Kyverno

In this demo we will show how to use *Sveltos* to automatically deploy, in any matching CAPI powered clusters, following:
1. Kubernetes Gateway API (with Contour);
2. Kyverno and a ClusterPolicy to prevent any ServiceAccount/User/Group but cluster-admin, to modify any Gateway instance.

## Kubernetes Gateway API

From https://gateway-api.sigs.k8s.io

"Whether it’s roads, power, data centers, or Kubernetes clusters, infrastructure is built to be shared. However, shared infrastructure raises a common challenge - how to provide flexibility to users of the infrastructure while maintaining control by owners of the infrastructure?"

When deploying Kubernetes Gateway API we would want to achieve:
- Cluster admin to create the Gateway instance derived from a GatewayClass, defining what can attach (via Route attachment process) to the Gateway;
- Different teams to deploy their applications in the cluster and expose their services via same Gateway.

Doing so, then:
- Centralized policies such as TLS can be enforced on the Gateway by the cluster admin;
- Separate teams only worry about their services.

Ideally as soon as a new Cluster is created, irrespective of which admin created the cluster, we want to enforce above Gateway approach.
  
Here is where Sveltos can help you. Create a ClusterProfile that:
- selects all the clusters you intend to configure with Gateway API;
- deploys Kyverno helm chart;
- deploys a Kyverno ClusterPolicy preventing anybody (but cluster-admin) to update any Gateway instance;
- deploys Gateway API CRDs and Contour.

Let's create all necessary resources.

This demo only assumption is that you have a management cluster with ClusterAPI, Sveltos and at least one CAPI cluster with labels *key:production*.

## ConfigMaps containing needed Kubernetes resources

Create a ConfigMap with content of following:

```
wget https://raw.githubusercontent.com/projectsveltos/demos/main/api-gateway/contour-gateway-provisioner.yaml
```

```
kubectl create configmap contour-gateway-provisioner --from-file=contour-gateway-provisioner.yaml
```

Above ConfigMap contains following Kubernetes resources:
1. Namespace *projectcontour* to run the Gateway provisioner;
2. Contour CRDs;
3. Gateway API CRDs;
4. Gateway provisioner RBAC resources;
5. Gateway provisioner Deployment.

Create a ConfigMap with GatewayClass:

```
wget https://raw.githubusercontent.com/projectsveltos/demos/main/api-gateway/gateway-class.yaml
```

```
kubectl create configmap gatewayclass --from-file=gateway-class.yaml
```

Create a ConfigMap with Gateway instance:

```
wget https://raw.githubusercontent.com/projectsveltos/demos/main/api-gateway/gateway.yaml
```

```
kubectl create configmap gateway--from-file=gateway.yaml
```

Summarizing, the above created ConfigMaps containing following Kubernetes resources:
1. A GatewayClass named contour controlled by the Gateway provisioner (via the projectcontour.io/gateway-controller string)
2. A Gateway resource named contour in the projectcontour namespace, using the contour GatewayClass
Contour;
3. Envoy resources in the projectcontour namespace to implement the Gateway, i.e. a Contour deployment, an Envoy daemonset, an Envoy service, etc.

Finally create a ConfigMap containing a Kyverno ClusterPolicy which prevents any ServiceAccount (but cluster-admin) from creating/updating/deleting Gateway.

```
kubectl create -f https://raw.githubusercontent.com/projectsveltos/demos/main/api-gateway/kyverno-policy.yaml
```

## ClusterProfile

```
kubectl create -f https://raw.githubusercontent.com/projectsveltos/demos/main/api-gateway/clusterprofile.yaml
```

This ClusterProfile:
1. Selects all ClusterAPI powered clusters with label *env:production*;
2. Deploy Kyverno helm chart version v.2.6.0;
3. References all ConfigMaps created above, so deploying all Kubernetes resources containined in those ConfigMaps.

## See all that is deployed

We can use sveltosctl to list all that has been deployed in the CAPI cluster:

```
./bin/sveltosctl show features                                                                     
+-------------------------------------+-------------------------------------------------------------+----------------+---------------------------------------------+---------+-------------------------------+-----------------------+
|               CLUSTER               |                        RESOURCE TYPE                        |   NAMESPACE    |                    NAME                     | VERSION |             TIME              |   CLUSTER PROFILES    |
+-------------------------------------+-------------------------------------------------------------+----------------+---------------------------------------------+---------+-------------------------------+-----------------------+
| default/sveltos-management-workload | helm chart                                                  | kyverno        | kyverno-latest                              | 2.6.0   | 2022-10-24 19:35:05 -0700 PDT | gateway-configuration |
| default/sveltos-management-workload | :Namespace                                                  |                | gateway-system                              | N/A     | 2022-10-24 19:35:25 -0700 PDT | gateway-configuration |
| default/sveltos-management-workload | rbac.authorization.k8s.io:Role                              | gateway-system | gateway-api-admission                       | N/A     | 2022-10-24 19:35:26 -0700 PDT | gateway-configuration |
| default/sveltos-management-workload | rbac.authorization.k8s.io:RoleBinding                       | projectcontour | contour-gateway-provisioner-leader-election | N/A     | 2022-10-24 19:35:26 -0700 PDT | gateway-configuration |
| default/sveltos-management-workload | apiextensions.k8s.io:CustomResourceDefinition               |                | contourconfigurations.projectcontour.io     | N/A     | 2022-10-24 19:35:25 -0700 PDT | gateway-configuration |
| default/sveltos-management-workload | rbac.authorization.k8s.io:ClusterRoleBinding                |                | contour-gateway-provisioner                 | N/A     | 2022-10-24 19:35:26 -0700 PDT | gateway-configuration |
| default/sveltos-management-workload | apiextensions.k8s.io:CustomResourceDefinition               |                | udproutes.gateway.networking.k8s.io         | N/A     | 2022-10-24 19:35:25 -0700 PDT | gateway-configuration |
| default/sveltos-management-workload | apps:Deployment                                             | gateway-system | gateway-api-admission-server                | N/A     | 2022-10-24 19:35:26 -0700 PDT | gateway-configuration |
| default/sveltos-management-workload | apiextensions.k8s.io:CustomResourceDefinition               |                | extensionservices.projectcontour.io         | N/A     | 2022-10-24 19:35:25 -0700 PDT | gateway-configuration |
| default/sveltos-management-workload | apiextensions.k8s.io:CustomResourceDefinition               |                | tlscertificatedelegations.projectcontour.io | N/A     | 2022-10-24 19:35:25 -0700 PDT | gateway-configuration |
| default/sveltos-management-workload | apiextensions.k8s.io:CustomResourceDefinition               |                | tcproutes.gateway.networking.k8s.io         | N/A     | 2022-10-24 19:35:25 -0700 PDT | gateway-configuration |
| default/sveltos-management-workload | :ServiceAccount                                             | gateway-system | gateway-api-admission                       | N/A     | 2022-10-24 19:35:26 -0700 PDT | gateway-configuration |
| default/sveltos-management-workload | apps:Deployment                                             | projectcontour | contour-gateway-provisioner                 | N/A     | 2022-10-24 19:35:26 -0700 PDT | gateway-configuration |
| default/sveltos-management-workload | gateway.networking.k8s.io:Gateway                           | projectcontour | contour                                     | N/A     | 2022-10-24 19:35:27 -0700 PDT | gateway-configuration |
| default/sveltos-management-workload | apiextensions.k8s.io:CustomResourceDefinition               |                | httpproxies.projectcontour.io               | N/A     | 2022-10-24 19:35:25 -0700 PDT | gateway-configuration |
| default/sveltos-management-workload | apiextensions.k8s.io:CustomResourceDefinition               |                | gatewayclasses.gateway.networking.k8s.io    | N/A     | 2022-10-24 19:35:25 -0700 PDT | gateway-configuration |
| default/sveltos-management-workload | apiextensions.k8s.io:CustomResourceDefinition               |                | tlsroutes.gateway.networking.k8s.io         | N/A     | 2022-10-24 19:35:25 -0700 PDT | gateway-configuration |
| default/sveltos-management-workload | batch:Job                                                   | gateway-system | gateway-api-admission-patch                 | N/A     | 2022-10-24 19:35:26 -0700 PDT | gateway-configuration |
| default/sveltos-management-workload | gateway.networking.k8s.io:GatewayClass                      |                | contour                                     | N/A     | 2022-10-24 19:35:27 -0700 PDT | gateway-configuration |
| default/sveltos-management-workload | apiextensions.k8s.io:CustomResourceDefinition               |                | contourdeployments.projectcontour.io        | N/A     | 2022-10-24 19:35:25 -0700 PDT | gateway-configuration |
| default/sveltos-management-workload | apiextensions.k8s.io:CustomResourceDefinition               |                | gateways.gateway.networking.k8s.io          | N/A     | 2022-10-24 19:35:25 -0700 PDT | gateway-configuration |
| default/sveltos-management-workload | admissionregistration.k8s.io:ValidatingWebhookConfiguration |                | gateway-api-admission                       | N/A     | 2022-10-24 19:35:26 -0700 PDT | gateway-configuration |
| default/sveltos-management-workload | :Namespace                                                  |                | projectcontour                              | N/A     | 2022-10-24 19:35:26 -0700 PDT | gateway-configuration |
| default/sveltos-management-workload | :ServiceAccount                                             | projectcontour | contour-gateway-provisioner                 | N/A     | 2022-10-24 19:35:26 -0700 PDT | gateway-configuration |
| default/sveltos-management-workload | rbac.authorization.k8s.io:ClusterRole                       |                | contour-gateway-provisioner                 | N/A     | 2022-10-24 19:35:26 -0700 PDT | gateway-configuration |
| default/sveltos-management-workload | rbac.authorization.k8s.io:ClusterRole                       |                | gateway-api-admission                       | N/A     | 2022-10-24 19:35:26 -0700 PDT | gateway-configuration |
| default/sveltos-management-workload | rbac.authorization.k8s.io:ClusterRoleBinding                | gateway-system | gateway-api-admission                       | N/A     | 2022-10-24 19:35:26 -0700 PDT | gateway-configuration |
| default/sveltos-management-workload | rbac.authorization.k8s.io:RoleBinding                       | gateway-system | gateway-api-admission                       | N/A     | 2022-10-24 19:35:26 -0700 PDT | gateway-configuration |
| default/sveltos-management-workload | batch:Job                                                   | gateway-system | gateway-api-admission                       | N/A     | 2022-10-24 19:35:26 -0700 PDT | gateway-configuration |
| default/sveltos-management-workload | rbac.authorization.k8s.io:Role                              | projectcontour | contour-gateway-provisioner                 | N/A     | 2022-10-24 19:35:26 -0700 PDT | gateway-configuration |
| default/sveltos-management-workload | apiextensions.k8s.io:CustomResourceDefinition               |                | httproutes.gateway.networking.k8s.io        | N/A     | 2022-10-24 19:35:25 -0700 PDT | gateway-configuration |
| default/sveltos-management-workload | apiextensions.k8s.io:CustomResourceDefinition               |                | referencegrants.gateway.networking.k8s.io   | N/A     | 2022-10-24 19:35:25 -0700 PDT | gateway-configuration |
| default/sveltos-management-workload | apiextensions.k8s.io:CustomResourceDefinition               |                | referencepolicies.gateway.networking.k8s.io | N/A     | 2022-10-24 19:35:25 -0700 PDT | gateway-configuration |
| default/sveltos-management-workload | :Service                                                    | gateway-system | gateway-api-admission-server                | N/A     | 2022-10-24 19:35:26 -0700 PDT | gateway-configuration |
| default/sveltos-management-workload | kyverno.io:ClusterPolicy                                    |                | no-gateway                                  | N/A     | 2022-10-24 19:35:26 -0700 PDT | gateway-configuration |
+-------------------------------------+-------------------------------------------------------------+----------------+---------------------------------------------+---------+-------------------------------+-----------------------+
```

## Testing the Gateway API

Deploy the test application in the CAPI cluster: 

```
KUBECONFIG=<CAPI Cluster kubeconfig> kubectl apply -f https://raw.githubusercontent.com/projectcontour/contour/v1.23.0/examples/example-workload/gatewayapi/kuard/kuard.yaml
```

This command creates:

1. A Deployment named kuard in the default namespace to run kuard as the test application;
2. A Service named kuard in the default namespace to expose the kuard application on TCP port 80;
3. An HTTPRoute named kuard in the default namespace, attached to the contour Gateway, to route requests for local.projectcontour.io to the kuard service.


Port-forward from your local machine to the Envoy service:

```
KUBECONFIG=<CAPI Cluster kubeconfig> kubectl -n projectcontour port-forward service/envoy-contour 8888:80
```

In another terminal, make a request to the application via the forwarded port (note, local.projectcontour.io is a public DNS record resolving to 127.0.0.1 to make use of the forwarded port):

```
curl -i http://local.projectcontour.io:8888
```

## See Kyverno policy in action

Create a *ClusterRole* giving it all permissions:

```
KUBECONFIG=<CAPI CLuster kubeconfig>  kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: capi-cluster-admin
rules:
- apiGroups:
  - '*'
  resources:
  - '*'
  verbs:
  - '*'
- nonResourceURLs:
  - '*'
  verbs:
  - '*'
EOF
```

Create a *ClusterRoleBinding* associating user _test_ to above ClusterRole.

```
KUBECONFIG=<CAPI CLuster kubeconfig>  kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: capi-cluster-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: capi-cluster-admin
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: User
  name: test
EOF
```

Even though user test has RBAC permissions, if it tries to modify Gateway configuration, the update gets rejected by Kyverno webhook because of the Kyverno ClusterPolicy installed by ClusterProfile.

```
KUBECONFIG=<CAPI CLuster kubeconfig> kubectl edit gateway -n projectcontour   contour  --as=test
```

```
error: gateways.gateway.networking.k8s.io "contour" could not be patched: admission webhook "validate.kyverno.svc-fail" denied the request: 

policy Gateway/projectcontour/contour for resource violation: 

no-gateway:
  block-gateway-updates: Gateway's configurations is managed by management cluster
    admin.
```