# install-k8s 

## Update
    sudo yum -y update && sudo systemctl reboot

    sudo tee /etc/yum.repos.d/kubernetes.repo<<EOF
    [kubernetes]
    name=Kubernetes
    baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
    enabled=1
    gpgcheck=1
    repo_gpgcheck=1
    gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
    EOF

    sudo yum -y install epel-release vim git curl wget kubelet kubeadm kubectl --disableexcludes=kubernetes
    kubectl version --client

## Set SELinux in permissive mode (effectively disabling it)
    sudo setenforce 0
    sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

    sudo systemctl enable --now kubelet

    sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
    sudo swapoff -a

## Configure persistent loading of modules
    sudo tee /etc/modules-load.d/containerd.conf <<EOF
    overlay
    br_netfilter

## Load at runtime
    sudo modprobe overlay
    sudo modprobe br_netfilter

## Ensure sysctl params are set
    sudo tee /etc/sysctl.d/kubernetes.conf<<EOF
    net.bridge.bridge-nf-call-ip6tables = 1
    net.bridge.bridge-nf-call-iptables = 1
    net.ipv4.ip_forward = 1
    EOF

## Reload configs
    sudo sysctl --system

## Install required packages
    sudo yum install -y yum-utils device-mapper-persistent-data lvm2

## Add Docker repo
    sudo yum-config-manager \
      --add-repo \
      https://download.docker.com/linux/centos/docker-ce.repo

## Install containerd
    sudo yum update -y && sudo yum install -y containerd.io

## Configure containerd and start service
    sudo mkdir -p /etc/containerd
    containerd config default | sudo tee /etc/containerd/config.toml

## Restart containerd
    sudo systemctl restart containerd
    sudo systemctl enable containerd

### Execute on Master Node
    sudo firewall-cmd --add-port={6443,2379-2380,10250,10251,10252,5473,179}/tcp --permanent
    sudo firewall-cmd --add-port={4789,8285,8472}/udp --permanent
    sudo firewall-cmd --reload

### Execute on Worker Node
    sudo firewall-cmd --add-port={10250,30000-32767,5473,179}/tcp --permanent
    sudo firewall-cmd --add-port={4789,8285,8472}/udp --permanent
    sudo firewall-cmd --reload


    lsmod | grep br_netfilter

    sudo kubeadm config images pull
