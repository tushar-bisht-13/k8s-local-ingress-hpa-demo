# k8s-local-ingress-hpa-demo

# KIND Cluster Setup Guide

## 1. Installing KIND and kubectl
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

## 2. Setting Up the KIND Cluster
Create a kind-cluster-config.yaml file:

```yaml

kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  image: kindest/node:v1.31.2
  kubeadmConfigPatches:                # give it the ingress‑ready label, too
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
Verify the cluster:

```bash

kubectl get nodes
kubectl cluster-info
```
## 3. Accessing the Cluster
Use kubectl to interact with the cluster:
```bash

kubectl cluster-info
```

## 4. Deleting the Cluster
Delete the KIND cluster:
```bash

kind delete cluster --name my-kind-cluster
```

## 6. Notes

Multiple Clusters: KIND supports multiple clusters. Use unique --name for each cluster.
Custom Node Images: Specify Kubernetes versions by updating the image in the configuration file.
Ephemeral Clusters: KIND clusters are temporary and will be lost if Docker is restarted.

