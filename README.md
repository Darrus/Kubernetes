# Kubernetes

## How to setup Kubernetes in VirtualBox running CentOS
### 1) Setup epel repository
```
sudo dnf config-manager --set-enabled crb
sudo dnf install epel-release epel-next-release
```
crb (CodeReady Linux Builder) repository contains additional packages for use by developers.

Extra Packages for Enterprise Linux (EPEL) is an initiative within the Fedora Project to provide high quality additional packages for CentOS Stream and Red Hat Enterprise Linux (RHEL).

EPEL Next is an additional repository that allows package maintainers to alternatively build against CentOS Stream. This is sometimes necessary when CentOS Stream contains an upcoming RHEL library rebase, or if an EPEL package has a minimum version build requirement that is already in CentOS Stream but not yet in RHEL.

### 2) Install kubernetes components
```
cd /etc/yum.repos.d
touch kubernetes.repo
```
> kubernetes.repo
```
[kubernetes] 
name=Kubernetes 
baseurl=https://pkgs.k8s.io/core:/stable:/v1.30/rpm/
enabled=1 
gpgcheck=1 
repo_gpgcheck=1 
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.30/rpm/repodata/repomd.xml.key
```
Setup repo to grab the kubernetes packages

```
sudo dnf install kubectl kubeadm kubelet
```
kubelet: The kubelet is the primary “node agent” that runs on each node. It can register the node with the apiserver using one of: the hostname; a flag to override the hostname; or specific logic for a cloud provider.

kubectl: The Kubernetes command-line tool, kubectl, allows you to run commands against Kubernetes clusters. You can use kubectl to deploy applications, inspect and manage cluster resources, and view logs.

kubeadm: Using kubeadm, you can create a minimum viable Kubernetes cluster that conforms to best practices.

### 3) Install Docker
```
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo dnf install docker-ce docker-ce-cli
```
We will be using Docker(docker uses containerd underlying) as our container runtime.

### 4) Configure CRI for Kubernetes
* #### Change Docker CGroup Driver to systemd
cgroup (Control Group) is a Linux kernel feature that allows system administrators to allocate and manage resources such as CPU, memory, I/O, and network bandwidth among a group of processes.
```
cd /etc/docker
touch daemon.json
```

> daemon.json
```
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": { "max-size": "100m" },
  "storage-driver": "overlay2"
}
```

- cgroupfs provides a low-level, file system-based interface for manual cgroup management.
- systemd offers a high-level, command-line-based interface for simplified cgroup management and integration with system services.

* #### Install cri-dockerd
cri-dockerd is a container runtime interface (CRI) implementation for Docker. It allows you to use Docker as the container runtime for Kubernetes.

Install wget
```
sudo dnf install wget
```

Download cri-dockerd bin and install it
```
wget https://github.com/Mirantis/cri-dockerd/releases/download/v0.3.15/cri-dockerd-0.3.15.amd64.tgz
tar -xvzf cri-dockerd-0.3.15.amd64.tgz
cd cri-dockerd
sudo install -o root -g root -m 0755 cri-dockerd /usr/local/bin/cri-dockerd
```

Setup service and socket for cri-dockerd
```
wget https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.service
wget https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.socket
sudo cp cri-docker.service /etc/systemd/system
sudo cp cri-docker.socket   /etc/systemd/system
sed -i -e 's,/usr/bin/cri-dockerd,/usr/local/bin/cri-dockerd,' /etc/systemd/system/cri-docker.service
```

Disable SELINUX temporarily
```
setenforce 0
```
Disable SELINUX permanently
```
sed -i --follow-symlinks 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
```
Security-Enhanced Linux (SELinux) is a security architecture for Linux® systems that allows administrators to have more control over who can access the system.

Enable the service and start it
```
systemctl daemon-reload
systemctl enable cri-docker.service
systemctl enable --now cri-docker.socket
systemctl start cri-docker.service
```

Remove containerd configurations
```
sudo rm /etc/containerd/config.toml
systemctl restart containerd
```

### 3) Disable swap
swap is a reserved space on a hard drive that is used as an extension of the computer's physical RAM (Random Access Memory). When the system runs low on physical RAM, it uses the swap space to temporarily store data that doesn't fit in RAM.
```
sudo swapoff -a
```
> The above command only temporaily disable swap

