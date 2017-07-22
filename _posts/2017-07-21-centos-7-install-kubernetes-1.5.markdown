---
layout: post
title:  "Centos 7 install kubernetes 1.5"
date:   2017-07-21 14:17:06 +0530
category: containers
comments: true
canonical_url: containers/centos-7-kubernetes-1-5-install.html
tags: [kubernetes, docker, containers, kubernetes 1.5, centos 7, docker cluster, calico]
shortinfo: This post list down the steps to install kubernetes 1.5 on centos 7.
description: steps to install kubernetes 1.5 on centos 7 with docker containerized application.
order: 1
---
## Kubernetes, docker and rocket
Kubernetes is an open source system for managing containerized applications across multiple hosts, providing basic mechanisms for deployment, maintenance, and scaling of applications.

Kubernetes currently has apis for docker and rocket containerized solutions.

Kubernetes 1.5 has been released. Changelog is available on https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG.md#downloads-for-v157


&nbsp;

## Centos 7 server setup
Install required packages, diable iptables, setup iptables and time.
```bash
setenforce 0
yum install  -y wget  vim-enhanced

# disable selinux
vi /etc/selinux/config
    SELINUX=disabled

yum install ntpdate
ntpdate pool.ntp.org

# allow kubernetes ports
iptables -A INPUT -p tcp --dport 8080 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT
iptables -A INPUT -p tcp --dport 2379 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT
iptables -A INPUT -p tcp --dport 8080 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT
```

### Kubernetes services and dependencies
Install kubernetes nodes and master packages on all nodes. Copy kubectl on nodes to access kubernetes API.
```bash
yum install -y docker kubernetes-node kubernetes-master libgo-devel
# copy kubectl
curl -O https://storage.googleapis.com/kubernetes-release/release/v1.5.2/bin/linux/amd64/kubectl
mv kubectl /usr/local/bin
chmod +x /usr/local/bin/kubectl
```

### Install SDN - calico
On all nodes install calico as SDN for inter-host docker container communication.

```bash
# start docker service
systemctl start docker
```
#### download calico plugins and dependencies
```bash
wget -N -P /opt/cni/bin https://github.com/projectcalico/calico-cni/releases/download/v1.4.2/calico
wget -N -P /opt/cni/bin https://github.com/projectcalico/calico-cni/releases/download/v1.4.2/calico-ipam

chmod +x /opt/cni/bin/calico /opt/cni/bin/calico-ipam

wget https://github.com/containernetworking/cni/releases/download/v0.3.0/cni-v0.3.0.tgz
tar zxvf cni-v0.3.0.tgz

cp loopback /opt/cni/bin/

chmod 755 /opt/cni/bin/loopback

mkdir -p /etc/cni/net.d

# get calicoctl
wget https://github.com/projectcalico/calicoctl/releases/download/v1.2.1/calicoctl

# set calico configuration
vim /etc/cni/net.d/10-calico.conf
>>
{
    "name": "calico-k8s-network",
    "type": "calico",
    "etcd_endpoints": "http://192.168.40.51:2379,http://192.168.40.52:2379,http://192.168.40.53:2379",
    "log_level": "DEBUG",
    "ipam": {
        "type": "calico-ipam",
         "assign_ipv4": "true"
    },
    "kubernetes": {
        "k8s_api_root": "http://naukri-dev-cluster.infoedge.com:8080"
    }
}

```

#### set calico env variable
```bash
echo "export ETCD_ENDPOINTS=http://<kube_master>:2379" >> ~/.bashrc
export ETCD_ENDPOINTS=http://<kube_master>:2379
export ETCD_ENDPOINTS=http://192.168.40.51:2379,http://192.168.40.52:2379,http://192.168.40.53:2379
```

&nbsp;
#### Kubernetes master setup

##### Etcd setup
On master server, install and run etcd server.
```bash
# creating user, directories and assign permissions
mkdir /var/lib/etcd;
mkdir /etc/etcd;
groupadd -r etcd;
useradd -r -g etcd -d /var/lib/etcd -s /sbin/nologin -c "etcd user" etcd;
chown -R etcd:etcd /var/lib/etcd

# install etcd
#ETCD_VERSION=`curl -s -L https://github.com/coreos/etcd/releases/latest | grep linux-amd64\.tar\.gz | grep href | cut -f 6 -d '/' | sort -u`;
ETCD_VERSION=v3.1.3
DOWNLOAD_URL=https://storage.googleapis.com/etcd
ETCD_DIR=/opt/etcd-$ETCD_VERSION;
mkdir $ETCD_DIR;
curl -L https://github.com/coreos/etcd/releases/download/$ETCD_VERSION/etcd-$ETCD_VERSION-linux-amd64.tar.gz | tar xz --strip-components=1 -C $ETCD_DIR;

ln -sf $ETCD_DIR/etcd /usr/bin/etcd && ln -sf $ETCD_DIR/etcdctl /usr/bin/etcdctl;
etcd --version
```

Setup etcd system command
```bash
cat << EOT > /usr/lib/systemd/system/etcd.service
[Unit]
Description=etcd service
After=network.target

