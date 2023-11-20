## Environment

### Creating a Management Cluster

To create a management cluster using Kind and the latest version of Projectsveltos, follow these steps:

```
git clone https://github.com/projectsveltos/addon-controller
cd addon-controller
make quickstart
```

This will create a management cluster using __Kind__ and installs latest version of __Projectsvelts__

### Registering Civo Clusters

[Civo](https://www.civo.com) was used to create managed cluster:

1. create clusters using Civo UI
2. download Kubeconfig of each Civo cloud
3. use sveltosctl to register Civo clusters to be managed by Projectsveltos
```
sveltosctl/bin/sveltosctl register cluster --namespace=civo --cluster=cluster1 --kubeconfig=<Path to CIVO Cluster Kubeconfig>
```

### Verifying Registration
To verify that the Civo clusters have been registered with Projectsveltos, run the following command:

```
kubectl get sveltosclusters -A
```

This will display a list of registered clusters, including their namespace, name, readiness status, and version. In our case:


```
kubectl get sveltosclusters -A
NAMESPACE   NAME       READY   VERSION
civo        cluster1   true    v1.26.4+k3s1
civo        cluster2   true    v1.27.1+k3s1
civo        cluster3   true    v1.28.2+k3s1
```

Edit each of the SveltosCluster instance by adding the label `env: fv`

## Configure Projectsveltos to detect crashing pods

Projectsveltos can be configured to monitor managed clusters and report events, such as pods entering a crashloopbackoff state. 
In this case, we want to instruct Projectsveltos to detect crashing pods and deploy a job that will collect logs and resources.

### EventSource and EventBasedAddOn CustomResourceDefinition

![Projectsveltos notifications](collect_logs_resources.gif)

The EventSource CRD is a mechanism for instructing Projectsveltos what to monitor.

Define a __EventSource__ instance with a Lua function. Following will instruct Projectsveltos to:

1. watch for Pods
2. consider only Pods in CrashLoopBackOff

```yaml
apiVersion: lib.projectsveltos.io/v1alpha1
kind: EventSource
metadata:
 name: crashing-pod
spec:
 group: ""
 version: "v1"
 kind: "Pod"
 collectResources: true
 script: |
  function evaluate()
     hs = {}
     hs.matching = false
     hs.message = ""
     if obj.status.containerStatuses then
        local containerStatuses = obj.status.containerStatuses
        for _, containerStatus in ipairs(containerStatuses) do
          if containerStatus.state.waiting and containerStatus.state.waiting.reason == "CrashLoopBackOff" then
            hs.matching = true
            hs.message = obj.metadata.namespace .. "/" .. obj.metadata.name .. ":" .. containerStatus.state.waiting.message
            if containerStatus.lastState.terminated and containerStatus.lastState.terminated.reason then
              hs.message = hs.message .. "\nreason:" .. containerStatus.lastState.terminated.reason
            end
          end
        end
     end
     return hs
  end
```

A __EventBasedAddOn__ instance references a __EventSource__, selects a subset of managed clusters where Projectsveltos will monitor for events. It also tells Projectsveltos what applications to deploy when event is detected.

Define a __ClusterHealthCheck__ instance:

```yaml
apiVersion: lib.projectsveltos.io/v1alpha1
kind: EventBasedAddOn
metadata:
 name: hc
spec:
 sourceClusterSelector: env=fv
 eventSourceName: crashing-pod
 oneForEvent: true
 stopMatchingBehavior: LeavePolicies
 policyRefs:
 - name: k8s-collector
   namespace: default
   kind: ConfigMap
```

The ConfigMap __default/k8s-collector__ can be created with following command:

1. kubectl apply -f https://raw.githubusercontent.com/projectsveltos/demos/main/collect-logs/configmap.yaml

## Create Pods

Create following Pod in one of your managed cluster. 

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: memory-demo
  namespace: default
spec:
  containers:
    - name: memory-demo-ctr
      image: polinux/stress
      resources:
        requests:
          memory: 100Mi
        limits:
          memory: 200Mi
      command: ["stress"]
      args: ["--vm", "1", "--vm-bytes", "250M", "--vm-hang", "1"]
```

This Pod will keep going in out of memory.

When Projectsveltos detects there is a Pod in a crashloopbackoff state, it will deploy the content of ConfigMap ```default/k8s-collector``` in the corresponding managed cluster.

That will create:

1. a PersistentVolumeClaim in the Civo cluster
2. deploys a Job ```k8s-collector``` which will collect logs and resources and store them in the mounted volume.

To know more about the [k8s-collector](https://github.com/gianlucam76/k8s_collector)

```bash
kubectl logs -n default k8s-collector-memory-demo-hngld 
I1120 14:54:01.480857       1 collect.go:53] "getting ConfigMap " configmap="default/k8s-collector-default-memory-demo"
I1120 14:54:01.498416       1 collect.go:88] "collecting logs"
I1120 14:54:01.508153       1 logs.go:74] "found 3 pods"
I1120 14:54:01.553703       1 resources.go:42] "collecting resources" gvk=":v1:Pod"
I1120 14:54:01.588363       1 resources.go:104] "collected 3 resources" gvk=":v1:Pod"
I1120 14:54:01.594969       1 resources.go:159] "storing resource in /collection/resources/default/Pod/install-traefik2-nodeport-cl-5xdpg.yaml" gvk=":v1:Pod" kind="Pod" resource="default install-traefik2-nodeport-cl-5xdpg"
I1120 14:54:01.599912       1 resources.go:159] "storing resource in /collection/resources/default/Pod/k8s-collector-memory-demo-hngld.yaml" gvk=":v1:Pod" kind="Pod" resource="default k8s-collector-memory-demo-hngld"
I1120 14:54:01.602357       1 resources.go:159] "storing resource in /collection/resources/default/Pod/memory-demo.yaml" gvk=":v1:Pod" kind="Pod" resource="default memory-demo"
I1120 14:54:01.602732       1 resources.go:42] "collecting resources" gvk="apps:v1:Deployment"
I1120 14:54:01.626599       1 resources.go:104] "collected 0 resources" gvk="apps:v1:Deployment"
...
```
