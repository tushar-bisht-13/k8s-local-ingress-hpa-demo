# k8s-local-ingress-hpa-demo

## To Get Started

- Clone the repository:

 ```bash
 git clone https://github.com/your-username/k8s-local-ingress-hpa-demo.git
 cd k8s-local-ingress-hpa-demo
 ```

## KIND Cluster Setup Guide

Prerequisite: Ensure Docker is installed on your system.

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

kind create cluster --config kind-config/kind-cluster-config.yaml --name my-kind-cluster
```

### iii. Verify the cluster
Use kubectl to verify the cluster:
```bash

kubectl get nodes
kind get clusters
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

```bash

kubectl get pods -n kube-system -l "k8s-app=metrics-server"
```

Expected output:
```bash
NAME                              READY   STATUS    RESTARTS   AGE

metrics-server-6966c7877d-44qc9   1/1     Running   0          24m
```

#### Verify:
Installs the latest tagged release:

```bash

kubectl top nodes
kubectl top pod
```
### ii. Install NGINX Controller:

#### Apply nginx controller from github repo:
```bash

kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.9.4/deploy/static/provider/kind/deploy.yaml
```

#### Verify if nginx pods are running:
```bash

kubectl get pods -n ingress-nginx --watch
```
Expected Output:
```bash

NAME                                        READY   STATUS      RESTARTS   AGE
ingress-nginx-admission-create-2phmn        0/1     Completed   0          54m
ingress-nginx-admission-patch-zm9s2         0/1     Completed   0          54m
ingress-nginx-controller-68c9d85487-2pszr   1/1     Running     0          54m
```
### iii. Apply Kubernetes manifest files:

#### The k8s-manifests/ folder contains:
```bash
📂 k8s-manifests
├─ page1-deployment.yaml      # nginx:alpine serving page 1
├─ page2-deployment.yaml      # nginx:alpine serving page 2
├─ page1-service.yaml         # ClusterIP svc
├─ page2-service.yaml         # ClusterIP svc
├─ page1-hpa.yaml             # Auto‑scale to 1–5 replicas @ 50 % CPU
├─ page2-hpa.yaml             # Auto‑scale to 1–5 replicas @ 50 % CPU
└─ ingress.yaml               # Path‑based routing: /page1 → svc‑1, /page2 → svc‑2
```

#### Apply Manifests:
```bash
kubectl apply -f k8s-manifests/
```

#### Verify all resources are present and in Running STATUS:
```bash
kubectl get all && kubectl get ingress
```

#### Verify Ingress Routing:
```bash
curl http://localhost/page1   # Expect: "This to Page 1"
curl http://localhost/page2   # Expect: "This to Page 2"
```

### iv. Test HPA:

#### Create a busybox container:
```bash
kubectl run loadpoke --image=busybox --restart=Never -i --tty -- sh
```

#### Execute below script to add load to page1 pod:
```bash
while true; do wget -q -O- http://page1-service.default.svc.cluster.local >/dev/null; done
```

#### Moniter HPA to see cpu and memory utilization in real time and HPA scaling up and Down based on the set utilization threshold:
```bash
watch -n4 kubectl describe hpa page1-hpa
```

### v. Delete the applied k8s resources and Kind Cluster:

#### Delete K8s resources:
```bash
kubectl delete -f k8s-manifests/
```

#### Delete Kind Cluster:
```bash
kind delete cluster --name my-kind-cluster
```

## Notes

Multiple Clusters: KIND supports multiple clusters. Use unique --name for each cluster.
Custom Node Images: Specify Kubernetes versions by updating the image in the configuration file.
Ephemeral Clusters: KIND clusters are temporary and will be lost if Docker is restarted.



