
# Setup Talos Kubernetes Cluster
## Setup client computer
```
brew install kubectl
echo 'source <(kubectl completion bash)' >>~/.bash_profile
brew install siderolabs/tap/talosctl
brew install argocd
brew install helm
```  

## Setup Talos VM template
### Download Talos nocloud VM image
On the Proxmox host:
```
cd /var/lib/vz/template/
wget https://factory.talos.dev/image/376567988ad370138ad8b2698212367b8edcb69b5fd68c80be1f2ec7d603b4ba/v1.12.6/nocloud-amd64.raw.xz
xz -d nocloud-amd64.raw.xz
```
Image was created by: https://factory.talos.dev/

 ### Create VM template
```
qm create 9000 --name talos-template --memory 2048 --cores 2 --net0 virtio,bridge=vmbr0
qm importdisk 9000 nocloud-amd64.raw local-lvm
qm set 9000 --args "-cpu host"
qm set 9000 --scsihw virtio-scsi-pci --scsi0 local-lvm:vm-9000-disk-0
qm set 9000 --boot order=scsi0
qm set 9000 --ide2 local-lvm:cloudinit
qm set 9000 --serial0 socket --vga serial0
qm set 9000 --agent enabled=0
qm template 9000
```

### Clone the template for control plane nodes
```
# Clone VMs
qm clone 9000 101 --name talos-cp-1 --full
qm clone 9000 102 --name talos-cp-2 --full
qm clone 9000 103 --name talos-cp-3 --full

# Resize the disks
qm resize 101 scsi0 10G
qm resize 102 scsi0 10G
qm resize 103 scsi0 10G

# Set resources for control plane nodes
qm set 101 --memory 2048 --cores 2
qm set 102 --memory 2048 --cores 2
qm set 103 --memory 2048 --cores 2

# Set reserved MAC addresses
qm set 101 -net0 virtio=BC:24:11:86:DD:3B,bridge=vmbr0
qm set 102 -net0 virtio=BC:24:11:88:BC:30,bridge=vmbr0
qm set 103 -net0 virtio=BC:24:11:26:4D:00,bridge=vmbr0

# Start the VMs
qm start 101
qm start 102
qm start 103
```

## Generate Talos configuration
```
talosctl gen config proxmox-cluster https://192.168.23.150:6443
vim vip-patch.yaml

machine:
  network:
    interfaces:
      - interface: eth0
        dhcp: true
        vip:
          ip: 192.168.23.150
````

```
talosctl machineconfig patch controlplane.yaml --patch @vip-patch.yaml -o controlplane-vip.yaml
```
  
  

## Configure Talos Control-Plane
```
talosctl apply-config --insecure --nodes 192.168.23.151 --file controlplane-vip.yaml
talosctl apply-config --insecure --nodes 192.168.23.152 --file controlplane-vip.yaml
talosctl apply-config --insecure --nodes 192.168.23.153 --file controlplane-vip.yaml
```
  

## Setup local Talos config on client computer
```
cp talosconfig ~/.talos/config
```
  

## Bootstrap the first control plane node
```
talosctl config endpoint 192.168.23.150
talosctl config node 192.168.23.151
talosctl bootstrap --nodes 192.168.23.151
```
  
## Get the kubeconfig
```
talosctl kubeconfig
```
## Check kubectl connectivity
```
kubectl get nodes -o wide
kubectl get pods -A
```

# Create worker node
```
# Clone worker nodes
qm clone 9000 201 --name talos-worker-1 --full
qm clone 9000 202 --name talos-worker-2 --full
qm clone 9000 203 --name talos-worker-3 --full

# Resize disks
qm resize 201 scsi0 50G
qm resize 202 scsi0 50G
qm resize 203 scsi0 50G

# Set resources for worker nodes
qm set 201 --memory 8192 --cores 4
qm set 202 --memory 8192 --cores 4
qm set 203 --memory 8192 --cores 4

# Set reserved MAC addresses
qm set 201 -net0 virtio=BC:24:11:F5:E6:25,bridge=vmbr0
qm set 202 -net0 virtio=BC:24:11:79:12:75,bridge=vmbr0
qm set 203 -net0 virtio=BC:24:11:E3:C7:F1,bridge=vmbr0

# Start worker VMs
qm start 201
qm start 202
qm start 203
```
# Apply worker configuration
```
talosctl apply-config --insecure --nodes 192.168.23.161 --file worker.yaml
talosctl apply-config --insecure --nodes 192.168.23.162 --file worker.yaml
talosctl apply-config --insecure --nodes 192.168.23.163 --file worker.yaml
```

# Install Cilium as the CNI
```
cilium install --helm-set ipam.mode=kubernetes

# For storage, use the Proxmox CSI driver or local-path-provisioner
# Local path provisioner is simpler for homelabs
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.26/deploy/local-path-storage.yaml
```

# Test Pod
```
kubectl run pingpong --image alpine ping 8.8.8.8
kubectl get pods -o wide
kubectl logs pod/pingpong -f
kubectl delete pod/pingpong
```

# Install certmanager
```
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.20.0/cert-manager.yaml
```

# Install MetalLB
```
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.5/config/manifests/metallb-native.yaml
```

# Install Traefik
```
helm repo add traefik https://helm.traefik.io/traefik
helm repo update traefik
helm show values traefik/traefik > values.yaml
vim custom-values.yaml

ingressRoute:
  dashboard:
    enabled: true

kubectl create namespace traefik
helm upgrade --install traefik traefik/traefik -f custom-values.yaml -n traefik

vim traefik-dashboard.yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: traefik-dashboard
  namespace: traefik
spec:
  entryPoints:
    - web
  routes:
    - match: PathPrefix(`/dashboard`) || PathPrefix(`/api`)
      kind: Rule
      services:
        - name: api@internal
          kind: TraefikService

kubectl apply -f traefik-dashboard.yaml
kubectl -n traefik get services
http://192.168.23.211/dashboard/

```

# Install ArgoCD
```
# Create the ArgoCD namespace
kubectl create namespace argocd

# Fetch Argo CD CRDs
kubectl apply --server-side --force-conflicts -k https://github.com/argoproj/argo-cd/manifests/crds\?ref\=stable

# Install ArgoCD using the official manifests
kubectl apply -n argocd --server-side --force-conflicts -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Get the auto-generated admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# Temporary port forward
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Log in to your ArgoCD instance
argocd login localhost:8080 --insecure

# Add local cluster
argocd cluster add admin@proxmox-cluster

kubectl -n argocd get svc
kubectl -n argocd edit deployment argocd-server
containers:
- name: argocd-server
  args:
    - /usr/local/bin/argocd-server
++    - --insecure
kubectl rollout restart deployment argocd-server -n argocd
```

# kubctl Cheat Sheet
## List Cluster Nodes
```kubectl get nodes -o wide```
## List Namespaces
```kubectl get namespace```
## List Pods in Namespace
```kubectl -n argocd get pods```
## List Services in Namespace
```kubectl -n argocd get services```
## Edit Deployment
```kubectl -n argocd edit deployment argocd-server```
## Restart Deployment
```kubectl rollout restart deployment argocd-server -n argocd```
