# kubernetes-cluster-install-met-rancher
#Loadbalancer:
#Install HAProxy
	sudo apt update && sudo apt install -y haproxy

#Configure HAProxy
#Edit the config:
	sudo nano /etc/haproxy/haproxy.cfg

#Add:
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
	    server master-2 <Ip-Master-2>6443 check
	    server master-3 <Ip-Master-3>6443 check

#Restart and enable HAProxy:
	sudo systemctl restart haproxy
	sudo systemctl enable haproxy

#Verify:
nc <ip-loadBalancer> -v





Step 2: Prepare All Nodes (except Load Balancer)
Disable Swap
	sudo swapoff -a
	sudo sed -i '/swap/d' /etc/fstab
Configure Kernel Parameters
	cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
	net.bridge.bridge-nf-call-iptables  = 1
	net.ipv4.ip_forward                 = 1
	net.bridge.bridge-nf-call-ip6tables = 1
	EOF
	sudo sysctl --system
Load Required Kernel Modules
	sudo modprobe overlay
	sudo modprobe br_netfilter

	cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
	overlay
	br_netfilter
	EOF

Step 3: Add Kubernetes Repository
	sudo mkdir -p -m 755 /etc/apt/keyrings

	curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.33/deb/Release.key   | sudo gpg --dearmor -o 	/etc/apt/keyrings/kubernetes-apt-keyring.gpg

	echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] 	https://pkgs.k8s.io/core:/stable:/v1.33/deb/ /"   | sudo tee /etc/apt/sources.list.d/kubernetes.list

Step 4: Install Kubernetes Components
	sudo apt-get update
	sudo apt-get install -y kubelet kubeadm kubectl
	sudo apt-mark hold kubelet kubeadm kubectl

Step 5: Install & Configure Container Runtime (containerd)
	sudo apt-get install -y containerd
	sudo mkdir -p /etc/containerd

	containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
	sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

	sudo systemctl restart containerd
	sudo systemctl enable containerd
	Enable kubelet:
	sudo systemctl enable kubelet

Step 6: Initialize Kubernetes Control Plane
	Run this on one control plane node:
	sudo kubeadm init   --control-plane-endpoint "<IP-Master-1>:6443"   --upload-certs   --pod-network-	cidr=10.244.0.0/16
	Save the join command from the output.




Step 7: Configure kubectl on Primary Master
	mkdir -p $HOME/.kube
	sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
	sudo chown $(id -u):$(id -g) $HOME/.kube/config

Step 8: Join Additional Nodes
	Gebruik de output gegevens van stap 6
	Voorbeeld voor Masters:
		kubeadm join 192.168.2.10:6443 --token <token>   --discovery-token-ca-cert-hash 				sha256:<hash>   --control-plane --certificate-key <cert-key>
	voorbeeld voor workers:
		kubeadm join 192.168.2.10:6443 --token <token>   --discovery-token-ca-cert-hash 				sha256:<hash>	

Step 9: Deploy Network Plugin (Calico)
	Run op Primary-Master-Node:
	kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/calico.yaml
	Check status:
		kubectl get nodes
		If any node is NotReady, restart kubelet:
		sudo systemctl restart kubelet




Step 10: Verify ETCD Cluster
	Install etcd client
		sudo apt update && sudo apt install -y etcd-client
	Check members
		ETCDCTL_API=3 etcdctl member list   --endpoints=https://127.0.0.1:2379   		--			cacert=/etc/kubernetes/pki/etcd/ca.crt   --cert=/etc/kubernetes/pki/etcd/server.crt   				--key=/etc/kubernetes/pki/etcd/server.key

Rancher installeren:

nodig: 
	3 – Masters
	3 – Workers
	Kubectl
Step 1. Voeg de Rancher Helm Repository toe
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
helm repo update
Step 2. Maak de namespace voor Rancher aan
kubectl create namespace cattle-system
Step 3. Installeer rancher
helm install rancher rancher-latest/rancher \
  --namespace cattle-system \
  --set replicas=1 \
  --set ingress.enabled=false \
  --set bootstrapPassword=admin
Step 4: Check Rancher Service
kubectl get svc -n cattle-system
step 5:installeer cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/<VERSION>/cert-manager.crds.yaml
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install cert-manager jetstack/cert-manager \
--namespace cert-manager \
--create-namespace \
--set crds.enabled=trued


Step 6: Convert Rancher Service to NodePort
kubectl patch svc rancher -n cattle-system -p '{"spec": {"type": "NodePort"}}'
kubectl get svc -n cattle-system

Step 7: Rancher in de Browser
	ga in de browser naar https://<Primary-master-IP>:
	
