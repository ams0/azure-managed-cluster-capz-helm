# CAPI Azure

## Prerequisites

- [Kind](https://kind.sigs.k8s.io/)
- [ClusterCTL](https://cluster-api.sigs.k8s.io/clusterctl/overview.html)
- [Helm](https://helm.sh)

## Deploy

Edit `clusterctl.env` with subscription/tenant IDs and parse it:

```
source clusterctl.env
```

Create a KIND cluster:

```
kind create cluster --name capi
clusterctl init --infrastructure azure
```

Clone this repo and edit values, then deploy:

```
git clone https://github.com/ams0/azure-managed-cluster-capz-helm.git
cd azure-managed-cluster-capz-helm

helm install --set subscriptionID=<your sub id>  aks charts/azure-managed-cluster/ 
```

Check the status with:
```
kubectl get cluster-api
kubectl  logs -n capz-system -l control-plane=capz-controller-manager -c manager -f
```

Get the credentials

```
kubectl get secret {cluster-name}-kubeconfig -o yaml -o jsonpath={.data.value} | base64 --decode > akscapi.kubeconfig
```

Test the cluster!

```
kubectl --kubeconfig=subscriptionID cluster-info
```

Deploy a second cluster

```
helm install --set subscriptionID=<your sub id> -f values2.yaml aks2 charts/azure-managed-cluster/
```

Clean up:

```
helm delete aks
helm delete aks2

kind delete clusters capi
```