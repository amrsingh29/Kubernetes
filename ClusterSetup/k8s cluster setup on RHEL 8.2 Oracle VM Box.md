# Kubernetes cluster Setup on REHL 8.2 & CentOS 7 Using Kubeadm and Containerd

## BMC specific configuration on each nodes

### Enable password based authentication for discovery & monitoring
```
vi /etc/ssh/sshd_config
Set PasswordAuthentication yes
systemctl restart sshd
```
### Create discovery user and provide root access to discovery user
```
useradd discovery
passwd discovery
visudo
# Set following for discovery user
discovery ALL=(ALL) ALL
```

### Create patrol user and provide root access to patrol user
```
useradd patrol
passwd patrol
visudo
# Set following for patrol user
patrol ALL=(ALL) ALL
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
vi /etc/hosts
## Add entries
172.31.35.57 master ip-172-31-35-57.ap-southeast-1.compute.internal
172.31.40.243 worker ip-172-31-40-243.ap-southeast-1.compute.internal
```

### Firewall setting
You can decide to either disable firewall on all nodes or can update firewall with below mentioned reules.
#### Setup fireall setting
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
#### Disable the firewalld on all nodes
	systemctl stop firewalld
	systemctl disable firewalld
	systemctl status firewalld

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

### Forwarding IPv4 and letting iptables see bridged traffic
https://kubernetes.io/docs/setup/production-environment/container-runtimes/#forwarding-ipv4-and-letting-iptables-see-bridged-traffic
```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system
```
Verify that the br_netfilter, overlay modules are loaded by running the following commands:
```
lsmod | grep br_netfilter
lsmod | grep overlay
```
Verify that the net.bridge.bridge-nf-call-iptables, net.bridge.bridge-nf-call-ip6tables, and net.ipv4.ip_forward system variables are set to 1 in your sysctl config by running the following command:
```
sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward
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
**_NOTE:_** Make sure the pod network CIDR is different than host network CIDR.
```
kubeadm init --apiserver-advertise-address=192.168.56.39 --pod-network-cidr=10.0.0.0/8
```

## Update the IP table to add the kubernetes service IP to route traiffce to node IP
https://stackoverflow.com/questions/39872332/how-to-fix-weave-net-crashloopbackoff-for-the-second-node
Error we reveice if IP table is not updated.
```

[root@master ~]# kubectl get pods -n kube-system
NAME                             READY   STATUS             RESTARTS      AGE
coredns-5dd5756b68-lqhdp         1/1     Running            0             15m
coredns-5dd5756b68-tbtwd         1/1     Running            0             15m
etcd-master                      1/1     Running            0             15m
kube-apiserver-master            1/1     Running            0             15m
kube-controller-manager-master   1/1     Running            0             15m
kube-proxy-2gr7r                 1/1     Running            0             15m
kube-proxy-q89nl                 1/1     Running            0             13m
kube-scheduler-master            1/1     Running            0             15m
weave-net-fgwx7                  2/2     Running            1 (14m ago)   14m
weave-net-m4xrk                  1/2     CrashLoopBackOff   5 (58s ago)   7m8s

```
Find the Kubernetes service IP
```
# kubectl get svc -n kube-system

NAME        TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
kubernetes  ClusterIP   10.96.0.1        <none>        443/TCP   14m
```

Since IP of kube-dns is 10.96.0.10 then, the valid range for iptable rule would be 10.96.0.1/32
```
# iptables -t nat -I KUBE-SERVICES -d 10.96.0.1/32 -p tcp -m comment --comment "default/kubernetes:https cluster IP" -m tcp --dport 443 -j KUBE-MARK-MASQ
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
#### Wait for weave to install & check the status
```
kubectl get pods -n kube-system
```
Output:
```
[root@master ~]# kubectl get pods -n kube-system
NAME                             READY   STATUS    RESTARTS        AGE
coredns-5dd5756b68-clzhk         1/1     Running   0               21m
coredns-5dd5756b68-n89gw         1/1     Running   0               21m
etcd-master                      1/1     Running   0               21m
kube-apiserver-master            1/1     Running   0               21m
kube-controller-manager-master   1/1     Running   2 (9m34s ago)   21m
kube-proxy-58gtd                 1/1     Running   0               21m
kube-scheduler-master            1/1     Running   2 (9m34s ago)   21m
weave-net-mhxzl                  2/2     Running   1 (18m ago)     19m
```


## Run on Slave Nodes 
Run the join command obtained from kubeadm init output on all Workers nodes. Example
```
kubeadm join \
192.168.56.2:6443 --token … --discovery-token-ca-cert-hash sha256 . . . .
```
Output:

```
[root@worker ~]# kubeadm join 192.168.56.49:6443 --token rfi1wo.whirxh99po40b6m8 \
>         --discovery-token-ca-cert-hash sha256:f1971898756dadd83c55c27xxxxxxxxxxxxxxxxx96f8c74
[preflight] Running pre-flight checks
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Starting the kubelet
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.

```

## Test the setup
```
kubectl get nodes
```
Output:
```
[root@master ~]# kubectl get nodes
NAME     STATUS   ROLES           AGE     VERSION
master   Ready    control-plane   24m     v1.28.2
worker   Ready    <none>          2m29s   v1.28.2
```
Check all running pods
```
kubectl get pods -A
```
Output:
```
[root@master ~]# kubectl get pods -A
NAMESPACE     NAME                             READY   STATUS    RESTARTS      AGE
kube-system   coredns-5dd5756b68-clzhk         1/1     Running   0             25m
kube-system   coredns-5dd5756b68-n89gw         1/1     Running   0             25m
kube-system   etcd-master                      1/1     Running   0             25m
kube-system   kube-apiserver-master            1/1     Running   0             25m
kube-system   kube-controller-manager-master   1/1     Running   2 (13m ago)   25m
kube-system   kube-proxy-58gtd                 1/1     Running   0             25m
kube-system   kube-proxy-dg4gm                 1/1     Running   0             3m12s
kube-system   kube-scheduler-master            1/1     Running   2 (13m ago)   25m
kube-system   weave-net-mhxzl                  2/2     Running   1 (22m ago)   23m
kube-system   weave-net-q948n                  2/2     Running   0             3m12s
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
