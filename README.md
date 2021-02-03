# Install k8s cluster with Ubuntu (OS), kubeadm, containerd (CRI), and calico (CNI)
Install latest k8s cluster (v1.20) with Ubuntu OS (20.04.1 LTS (Focal Fossa)) using kubeadm with containerd as CRI and calico as CNI plugin

## Prerequisites
> https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#before-you-begin

## Steps
## 1. Ensure that the `br_netfilter` module is loaded on the controlplane and worker node
> https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#letting-iptables-see-bridged-traffic
```
# Ensure that the module is loaded
lsmod | grep br_netfilter

# If the module is not loaded yet, follow the below steps
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

sudo sysctl --system
```

## 2. Install container runtime (containerd) on the controlplane and worker node
> https://kubernetes.io/docs/setup/production-environment/container-runtimes/#containerd
```
# -------------------------- prerequisites -------------------------------
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Setup required sysctl params, these persist across reboots.
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system

# -------------------------- Install and configure containerd -------------------------------
# Install containerd
sudo apt-get update && sudo apt-get install -y containerd

# Configure containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

# Restart containerd
sudo systemctl restart containerd
```

## 3. Install kubeadm, kubelet, and kubectl on the controlplane and worker node
> https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl
```
sudo apt-get update && sudo apt-get install -y apt-transport-https curl

curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
sudo apt-get update

sudo apt-get install -y kubelet kubeadm kubectl

sudo apt-mark hold kubelet kubeadm kubectl
```

## 4. Install controlplane components on the controlplane node using kubeadm
> https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#initializing-your-control-plane-node
```
sudo kubeadm init

# After installation has finished, setup kubeconfig file
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# IMPORTANT (copy the generated kubeadm join command)
#kubeadm join <control-plane-host>:<control-plane-port> --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

## 5. Join the worker node to the cluster
```
# Paste the generated kubeadm join command from the controlplane node
sudo kubeadm join <control-plane-host>:<control-plane-port> --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

## 6. Enable pod networking by installing CNI plugin (calico)
> https://v1-17.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#pod-network
```
controlplane$ kubectl apply -f https://docs.projectcalico.org/v3.11/manifests/calico.yaml
```

## 7. Ensure that the controlplane and worker node are ready
```
# Ensure that the controlplane and worker node status are "Ready"
kubectl get node
```

## Reference
[1] https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

[2] https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/
