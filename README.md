# Install HA stacked k8s cluster with Ubuntu (OS), kubeadm, containerd (CRI), and calico (CNI)
Install latest k8s cluster (v1.20) with Ubuntu OS (20.04.1 LTS (Focal Fossa)) using kubeadm with containerd as CRI and calico as CNI plugin in HA setup (2 controlplane and worker)

## Prerequisites
> https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/#before-you-begin

## Steps
## 1. Create L4 load balancer for the controlplane nodes (kube-apiserver) 
- LB listening on TCP port 6443 and redirect traffic to the controlplane nodes kube-apiserver (TCP port 6443)
> https://github.com/raykrishardi/kubernetes-the-hard-way/blob/master/docs/08-bootstrapping-kubernetes-controllers.md#the-kubernetes-frontend-load-balancer
```
# Install HAProxy
sudo apt-get update && sudo apt-get install -y haproxy

# Change the load balancer IP address (10.0.1.235:6443) and target groups FQDN and IP address (master-1 192.168.5.11:6443, master-2 192.168.5.12:6443)
cat <<EOF | sudo tee /etc/haproxy/haproxy.cfg 
frontend kubernetes
    bind 10.0.1.235:6443
    option tcplog
    mode tcp
    default_backend kubernetes-master-nodes

backend kubernetes-master-nodes
    mode tcp
    balance roundrobin
    option tcp-check
    server master-1 192.168.5.11:6443 check fall 3 rise 2
    server master-2 192.168.5.12:6443 check fall 3 rise 2
EOF

sudo service haproxy restart
```

## 2. Ensure that the `br_netfilter` module is loaded on the controlplane and worker nodes
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

## 3. Install container runtime (containerd) on the controlplane and worker nodes
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
{
  # Install containerd
  sudo apt-get update && sudo apt-get install -y containerd

  # Configure containerd
  sudo mkdir -p /etc/containerd
  containerd config default | sudo tee /etc/containerd/config.toml

  # Restart containerd
  sudo systemctl restart containerd
}
```

## 4. Install kubeadm, kubelet, and kubectl on the controlplane and worker nodes
> https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl
```

sudo apt-get update && sudo apt-get install -y apt-transport-https curl

curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF

{
  sudo apt-get update
  sudo apt-get install -y kubelet kubeadm kubectl
  sudo apt-mark hold kubelet kubeadm kubectl
}
```

## 5. Install controlplane components on the MAIN controlplane node using kubeadm
> https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/#stacked-control-plane-and-etcd-nodes
```
LOAD_BALANCER_DNS=10.0.1.235
LOAD_BALANCER_PORT=6443
sudo kubeadm init --control-plane-endpoint "${LOAD_BALANCER_DNS}:${LOAD_BALANCER_PORT}" --upload-certs


# After installation has finished, setup kubeconfig file
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# IMPORTANT (copy the generated kubeadm join command for controlplane and worker nodes)
#You can now join any number of the control-plane node running the following command on each as root:

#  kubeadm join 10.0.1.235:6443 --token 8hctpv.j9i2fof2qrsv2ncw \
    --discovery-token-ca-cert-hash sha256:98e192fc50e45a19b47d2d3252bec7becc2b4731683ec20ae755a04a77a9c8da \
    --control-plane --certificate-key 093d67c4b78c352daaa29b5a1266eb9a0eb357d285c332cd20bd1da3120ed023

# Then you can join any number of worker nodes by running the following on each as root:

# kubeadm join 10.0.1.235:6443 --token 8hctpv.j9i2fof2qrsv2ncw \
    --discovery-token-ca-cert-hash sha256:98e192fc50e45a19b47d2d3252bec7becc2b4731683ec20ae755a04a77a9c8da

```

## 6.1. Join the other controlplane nodes to the cluster
```
# Paste the generated kubeadm join command from the main controlplane node
sudo kubeadm join 10.0.1.235:6443 --token 8hctpv.j9i2fof2qrsv2ncw \
    --discovery-token-ca-cert-hash sha256:98e192fc50e45a19b47d2d3252bec7becc2b4731683ec20ae755a04a77a9c8da \
    --control-plane --certificate-key 093d67c4b78c352daaa29b5a1266eb9a0eb357d285c332cd20bd1da3120ed023
```

## 6.2. Join the worker nodes to the cluster
```
# Paste the generated kubeadm join command from the main controlplane node
sudo kubeadm join 10.0.1.235:6443 --token 8hctpv.j9i2fof2qrsv2ncw \
    --discovery-token-ca-cert-hash sha256:98e192fc50e45a19b47d2d3252bec7becc2b4731683ec20ae755a04a77a9c8da
```

## 7. Enable pod networking by installing CNI plugin (calico)
> https://v1-17.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#pod-network
```
controlplane$ kubectl apply -f https://docs.projectcalico.org/v3.11/manifests/calico.yaml
```

## 8. Ensure that the controlplane and worker nodes are ready
```
# Ensure that the controlplane and worker node status are "Ready"
kubectl get node
```

## Reference
[1] https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

[2] https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/

[3] https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/
