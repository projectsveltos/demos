In this demo we will show how to use Sveltos to automatically deploy, in any matching clusters, Kubernetes Gateway API (with Contour) and use Sveltos event drivern framework to autogenerate HTPPRoute instances.

We already covered [here](https://medium.com/@projectsveltos/how-to-deploy-l4-and-l7-routing-on-multiple-kubernetes-clusters-securely-and-programmatically-930ebe65fa8c) how to deploy L4 and L7 routing on multiple Kubernetes clusters securely and programmatically with Sveltos.

With event driven framework, we are now taking a step forward: programmatically generate/update HTTPRoutes.

In order to achieve that:

1. define a Sveltos Event as creation/deletion of specific Service instances (in our example, the Service instances we are interested in are in the namespace *eng* and are exposing port *80*);
2. define what add-ons to deploy in response to such events: an HTTPRoute instance defined as a template. Sveltos will instantiate this template using information from Services in the managed clusters that are part of the event defined in #1. 

## Kubernetes API Gateway

In the management cluster crepare ConfigMaps and Secret containing Kubernetes API Gateway.

### Gateway API and Contour CRDs

Download this file

```bash
wget https://raw.githubusercontent.com/projectsveltos/demos/main/httproute/gateway-class.yaml
```

which contains:

1. Namespace projectcontour to run the Gateway provisioner
1. Contour CRDs
1. Gateway API CRDs
1. Gateway provisioner RBAC resources
1. Gateway provisioner Deployment

and create a ConfigMap in the management cluster containing content of that file[^1].

```bash
kubectl create secret generic contour-gateway-provisioner-secret --from-file=contour-gateway-provisioner.yaml --type=addons.projectsveltos.io/cluster-profile
```

### GatewayClass

In the management cluster, create a ConfigMap with GatewayClass:

```bash
wget https://raw.githubusercontent.com/projectsveltos/demos/main/httproute/gateway-class.yaml

kubectl create configmap gatewayclass --from-file=gateway-class.yaml
```

### Gateway

and another ConfigMap with Gateway instance:

```bash
wget https://raw.githubusercontent.com/projectsveltos/demos/main/httproute/gateway.yaml

kubectl create configmap gateway --from-file=gateway.yaml
```

## ClusterProfile

Create a ClusterProfile that deploys Gateway API and Contour resources. We want those resources installed once per cluster.

```yaml
apiVersion: config.projectsveltos.io/v1alpha1
kind: ClusterProfile
metadata:
 name: gateway-configuration
spec:
 clusterSelector: env=fv
 eventSourceName: eng-http-service
 oneForEvent: true
 policyRefs:
 - name: contour-gateway-provisioner-secret
   namespace: default
   kind: Secret
 - name: gateway
   namespace: default
   kind: ConfigMap
 - name: gatewayclass
   namespace: default
   kind: ConfigMap
```

## EventSource

Define a Sveltos Event as creation/deletion of specific Service instances (in our example, the Service instances we are interested in are in the namespace *eng* and are exposing port *80*);

```yaml
apiVersion: lib.projectsveltos.io/v1alpha1
kind: EventSource
metadata:
 name: eng-http-service
spec:
 collectResources: true
 group: ""
 version: "v1"
 kind: "Service"
 namespace: eng
 script: |
   function evaluate()
     hs = {}
     hs.matching = false
     if obj.spec.ports ~= nil then
       for _,p in pairs(obj.spec.ports) do
         if p.port == 80 then
           hs.matching = true
         end
       end
     end
     return hs
   end
```

## EventBasedAddOn

Define an EventBasedAddOn instance that reacts to events defined above and deploy HTTPRoute as response.
The HTTPRoute is defined as a template in a ConfigMap.
Sveltos will instantiate it using information from Services in managed cluster matching the EventSource (Resource is a Service instance in the managed cluster matching EventSource).

An HTTPRoute instance will be created for each Service matching the EventSource.

```yaml
apiVersion: lib.projectsveltos.io/v1alpha1
kind: EventBasedAddOn
metadata:
 name: service-network-policy
spec:
 clusterSelector: env=fv
 eventSourceName: eng-http-service
 oneForEvent: true
 policyRefs:
 - name: http-routes
   namespace: default
   kind: ConfigMap
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: http-routes
  namespace: default
data:
  http-route.yaml: |
    kind: HTTPRoute
    apiVersion: gateway.networking.k8s.io/v1beta1
    metadata:
      name: {{ .Resource.metadata.name }}
      namespace: {{ .Resource.metadata.namespace }}
      labels:
        {{ range $key, $value := .Resource.spec.selector }}
        {{ $key }}: {{ $value }}
        {{ end }}
    spec:
      parentRefs:
      - group: gateway.networking.k8s.io
        kind: Gateway
        name: contour
        namespace: projectcontour
      hostnames:
      - "local.projectcontour.io"
      rules:
      - matches:
        - path:
            type: PathPrefix
            value: /{{ .Resource.metadata.name }}
        backendRefs:
        - kind: Service
          name: {{ .Resource.metadata.name }}
          port: {{ (index .Resource.spec.ports 0).port }}
```

## Install kuard app in the managed cluster

Now deploy kuard configuration in the managed cluster. This contains a Service instance that is a match for EventSource we defined above.

```bash
KUBECONFIG=<Managed Cluster kubeconfig> kubectl create ns eng
```

```bash
KUBECONFIG=<Managed Cluster kubeconfig> kubectl -n https://raw.githubusercontent.com/projectsveltos/demos/main/httproute/kuard.yaml
```

As soon as this is posted, *sveltos-agent* running in the managed cluster detects the event, reports it to the management cluster. In the management cluster, *event-manager* pods reacts by instantiating necessary configuration.

We can see that HTTPRoute has been created in the managed cluster as consequence:

```bash
KUBECONFIG=<Managed Cluster kubeconfig>   kubectl get httproutes -A

NAMESPACE   NAME    HOSTNAMES                     AGE
default     kuard   ["local.projectcontour.io"]   22s
```

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  annotations:
    projectsveltos.io/hash: sha256:50470c676add0df13162d88cdc32d3e62dfbaae88ac91777840bd9d124526d5a
  creationTimestamp: "2023-03-19T22:36:02Z"
  generation: 1
  labels:
    app: kuard
    projectsveltos.io/reference-kind: ConfigMap
    projectsveltos.io/reference-name: sveltos-xqibvmocxhizaa6assc5
    projectsveltos.io/reference-namespace: projectsveltos
  name: kuard
  namespace: eng
  ownerReferences:
  - apiVersion: config.projectsveltos.io/v1alpha1
    kind: ClusterProfile
    name: sveltos-98sqznlc2gymvmfbci86
    uid: 2bca609e-db2a-4d0a-bab1-005af167dec4
  resourceVersion: "1904"
  uid: 06748e1b-9120-47dd-af2c-e382e4856e62
spec:
  hostnames:
  - local.projectcontour.io
  parentRefs:
  - group: gateway.networking.k8s.io
    kind: Gateway
    name: contour
    namespace: projectcontour
  rules:
  - backendRefs:
    - group: ""
      kind: Service
      name: kuard
      port: 80
      weight: 1
    matches:
    - path:
        type: PathPrefix
        value: /kuard
```


[^1]: This is same file available [here](https://projectcontour.io/quickstart/contour-gateway-provisioner.yaml).  There is currently an issues with Sveltos processing sections containing just comment and empty spaces. That is why we copied it locally, to simply remove this section.

```yaml
# This file is generated from the individual YAML files by generate-provisioner-deployment.sh. Do not
# edit this file directly but instead edit the source files and re-render.
#
# Generated from:
#       examples/contour/01-crds.yaml
#       examples/gateway/00-crds.yaml
#       examples/gateway/00-namespace.yaml
#       examples/gateway/01-admission_webhook.yaml
#       examples/gateway/02-certificate_config.yaml
#       examples/gateway-provisioner/00-common.yaml
#       examples/gateway-provisioner/01-roles.yaml
#       examples/gateway-provisioner/02-rolebindings.yaml
#       examples/gateway-provisioner/03-gateway-provisioner.yaml

---
```