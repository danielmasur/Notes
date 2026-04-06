# Setup Talos Kubernetes Cluster
# TOC
TODO

# Setup client computer
## Debian
```
curl -sL https://talos.dev/install | sh
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"; mv kubectl /usr/local/bin/; chmod 755 /usr/local/bin/kubectl
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
curl -L --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-amd64.tar.gz
tar xzvf cilium-linux-amd64.tar.gz
sudo mv cilium /usr/local/bin/
```
## MAC
```
brew install kubectl
echo 'source <(kubectl completion bash)' >>~/.bash_profile
brew install siderolabs/tap/talosctl
brew install argocd
brew install helm
```  

# Setup Talos VM template
## Download Talos nocloud VM image
On the Proxmox host:
```
cd /var/lib/vz/template/
wget https://factory.talos.dev/image/613e1592b2da41ae5e265e8789429f22e121aab91cb4deb6bc3c0b6262961245/v1.12.6/nocloud-amd64.raw.xz
xz -d nocloud-amd64.raw.xz
```
Image was created by: https://factory.talos.dev/

## Create VM template
On the Proxmox host:
```
qm create 9000 --name talos-template --memory 2048 --cores 2 --net0 virtio,bridge=vmbr0
qm importdisk 9000 nocloud-amd64.raw ssd-1gb
qm set 9000 --args "-cpu host"
qm set 9000 --scsihw virtio-scsi-pci --scsi0 ssd-1gb:vm-9000-disk-0
qm set 9000 --boot order=scsi0
qm set 9000 --ide2 ssd-1gb:cloudinit
qm set 9000 --serial0 socket --vga std
qm set 9000 --agent enabled=0
qm template 9000
```

# Create VM's
## Clone the template for control-plane nodes
On the Proxmox host:
```
# Clone VMs
qm clone 9000 101 --name talos-cp-1 --full
qm clone 9000 102 --name talos-cp-2 --full
qm clone 9000 103 --name talos-cp-3 --full

# Resize the disks
qm resize 101 scsi0 50G
qm resize 102 scsi0 50G
qm resize 103 scsi0 50G

# Set resources for control-plane nodes
qm set 101 --memory 2048 --cores 2
qm set 102 --memory 2048 --cores 2
qm set 103 --memory 2048 --cores 2

# Set reserved MAC addresses
qm set 101 -net0 virtio=BC:24:11:86:DD:3B,bridge=vmbr0
qm set 102 -net0 virtio=BC:24:11:88:BC:30,bridge=vmbr0
qm set 103 -net0 virtio=BC:24:11:26:4D:00,bridge=vmbr0

# Start VMs

qm start 101
qm start 102
qm start 103
```

## Clone the template for worker nodes
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

# Start VMs
qm start 201
qm start 202
qm start 203
```

# Setup Talos CLuster
## Generate Talos Cluster Config
```
talosctl gen config talos-proxmox https://192.168.23.151:6443
export TALOSCONFIG=$(pwd)/talosconfig
cp talosconfig ~/.talos/config
```
## Patch machine config
vim cluster-patch.yaml
```
machine:
  kernel:
    modules:
      - name: nbd
      - name: iscsi_tcp
      - name: iscsi_generic
      - name: configfs
      - name: dm_crypt
  kubelet:
    extraMounts:
      # Longhorn data directory
      - destination: /var/lib/longhorn
        type: bind
        source: /var/lib/longhorn
        options:
          - bind
          - rshared
          - rw
  sysctls:
    # Recommended for Longhorn
    vm.max_map_count: "262144"
    net.ipv4.ip_forward: "1"
    net.ipv6.conf.all.forwarding: "1"
    net.ipv4.conf.all.rp_filter: "0"
    net.ipv4.conf.default.rp_filter: "0"

cluster:
  network:
    # Disable the default Flannel CNI
    cni:
      name: none
  proxy:
    # Cilium can replace kube-proxy entirely
    disabled: true
