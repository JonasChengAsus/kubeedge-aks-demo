# AKS Cluster Provisioning

## Create Service Principal

To manually create a service principal, the `--skip-assignment` parameter prevents any additional default assignments being assigned

```bash
az ad sp create-for-rbac --skip-assignment --name myAKSClusterServicePrincipal
```

The output is similar to the following example. Make a note of your own appId and password. These values are used when you create an AKS cluster in the next section.

```json
{
  "appId": "559513bd-0c19-4c1a-87cd-851a26afd5fc",
  "displayName": "myAKSClusterServicePrincipal",
  "name": "http://myAKSClusterServicePrincipal",
  "password": "e763725a-5eee-40e8-a466-dc88d980f415",
  "tenant": "72f988bf-86f1-41af-91ab-2d7cd011db48"
}
```

## Create AKS Cluster

```bash
export RESOURCE_GROUP=MY_RESOURCE_GROUP
export MANAGED_CLUSTER=MY_MANAGED_CLUSTER
export SP_APP_ID=559513bd-0c19-4c1a-87cd-851a26afd5fc
export SP_PASSWORD=e763725a-5eee-40e8-a466-dc88d980f415
# ACR Resource ID can get via command:
# az acr show -n MY_ACR_REGISTRY
export ACR_RESOURCE_ID=MY_ACR_RESOURCE_ID

az aks create -g $RESOURCE_GROUP -n $MANAGED_CLUSTER --service-principal $SP_APP_ID --client-secret $SP_PASSWORD --node-count 1 --kubernetes-version 1.16.7 --ssh-key-value /path/to/publickey 
```

## Get Credentials to Access the Cluster

```bash
az aks get-credentials -g $RESOURCE_GROUP -n $MANAGED_CLUSTER
```

## Test AKS Cluster is Accessible

```bash
kubectl cluster-info
# there should be at least one master and one agentpool node in Ready status
kubectl get nodes
```

## (Optional) Access Dashboard

```bash
# Terminal 1, get token to log in dashboard within the same terminal context
# 1. set context to kube-system
kubectl config set-context --current --namespace=kube-system
# 2. recreate clusterrolebinding
# https://github.com/Azure/AKS/issues/601
kubectl delete clusterrolebinding kubernetes-dashboard
kubectl create clusterrolebinding kubernetes-dashboard --clusterrole=cluster-admin --serviceaccount=kube-system:kubernetes-dashboard
# 3. get token
kubectl get secret $(kubectl get sa kubernetes-dashboard -o jsonpath="{.secrets[0].name}") -o jsonpath="{.data.token}" | base64 --decode
```

```bash
# Terminal 2, open a new terminal to creates a proxy between your system and the K8S API, and opens a web browser to the K8S dashboard. 
# If a web browser doesn't open to the Kubernetes dashboard, copy and paste the URL address noted in the Azure CLI, typically http://127.0.0.1:8001.
export RESOURCE_GROUP=MY_RESOURCE_GROUP
export MANAGED_CLUSTER=MY_MANAGED_CLUSTER

az aks browse -g $RESOURCE_GROUP -n $MANAGED_CLUSTER
```

# Create KubeEdge CloudCore Service in AKS Cluster

## Pull Script from Github

```bash
git clone https://github.com/kubeedge/kubeedge.git
```

## Build CloudCore Image

Ensure your k8s cluster can pull edge controller image. If the
image not exist. We can make one, and push to your registry.

```bash
cd kubeedge
make cloudimage
```

If you use existing ACR, please allows you to authorize an existing ACR in your subscription and configures the appropriate ACRPull role for the service principal. 

```bash
az aks update -g $RESOURCE_GROUP -n $MANAGED_CLUSTER --attach-acr $ACR_RESOURCE_ID
```

## Modify Deployment Manifest

Replace container image name `kubeedge/cloudcore:v1.2.1` in `kubeedge/build/cloud/07-deployment.yaml` according to the image you built in previous step

## Generate TLS Certs

Then, we need to generate the tls certs. It then will give us
`06-secret.yaml` if succeeded.

```bash
cd kubeedge/build/cloud
sudo ../tools/certgen.sh buildSecret | tee ./06-secret.yaml
```

## Create K8S Resources

Then, we create k8s resources from the manifests in name order. Before creating, check the content of each manifest to make sure it meets your
environment.

```bash
for resource in $(ls *.yaml); do kubectl create -f $resource; done
# create CRD for devices.devices.kubeedge.io
kubectl create -f https://raw.githubusercontent.com/kubeedge/kubeedge/master/build/crds/devices/devices_v1alpha1_device.yaml
# create CRD for devicemodels.devices.kubeedge.io
kubectl create -f https://raw.githubusercontent.com/kubeedge/kubeedge/master/build/crds/devices/devices_v1alpha1_devicemodel.yaml
# create CRD for clusterobjectsyncs.reliablesyncs.kubeedge.io
kubectl create -f https://raw.githubusercontent.com/kubeedge/kubeedge/master/build/crds/reliablesyncs/cluster_objectsync_v1alpha1.yaml
# create CRD for objectsyncs.reliablesyncs.kubeedge.io
kubectl create -f https://raw.githubusercontent.com/kubeedge/kubeedge/master/build/crds/reliablesyncs/objectsync_v1alpha1.yaml
```

Last, expose cloud hub to outside of K8S cluster, so that edge core can connect to.

```bash
kubectl create -f 08-service.yaml
```

# Destroy AKS Cluster

```bash
az aks delete -g $RESOURCE_GROUP -n $MANAGED_CLUSTER
```