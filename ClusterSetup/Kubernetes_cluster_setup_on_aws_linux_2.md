# Kubernetes cluster Setup on AWS Using Kubeadm and Containerd

## BMC specific configuration on each nodes

### Enable password based authentication for discovery & monitoring
```
vi /etc/ssh/sshd_config
Set PasswordAuthentication yes
systemctl restart sshd
```
### Provide root access to discovery user
```
visudo
# Set following for discovery user
discovery ALL=(ALL) ALL
```

## Prerequisites

- A compatible Linux hosts:  2 GB or more of RAM per machine and 2 CPUs or more 
- 3 - Ubuntu 20.04 LTS Serves:  1x Manager (4GB RAM, 2 vCPU)t2.medium type, 2x Workers (1 GB, 1 Core) t2.micro type 
- Full network connectivity between all machines in the cluster
  
### Check MAC address of your NIC
```
ifconfig -a
```
### Check the product UUID on your host
```
cat /sys/class/dmi/id/product_uuid
```

## Run on all nodes of the cluster as root user

### Set Unique hostname for each host. 
Change hostname of the machines using hostnamectl. For master nodes, run
```
hostnamectl set-hostname master
hostnamectl set-hostname slave-01
hostnamectl set-hostname slave-02
```

### Make a hostname entry for all nodes:
```
vi /ect/hosts
## Add entries
172.31.35.57 master ip-172-31-35-57.ap-southeast-1.compute.internal
172.31.40.243 worker ip-172-31-40-243.ap-southeast-1.compute.internal
```

- Certain ports are open on your machines(https://kubernetes.io/docs/reference/ports-and-protocols/)
  - On Master Node
	```
	6443/tcp for Kubernetes API Server
	2379-2380 for etcd server client API
	6783/tcp,6784/udp for Weavenet CNI
	10248-10260 for Kubelet API, Kube-scheduler, Kube-controller-manager, Read-Only Kubelet API, Kubelet health
	80,8080,443 Generic Ports
	30000-32767 for NodePort Services
	```
  - On Slave Nodes
	```
	6783/tcp,6784/udp for Weavenet CNI
	10248-10260 for Kubelet API etc
	30000-32767 for NodePort Services
	```

### Disable SWAP
You MUST disable swap in order for the kubelet to work properly 
```
swapoff -a
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

### Set SELinux in permissive mode (effectively disabling it)
```
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```

### Install Containerd
You must download the [containerd](https://github.com/containerd/containerd/blob/main/docs/getting-started.md) and extract in /usr/local
Check the latest version of containerd [here](https://github.com/containerd/containerd/releases).
```
cd /usr/local
CONTAINERD_VERSION="1.7.3"
wget https://github.com/containerd/containerd/releases/download/v$CONTAINERD_VERSION/containerd-$CONTAINERD_VERSION-linux-amd64.tar.gz
tar Cxzvf /usr/local containerd-$CONTAINERD_VERSION-linux-amd64.tar.gz
wget https://raw.githubusercontent.com/containerd/containerd/main/containerd.service
mkdir -p /usr/local/lib/systemd/system
mv containerd.service /usr/local/lib/systemd/system/containerd.service
systemctl daemon-reload
systemctl enable --now containerd
```

### Install Runc
Check the latest version of runc [here](https://github.com/opencontainers/runc/releases).
```
RUNC_VERSION="1.1.8"
wget https://github.com/opencontainers/runc/releases/download/v$RUNC_VERSION/runc.amd64
install -m 755 runc.amd64 /usr/local/sbin/runc
```

### Install CNI
Check the latest version of CNI [here](https://github.com/containernetworking/plugins/releases).
```
CNI_PLUGIN_VERSION="1.3.0"
mkdir -p /opt/cni/bin
wget https://github.com/containernetworking/plugins/releases/download/v$CNI_PLUGIN_VERSION/cni-plugins-linux-amd64-v$CNI_PLUGIN_VERSION.tgz
tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v$CNI_PLUGIN_VERSION.tgz
```

### Install CRICTL
Check the latest version of CRICTL [here](https://github.com/kubernetes-sigs/cri-tools/releases).
```
VERSION="v1.27.1" 
wget https://github.com/kubernetes-sigs/cri-tools/releases/download/$VERSION/crictl-$VERSION-linux-amd64.tar.gz
tar Cxzvf /usr/local crictl-$VERSION-linux-amd64.tar.gz
rm -f crictl-$VERSION-linux-amd64.tar.gz

cat <<EOF | sudo tee /etc/crictl.yaml
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 2
debug: false
pull-image-on-create: false
EOF
```

### Forwarding IPv4 and letting iptables see bridged traffic
https://kubernetes.io/docs/setup/production-environment/container-runtimes/#forwarding-ipv4-and-letting-iptables-see-bridged-traffic
```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF

sysctl --system
sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward
modprobe br_netfilter
sysctl -p /etc/sysctl.conf
```

### Install kubectl, kubelet and kubeadm
```
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF

yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
systemctl enable --now kubelet
```

## Run on Master Node and follow the instructions
```
kubeadm config images pull
```
**_NOTE:_** Make sure the pod network CIDR is different than host network CIDR
```
kubeadm init --pod-network-cidr=192.168.0.0/16
```
### To star using cluster using kubectl using regualr user
Switch to regular user and run following commands
```
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```

### Install any CNI plugin. We will use weavenet
Check the latest version of CNI weavenet plugin [here](https://github.com/weaveworks/weave/releases)
```
kubectl apply -f https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml
```

## Run on Slave Nodes 
Run the join command obtained from kubeadm init output on all Workers nodes. Example
```
kubeadm join \
192.168.56.2:6443 --token … --discovery-token-ca-cert-hash sha256 . . . .
```

## Test the setup
```
kubectl get nodes
kubectl get pods -A
```

## Run a demo app
```
kubectl run nginx --image=nginx --port=80 
kubectl expose pod nginx --port=80 --type=NodePort
```

## References
- https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
- https://kubernetes.io/docs/setup/production-environment/container-runtimes/#containerd
- https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/
- https://www.mirantis.com/blog/how-install-kubernetes-kubeadm/
- https://www.mankier.com/1/kubeadm-init
- https://kubernetes.io/docs/setup/production-environment/container-runtimes/#docker
- https://github.com/containerd/containerd/blob/main/docs/getting-started.md
- https://kubernetes.io/docs/reference/networking/ports-and-protocols/
- https://www.weave.works/docs/net/latest/kubernetes/kube-addon/#install
- https://github.com/skooner-k8s/skooner
- https://www.weave.works/docs/net/latest/kubernetes/kube-addon/#eks
- https://github.com/kubernetes-sigs/cri-tools/blob/master/docs/crictl.md
- https://www.mankier.com/1/kubeadm-init
