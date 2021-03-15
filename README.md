# install-k8s 

## Update System
    sudo yum -y update && sudo systemctl reboot

## Add kubernetes repo
    sudo tee /etc/yum.repos.d/kubernetes.repo<<EOF
    [kubernetes]
    name=Kubernetes
    baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
    enabled=1
    gpgcheck=1
    repo_gpgcheck=1
    gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
    EOF


## Install k8s packages and etc
    sudo yum -y install epel-release vim git curl wget kubelet kubeadm kubectl --disableexcludes=kubernetes

## Start kubelet
    sudo systemctl start kubelet
    sudo systemctl enable --now kubelet
    kubectl version --client

## Set SELinux in permissive mode (effectively disabling it)
    sudo setenforce 0
    sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config


## Turn off swap on fstab
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

## Initialize master node
    lsmod | grep br_netfilter

## Pull kubernetes images
    sudo kubeadm config images pull

## Install using kubeadm
    export CALICO_IPV4POOL_CIDR=172.16.0.0
    export MASTER_IP=192.168.1.190

    sudo kubeadm init \
      --pod-network-cidr=$CALICO_IPV4POOL_CIDR/12 \
      --apiserver-advertise-address=$MASTER_IP \
      --apiserver-cert-extra-sans=$MASTER_IP

## Setup and verify k8s cluster
    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config

    kubectl cluster-info

## Install CNI (calico)
    kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml

## See k8s workloads
    kubectl get pods --all-namespaces
    kubectl get nodes -o wide

## Install kubernetes dashboard
    git clone https://github.com/achikam/install-k8s.git
    chmod +x ~/install-k8s/dashboard.sh
    sudo ln -s ~/install-k8s/dashboard.sh /usr/local/bin/dashboard
    dashboard start
    kubectl get pods -n kubernetes-dashboard
    kubectl get svc -n kubernetes-dashboard
*Notice the port of kubernetes dashboard service

## Access kubernetes dashboard
Access 
    [http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/](http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/)
or 
    __http://[master_node_IP]:[port_service]__
