## Kubernetes Setup Using Kubeadm In AWS EC2 Ubuntu Serverse.
##### Prerequisite
+ AWS Acccount.
+ Create 3 - Ubuntu Servers -- 18.04.
+ 1 Master (4GB RAM , 2 Core)  t2.medium
+ 2 Workers  (1 GB, 1 Core)     t2.micro
+ Create Security Group and open required ports for kubernetes.
   + Open all port for this illustration
+ Attach Security Group to EC2 Instance/nodes.

## Assign hostname &  login as ‘root’ user because the following set of commands need to be executed with ‘sudo’ permissions. Note that if running on first launch, an EC2 instance runs scripts in root mode so it's not needed. But if running after launch then you need to switch to root user.
```sh
#!/bin/bash
# Load the necessary module
modprobe br_netfilter

# Set hostname to 'master'
hostnamectl set-hostname master
  
# Update all installed packages.
apt update -y
apt upgrade -y

# if you get an error similar to
# '[ERROR Swap]: running with swap on is not supported. Please disable swap', disable swap:
swapoff -a

# Install some utils
apt install -y apt-transport-https curl containerd

# Add Kubernetes apt repository :
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
cat <<EOT | tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOT
apt update -y

# Install kubelet, kubeadm and kubectl, and pin their version:
KUBE_VERSION="1.21.0-00"
apt install -y kubelet=$KUBE_VERSION kubeadm=$KUBE_VERSION kubectl=$KUBE_VERSION
apt-mark hold kubelet kubeadm kubectl

# Enable and Start containerd service
systemctl enable containerd
systemctl restart containerd

# Ensure net.bridge.bridge-nf-call-iptables is set to 1 in your sysctl config, e.g.
cat <<EOT | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOT
sysctl --system

# Set up the Docker daemon
cat <<EOT | tee /etc/containerd/config.toml
[plugins]
  [plugins.cri.containerd]
    [plugins.cri.containerd.default_runtime]
      runtime_type = "io.containerd.runtime.v1.linux"
      runtime_engine = "/usr/local/bin/runc"
      runtime_root = ""
    [plugins.cri.containerd.untrusted_workload_runtime]
      runtime_type = "io.containerd.runtime.v1.linux"
      runtime_engine = "/usr/local/bin/runsc"
      runtime_root = "/run/containerd/runsc"
EOT

# Restart containerd:
systemctl restart containerd

# Enable and start kubelet service
systemctl enable kubelet
systemctl start kubelet

# Print success message
echo "Your script ran successfully" > /tmp/script-success.txt
```
## Exit as root user & execute the below commands as normal ubuntu user
```sh
sudo su - ubuntu
```

## Initialised the control plane.
``` sh
# Initialize Kubernates master by executing below commond.
sudo kubeadm init

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
## To verify, if kubectl is working or not, run the following command.
kubectl get pods -A
```sh
#deploy the network plugin - weave network
kubectl apply -f https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml
kubectl get pods -A
kubectl get node
```
```sh
#deploy the network plugin - weave network
kubectl apply -f https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml
kubectl get pods -A
kubectl get node
```
## Copy kubeadm join token from the master and execute in Worker Nodes to join to cluster
```sh
kubeadm join 172.31.10.12:6443 --token cdm6fo.dhbrxyleqe5suy6e \
        --discovery-token-ca-cert-hash sha256:1fc51686afd16c46102c018acb71ef9537c1226e331840e7d401630b96298e7d
```

echo "User data script executed successfully" >> /var/log/user-data.log