[Service]
Type=notify
WorkingDirectory=/var/lib/etcd/
EnvironmentFile=-/etc/etcd/etcd.conf
User=etcd
ExecStart=/usr/bin/etcd
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOT
```

```bash
cat << EOT > /etc/etcd/etcd.conf
# [member]
ETCD_NAME="dev0"
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
#ETCD_SNAPSHOT_COUNTER="10000"
#ETCD_HEARTBEAT_INTERVAL="100"
#ETCD_ELECTION_TIMEOUT="1000"
ETCD_LISTEN_PEER_URLS="http://0.0.0.0:2380"
ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379"
#ETCD_LISTEN_CLIENT_URLS="http://localhost:2379"
ETCD_ADVERTISE_CLIENT_URLS="http://0.0.0.0:2379"
#ETCD_MAX_SNAPSHOTS="5"
#ETCD_MAX_WALS="5"
#ETCD_CORS=""
#
#[cluster]
#ETCD_INITIAL_ADVERTISE_PEER_URLS="http://localhost:2380,http://localhost:7001"
# if you use different ETCD_NAME (e.g. test), set ETCD_INITIAL_CLUSTER value for this name, i.e. "test=http://..."
ETCD_INITIAL_CLUSTER="dev0=http://192.168.2.179:2380,dev6=http://192.168.40.101:2380,dev3=http://192.168.40.51:2380,dev4=http://192.168.40.52:2380,dev5=http://192.168.40.53:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
#ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
#ETCD_ADVERTISE_CLIENT_URLS="http://localhost:2379,http://localhost:4001"
#ETCD_DISCOVERY=""
#ETCD_DISCOVERY_SRV=""
#ETCD_DISCOVERY_FALLBACK="proxy"
#ETCD_DISCOVERY_PROXY=""
#
#[proxy]
#ETCD_PROXY="off"
#
#[security]
#ETCD_CA_FILE=""
#ETCD_CERT_FILE=""
#ETCD_KEY_FILE=""
#ETCD_PEER_CA_FILE=""
#ETCD_PEER_CERT_FILE=""
#ETCD_PEER_KEY_FILE=""
EOT
```

##### start etcd service
```bash
systemctl start etcd
systemctl enable etcd;
```


## Kubernetes config
Configure kubernetes service configuration variables. Change IPs and other rules for cluster configuration.
### Update global config
```bash
# file - /etc/kubernetes/config
KUBE_LOGTOSTDERR="--logtostderr=true"
KUBE_LOG_LEVEL="--v=0"
KUBE_ALLOW_PRIV="--allow-privileged=true"
KUBE_MASTER="--master=http://kube_master:8080"
```
#### Update proxy  config
```bash
# file - /etc/kubernetes/proxy
# Add your own!
KUBE_PROXY_ARGS=""
```

### Master config
#### Update apiserver config
```bash
# file - /etc/kubernetes/apiserver
# The address on the local server to listen to.
KUBE_API_ADDRESS="--insecure-bind-address=0.0.0.0"

# The port on the local server to listen on.
KUBE_API_PORT="--port=8080"

# Port minions listen on
KUBELET_PORT="--kubelet-port=10250"

# Comma separated list of nodes in the etcd cluster
KUBE_ETCD_SERVERS="--etcd-servers=http://<kube-master>:2379"
ENABLE_ETCD_QUORUM_READS="false"
# Address range to use for services
KUBE_SERVICE_ADDRESSES="--service-cluster-ip-range=10.254.0.0/16"

# default admission control policies
KUBE_ADMISSION_CONTROL="--admission-control=NamespaceLifecycle,NamespaceExists,LimitRanger,PersistentVolumeLabel,DefaultStorageClass,ResourceQuota"

# Add your own!
KUBE_API_ARGS="--allow_privileged=true --apiserver-count=1 --runtime-config=batch/v2alpha1=true"
```
#### Update scheduler config
```bash
# file - /etc/kubernetes/scheduler
# Add your own!
KUBE_SCHEDULER_ARGS=""
```

#### Update controller manager config
```bash
# file - /etc/kubernetes/controller-manager
# Add your own!
KUBE_CONTROLLER_MANAGER_ARGS="--concurrent-namespace-syncs=5 --concurrent-service-syncs=20 --concurrent-rc-syncs=10 --terminated-pod-gc-threshold=50"
```


### Minion config



#### Update kubelet config
```bash
# file - /etc/kubernetes/kubelet
KUBELET_ADDRESS="--address=0.0.0.0"
KUBELET_HOSTNAME="--hostname-override=kube-node1"
KUBELET_API_SERVER="--api-servers=http://<kube-master>:8080"
KUBELET_POD_INFRA_CONTAINER="--pod-infra-container-image=registry.access.redhat.com/rhel7/pod-infrastructure:latest"

KUBELET_ARGS="--network-plugin=cni --cni-conf-dir=/etc/cni/net.d --cni-bin-dir=/opt/cni/bin --cluster-dns=kube_master --cluster-domain=kube.cluster"
```

## Start master services
``` bash
systemctl start kube-apiserver
systemctl start kube-controller-manager
systemctl start kube-scheduler
systemctl start kube-proxy
```


## Start node services
``` bash
systemctl start docker
systemctl start kubelet
systemctl start kube-proxy
```

## Setting calico
``` bash
cat << EOT > /tmp/ca.yaml
- apiVersion: v1
  kind: ipPool
  metadata:
    cidr: 10.11.0.0/16
  spec:
    ipip:
      enabled: true
      mode: cross-subnet
    nat-outgoing: true
#    disabled: true
EOT

./calicoctl create -f ca.yml
```


#### run calico
```bash
./calicoctl node run
```

## Check cluster config
On kube master run following command to check cluster config/health

``` bash
kubectl get nodes -o wide
# NAME              STATUS                        AGE       EXTERNAL-IP
# node-1            Ready                         1d       <none>

kubectl get namespaces
# NAME                                  STATUS    AGE
# default                               Active    1d
# kube-system                           Active    1d

```