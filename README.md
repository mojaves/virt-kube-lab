# Kubernetes on Virtual Machines on your home lab

## Introduction
WRITEME

## Definitions
Entities used in this document:

* `user` is the developer box. May be the same box as `host`, although we assume it isn't.
* `host` is the box on which VMs run. We assume the developer connects to it from `client` using SSH.
* `guest` is the OS running on any VM.
* `gateway` is the host on which dnsmasq run (see `host/lan` setup below). May be the same box as `host`, although we assume it isn't.

## hardware
TODO

## setup

### general

In this document, we assume you have a checkout of the `virt-kube-lab` repo on the `user` box, and that all commands are run within the root directory of the repo:
```bash
user# git clone https://github.com/mojaves/virt-kube-lab.git
user# cd virt-kube-lab
```

### host

WRITEME
Host distribution: CentOS 7

set up bridge and export its name:

```bash
# host
export VM_BRIDGE="k8sbr0"
```

### lan

### Required packages
```bash
# host
yum -y install \
libguestfs \
libguestfs-xfs \
libguestfs-tools \
libguestfs-tools-c \
jq
```

### Provision golden image for virtual machines
```bash
# host
virt-builder -o /var/lib/libvirt/images/c7-base.qcow2 --size=80G --format qcow2 --ssh-inject root:file:kojiro-kube-lan.pub --update --selinux-relabel --root-password file:rootpw centos-7.6
```

### Initial steps

```bash
# host
export VM_NAME="c7-test-vm"
```

### Provision virtual machines

Clone the disks:
```bash
# host
cp -a /var/lib/libvirt/images/c7-base.qcow2 /var/lib/libvirt/images/${VM_NAME}.qcow2
```


#### Provision for All-in-One:
```bash
# host
virt-install --name ${VM_NAME} --ram 6144 --vcpus 4 --cpu host --os-type linux --os-variant centos7.0 --disk path=/var/lib/libvirt/images/${VM_NAME}.qcow2,device=disk,bus=virtio,format=qcow2 --network bridge=${VM_BRIDGE},model=virtio --graphics none --console pty,target_type=serial --import
```

TODO: set up CPU passthrough (KVM L2)

### Set up DNS, IP
```bash
# TODO: dns
```

TODO: the jq queries are naive and fragile

Discover the network addresses of the box, using the main (/default) NIC
```bash
# host
VM_MACADDR=$(virsh qemu-agent-command ${VM_NAME} '{"execute":"guest-network-get-interfaces"}' | jq -r '.return[1] | .["hardware-address"]')
VM_IPADDR=$(virsh qemu-agent-command ${VM_NAME} '{"execute":"guest-network-get-interfaces"}' | jq -r '.return[1] | .["ip-addresses"][0] | .["ip-address"]')
echo -e "export VM_NAME=${VM_NAME}\nexport VM_MACADDR=${VM_MACADDR}\nexport VM_IPADDR=${VM_IPADDR}"
```

Now copy paste the output of last command (`VM_MACADDR` and `VM_IPADDR` variable definition) on `client`

Set the user-friendly hostname:
```bash
# user
ssh -oStrictHostKeyChecking=no root@${VM_IPADDR} hostnamectl set-hostname ${VM_NAME}.kube.lan
```

### Install base packages
```bash
# user
ssh -T root@${VM_IPADDR} yum -y install $( cat packages/centos7-guest-base.txt )
```

## Pick your distribution

Below the instructions for vanilla Kubernetes (K8S) and Openshift Origin (OKD)

## Kubernetes (K8S)

### Configure for Kubeadm

The following [script](scripts/-kube-box-setup.sh) demonstrates the needed/recommended steps to setup a box on which we wanna run kubernetes.
The script requires root privileges.
The script is built for convenience/fast setup. The steps are taken from the kubernetes documentation, so they are believed to be correct,
but there is no error check or recovery, so **YOU SHOULD NEVER RUN THIS SCRIPT UNAUDITED OR ON A PRODUCTION, OR OTHERWISE IMPORTANT, BOX**.

```bash
# !/bin/bash
set -e

## REPOS
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kube*
EOF

## SELinux
# Set SELinux in permissive mode (effectively disabling it) - still needed as k8s 1.13, unfortunately.
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

## Kernel
# setup kernel parameters needed/recommended by k8s
cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
sysctl --system
# setup kernel modules needed/recommended by k8s
cat <<EOF > /etc/modules-load.d/k8s.conf
br_netfilter
EOF
modprobe br_netfilter

## Firewalld
systemctl stop firewalld
systemctl disable firewalld
systemctl mask firewalld

## Reset iptables
iptables -P INPUT ACCEPT
iptables -P FORWARD ACCEPT
iptables -P OUTPUT ACCEPT
iptables -t nat -F
iptables -t mangle -F
iptables -F
iptables -X

## Disable swap - to avoid annoyances with kubelet
cp /etc/fstab /etc/fstab.orig
grep -v swap /etc/fstab.orig > /etc/fstab
```

To run the script on the provisioned VM:
```
# user
ssh -T root@${VM_IPADDR} < kube-box-setup.sh
```

### Install required packages

```bash
# user
ssh root@${VM_NAME} yum install -y $( cat packages/centos7-guest-container-base.txt )
ssh root@${VM_NAME} yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
```

```bash
# user
ssh root@${VM_NAME} systemctl enable --now docker
ssh root@${VM_NAME} systemctl enable --now kubelet
```

### Run kubeadm

```bash
# we will use flannel, so use parameters recommended by flannel
# user
ssh root@${VM_NAME} kubeadm init --pod-network-cidr=10.244.0.0/16 --ignore-preflight-errors=swap
```

```bash
# TODO: setup flannel
```

### Configure the host ass All-in-One

TODO: (un)taint node


## Openshift Origin (OKD)

TODO
