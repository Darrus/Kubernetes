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
Docker will be use as the CRI (Container Runtime Interface) for kubernetes.

### 4) Configure Docker for Kubernetes
* #### Change Docker CGroup Driver to systemd
cgroup (Control Group) is a Linux kernel feature that allows system administrators to allocate and manage resources such as CPU, memory, I/O, and network bandwidth among a group of processes.
```
cd /etc/docker
touch daemon.json
sudo cat <<EOF | sudo tee /etc/docker/daemon.json
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

cgroupfs provides a low-level, file system-based interface for manual cgroup management.
systemd offers a high-level, command-line-based interface for simplified cgroup management and integration with system services.

* #### Disable swap
swap is a reserved space on a hard drive that is used as an extension of the computer's physical RAM (Random Access Memory). When the system runs low on physical RAM, it uses the swap space to temporarily store data that doesn't fit in RAM.
```
sudo swapoff -a
```
> The above command only temporaily disable swap
