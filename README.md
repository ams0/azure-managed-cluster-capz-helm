# CAPI Azure

## Prerequisites

- [Kind](https://kind.sigs.k8s.io/)
- [ClusterCTL](https://cluster-api.sigs.k8s.io/clusterctl/overview.html)
- [Helm](https://helm.sh)

## Deploy

Create a cluster

```
kind create cluster --name capi
clusterctl init --infrastructure azure
```

Edit values

```
helm install --set subscriptionID=<your sub id> .
```

Check the status with:
```
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
