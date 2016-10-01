---
layout: post
title:  "Installing kubernetes 1.3 on centos 7!"
date:   2016-08-31 14:17:06 +0530
category: containers
comments: true
tags: [kubernetes, docker, containers]
shortinfo: This post list down the steps to install kubernetes 1.3 on centos 7.
description: steps to install kubernetes 1.3 on centos 7 with docker containerized application.
---


# Kubernetes, docker and rocket
Kubernetes is an open source system for managing containerized applications across multiple hosts, providing basic mechanisms for deployment, maintenance, and scaling of applications.

Kubernetes currently has apis for docker and rocket containerized solutions.

Kubernetes 1.3 has been released. Kubernetes 1.3 features can be found on [link](http://blog.kubernetes.io/2016/07/kubernetes-1.3-bridging-cloud-native-and-enterprise-workloads.html).

&nbsp;

#### Centos 7 installation
There isnt straight forward installation for kubernetes, you can install using kubernetes scripts for server flavours.

I list down steps to install `kubernetes 1.3` on `centos 7`.

To fully test kubernetes, we need 3 servers, one  kubernetes master and other 2 as nodes or minions.

&nbsp;

#### Kubernetes services and dependencies:
> 1. master services
    * kube-apiserver
    * kube-controller-manager
    * kube-scheduler
>
> 2. minion services
    * kube-proxy
    * kubelet
>
> 3. dependencies
    * etcd - shores all the data for kubernetes as key value store.
    * flanneld - flanneld is `SDN` which routes traffic across different minion containers.
&nbsp;

# Kubernetes servers and versions

&nbsp;

#### Servers/hosts

```
# master
192.168.166.206 kube-master
# minions
192.168.166.61 kube-minion1
192.168.166.173 kube-minion2
```
&nbsp;

#### kubernetes and dependency versions
```
K8S_VERSION=1.3.6
FLANNEL_VERSION=${FLANNEL_VERSION:-"0.5.5"}
ETCD_VERSION=${ETCD_VERSION:-"2.2.1"}
```

&nbsp;

## Download kubernetes
```bash
cd /usr/src/
scurl -o kubernetes.tar.gz https://github.com/kubernetes/kubernetes/releases/download/v${K8S_VERSION}/kubernetes.tar.gz

tar -xf kubernetes.tar.gz

export KUBERNETES_SRC=/usr/src/kubernetes/
export KUBERNETES_HOME=/usr/src/kubeSystem
```

&nbsp;

# Kubernetes installation

### Uninstall previous version

```bash
# stop services
for SERVICES in etcd flanneld kube-apiserver kube-controller-manager kube-scheduler kube-proxy kubelet flanneld docker; do
    systemctl stop $SERVICES
done

# uninstall packages
yum -y remove flannel kubernetes docker-logrotate docker docker-selinux

# remove etcd data dir, with caution you lost all cluster info
rm -rf /opt/etcd/
```

&nbsp;

## Master and minion servers
Installations for master and minion servers.

Master keeps all the cluster info and schedule pods. Minions are nodes that actually run container services.

&nbsp;

### Common packages on master and minion servers

#### install network and utility packages
```bash
yum -y install ntp net-tools telnet screen wget bridge-utils

systemctl start ntpd
systemctl enable ntpd
```

&nbsp;

#### installing flannel
```bash
cd /opt/kubernetes/

wget https://github.com/coreos/flannel/releases/download/v0.5.5/flannel-0.5.5-linux-amd64.tar.gz

tar -xf flannel-0.5.5-linux-amd64.tar.gz
cp flannel-0.5.5/flanneld bin/
ln -s /opt/kubernetes/bin/flanneld /usr/local/bin/
rm -rf flannel-0.5.5*
```

&nbsp;

#### Kubernetes systemd and binary files
```bash
mkdir -p /opt/kubernetes/bin/ /etc/kubernetes/

# copy all config/system and binary files on all servers
# getting systemd init files
cd $KUBERNETES_HOME
git clone https://github.com/akash-git/akash-git.github.io.git
cd akash-git.github.io

# copy systemd and config
cp systemd/* /usr/lib/systemd/system/
cp config/* /etc/kubernetes/

# copy kubernetes binary files on servers
# usually copy on single server and scp on other servers.
cd $KUBERNETES_SRC/server/kubernetes/server/bin
cp kube-apiserver kube-controller-manager kube-dns kubelet kube-proxy kube-scheduler /opt/kubernetes/bin/

# copy node files, docker prestart scripts
cp $KUBERNETES_SRC/cluster/centos/node/bin/* /opt/kubernetes/bin
```

&nbsp;

### master setup

#### installing etcd
```bash
mkdir -p /var/lib/etcd/ /etc/etcd/
cd /opt/kubernetes/
wget https://github.com/coreos/etcd/releases/download/v2.2.1/etcd-v2.2.1-linux-amd64.tar.gz
tar -xf etcd-v2.2.1-linux-amd64.tar.gz
cp etcd-v2.2.1-linux-amd64/etcd* bin/
ln -s /opt/kubernetes/bin/etcd /usr/local/bin/
ln -s /opt/kubernetes/bin/etcdctl /usr/local/bin/
rm -rf etcd-v2.2.1-linux-amd64*

systemctl daemon-reload
systemctl enable etcd
systemctl start etcd

# save flannel config to etcd
curl -L http://kube-master:2379/v2/keys/coreos.com/network/config -XPUT --data '{"Network": "10.254.0.0/16","SubnetLen": 24,"SubnetMin": "10.254.50.0","SubnetMax": "10.254.199.0","Backend": {"Type": "vxlan","VNI": 1}}'
```

&nbsp;

#### configure master
update master ip in `/etc/kubernetes/kube-apiserver`

```bash
# few changes according to your server IPs
# update kube-apiserver configuration
vi /etc/kubernetes/kube-apiserver
update > KUBE_ADVERTISE_ADDR="--advertise-address=192.168.166.206"
```

&nbsp;

#### starting kubernetes services on master
```bash
for SERVICES in etcd flanneld kube-apiserver kube-controller-manager kube-scheduler; do
    systemctl restart $SERVICES
    systemctl enable $SERVICES
    systemctl status $SERVICES
done
```

&nbsp;

### minion setups

#### install docker and setup kubectl
```bash
# installing docker
tee /etc/yum.repos.d/docker.repo <<-'EOF'
[dockerrepo]
name=Docker Repository
baseurl=https://yum.dockerproject.org/repo/main/centos/7/
enabled=1
gpgcheck=1
gpgkey=https://yum.dockerproject.org/gpg
EOF

yum install -y docker-engine-1.10.3-1.el7.centos

# installing kubectl
curl -O https://storage.googleapis.com/kubernetes-release/release/v1.2.4/bin/linux/amd64/kubectl
chmod +x kubectl
mv kubectl /usr/local/bin/kubectl
```

&nbsp;

#### configure docker
update `/usr/lib/systemd/system/docker.service` for your configurations

```bash
# add docker options, add other configurations like insecure-registry
vi /etc/sysconfig/docker
update > DOCKER_OPTIONS="-H tcp://127.0.0.1:4243 -H unix:///var/run/docker.sock"
```

&nbsp;

#### configure minion
update kublet configuration in file `/etc/kubernetes/kubelet`.
Change `NODE_HOSTNAME/NODE_ADDRESS` to respective server IPs.

```bash
vi /etc/kubernetes/kubelet
update > NODE_HOSTNAME="--hostname-override=192.168.166.61"
update > NODE_ADDRESS="--address=192.168.166.61"
```

&nbsp;

#### starting kubernetes services on minion1/minion2....
```bash
for SERVICES in kube-proxy kubelet flanneld docker; do
    systemctl restart $SERVICES
    systemctl enable $SERVICES
    systemctl status $SERVICES
done
```

&nbsp;

### test kubernetes setup
Kubernetes should be up now, you can test setup using `kubectl` command.

Try to get nodes information like below.

#### kubernetes commands

```bash
# get all nodes
kubectl get nodes
```

&nbsp;

## Troubleshoot

#### docker doesnt get flannel ips
It can happen if docker was started before flannel service.

To fix this, remove `docker0` bridge and restart flanneld and docker daemon.

```bash
#delete existing bridge
systemctl stop docker
ip link set dev docker0 down
brctl delbr docker0
rm /run/flannel/docker
systemctl restart flanneld
systemctl restart docker
```