```
```
talosctl machineconfig patch controlplane.yaml --patch @cluster-patch.yaml
talosctl machineconfig patch worker.yaml --patch @cluster-patch.yaml
```
## Bootstrap Nodes
```
talosctl apply-config -i -n 192.168.23.151 --file controlplane.yaml
talosctl apply-config -i -n 192.168.23.152 --file controlplane.yaml
talosctl apply-config -i -n 192.168.23.153 --file controlplane.yaml
talosctl apply-config -i -n 192.168.23.161 --file worker.yaml
talosctl apply-config -i -n 192.168.23.162 --file worker.yaml
talosctl apply-config -i -n 192.168.23.163 --file worker.yaml
```
## Bootstrap Cluster
```
talosctl config endpoint 192.168.23.151
talosctl config node 192.168.23.151
talosctl bootstrap -n 192.168.23.151
```

## Copy kube config
```
talosctl kubeconfig
sleep 60
watch kubectl get nodes -o wide
```
## Check Cluster
```
kubectl get pods -A | grep -v Running
```
# Install Cilium
```
helm repo add cilium https://helm.cilium.io
helm repo update
helm install cilium cilium/cilium \
  --namespace kube-system \
  --set ipam.mode=kubernetes \
  --set kubeProxyReplacement=true \
  --set l2announcements.enabled=true \
  --set externalIPs.enabled=true \
  --set devices=eth0 \
  --set securityContext.capabilities.ciliumAgent="{CHOWN,KILL,NET_ADMIN,NET_RAW,IPC_LOCK,SYS_ADMIN,SYS_RESOURCE,DAC_OVERRIDE,FOWNER,SETGID,SETUID}" \
  --set securityContext.capabilities.cleanCiliumState="{NET_ADMIN,SYS_ADMIN,SYS_RESOURCE}" \
  --set cgroup.autoMount.enabled=false \
  --set cgroup.hostRoot=/sys/fs/cgroup \
  --set k8sServiceHost=localhost \
  --set k8sServicePort=7445
sleep 60
watch kubectl -n kube-system get pods -l k8s-app=cilium
kubectl apply -f - <<EOF
apiVersion: cilium.io/v2
kind: CiliumLoadBalancerIPPool
metadata:
  name: default-pool
spec:
  blocks:
    stop: 192.168.23.219
EOF
cilium status

cat <<EOF > l2-announcement.yaml
apiVersion: cilium.io/v2alpha1
kind: CiliumL2AnnouncementPolicy
metadata:
  name: default-l2-policy
spec:
  # Which nodes can announce LoadBalancer IPs
  nodeSelector:
    matchExpressions:
    - key: node-role.kubernetes.io/control-plane
      operator: DoesNotExist
  # Which interfaces to use for ARP announcements
  interfaces:
  - ^eth[0-9]+
  externalIPs: true
  loadBalancerIPs: true
EOF
kubectl apply -f l2-announcement.yaml
helm upgrade cilium cilium/cilium \
  --namespace kube-system \
  --reuse-values \
  --set devices=""

```
### Restarting cilium
```
kubectl -n kube-system rollout restart ds cilium
```
### Debug Cilium
```
# Check overall status
kubectl -n kube-system exec ds/cilium -- cilium status

# Check IP pool utilization
kubectl get ciliumloadbalancerippool -o wide

# Check which services have IPs assigned
kubectl get svc -A --field-selector spec.type=LoadBalancer

# Check L2 announcement status
kubectl get ciliuml2announcementpolicy

# View Cilium's service state
kubectl exec -n kube-system ds/cilium -- cilium service list

# Check which node is handling a specific service IP
kubectl exec -n kube-system ds/cilium -- cilium bpf lb list

# Check if the IP pool has available addresses
kubectl describe ciliumloadbalancerippool default-pool

# Check Cilium operator logs
kubectl logs -n kube-system -l app.kubernetes.io/name=cilium-operator --tail=50

# Verify L2 announcement policy exists
kubectl get ciliuml2announcementpolicy

# Check Cilium agent logs for LB-related messages
kubectl logs -n kube-system -l k8s-app=cilium --tail=50 | grep -i "lb\|loadbalancer\|l2"

# Delete l2 annoucement policy
kubectl delete ciliuml2announcementpolicy default-l2-policy

# Monitor traffic / events in real-time
kubectl -n kube-system exec ds/cilium -- cilium monitor
```
# Enable Hubble in your Cilium deployment
```
helm upgrade cilium cilium/cilium \
  -n kube-system \
  --reuse-values \
  --set hubble.enabled=true \
  --set hubble.ui.enabled=true \
  --set hubble.relay.enabled=true
kubectl patch svc hubble-ui -n kube-system \
  --type merge \
  -p '{"spec":{"type":"LoadBalancer"}}'

cat <<EOF > hubble-traefik.yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: hubble-ui
  namespace: kube-system
spec:
  entryPoints:
    - web
    - websecure
  routes:
    - kind: Rule
      match: PathPrefix(`/hubble`)
      services:
        - name: hubble-ui
          kind: Service
          port: 80
