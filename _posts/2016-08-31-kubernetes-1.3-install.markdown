---
layout: post
title:  "Installing kubernetes 1.3 on centos 7!"
date:   2016-08-31 14:17:06 +0530
category: containers
tags: [kubernetes, docker, containers]
---

Kubernetes 1.3 has been released. Kubernetes 1.3 features can be found on [link](http://blog.kubernetes.io/2016/07/kubernetes-1.3-bridging-cloud-native-and-enterprise-workloads.html).

There isnt straight forward installation for kubernetes, you can install using kubernetes scripts for server flavours.

I list down steps to install ```kubernetes _1.3_``` on ```centos _7_```. 

To fully test kubernetes, we need 3 servers, one  kubernetes master and other 2 as nodes or minions.

>Kubernetes services and dependencies:
>- master services
>   + kube-apiserver
>   + kube-controller-manager
>   + kube-scheduler
>- minion services
    + kube-proxy
    + kubelet
>- dependencies
    + etcd - shores all the data for kubernetes as key value store.
    + flanneld - flanneld is ```SDN``` which routes traffic across different minion containers.

#### Servers/hosts

```
# master
192.168.166.206 kube-master
# minions
192.168.167.61 kube-minion1
192.168.166.173 kube-minion2
```

#### kubernetes and dependency versions
```
K8S_VERSION=1.3.6
FLANNEL_VERSION=${FLANNEL_VERSION:-"0.5.5"}
ETCD_VERSION=${ETCD_VERSION:-"2.2.1"}
```


#### Download kubernetes
```bash
cd /usr/src/
scurl -o kubernetes.tar.gz https://github.com/kubernetes/kubernetes/releases/download/v${K8S_VERSION}/kubernetes.tar.gz

tar -xf kubernetes.tar.gz

export KUBERNETES_SRC=/usr/src/kubernetes
```

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


## Setup master and minion servers
### common installations for master and minion servers
#### install network and utility packages
```bash
yum -y install ntp net-tools telnet screen wget bridge-utils

systemctl start ntpd
systemctl enable ntpd
```


#### Kubernetes systemd and binary files
```bash
# copy all config/system and binary files on all servers
# copy systemd and config
cp systemd/* /usr/lib/systemd/system/
cp config/* /etc/kubernetes/

# kubernetes binary files

mkdir -p /opt/kubernetes/bin/
copy binary files
    copy node files
        cp $KUBERNETES_SRC/cluster/centos/node/bin/* /opt/kubernetes/bin

    copy kubernetes binary files
        cd $KUBERNETES_SRC/server/kubernetes/server/bin
        cp kube-apiserver kube-controller-manager kube-dns kubelet kube-proxy kube-scheduler /opt/kubernetes/bin/

```



#### installing flannel
```bash
cd /opt/kubernetes/

wget https://github.com/coreos/flannel/releases/download/v0.5.5/flannel-0.5.5-linux-amd64.tar.gz

tar -xf flannel-0.5.5-linux-amd64.tar.gz
cp flannel-0.5.5/flanneld bin/
ln -s /opt/kubernetes/bin/flanneld /usr/local/bin/
rm -rf flannel-0.5.5*
```



## master setup
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

curl -L http://kube-master:2379/v2/keys/coreos.com/network/config -XPUT --data '{"Network": "10.254.0.0/16","SubnetLen": 24,"SubnetMin": "10.254.50.0","SubnetMax": "10.254.199.0","Backend": {"Type": "vxlan","VNI": 1}}'
```

#### configure master
```
 - update kube-apiserver configuration
    + set KUBE_ADVERTISE_ADDR to server IP
```

#### starting master services
```bash
for SERVICES in etcd flanneld kube-apiserver kube-controller-manager kube-scheduler; do
    systemctl restart $SERVICES
    systemctl enable $SERVICES
    systemctl status $SERVICES
done
```


## minion setups

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

#### setting docker
```bash
@todo merge /usr/lib/systemd/system/docker.service
with docker-systemd.service file

# add docker options
vi /etc/sysconfig/docker
DOCKER_OPTIONS="-H tcp://127.0.0.1:4243 -H unix:///var/run/docker.sock"
```

#### configure minion
```
 - update kublet configuration
    + set NODE_HOSTNAME to server IP
    + set NODE_ADDRESS to server IP


```

#### starting services
```bash
for SERVICES in kube-proxy kubelet flanneld docker; do
    systemctl restart $SERVICES
    systemctl enable $SERVICES
    systemctl status $SERVICES
done
```


## kubernetes commands
```bash
# get all nodes
kubectl get nodes

```

## Troubleshoot
##### docker doesnt get flannel ips

```bash
#delete existing bridge
systemctl stop docker
ip link set dev docker0 down
brctl delbr docker0
rm /run/flannel/docker
systemctl restart flanneld
systemctl restart docker
```
