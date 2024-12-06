# Install Sveltos

helm repo add projectsveltos https://projectsveltos.github.io/helm-charts
helm repo update
helm install projectsveltos projectsveltos/projectsveltos -n projectsveltos --create-namespace

# Register cluster

kubect create ns gke
sveltosctl register cluster --namespace=gke --cluster=cluster --kubeconfig=gke_kubeconfig  --labels=env=production 

kubect create ns civo
sveltosctl register cluster --namespace=civo --cluster=cluster --kubeconfig=gke_kubeconfig

# Deploy Kyverno Helm chart

kubectl apply -f clusterprofile-deploy-kyverno.yaml

this deploys kyverno to GKE cluster

sveltosctl show addons

# Add label to Civo cluster

kubectl label sveltoscluster -n civo cluster env=production

Kyverno is now also deployed to civo

# Deploy Kyverno ClusterPolicy

kubectl apply -f disallow-empty-ingress-host.yaml
kubectl apply -f disallow-latest-tag.yaml
kubectl apply -f clusterprofile-deploy-kyverno-resources.yaml

# Deploy Dashboard

kubectl apply -f https://raw.githubusercontent.com/projectsveltos/sveltos/main/manifest/dashboard-manifest.yaml 

kubectl create sa platform-admin
kubectl create clusterrolebinding platform-admin-access --clusterrole cluster-admin --serviceaccount default:platform-admin
kubectl create token platform-admin 


kubectl -n projectsveltos  port-forward service/dashboard   8080:80

