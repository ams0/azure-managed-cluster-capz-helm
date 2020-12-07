
# Create AKS clusters with a Helm chart? It’s possible with ClusterAPI!

ClusterAPIs are Kubernetes-native APIs to manage the lifecycle of clusters and nodes as CRDs. 

*Code available at [https://github.com/ams0/azure-managed-cluster-capz-helm](https://github.com/ams0/azure-managed-cluster-capz-helm)*

I’ve been following from a distance the [ClusterAPI](https://github.com/kubernetes-sigs/cluster-api) project for some time now, but my interest got suddenly renewed when I find out that the [Azure provider](https://github.com/kubernetes-sigs/cluster-api-provider-azure/) now [supports ](https://github.com/kubernetes-sigs/cluster-api-provider-azure/blob/master/docs/book/src/topics/managedcluster.md)AKS managed clusters. So I set out to come up with some simple instructions on how to create AKS clusters programmatically; however, since creating clusters requires a number of templates (4, as of now), I decided to write a simple Helm chart to easily create clusters. 

### Preparation

First, you’ll need to [get ](https://cluster-api.sigs.k8s.io/user/quick-start.html#install-clusterctl)the clusterctl command line (I recommend to use the latest at the time of writing, [v0.3.11](https://github.com/kubernetes-sigs/cluster-api/releases/tag/v0.3.11)); then, you’ll need to have a Kubernetes cluster to store the CRDs for your AKS clusters. You can use any cluster (another AKS cluster for example) or to keep things simple, you can spin up a local [KIND ](https://kind.sigs.k8s.io/)cluster, and that’s what we will do here (my preferred way to obtain kind and other tools is to use the gofish [package manager](https://gofi.sh/)):

    kind create cluster --name capi

Next, use clusterctl to install the ClusterAPI components:

    clusterctl init --infrastructure azure

Wait until all components are deployed in the various capi-*-system namespaces.

<iframe src="https://medium.com/media/eff76545b130976ca4be4b5e51da70ce" frameborder=0></iframe>

Now clone the repo and edit the values in cluster.env :

    git clone https://github.com/ams0/azure-managed-cluster-capz-helm.git
    cd azure-managed-cluster-capz-helm
    
    source clusterctl.env

### Deploy a cluster

Ready to roll! Deploy your first cluster with Helm:

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
    --set agentpools[1].osDiskSizeGB=10

If you like you can use a values.yaml file:

    helm install capz1 charts/azure-managed-cluster/ --values aks1.yaml

The options available are a bit limited at the moment, but the ClusterAPI azure provider will surely cover all the missing cluster options soon. Note that the cluster is created with aSystemAssigned managed identity, the SP from cluster.env is the one needed to *create *the cluster itself, not the one used by the cluster to interact with Azure via the cloud-controller-manager.

You can check the status of the deployment in several ways; in addition to the obvious az aks list to check the Azure side, you can check the cluster-api object and get the logs of the capz-controller-manager pod:

    kubectl get cluster-api
    kubectl  logs -n capz-system -l control-plane=capz-controller-manager -c manager -f

### Access the cluster

You don’t need to access the Azure API to get the credentials for the cluster, as its kubeconfig config file is stored as a secret in the namespace where the cluster CRDs live.

    kubectl get secret {cluster-name}-kubeconfig -o yaml \
    -o jsonpath={.data.value} | base64 --decode > aks1.kubeconfig

    kubectl --kubeconfig=aks1.kubeconfig cluster-info

That’s it! 

### Adding clusters

![](https://cdn-images-1.medium.com/max/2000/0*1VXPMdndEPZHOLMp.jpg)

Let’s go crazy and add a second cluster:

    helm install capz2 charts/azure-managed-cluster/ --values aks2.yaml

And so on! Now for some magic: you can delete the KIND cluster and reapply exactly the same Helm charts/ClusterAPI CRDs and the controller will just reconcile the desired state (2 clusters) with the current situation in Azure (2 clusters) and happily report no change needed. So as you can see this is a great step towards complete automation of AKS infrastructure, from start to finish (you can just add a GitOps configuration repository to the clusters, even via Azure Arc and now you have a full end-to-end automation from clusters to applications). I will expand on the concept in a later article.

I hope you find ClusterAPI as exciting as I do, and I’m looking forward to contribute to the project.


