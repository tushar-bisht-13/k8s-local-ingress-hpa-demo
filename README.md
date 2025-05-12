# k8s-local-ingress-hpa-demo

## 1. KIND Cluster Setup Guide

### i. Installing KIND and kubectl
Install KIND and kubectl using the provided script:
```bash

#!/bin/bash

[ $(uname -m) = x86_64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.27.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

VERSION="v1.30.0"
URL="https://dl.k8s.io/release/${VERSION}/bin/linux/amd64/kubectl"
INSTALL_DIR="/usr/local/bin"

curl -LO "$URL"
chmod +x kubectl
sudo mv kubectl $INSTALL_DIR/
kubectl version --client

rm -f kubectl
rm -rf kind

echo "kind & kubectl installation complete."
```

### ii. Setting Up the KIND Cluster
Create a kind-cluster-config.yaml file:

```yaml

kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  image: kindest/node:v1.31.2
  kubeadmConfigPatches:                # give it the ingress‑ready label
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
- role: worker
  image: kindest/node:v1.31.2

```
Create the cluster using the configuration file:

```bash

kind create cluster --config kind-cluster-config.yaml --name my-kind-cluster
```

### iii. Verify the cluster
Use kubectl to verify the cluster:
```bash

kubectl get nodes
kubectl cluster-info
```

### iv. Deleting the Cluster
Delete the KIND cluster:
```bash

kind delete cluster --name my-kind-cluster
```
## 2. Kubernetes resource deployment

### i. Install Mertic Server:

#### Install the latest tagged release:
```bash

kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

#### Add kind‑friendly flags (skip certs / prefer InternalIP)

By default Metrics Server expects each node’s kubelet to have a public
certificate. In local clusters (kind, Minikube, k3d) that isn’t the case, so we patch two flags:

```bash

kubectl patch deployment metrics-server -n kube-system \
  --type=json \
  -p='[
    {"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"},
    {"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname"}
  ]'
```

#### Wait until the mertics-server pod is in Running state:
Installs the latest tagged release:

```bash

kubectl get pods -n kube-system -l "k8s-app=metrics-server"
```

Expected output:

NAME                              READY   STATUS    RESTARTS   AGE
metrics-server-6966c7877d-44qc9   1/1     Running   0          24m

#### Verify:
Installs the latest tagged release:

```bash

kubectl top nodes
kubectl top pod
```
### ii. Install NGINX Controller:

## 6. Notes

Multiple Clusters: KIND supports multiple clusters. Use unique --name for each cluster.
Custom Node Images: Specify Kubernetes versions by updating the image in the configuration file.
Ephemeral Clusters: KIND clusters are temporary and will be lost if Docker is restarted.

