# CAPI Azure

## Prerequisites
- [Docker Desktop](https://www.docker.com/)
- [Kind](https://kind.sigs.k8s.io/)
- [ClusterCTL](https://cluster-api.sigs.k8s.io/clusterctl/overview.html) Version 0.4.4 or older
- [Helm](https://helm.sh) version 3

## Prerequisites Installations
Docker Desktop
https://www.docker.com/products/docker-desktop 

Install Kind
```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.11.1/kind-linux-amd64
chmod +x ./kind
mv ./kind /some-dir-in-your-PATH/kind
```
Install Clusterctl
```bash
curl -L https://github.com/kubernetes-sigs/cluster-api/releases/download/v0.4.4/clusterctl-linux-amd64 -o clusterctl
chmod +x ./clusterctl
sudo mv ./clusterctl /usr/local/bin/clusterctl
clusterctl version
```
Install Helm3
```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```
## Deploy

Clone this repo

```bash
git clone https://github.com/ams0/azure-managed-cluster-capz-helm.git
cd azure-managed-cluster-capz-helm

```

Create an Azure Service Principal
```bash
az ad sp create-for-rbac -n "capz" --role Contributor
```

Save outuput of this command somewhere.AppId is the AZURE_CLIENT_ID used on the next step. and Password is the CLIENT_SECRET


Create and Update Environment variables
Edit your ~/.bashrc file to include Azure Service Principal environment variables or just export on your current terminal session
```bash
export AZURE_CLIENT_ID=""

export AZURE_CLIENT_SECRET=""

export AZURE_SUBSCRIPTION_ID=""

export AZURE_TENANT_ID=""
```

Load Environment variables
```bash
source clusterctl.env
```

Create a KIND cluster:

```bash
kind create cluster --name capi-helm
```

Create a secret to include the password of the Service Principal identity created in Azure
This secret will be referenced by the AzureClusterIdentity used by the AzureCluster

```bash
kubectl create secret generic "${AZURE_CLUSTER_IDENTITY_SECRET_NAME}" --from-literal=clientSecret="${AZURE_CLIENT_SECRET}"
```

Initialize Cluster API and install Azure CAPZ provider version 0.5.3(uses alphav4 capi)

```bash
clusterctl init --infrastructure azure:v0.5.3
```

Deploy a cluster with Helm (please customize parameters as required)

Requirement: SSH public key~/.ssh/id_rsa.pub, to create a key use command "ssh-keygen -t rsa"

```bash
helm install capz1 charts/azure-managed-cluster/  \
--namespace default \
--set subscriptionID="${AZURE_SUBSCRIPTION_ID}" \
--set identity.clientId="${AZURE_CLIENT_ID}" \
--set identity.tenantId="${AZURE_TENANT_ID}" \
--set cluster.resourceGroupName=aksclusters \
--set cluster.nodeResourceGroupName=capz1 \
--set cluster.name=aks1 \
--set controlplane.sshPublicKey="$(cat ~/.ssh/id_rsa.pub)" \
--set agentpools[0].name=capz1np0 \
--set agentpools[0].mode=System \
--set agentpools[0].nodecount=1 \
--set agentpools[0].sku=Standard_B2s \
--set agentpools[0].osDiskSizeGB=100 \
--set agentpools[1].name=capz1np1 \
--set agentpools[1].mode=User \
--set agentpools[1].nodecount=1 \
--set agentpools[1].sku=Standard_B2s \
--set agentpools[1].osDiskSizeGB=100
```

or more simply (after you edit the values file with your own values):

```bash
helm install capz1 charts/azure-managed-cluster/ --values aks1.yaml \
--namespace default \
--set controlplane.sshPublicKey="$(cat ~/.ssh/id_rsa.pub)" \
--set subscriptionID="${AZURE_SUBSCRIPTION_ID}" \
--set identity.clientId="${AZURE_CLIENT_ID}" \
--set identity.tenantId="${AZURE_TENANT_ID}"
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
Create second namespace
```bash
kubectl create namespace default2
```

```bash
helm install capz2 charts/azure-managed-cluster/  \
--namespace default2 \
--set subscriptionID="${AZURE_SUBSCRIPTION_ID}" \
--set identity.clientId="${AZURE_CLIENT_ID}" \
--set identity.tenantId="${AZURE_TENANT_ID}" \
--set cluster.resourceGroupName=aksclusters \
--set cluster.nodeResourceGroupName=capz2 \
--set cluster.name=aks2 \
--set controlplane.sshPublicKey="$(cat ~/.ssh/id_rsa.pub)" \
--set agentpools[0].name=capz2np0 \
--set agentpools[0].mode=System \
--set agentpools[0].nodecount=1 \
--set agentpools[0].sku=Standard_B2s \
--set agentpools[0].osDiskSizeGB=100 \
--set agentpools[1].name=capz2np1 \
--set agentpools[1].mode=User \
--set agentpools[1].nodecount=1 \
--set agentpools[1].sku=Standard_B2s \
--set agentpools[1].osDiskSizeGB=100
```

or more simply (after you edit the values file with your own values):

```bash
helm install capz2 charts/azure-managed-cluster/ --values aks2.yaml \
--namespace default2 \
--set controlplane.sshPublicKey="$(cat ~/.ssh/id_rsa.pub)" \
--set subscriptionID="${AZURE_SUBSCRIPTION_ID}" \
--set identity.clientId="${AZURE_CLIENT_ID}" \
--set identity.tenantId="${AZURE_TENANT_ID}"
```

Clean up:

```bash
helm delete capz1
helm delete capz2 -n default2
kubectl delete namespace default2

kind delete clusters capi-helm
```
