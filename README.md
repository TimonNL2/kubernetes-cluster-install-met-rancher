# Kubernetes Cluster Installatie met Rancher

Deze guide beschrijft de complete installatie van een Kubernetes cluster met HAProxy load balancer en Rancher management.

## Infrastructuur Vereisten

- **Load Balancer**: 1 node met HAProxy
- **Control Plane (Masters)**: 3 nodes
- **Workers**: 3 nodes
- **kubectl**: Geïnstalleerd op management machine

## Inhoudsopgave

1. [HAProxy Load Balancer Setup](#stap-1-haproxy-load-balancer-setup)
2. [Alle Nodes Voorbereiden](#stap-2-alle-nodes-voorbereiden-behalve-load-balancer)
3. [Kubernetes Repository Toevoegen](#stap-3-kubernetes-repository-toevoegen)
4. [Kubernetes Components Installeren](#stap-4-kubernetes-components-installeren)
5. [Container Runtime Installeren](#stap-5-container-runtime-containerd-installeren--configureren)
6. [Control Plane Initialiseren](#stap-6-kubernetes-control-plane-initialiseren)
7. [kubectl Configureren](#stap-7-kubectl-configureren-op-primary-master)
8. [Additional Nodes Toevoegen](#stap-8-additional-nodes-toevoegen)
9. [Network Plugin Deployen](#stap-9-network-plugin-calico-deployen)
10. [ETCD Cluster Verificeren](#stap-10-etcd-cluster-verificeren)
11. [Rancher Installeren](#rancher-installeren)

---

## Stap 1: HAProxy Load Balancer Setup

### HAProxy Installeren

```bash
sudo apt update && sudo apt install -y haproxy
```

### HAProxy Configureren

Edit de config:

```bash
sudo nano /etc/haproxy/haproxy.cfg
```

Voeg toe aan het einde van het bestand:

```haproxy
frontend kubernetes-frontend
    bind <ip-loadBalancer>:6443
    mode tcp
    option tcplog
    default_backend kubernetes-backend

backend kubernetes-backend
    mode tcp
    balance roundrobin
    option tcp-check
    server master-1 <Ip-Master-1>:6443 check
    server master-2 <Ip-Master-2>:6443 check
    server master-3 <Ip-Master-3>:6443 check
```

**Let op**: Vervang `<ip-loadBalancer>`, `<Ip-Master-1>`, `<Ip-Master-2>`, en `<Ip-Master-3>` met de daadwerkelijke IP-adressen.

### HAProxy Herstarten en Enablen

```bash
sudo systemctl restart haproxy
sudo systemctl enable haproxy
```

### Verificatie

Test de connectiviteit naar de load balancer:

```bash
nc <ip-loadBalancer> 6443 -v
```

---

## Stap 2: Alle Nodes Voorbereiden (behalve Load Balancer)

**Voer deze stappen uit op alle master en worker nodes.**

### Swap Uitschakelen

```bash
sudo swapoff -a
sudo sed -i '/swap/d' /etc/fstab
```

### Kernel Parameters Configureren

```bash
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sudo sysctl --system
```

### Benodigde Kernel Modules Laden

```bash
sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF
```

---

## Stap 3: Kubernetes Repository Toevoegen

```bash
sudo mkdir -p -m 755 /etc/apt/keyrings

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.33/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.33/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

---

## Stap 4: Kubernetes Components Installeren

```bash
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

---

## Stap 5: Container Runtime (containerd) Installeren & Configureren

### Containerd Installeren

```bash
sudo apt-get install -y containerd
sudo mkdir -p /etc/containerd
```

### Containerd Configureren

```bash
containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
```

### Containerd Herstarten en Enablen

```bash
sudo systemctl restart containerd
sudo systemctl enable containerd
```

### kubelet Enablen

```bash
sudo systemctl enable kubelet
```

---

## Stap 6: Kubernetes Control Plane Initialiseren

**Voer dit uit op één control plane node (primary master):**

```bash
sudo kubeadm init \
  --control-plane-endpoint "<IP-Master-1>:6443" \
  --upload-certs \
  --pod-network-cidr=10.244.0.0/16
```

**Let op**: Vervang `<IP-Master-1>` met het daadwerkelijke IP-adres van de primary master node.

**Belangrijk**: Bewaar de join commands uit de output! Je hebt deze nodig voor stap 8.

---

## Stap 7: kubectl Configureren op Primary Master

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

---

## Stap 8: Additional Nodes Toevoegen

Gebruik de output gegevens van stap 6.

### Masters Toevoegen

```bash
kubeadm join <IP-Master-1>:6443 --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --control-plane --certificate-key <cert-key>
```

### Workers Toevoegen

```bash
kubeadm join <IP-Master-1>:6443 --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

**Let op**: Vervang `<IP-Master-1>`, `<token>`, `<hash>`, en `<cert-key>` met de daadwerkelijke waarden uit de kubeadm init output.

---

## Stap 9: Network Plugin (Calico) Deployen

**Voer uit op de Primary Master Node:**

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/calico.yaml
```

### Status Controleren

```bash
kubectl get nodes
```

Als een node `NotReady` is, herstart kubelet:

```bash
sudo systemctl restart kubelet
```

---

## Stap 10: ETCD Cluster Verificeren

### ETCD Client Installeren

```bash
sudo apt update && sudo apt install -y etcd-client
```

### Members Controleren

```bash
ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

---

## Rancher Installeren

### Vereisten

- 3 Masters (control plane nodes)
- 3 Workers
- kubectl geconfigureerd

### Stap 1: Rancher Helm Repository Toevoegen

```bash
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
helm repo update
```

### Stap 2: Namespace voor Rancher Aanmaken

```bash
kubectl create namespace cattle-system
```

### Stap 3: Rancher Installeren

```bash
helm install rancher rancher-latest/rancher \
  --namespace cattle-system \
  --set replicas=1 \
  --set ingress.enabled=false \
  --set bootstrapPassword=admin
```

### Stap 4: Rancher Service Controleren

```bash
kubectl get svc -n cattle-system
```

### Stap 5: cert-manager Installeren

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/<VERSION>/cert-manager.crds.yaml

helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set crds.enabled=true
```

**Let op**: Vervang `<VERSION>` met de gewenste cert-manager versie (bijv. v1.13.0).

### Stap 6: Rancher Service naar NodePort Converteren

```bash
kubectl patch svc rancher -n cattle-system -p '{"spec": {"type": "NodePort"}}'
kubectl get svc -n cattle-system
```

### Stap 7: Rancher in de Browser Openen

Ga in de browser naar:

```
https://<Primary-master-IP>:<NodePort>
```

Gebruik de NodePort die je ziet in de output van `kubectl get svc -n cattle-system`.

**Default inloggegevens**:
- Username: `admin`
- Password: `admin` (zoals ingesteld bij installatie)

---

## Troubleshooting

### Node is NotReady

```bash
sudo systemctl restart kubelet
kubectl get nodes
```

### HAProxy werkt niet

```bash
sudo systemctl status haproxy
sudo journalctl -u haproxy -f
```

### Rancher pods starten niet

```bash
kubectl get pods -n cattle-system
kubectl describe pod <pod-name> -n cattle-system
kubectl logs <pod-name> -n cattle-system
```

### ETCD cluster problemen

```bash
kubectl get pods -n kube-system | grep etcd
kubectl logs -n kube-system etcd-<master-name>
```

---

## Nuttige Commands

### Cluster Status

```bash
kubectl get nodes -o wide
kubectl get pods -A
kubectl cluster-info
```

### Token Opnieuw Genereren

Als je join tokens verlopen zijn:

```bash
kubeadm token create --print-join-command
```

Voor control plane nodes:

```bash
kubeadm init phase upload-certs --upload-certs
```

### Kubernetes Configuratie Resetten (VOORZICHTIG!)

```bash
sudo kubeadm reset
sudo rm -rf /etc/cni/net.d
sudo rm -rf $HOME/.kube/config
```

---

## Licentie

Dit is een community guide voor educatieve doeleinden.