EOF
kubectl apply -f hubble-traefik.yaml
```
# Install Longhorn

cat <<EOF > longhorn-values.yaml
```
# longhorn-values.yaml
defaultSettings:
  # Default data path on Talos nodes
  defaultDataPath: /var/lib/longhorn
  # Default replica count
  defaultReplicaCount: 3
  # Storage over-provisioning percentage
  storageOverProvisioningPercentage: 100
  # Storage minimal available percentage
  storageMinimalAvailablePercentage: 15
  # Default data locality
  defaultDataLocality: best-effort
  # Create default disk on nodes
  createDefaultDiskLabeledNodes: true
  # Node drain policy
  nodeDrainPolicy: block-for-eviction
  # Guaranteed instance manager CPU
  guaranteedInstanceManagerCPU: 12

persistence:
  # Default storage class
  defaultClass: true
  defaultClassReplicaCount: 3
  defaultFsType: ext4

# Resource limits
longhornManager:
  resources:
    requests:
      cpu: 250m
      memory: 256Mi

longhornDriver:
  resources:
    requests:
      cpu: 100m
      memory: 128Mi

# Enable the UI
longhornUI:
  replicas: 2
EOF
```

```
helm repo add longhorn https://charts.longhorn.io
helm repo update
helm install longhorn longhorn/longhorn \
  --namespace longhorn-system \
  --create-namespace \
  --values longhorn-values.yaml
kubectl label namespace longhorn-system \
  pod-security.kubernetes.io/enforce=privileged \
  pod-security.kubernetes.io/audit=privileged \
  pod-security.kubernetes.io/warn=privileged \
  --overwrite
watch kubectl get pods -n longhorn-system
# Set default storage
kubectl patch storageclass longhorn -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

# Create "disks"
kubectl -n longhorn-system patch nodes.longhorn.io talos-worker-1   --type merge -p '{
    "spec": {
      "disks": {
        "longhorn-disk-1": {
          "path": "/var/lib/longhorn",
          "allowScheduling": true,
          "allowVolumeCreation": true
        }
      }
    }
  }'
```
# Install certmanager
```
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true
kubectl get pods -n cert-manager
```
## Test cert-manager
```
cat <<EOF > test-resources.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: cert-manager-test
---
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: test-selfsigned
  namespace: cert-manager-test
spec:
  selfSigned: {}
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: selfsigned-cert
  namespace: cert-manager-test
spec:
  dnsNames:
    - example.com
  secretName: selfsigned-cert-tls
  issuerRef:
    name: test-selfsigned
EOF
kubectl apply -f test-resources.yaml
kubectl describe certificate -n cert-manager-test
kubectl delete -f test-resources.yaml
```
## Configure cert-manager
TODO
## Debug cert-manager
```
kubectl get pods -n cert-manager
kubectl logs -n cert-manager deploy/cert-manager
kubectl logs -n cert-manager deploy/cert-manager-webhook
kubectl logs -n cert-manager deploy/cert-manager-cainjector
kubectl get issuers,clusterissuers --all-namespaces
# Check cert
kubectl get certificates --all-namespaces
kubectl describe certificate <name> -n <namespace>

cilium service list
```
# Install Traefik
```
cat <<EOF > traefik-values.yaml
deployment:
  replicas: 2

service:
  type: LoadBalancer

additionalArguments:
  - "--entrypoints.web.http.redirections.entrypoint.to=websecure"
  - "--entrypoints.web.http.redirections.entrypoint.scheme=https"

ports:
  web:
    port: 80
    expose:
      default: true

  websecure:
    port: 443
    expose:
      default: true

ingressRoute:
  dashboard:
    enabled: true

providers:
  kubernetesCRD:
    enabled: true
  kubernetesIngress:
    enabled: true
EOF
helm repo add traefik https://traefik.github.io/charts
helm repo update
helm install traefik traefik/traefik \
  -n traefik \
  --create-namespace \
  -f traefik-values.yaml
kubectl -n traefik get pods
kubectl -n traefik get svc
kubectl patch svc traefik -n traefik \
  --type=json \
  -p='[
    {
      "op": "replace",
      "path": "/spec/selector",
      "value": {
        "app.kubernetes.io/instance": "traefik-traefik",
        "app.kubernetes.io/name": "traefik"
      }
    }
  ]'
helm upgrade traefik traefik/traefik \
  -n traefik \
  --reuse-values \
  --set "ingressRoute.dashboard.entryPoints={web,websecure}"
