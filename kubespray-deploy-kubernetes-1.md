---
layout: post
title: kubespray容器化部署kubernetes高可用集群(1)
categories: [kubernetes, docker]
description: kubespray容器化部署kubernetes高可用集群
keywords: kubespray,kubernetes,docker
---

> 捣鼓kubernetes有一段时间了，先后用过`yum`,`kubeadm`,`custom`等方式，但都不尽如人意，不是缺胳膊就是短腿，后来有幸翻到[漠然](https://mritd.me/)大神的[快速部署kubernetes高可用集群](https://mritd.me/2017/03/03/set-up-kubernetes-ha-cluster-by-kubespray/)，并且请教了无数次才最终搭建成功，在此，再次膜拜漠然大神，同时该篇blog也参考了漠神的的博客，主要是为了记录下来，方便以后查看。

<!--more-->
# 一、基础环境

* docker版本1.12.6
* CentOS 7

## 1.准备好要部署的机器

| IP | ROLE |
| :--- | :--- |
| 172.30.33.89 | k8s-registry-lb |
| 172.30.33.90 | k8s-master01-etcd01 |
| 172.30.33.91 | k8s-master02-etcd02 |
| 172.30.33.92 | k8s-master03-etcd03 |
| 172.30.33.93 | k8s-node01-ingress01 |
| 172.30.33.94 | k8s-node02-ingress02 |
| 172.30.33.31 | ansible-client |


## 2.准备部署机器 [ansible-client](https://kevinguo.me/2017/07/06/ansible-client/)

## 3.准备所需要镜像,由于被墙，所需镜像可以在百度云去下载，[点击这里](http://pan.baidu.com/s/1i5sUzmH)

| IMAGE | VERSION |
| :--- | :--- |
| quay.io/coreos/hyperkube| v1.6.7_coreos.0 |
| quay.io/coreos/etcd | v3.1.10 |
| calico/ctl | v1.1.3 |
| calico/node | v2.4.1 |
| calico/cni | v1.10.0 |
| calico/kube-policy-controller | v0.7.0 |
| quay.io/calico/routereflector | v0.3.0 |
| gcr.io/google_containers/kubernetes-dashboard-amd64 | v1.6.3 |
| gcr.io/google_containers/nginx-ingress-controller | 0.9.0-beta.11 |
| gcr.io/google_containers/defaultbackend | 1.3 |
| gcr.io/google_containers/cluster-proportional-autoscaler-amd64 | 1.1.1 |
| gcr.io/google_containers/fluentd-elasticsearch | 1.22 |
| gcr.io/google_containers/kibana | v4.6.1 |
| gcr.io/google_containers/elasticsearch | v2.4.1 |
| gcr.io/google_containers/k8s-dns-sidecar-amd64 | 1.14.4 |
| gcr.io/google_containers/k8s-dns-kube-dns-amd64 | 1.14.4 |
| gcr.io/google_containers/k8s-dns-dnsmasq-nanny-amd64 | 1.14.4 |
| andyshinn/dnsmasq | 2.72 |
| nginx | 1.11.4-alpine |
| gcr.io/google_containers/heapster-grafana-amd64 | v4.4.1 |
| gcr.io/google_containers/heapster-amd64 | v1.4.0 |
| gcr.io/google_containers/heapster-influxdb-amd64 | v1.1.1 |
| gcr.io/google_containers/pause-amd64 | 3.0 |
| lachlanevenson/k8s-helm | v2.2.2 |
| gcr.io/kubernetes-helm/tiller | v2.2.2 |

## 4.load所有下载的镜像
```bash
# 在ansible-client上操作
$ IP=(172.30.33.89 172.30.33.90 172.30.33.91 172.30.33.92 172.30.33.93 172.30.33.94)
$ for i in ${IP[*]}; do scp -r kubespray_images_v1.6.7 $i:~/; done
# 对所有要部署的节点操作
$ IMAGES=$(ls -l ~/kubespray_images_v1.6.7|awk -F' ' '{ print $9 }')
$ for x in ${images[*]}; do docker load -i kubespray_images_v1.6.7/$x; done
```
# 二、搭建集群

## 1.获取kubespray源码
```bash
$ git clone https://github.com/kubernetes-incubator/kubespray.git
```

## 2.编辑配置文件

``` sh
$ vim ~/kubespray/inventory/group_vars/k8s-cluster.yml

---
# 启动集群的基础系统,支持ubuntu, coreos, centos, none
bootstrap_os: centos

# etcd数据存放位置
etcd_data_dir: /var/lib/etcd

# kubernetes所需二进制文件将要安装的位置
bin_dir: /usr/local/bin

# kubrnetes配置文件存放目录
kube_config_dir: /etc/kubernetes
# 生成证书和token的脚本的存放位置
kube_script_dir: "{ { bin_dir } }/kubernetes-scripts"
# kubernetes manifest文件存放目录
kube_manifest_dir: "{ { kube_config_dir } }/manifests"
# kubernetes 命名空间
system_namespace: kube-system

# 日志存放位置
kube_log_dir: "/var/log/kubernetes"

# kubernetes证书存放位置
kube_cert_dir: "{ { kube_config_dir } }/ssl"

# kubernetes token存放位置
kube_token_dir: "{ { kube_config_dir } }/tokens"

# basic auth 认证文件存放位置
kube_users_dir: "{ { kube_config_dir } }/users"

# 关闭匿名授权
kube_api_anonymous_auth: false

## kubernetes使用版本
kube_version: v1.6.7

# 安装过程中缓存文件下载位置(最少1G)
local_release_dir: "/tmp/releases"
# 重试次数，比如下载失败等情况
retry_stagger: 5

# 证书组
kube_cert_group: kube-cert

# 集群日志等级
kube_log_level: 2

# HTTP下api server的basic auth认证用户名密码
kube_api_pwd: "test123"
kube_users:
  kube:
    pass: "{ {kube_api_pwd} }"
    role: admin
  root:
    pass: "{ {kube_api_pwd} }"
    role: admin



## 开关认证 (basic auth, static token auth)
#kube_oidc_auth: false
#kube_basic_auth: false
#kube_token_auth: false


## Variables for OpenID Connect Configuration https://kubernetes.io/docs/admin/authentication/
## To use OpenID you have to deploy additional an OpenID Provider (e.g Dex, Keycloak, ...)

# kube_oidc_url: https:// ...
# kube_oidc_client_id: kubernetes
## Optional settings for OIDC
# kube_oidc_ca_file: { { kube_cert_dir } }/ca.pem
# kube_oidc_username_claim: sub
# kube_oidc_groups_claim: groups


# 网络插件 (calico, weave or flannel)
kube_network_plugin: calico

# 开启 kubernetes network policies
enable_network_policy: false

# Kubernetes 服务的地址范围.
kube_service_addresses: 10.233.0.0/18

# pod 地址范围
kube_pods_subnet: 10.233.64.0/18

# 网络节点大小分配
kube_network_node_prefix: 24

# api server 监听地址及端口
kube_apiserver_ip: "{ { kube_service_addresses|ipaddr('net')|ipaddr(1)|ipaddr('address') } }"
kube_apiserver_port: 6443 # (https)
kube_apiserver_insecure_port: 8080 # (http)

# 默认dns后缀
cluster_name: cluster.local
# 为使用主机网络的pods使用/etc/resolv.conf解析DNS的子域
ndots: 2
# DNS 组件dnsmasq_kubedns/kubedns
dns_mode: dnsmasq_kubedns
# dns模式，可以是 docker_dns, host_resolvconf or none
resolvconf_mode: docker_dns
# 部署netchecker来检测DNS和HTTP状态
deploy_netchecker: false
# skydns service IP配置
skydns_server: "{ { kube_service_addresses|ipaddr('net')|ipaddr(3)|ipaddr('address') } }"
dns_server: "{ { kube_service_addresses|ipaddr('net')|ipaddr(2)|ipaddr('address') } }"
dns_domain: "{ { cluster_name } }"

# docker 存储目录
docker_daemon_graph: "/var/lib/docker"

## A string of extra options to pass to the docker daemon.
## This string should be exactly as you wish it to appear.
## An obvious use case is allowing insecure-registry access
## to self hosted registries like so:
docker_options: "--insecure-registry={ { kube_service_addresses } } --graph={ { docker_daemon_graph } } --iptables=false --storage-driver=devicemapper"
docker_bin_dir: "/usr/bin"

# 组件部署方式
# Settings for containerized control plane (etcd/kubelet/secrets)
etcd_deployment_type: docker
kubelet_deployment_type: docker
cert_management: script
vault_deployment_type: docker

# K8s image pull policy (imagePullPolicy)
k8s_image_pull_policy: IfNotPresent

# Monitoring apps for k8s
efk_enabled: true

# Helm deployment
helm_enabled: false
```

## 3.生成集群配置
配置完基本集群参数后，还需要生成一个集群配置文件，用于指定需要在哪几台服务器安装，和指定 master、node 节点分布，以及 etcd 集群等安装在那几台机器上。

```bash
# 定义集群IP
$ IP=(172.30.33.89 172.30.33.90 172.30.33.91 172.30.33.92 172.30.33.93 172.30.33.94)
# 利用kubespray自带的python脚本生成配置
$ CONFIG_FILE=~/kubespray/inventory/inventory.cfg python3 ~/kubespray/contrib/inventory_builder/inventory.py ${ IP[*] }

```

生成的配置如下，最好是在配置上加上`ansible_user=root`,我最开始在搭建的时候没有指定，报错了
```bash
node1    ansible_user=root ansible_host=172.30.33.90 ip=172.30.33.90
node2    ansible_user=root ansible_host=172.30.33.91 ip=172.30.33.91
node3    ansible_user=root ansible_host=172.30.33.92 ip=172.30.33.92
node4    ansible_user=root ansible_host=172.30.33.93 ip=172.30.33.93
node5    ansible_user=root ansible_host=172.30.33.94 ip=172.30.33.94

[kube-master]
node1
node2
node3

[kube-node]
node1
node2
node3
node4
node5

[etcd]
node1
node2
node3

[k8s-cluster:children]
kube-node
kube-master

[calico-rr]
```

## 5.docker,efk,etcd配置修改

提前修改ansbile中有关docker,efk,etcd的配置，因为后面在部署的过程中，ansible会检测docker的版本并下载最新的版本，但是由于墙的原因，导致无法下载，会一直卡在下载的地方，所以这里，我们要提前修改，同时需要升级etcd的版本，默认的3.0.6的版本，存在不稳定因素。

修改docker配置,将下面关于docker安装的部分全部注释掉
```bash
vim ~/kubespray/roles/docker/tasks/main.yml

# - name: ensure docker repository public key is installed
#  action: "{ { docker_repo_key_info.pkg_key } }"
#  args:
#    id: "{ {item} }"
#    keyserver: "{ {docker_repo_key_info.keyserver} }"
#    state: present
#  register: keyserver_task_result
#  until: keyserver_task_result|succeeded
#  retries: 4
#  delay: "{ { retry_stagger | random + 3 } }"
#  with_items: "{ { docker_repo_key_info.repo_keys } }"
#  when: not (ansible_os_family in ["CoreOS", "Container Linux by CoreOS"] or is_atomic)

# - name: ensure docker repository is enabled
#  action: "{ { docker_repo_info.pkg_repo } }"
#  args:
#    repo: "{ {item} }"
#    state: present
#  with_items: "{ { docker_repo_info.repos } }"
#  when: not (ansible_os_family in ["CoreOS", "Container Linux by CoreOS"] or is_atomic) and (docker_repo_info.repos|length > 0)

# - name: Configure docker repository on RedHat/CentOS
#  template:
#    src: "rh_docker.repo.j2"
#    dest: "/etc/yum.repos.d/docker.repo"
#  when: ansible_distribution in ["CentOS","RedHat"] and not is_atomic

# - name: ensure docker packages are installed
#  action: "{ { docker_package_info.pkg_mgr } }"
#  args:
#    pkg: "{ {item.name} }"
#    force: "{ {item.force|default(omit)} }"
#    state: present
#  register: docker_task_result
#  until: docker_task_result|succeeded
#  retries: 4
#  delay: "{ { retry_stagger | random + 3 } }"
#  with_items: "{ { docker_package_info.pkgs } }"
#  notify: restart docker
#  when: not (ansible_os_family in ["CoreOS", "Container Linux by CoreOS"] or is_atomic) and (docker_package_info.pkgs|length > 0)

# 如果你是自己安装的docker,记得将这段注释掉,除非你觉得template中的docker.service你能用
#- name: Set docker systemd config
#  include: systemd.yml
```

修改efk配置，注释掉`KIBANA_BASE_URL`这段，否则后面你搭建efk之后，无法访问kibana
```bash
vim ~/kubespray/roles/kubernetes-apps/efk/kibana/templates/kibana-deployment.yml.j2

#          - name: "KIBANA_BASE_URL"
#            value: "{ { kibana_base_url } }"

```

修改download配置，更改etcd和kubedns版本

**1.6.7版本中,为了使用更高版本的calico node,我自己多添加了一个变量`calico_node_version`**

```bash
vim ~/kubespray/roles/download/defaults/main.yml
etcd_version: v3.1.10
calico_node_version: "v2.4.1"
kubedns_version: 1.14.4
calico_policy_version: "v0.7.0"
```

**注意: 如果你修改了kubedns_version版本，那么也需要修改/root/kubespray/roles/kubernetes-apps/ansible/defaults/main.yml文件中的kubedns_version版本**

## (可选)6.修改docker.service

```bash
# 如果你的docker.service中没有MountFlags则不需要这一步
# 注释掉/usr/lib/systemd/system/docker.service 中的MountFlags=slave
```

## 7.在ansible-client上一键部署

```bash
$ ansible-playbook -i ~/kubespray/inventory/inventory.cfg cluster.yml -b -v --private-key=~/.ssh/id_rsa
```

部署成功后如下
![](/images/posts/ansible-run.png)
相关node信息
![](/images/posts/nodes.png)
相关pod信息
![](/images/posts/pods.png)
