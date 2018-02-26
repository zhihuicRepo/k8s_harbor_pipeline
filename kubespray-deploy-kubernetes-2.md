---
layout: post
title: kubespray容器化部署kubernetes高可用集群(2)
categories: [kubernetes, docker]
description: kargo容器化部署kubernetes高可用集群
keywords: kargo,kubernetes,docker
---

> 上一篇已经使用kargo搭建了kubernetes高可用集群，这里重点通过剥析kargo生成的配置文件来更加细化的了解下kubernetes，方便后期对kubernetes的自定义。所有的配置文件，我会放到[github](https://github.com/chinakevinguo/kubernetes-custom/tree/master/kubernetes%20%E9%AB%98%E5%8F%AF%E7%94%A8%E9%9B%86%E7%BE%A4%E7%BB%84%E4%BB%B6)上

<!--more-->
# etcd service

kargo中也将etcd以容器的方式运行，不过不是放在manifest中，而是单独用systemd的方式管理起来，然后通过etcd cluster来实现高可用

## 主要文件

```bash
/usr/local/etcd # etcd server启动的shell脚本文件
/usr/local/etcdctl # etcd 命令,使用命令docker cp etcd1:/usr/local/bin/etcdctl /usr/local/bin/etcdctl将命令复制到本地
/etc/systemd/system/etcd.service # etcd 的service文件
/etc/etcd.env # etcd 的环境文件
/etc/ssl/etcd/ # etcd 证书文件存放目录
/etc/ssl/etcd/openssl.conf # 生成etcd证书所需要的openssl文件
/usr/local/bin/etcd-scripts/ # 生成etcd证书的脚本文件
```

### etcd shell脚本

```bash
$ vim /usr/local/etcd
#!/bin/bash
/usr/bin/docker run \
  --restart=on-failure:5 \
  --env-file=/etc/etcd.env \
  --net=host \
  -v /etc/ssl/certs:/etc/ssl/certs:ro \
  -v /etc/ssl/etcd/ssl:/etc/ssl/etcd/ssl:ro \
  -v /var/lib/etcd:/var/lib/etcd:rw \
    --memory=512M \
      --name=etcd1 \
  quay.io/coreos/etcd:v3.1.5 \
    /usr/local/bin/etcd \
    "$@"
```

### etcd.service

```bash
$ vim /etc/systemd/system/etcd.service
[Unit]
Description=etcd docker wrapper
Wants=docker.socket
After=docker.service

[Service]
User=root
PermissionsStartOnly=true
EnvironmentFile=/etc/etcd.env
ExecStart=/usr/local/bin/etcd
ExecStartPre=-/usr/bin/docker rm -f etcd1
ExecStop=/usr/bin/docker stop etcd1
Restart=always
RestartSec=15s
TimeoutStartSec=30s

[Install]
WantedBy=multi-user.target
```

### etcd.env

```bash
$ vim /etc/etcd.env
ETCD_DATA_DIR=/var/lib/etcd
ETCD_ADVERTISE_CLIENT_URLS=https://172.30.33.90:2379
ETCD_INITIAL_ADVERTISE_PEER_URLS=https://172.30.33.90:2380
ETCD_INITIAL_CLUSTER_STATE=existing
ETCD_LISTEN_CLIENT_URLS=https://172.30.33.90:2379,https://127.0.0.1:2379
ETCD_ELECTION_TIMEOUT=5000
ETCD_HEARTBEAT_INTERVAL=250
ETCD_INITIAL_CLUSTER_TOKEN=k8s_etcd
ETCD_LISTEN_PEER_URLS=https://172.30.33.90:2380
ETCD_NAME=etcd1
ETCD_PROXY=off
ETCD_INITIAL_CLUSTER=etcd1=https://172.30.33.90:2380,etcd2=https://172.30.33.91:2380,etcd3=https://172.30.33.92:2380

# TLS settings
ETCD_TRUSTED_CA_FILE=/etc/ssl/etcd/ssl/ca.pem
ETCD_CERT_FILE=/etc/ssl/etcd/ssl/member-node1.pem
ETCD_KEY_FILE=/etc/ssl/etcd/ssl/member-node1-key.pem
ETCD_PEER_TRUSTED_CA_FILE=/etc/ssl/etcd/ssl/ca.pem
ETCD_PEER_CERT_FILE=/etc/ssl/etcd/ssl/member-node1.pem
ETCD_PEER_KEY_FILE=/etc/ssl/etcd
```

## 安装步骤
安装步骤很简单

1.按照最后的步骤生成证书
2.在每个etcd server的节点上配置好上面的配置文件
3.将etcd.service配置成系统服务(其实就是将运行docker的shell脚本写到systemd中)
# calico service

## 主要文件

```bash
/usr/local/bin/calicoctl #calicoctl命令
/etc/systemd/system/calico-node.service #calico-node service文件
/etc/calico/calico.env # calico 环境文件
/etc/calico/certs/  #(将etcd/ssl目录下的ca.pem 复制成cert.crt, node-nodex.pem复制成cert.crt, node-nodex-key.pem复制成key.pem)
/etc/cni/net.d/10-calico.conf # calico cni config
/etc/kubernetes/node-kubeconfig.yaml # 为kubernetes 配置calico网络
/opt/cni/bin/ #(将hyperkube和calico/cni镜像中/opt/cni/bin/目录下的所有插件复制到宿主机的/opt/cni/bin/目录下，可通过-v挂载的方式)
```

### calicoctl shell脚本
运行这个脚本其实就是运行一个容器来进行查询
```bash
#!/bin/bash
/usr/bin/docker run -i --privileged --rm \
--net=host --pid=host \
-e ETCD_ENDPOINTS=https://172.30.33.90:2379,https://172.30.33.91:2379,https://172.30.33.92:2379 \
-e ETCD_CA_CERT_FILE=/etc/calico/certs/ca_cert.crt \
-e ETCD_CERT_FILE=/etc/calico/certs/cert.crt \
-e ETCD_KEY_FILE=/etc/calico/certs/key.pem \
-v /usr/bin/docker:/usr/bin/docker \
-v /var/run/docker.sock:/var/run/docker.sock \
-v /var/run/calico:/var/run/calico \
-v /etc/calico/certs:/etc/calico/certs:ro \
--memory=170M --cpu-shares=100 \
calico/ctl:v1.1.0 \
"$@"
```

### calico-node.service
这个系统服务，其实就是启动一个容器
```bash
[Unit]
Description=calico-node
After=docker.service
Requires=docker.service

[Service]
EnvironmentFile=/etc/calico/calico.env
ExecStartPre=-/usr/bin/docker rm -f calico-node
ExecStart=/usr/bin/docker run --net=host --privileged \
 --name=calico-node \
 -e HOSTNAME=${CALICO_HOSTNAME} \
 -e IP=${CALICO_IP} \
 -e IP6=${CALICO_IP6} \
 -e CALICO_NETWORKING_BACKEND=${CALICO_NETWORKING_BACKEND} \
 -e FELIX_DEFAULTENDPOINTTOHOSTACTION=RETURN \
 -e AS=${CALICO_AS} \
 -e NO_DEFAULT_POOLS=${CALICO_NO_DEFAULT_POOLS} \
 -e CALICO_LIBNETWORK_ENABLED=${CALICO_LIBNETWORK_ENABLED} \
 -e ETCD_ENDPOINTS=${ETCD_ENDPOINTS} \
 -e ETCD_CA_CERT_FILE=${ETCD_CA_CERT_FILE} \
 -e ETCD_CERT_FILE=${ETCD_CERT_FILE} \
 -e ETCD_KEY_FILE=${ETCD_KEY_FILE} \
 -v /var/log/calico:/var/log/calico \
 -v /run/docker/plugins:/run/docker/plugins \
 -v /lib/modules:/lib/modules \
 -v /var/run/calico:/var/run/calico \
 -v /etc/calico/certs:/etc/calico/certs:ro \
 --memory=500M --cpu-shares=300 \
 calico/node:v1.1.0

Restart=always
RestartSec=10s

ExecStop=-/usr/bin/docker stop calico-node

[Install]
WantedBy=multi-user.target
```

### calico.env 文件
系统服务所需的环境变量信息
```bash
ETCD_ENDPOINTS="https://172.30.33.90:2379,https://172.30.33.91:2379,https://172.30.33.92:2379"
ETCD_CA_CERT_FILE="/etc/calico/certs/ca_cert.crt"
ETCD_CERT_FILE="/etc/calico/certs/cert.crt"
ETCD_KEY_FILE="/etc/calico/certs/key.pem"
CALICO_IP="172.30.33.90"
CALICO_IP6=""
CALICO_NO_DEFAULT_POOLS="true"
CALICO_LIBNETWORK_ENABLED="true"
CALICO_HOSTNAME="k8s-master01"
```

### 10-calico.conf
Calico CNI插件需要有一个标准的CNI配置文件
```bash
{
  "name": "calico-k8s-network",
  "hostname": "k8s-master01",
  "type": "calico",
  "etcd_endpoints": "https://172.30.33.90:2379,https://172.30.33.91:2379,https://172.30.33.92:2379",
  "etcd_cert_file": "/etc/ssl/etcd/ssl/node-node1.pem",
  "etcd_key_file": "/etc/ssl/etcd/ssl/node-node1-key.pem",
  "etcd_ca_cert_file": "/etc/ssl/etcd/ssl/ca.pem",
  "log_level": "info",
  "ipam": {
    "type": "calico-ipam"
  },
  "kubernetes": {
    "kubeconfig": "/etc/kubernetes/node-kubeconfig.yaml"
  }
}
```

## 安装步骤

安装步骤也很简单

1.首先将hyperkube和calico/cni镜像中的CNI插件拷贝到宿主机本地的/opt/cni/bin/目录下
2.将和etcd通信的证书配置好
3.将calicoctl,calico.env,10-calico.conf文件配置好放到指定的目录
4.将calico-node配置成systemd管理的系统服务
5.配置kubelet和kube-proxy
# kubelet service

## 主要文件

```bash
/usr/local/bin/kubelet # 启动kubelet的shell脚本文件
/usr/local/bin/kubectl # kubectl二进制文件
/etc/systemd/system/kubelet.service # kubelet 的service文件
/etc/kubernetes/kubelet.env # kubelet service相关的环境文件
/etc/kubernetes/ssl # kubernetes证书文件存放目录
/etc/kubernetes/openssl.conf # 生成kubernetes证书所依赖的openssl文件
/usr/local/bin/kubernetes-scripts/ # 生成证书文件的shell脚本
/etc/kubernetes/node-kubeconfig.yaml #
```

### kubelet shell 脚本

```bash
$ vim /usr/local/bin/kubelet
#!/bin/bash
/usr/bin/docker run \
  --net=host \
  --pid=host \
  --privileged \
  --name=kubelet \
  --restart=on-failure:5 \
  --memory=512M \
  --cpu-shares=100 \
  -v /dev:/dev:rw \
  -v /etc/cni:/etc/cni:ro \
  -v /opt/cni:/opt/cni:ro \
  -v /etc/ssl:/etc/ssl:ro \
  -v /etc/resolv.conf:/etc/resolv.conf \
  -v /etc/pki/tls:/etc/pki/tls:ro \
  -v /etc/pki/ca-trust:/etc/pki/ca-trust:ro \
  -v /sys:/sys:ro \
  -v /var/lib/docker:/var/lib/docker:rw \
  -v /var/log:/var/log:rw \
  -v /var/lib/kubelet:/var/lib/kubelet:shared \
  -v /var/lib/cni:/var/lib/cni:shared \
  -v /var/run:/var/run:rw \
  -v /etc/kubernetes:/etc/kubernetes:ro \
  quay.io/coreos/hyperkube:v1.6.1_coreos.0 \
  ./hyperkube kubelet \
  "$@"
```

### kubelet.service 文件

```bash
$ vim /etc/systemd/system/kubelet.service
[Unit]
Description=Kubernetes Kubelet Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=docker.service docker.socket calico-node.service
Wants=docker.socket calico-node.service

[Service]
EnvironmentFile=/etc/kubernetes/kubelet.env
ExecStart=/usr/local/bin/kubelet \
                $KUBE_LOGTOSTDERR \
                $KUBE_LOG_LEVEL \
                $KUBELET_API_SERVER \
                $KUBELET_ADDRESS \
                $KUBELET_PORT \
                $KUBELET_HOSTNAME \
                $KUBE_ALLOW_PRIV \
                $KUBELET_ARGS \
                $DOCKER_SOCKET \
                $KUBELET_NETWORK_PLUGIN \
                $KUBELET_CLOUDPROVIDER
ExecStartPre=-/usr/bin/docker rm -f kubelet
ExecReload=/usr/bin/docker restart kubelet
Restart=always
RestartSec=10s

[Install]
WantedBy=multi-user.target
```

### kubelet.env

```bash
$ vim /etc/kubernetes/kubelet.env
# logging to stderr means we get it in the systemd journal
KUBE_LOGGING="--logtostderr=true"
KUBE_LOG_LEVEL="--v=2"
# The address for the info server to serve on (set to 0.0.0.0 or "" for all interfaces)
KUBELET_ADDRESS="--address=172.30.33.90"
# The port for the info server to serve on
# KUBELET_PORT="--port=10250"
# You may leave this blank to use the actual hostname
KUBELET_HOSTNAME="--hostname-override=k8s-master01"

KUBELET_ARGS="--pod-manifest-path=/etc/kubernetes/manifests \
--pod-infra-container-image=gcr.io/google_containers/pause-amd64:3.0 \
--kube-reserved cpu=100m,memory=512M \
--node-status-update-frequency=10s \
--enable-cri=False --cgroups-per-qos=False \
--enforce-node-allocatable=''  --cluster_dns=10.233.0.2 --cluster_domain=cluster.local --resolv-conf=/etc/resolv.conf --kubeconfig=/etc/kubernetes/node-kubeconfig.yaml --require-kubeconfig --node-labels=node-role.kubernetes.io/master=true,node-role.kubernetes.io/node=true"
KUBELET_NETWORK_PLUGIN="--network-plugin=cni --network-plugin-dir=/etc/cni/net.d --cni-bin-dir=/opt/cni/bin"
# Should this cluster be allowed to run privileged docker containers
KUBE_ALLOW_PRIV="--allow-privileged=true"
KUBELET_CLOUDPROVIDER=""
```

### node-kubeconfig.yaml

```yaml
apiVersion: v1
kind: Config
clusters:
- name: local
  cluster:
    certificate-authority: /etc/kubernetes/ssl/ca.pem
    server: http://127.0.0.1:8080
users:
- name: kubelet
  user:
    client-certificate: /etc/kubernetes/ssl/node-node1.pem
    client-key: /etc/kubernetes/ssl/node-node1-key.pem
contexts:
- context:
    cluster: local
    user: kubelet
  name: kubelet-cluster.local
current-context: kubelet-cluster.local
```

# kubernetes manifest文件

```bash
/etc/kubernetes/manifests
$ ls -l
-rw-r--r--. 1 root root 2234 Apr 12 15:26 kube-apiserver.manifest
-rw-r--r--. 1 root root 1331 Apr 12 15:26 kube-controller-manager.manifest
-rw-r--r--. 1 root root 1319 Apr 12 15:23 kube-proxy.manifest
-rw-r--r--. 1 root root  708 Apr 12 15:26 kube-scheduler.manifest
```

我们知道在kubernetes中，/etc/kubernetes/manifests 目录下的文件，会由kubelet来在文件所在的节点生成static pod，下面看看每个文件的详细信息。

## kube-apiserver.manifest
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kube-apiserver
  namespace: kube-system
  labels:
    k8s-app: kube-apiserver
    kargo: v2
spec:
  hostNetwork: true
  dnsPolicy: ClusterFirstWithHostNet
  containers:
  - name: kube-apiserver
    image: quay.io/coreos/hyperkube:v1.6.1_coreos.0
    imagePullPolicy: IfNotPresent
    resources:
      limits:
        cpu: 800m
        memory: 2000M
      requests:
        cpu: 100m
        memory: 256M
    command:
    - /hyperkube
    - apiserver
    - --advertise-address=172.30.33.90
    - --etcd-servers=https://172.30.33.90:2379,https://172.30.33.91:2379,https://172.30.33.92:2379
    - --etcd-quorum-read=true
    # etcd 证书认证
    - --etcd-cafile=/etc/ssl/etcd/ssl/ca.pem
    - --etcd-certfile=/etc/ssl/etcd/ssl/node-node1.pem
    - --etcd-keyfile=/etc/ssl/etcd/ssl/node-node1-key.pem
    - --insecure-bind-address=127.0.0.1
    - --apiserver-count=3
    - --admission-control=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota
    - --service-cluster-ip-range=10.233.0.0/18
    - --service-node-port-range=30000-32767
    # kubernetes apiserver证书认证
    - --client-ca-file=/etc/kubernetes/ssl/ca.pem
    - --tls-cert-file=/etc/kubernetes/ssl/apiserver.pem
    - --tls-private-key-file=/etc/kubernetes/ssl/apiserver-key.pem
    # token auth认证
    - --token-auth-file=/etc/kubernetes/tokens/known_tokens.csv
    - --basic-auth-file=/etc/kubernetes/users/known_users.csv
    # service account
    - --service-account-key-file=/etc/kubernetes/ssl/apiserver-key.pem
    - --secure-port=6443
    - --insecure-port=8080
    - --storage-backend=etcd3
    - --v=2
    - --allow-privileged=true
    - --anonymous-auth=False
    livenessProbe:
      httpGet:
        host: 127.0.0.1
        path: /healthz
        port: 8080
      initialDelaySeconds: 30
      timeoutSeconds: 10
    volumeMounts:
    - mountPath: /etc/kubernetes
      name: kubernetes-config
      readOnly: true
    - mountPath: /etc/ssl/certs
      name: ssl-certs-host
      readOnly: true
    - mountPath: /etc/ssl/etcd/ssl
      name: etcd-certs
      readOnly: true
  volumes:
  - hostPath:
      path: /etc/kubernetes
    name: kubernetes-config
  - hostPath:
      path: /etc/ssl/certs/
    name: ssl-certs-host
  - hostPath:
      path: /etc/ssl/etcd/ssl
    name: etcd-certs
```

## kube-controller-manager.manifest
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kube-controller-manager
  namespace: kube-system
  labels:
    k8s-app: kube-controller
spec:
  hostNetwork: true
  dnsPolicy: ClusterFirstWithHostNet
  containers:
  - name: kube-controller-manager
    image: quay.io/coreos/hyperkube:v1.6.1_coreos.0
    imagePullPolicy: IfNotPresent
    resources:
      limits:
        cpu: 250m
        memory: 512M
      requests:
        cpu: 100m
        memory: 100M
    command:
    - /hyperkube
    - controller-manager
    # master ip
    - --master=http://127.0.0.1:8080
    - --leader-elect=true
    - --service-account-private-key-file=/etc/kubernetes/ssl/apiserver-key.pem
    # 集群范围内的证书
    - --root-ca-file=/etc/kubernetes/ssl/ca.pem
    - --cluster-signing-cert-file=/etc/kubernetes/ssl/ca.pem
    - --cluster-signing-key-file=/etc/kubernetes/ssl/ca-key.pem
    - --enable-hostpath-provisioner=false
    - --node-monitor-grace-period=40s
    - --node-monitor-period=5s
    - --pod-eviction-timeout=5m0s
    - --v=2
    livenessProbe:
      httpGet:
        host: 127.0.0.1
        path: /healthz
        port: 10252
      initialDelaySeconds: 30
      timeoutSeconds: 10
    volumeMounts:
    - mountPath: /etc/kubernetes/ssl
      name: ssl-certs-kubernetes
      readOnly: true
    - mountPath: /etc/ssl/certs
      name: ssl-certs-host
      readOnly: true
  volumes:
  - hostPath:
      path: /etc/kubernetes/ssl
    name: ssl-certs-kubernetes
  - hostPath:
      path: /etc/ssl/certs
    name: ssl-certs-host
```

## kube-scheduler.manifest
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kube-scheduler
  namespace: kube-system
  labels:
    k8s-app: kube-scheduler
spec:
  hostNetwork: true
  dnsPolicy: ClusterFirstWithHostNet
  containers:
  - name: kube-scheduler
    image: quay.io/coreos/hyperkube:v1.6.1_coreos.0
    imagePullPolicy: IfNotPresent
    resources:
      limits:
        cpu: 250m
        memory: 512M
      requests:
        cpu: 80m
        memory: 170M
    command:
    - /hyperkube
    - scheduler
    - --leader-elect=true
    - --master=http://127.0.0.1:8080
    - --v=2
    livenessProbe:
      httpGet:
        host: 127.0.0.1
        path: /healthz
        port: 10251
      initialDelaySeconds: 30
      timeoutSeconds: 10
```

## kube-proxy.manifest
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kube-proxy
  namespace: kube-system
  labels:
    k8s-app: kube-proxy
spec:
  hostNetwork: true
  dnsPolicy: ClusterFirstWithHostNet
  containers:
  - name: kube-proxy
    image: quay.io/coreos/hyperkube:v1.6.1_coreos.0
    imagePullPolicy: IfNotPresent
    resources:
      limits:
        cpu: 500m
        memory: 2000M
      requests:
        cpu: 150m
        memory: 64M
    command:
    - /hyperkube
    - proxy
    - --v=2
    - --master=http://127.0.0.1:8080
    - --bind-address=172.30.33.90
    - --cluster-cidr=10.233.64.0/18
    - --proxy-mode=iptables
    securityContext:
      privileged: true
    volumeMounts:
    - mountPath: /etc/ssl/certs
      name: ssl-certs-host
      readOnly: true
    - mountPath: /etc/kubernetes/node-kubeconfig.yaml
      name: "kubeconfig"
      readOnly: true
    - mountPath: /etc/kubernetes/ssl
      name: "etc-kube-ssl"
      readOnly: true
    - mountPath: /var/run/dbus
      name: "var-run-dbus"
      readOnly: false
  volumes:
  - name: ssl-certs-host
    hostPath:
      path: /etc/pki/tls
  - name: "kubeconfig"
    hostPath:
      path: "/etc/kubernetes/node-kubeconfig.yaml"
  - name: "etc-kube-ssl"
    hostPath:
      path: "/etc/kubernetes/ssl"
  - name: "var-run-dbus"
    hostPath:
      path: "/var/run/dbus"
```

# 生成证书
> 上面的文件都配置好后，我们就需要生正认证所需的证书了，以前看过漠神的一篇[kubernetes双向TSL认证](https://mritd.me/2016/09/11/kubernetes-%E5%8F%8C%E5%90%91-TLS-%E9%85%8D%E7%BD%AE/)，有兴趣的可以去看下，说实话证书这块，我也不是很懂。

## 生成etcd证书

### 修改`make-ssl-etcd.sh`脚本
```bash
# 在脚本开头加上
MASTERS=(node1 node2 node3)
HOSTS=(node1 node2 node3 node4 node5)

# 修改ETCD member的for循环语句，修改成下面这样
for host in ${MASTERS[*]}

# 修改Node keys的for循环语句，修改成下面这样
for host in ${HOSTS[*]}
```
### 生成证书
```bash
$ cd /usr/local/bin/etcd-scripts/
$ ./make-ssl-etcd.sh -f /etc/ssl/etcd/openssl.conf -d ~/certs/etcd/
$ chown kube.root ~/certs/etcd/*
$ chmod 700 ~/certs/etcd/*
```

## 生成kubernetes证书

### 修改`make-ssl.sh`脚本
```bash
# 在脚本开头加上
MASTERS=(node1 node2 node3)
HOSTS=(node1 node2 node3 node4 node5)

# 修改ETCD member的for循环语句，修改成下面这样
for host in ${MASTERS[*]}

# 修改Node keys的for循环语句，修改成下面这样
for host in ${HOSTS[*]}
```

### 生成证书
```bash
$ cd /usr/local/bin/kubernetes-scripts/
$ ./make-ssl.sh -f /etc/kubernetes/openssl.conf -d ~/certs/kubernetes/
$ chown kube.kube-cert ~/certs/kubernetes/*
$ chmod 600 ~/certs/kubernetes/*
```

# node节点上的主要文件

以后所有节点就按照这个来改就行了

/etc/kubernetes/kubelet.env
/etc/kubernetes/node-kubeconfig.yaml
/etc/kubernetes/manifests/kube-proxy.manifest
/etc/kubernetes/manifests/nginx-proxy.yml

> **Note:** 最后要重点注意一下，kargo会在node节点上配置单独的nginx反向代理，代理到apiserver集群，而node-kubeconfig.yaml和kube-proxy.manifest文件的内容需要修改一下

## node-kubeconfig.yaml
```bash
apiVersion: v1
kind: Config
clusters:
- name: local
  cluster:
    certificate-authority: /etc/kubernetes/ssl/ca.pem
    # server: https://172.30.33.90:6443 默认生成的
    server: https://127.0.0.1:6443 #修改成这样，因为本地节点上的nginx反向代理是监听的这个地址，这样就确保了高可用
    ...
```

## kube-proxy.manifest
```bash
apiVersion: v1
...
  # - --master=https://172.30.33.90:6443 默认生成
    - --master=https://127.0.0.1:6443
    - --kubeconfig=/etc/kubernetes/node-kubeconfig.yaml
```

至此， 基本上可以搭建一个基础的HA kubernetes 集群了，下一章将主要讲解如何配置一些常用插件
