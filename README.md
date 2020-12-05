# CAPI Azure

## Prerequisites

- [Kind](https://kind.sigs.k8s.io/)
- [ClusterCTL](https://cluster-api.sigs.k8s.io/clusterctl/overview.html)
- [Helm](https://helm.sh)

## Deploy

Create a KIND cluster:

```bash
kind create cluster --name capi
clusterctl init --infrastructure azure
```

Clone this repo, edit `clusterctl.env` with subscription/tenant IDs and parse it:

```bash
git clone https://github.com/ams0/azure-managed-cluster-capz-helm.git
cd azure-managed-cluster-capz-helm

source clusterctl.env
```

Deploy a cluster with Helm

```bash
helm install capz1 charts/azure-managed-cluster/  \
--set subscriptionID=12c7e9d6-967e-40c8-8b3e-4659a4ada3ef \
--set cluster.resourceGroupName=aksclusters \
--set cluster.nodeResourceGroupName=capz1 \
--set cluster.name=aks1 \
--set controlplane.sshPublicKey="$(cat ~/.ssh/id_rsa.pub)" \
--set agentpools[0].name=capz1np0 \
--set agentpools[0].nodecount=1 \
--set agentpools[0].sku=Standard_B4ms \
--set agentpools[0].osDiskSizeGB=100 \
--set agentpools[1].name=capz1np1 \
--set agentpools[1].nodecount=1 \
--set agentpools[1].sku=Standard_B4ms \
--set agentpools[1].osDiskSizeGB=100 
```

or more simply (after you edit the values file with your own values):

```bash
helm install capz1 charts/azure-managed-cluster/ --values aks1.yaml
```

Check the status with:
```
kubectl get cluster-api
kubectl  logs -n capz-system -l control-plane=capz-controller-manager -c manager -f
```

Get the credentials

```
kubectl get secret {cluster-name}-kubeconfig -o yaml -o jsonpath={.data.value} | base64 --decode > aks1.kubeconfig
```

Test the cluster!

```
kubectl --kubeconfig=aks1.kubeconfig cluster-info
```

Deploy a second cluster

```
helm install capz2 charts/azure-managed-cluster/  \
--set subscriptionID=12c7e9d6-967e-40c8-8b3e-4659a4ada3ef \
--set cluster.resourceGroupName=aksclusters \
--set cluster.nodeResourceGroupName=capz2 \
--set cluster.name=aks2 \
--set controlplane.sshPublicKey="$(cat ~/.ssh/id_rsa.pub)" \
--set agentpools[0].name=capz2np0 \
--set agentpools[0].nodecount=1 \
--set agentpools[0].sku=Standard_B4ms \
--set agentpools[0].osDiskSizeGB=100 \
--set agentpools[1].name=capz2np1 \
--set agentpools[1].nodecount=1 \
--set agentpools[1].sku=Standard_B4ms \
--set agentpools[1].osDiskSizeGB=100 
```

or more simply (after you edit the values file with your own values):

```bash
helm install capz2 charts/azure-managed-cluster/ --values aks2.yaml
```

Clean up:

```bash
helm delete capz1
helm delete capz2

kind delete clusters capi
```