To permanently disable swap, you'll have to comment out a line in the file below
```
sudo vi /etc/fstab
```
Comment out `/dev/mapper/cs-swap` by adding a `#` infront of the line.

### 4) Enable IP Forwarding and iptables for processing
```
cd /etc/sysctl.d
sudo touch kubernetes.conf
sudo vi kubernetes.conf
```

> kubernetes.conf
```
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
```
These parameters determine whether packets crossing a bridge are sent to iptables for processing.

iptables is a firewall utility in Linux that allows you to manage incoming and outgoing network traffic based on predetermined security rules. 
```
net.ipv4.ip_forward = 1
```
This allows linux to forward the request to the correct NIC (Network Interface Card) in the event where the request comes in for the wrong NIC.

### 5) Open ports for kubernetes
#### Control plane
| Protocol | Direction | Port Range |	Purpose	| Used By |
| -------- | --------- | ---------- | ------- | ------- |
| TCP |	Inbound |	6443 | Kubernetes API server | All |
| TCP	| Inbound	| 2379-2380	| etcd server client API | kube-apiserver, etcd |
| TCP	| Inbound	| 10250	| Kubelet API	| Self, Control plane |
| TCP	| Inbound	| 10259	| kube-scheduler	| Self |
| TCP	| Inbound	| 10257	| kube-controller-manager	| Self |

Although etcd ports are included in control plane section, you can also host your own etcd cluster externally or on custom ports.

#### Worker node(s)
| Protocol | Direction | Port Range	| Purpose	| Used By |
| -------- | --------- | ---------- | ------- | ------- |
| TCP	| Inbound	| 10250	| Kubelet API	| Self, Control plane |
| TCP	| Inbound	| 10256	| kube-proxy	| Self, Load balancers |
| TCP	| Inbound	| 30000-32767	| NodePort Services	| All |

```
sudo firewall-cmd --add-port=6443/tcp --permanent
```

### 6) Initialize Master Node
#### Create config file for kubeadm
> kubeadm-master.yaml
```
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
localApiEndpoint:
  advertiseAddresss: <IP Address for the cluster members to reach>
nodeRegistration:
  criSocket: unix:///var/run/cri-dockerd.sock
  imagePullPolicy: IfNotPresent
  taints: []
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
controlPlaneEndpoint: <IP Address that all control planes shared with>
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
containerRuntimeEndpoint: unix:///var/run/cri-dockerd.sock
```

run `kubeadm init --config kubeadm-master.yaml`

Next run the following commands to configure kubectl to run on current user.
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

You can check if the master node is running by running `kubectl get nodes`

### 7) Setup Flannel as CNI
We will be using Flannel as our CNI (Control Network Interface) with Kubernetes.

```
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```

If you have configured the POD network CIDR, you'll need to configue kube-flannel.yml as well.

Currently the default network in flannel is `10.244.0.0/16`

### 8) Join worker nodes
Generate bootstrap token from the master node.
```
sudo kubeadm create token --print-join-command
```

Create the following kubeadm config
> kubeadm-worker.yaml
```
apiVersion: kubeadm.k8s.io/v1beta3
kind: JoinConfiguration
nodeRegistration:
  criSocket: unix:///var/run/cri-dockerd.sock
  imagePullPolicy: IfNotPresent
  taints: []
discovery:
  bootstrapToken:
    token: <Bootstrap token generated from master node>
    apiServerEndpoint: <Your master node ip and port>
    caCertHashes: ['<Cert hash generated with bootstrap token>']
```

Run the following command.
```
sudo kubeadm join --config kubeadm-worker.yaml
```

You can verify the node was created by running `kubectl get node` on your master node.

#### Patch worker node
You may have to patch the podCidr in the worker node, if the `--allocate-node-cidrs=true` properties was not set in the master controller node.

The config can be found in `/etc/kubernetes/manifests/`

To patch the node run the following command on the master node.
```
kubectl patch node <Node name> -p '{"spec":{"podCIDR":"<Pod network cidr>"}}'
```