```
# Install Forgejo
```
cat <<EOF > forgejo-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: forgejo-data
  namespace: forgejo
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: longhorn
EOF
kubectl create namespace forgejo
kubectl apply -f forgejo-pvc.yaml

cat <<EOF > forgejo-values.yaml
service:
  type: LoadBalancer

persistence:
  enabled: true
  size: 20Gi

postgresql:
  enabled: true

redis:
  enabled: true
  
ssh:
  enabled: true
EOF

helm install forgejo -f forgejo-values.yaml oci://code.forgejo.org/forgejo-helm/forgejo \
  --namespace forgejo \
  --create-namespace
  ```
## Ingres
```
cat <<EOF > forgejo-ssh.yaml
apiVersion: v1
kind: Service
metadata:
  name: forgejo-ssh
  namespace: forgejo
spec:
  type: LoadBalancer
  selector:
    app.kubernetes.io/name: forgejo
  ports:
    - name: ssh
      protocol: TCP
      port: 22
      targetPort: 22
EOF
cat <<EOF > forgejo-http.yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: forgejo
  namespace: forgejo
spec:
  entryPoints:
    - web
  routes:
    - match: PathPrefix(`/forgejo`)
      kind: Rule
      services:
        - name: forgejo-http
          port: 3000
EOF
kubectl apply -f forgejo-http.yaml
kubectl apply -f forgejo-ssh.yaml
kubectl patch svc forgejo-http -n forgejo \
  --type merge \
  -p '{"spec":{"type":"LoadBalancer"}}'

# Set gitea_admin password
kubectl get pods -n forgejo
kubectl exec -it -n forgejo forgejo-5d767997fb-2sl4v -- /bin/sh
forgejo admin user change-password -u gitea_admin -p password
```
## Sample repo for argocd

create hello-world repo and clone
```
mkdir -p hello-world/k8s
cat <<EOF > hello-world/k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-world
  namespace: hello-world
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello-world
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      containers:
      - name: web
        image: hashicorp/http-echo
        args:
          - "-text=Hello from Argo CD!"
        ports:
        - containerPort: 5678
EOF
cat <<EOF > hello-world/k8s/ingres.yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: hello-world
  namespace: hello-world
spec:
  entryPoints:
    - web
  routes:
    - match: PathPrefix(`/hello`)
      kind: Rule
      services:
        - name: hello-world
          port: 80
EOF
cat <<EOF > hello-world/k8s/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: hello-world
  namespace: hello-world
spec:
  selector:
    app: hello-world
  ports:
    - protocol: TCP
      port: 80
      targetPort: 5678
  type: ClusterIP
EOF
```
# Install argocd
```
kubectl create namespace argocd
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```
## Ingres
```
cat <<EOF > argocd-traefik-ingress.yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: argocd
  namespace: argocd
spec:
  entryPoints:
    - web
  routes:
    - match: PathPrefix('/argocd')
      kind: Rule
      services:
        - name: argocd-server
          port: 80
EOF
kubectl apply -f argocd-traefik-ingress.yaml
kubectl patch svc argocd-server-lb -n argocd \
  --type merge \
  -p '{"spec":{"type":"LoadBalancer"}}'
```

## Add secret Key
```
kubectl -n argocd create secret generic argocd-secret --from-literal=server.secretkey=$(openssl rand -hex 32)
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 --decode # eY05LkYBgubzammK
```

## Hello World!
```
cat <<EOF > argocd-hello-world.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: hello-world
  namespace: argocd
spec:
  project: default
  source:
    repoURL: 'http://192.168.23.213:3000/daniel/test.git'
    targetRevision: main
    path: hello-world/k8s
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: hello-world
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
EOF
kubectl apply -f argocd-hello-world.yaml
```
# Somethingsomething debug
## Clean board
```
qm stop 101
qm stop 102
qm stop 103
qm stop 201
qm stop 202
qm stop 203
qm destroy 101
qm destroy 102
qm destroy 103
qm destroy 201
qm destroy 202
qm destroy 203
```
## Logs
```
kubectl logs -n traefik -l app.kubernetes.io/name=traefik
```
## Network
### List services
```
kubectl get svc -n -A -o wide
```
### Somethingsomething
```
kubectl get endpointslice -n traefik
kubectl get endpoints -n traefik traefik
```
## Storage
### Persistent Volumes
#### List persistent volumes
```
kubectl get pv
```
#### Delete a persistent volume
```
kubectl delete pv <pv_name> --grace-period=0 --force
```
