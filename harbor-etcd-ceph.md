---
layout: post
title: Harbor+etcd+docker结合Ceph搭建高可用集群
categories: [harbor, docker, etcd]
description: Harbor+etcd+docker结合Ceph搭建高可用集群
keywords: harbor,docker,etcd
---

> 由于原有的etcd一直是以单机的环境运行，不仅没有共享存储，也没有集群环境，而且生产上的私有image仓库也是使用的docker private registry，没有任何高可用，存在很大的隐患，所以，这里我搭建了一个套由ceph fs作为共享存储，为harbor和etcd集群提供存储服务的环境，特意在此记录下来，免得以后忘记了。整体架构图如下:

![](/images/posts/overall-structure.png)

### docker 安装

1.安装依赖包

```bash
sudo yum install -y yum-utils device-mapper-persistent-data lvm2
```

2.添加docker stable 库

```bash
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```

3.关闭edge和test库

```bash
yum-config-manager --disable docker-ce-edge
yum-config-manager --disable docker-ce-test
```

4.安装docker-ce

```bash
yum install docker-ce -y
```

5.配置docker graph driver

```bash
vim /etc/docker/daemon.json

{
  "graph": "/data_docker/docker"
}
```

6.配置成开机启动，这时候先别启动docker

```bash
systemctl enable docker
```

### ceph 搭建

#### 在管理节点上操作

1.添加ceph源

```bash
[ceph-noarch]
name=Ceph noarch packages
baseurl=https://download.ceph.com/rpm/el7/noarch
enabled=1
gpgcheck=1
type=rpm-md
gpgkey=https://download.ceph.com/keys/release.asc
```

2.更新并安装`ceph-deploy`

```bash
$ sudo yum update && sudo yum install ceph-deploy
```

2.配置从部署机器到所有其他节点的免密钥登录，具体参考[这里](https://kevinguo.me/2017/07/06/ansible-client/)

#### 在节点上操作

1.安装epel源

```bash
$ yum install yum-plugin-priorities -y
$ yum install epel-release -y
```

2.校对时间，由于ceph使用Paxos算法保证数据一致性，所以安装前要先保证各个节点的时间同步

```bash
$ sudo yum install ntp ntpdate ntp-doc

$ ntpdate 0.cn.pool.ntp.org
```

3.开放所需端口或关闭防火墙

```bash
$ systemctl stop firewalld
$ sudo firewall-cmd --zone=public --add-port=6789/tcp --permanent
```

4.关闭selinux

```bash
$ sudo setenforce 0
```

#### 创建集群

1.由于ceph-deploy工具部署集群前需要创建一些集群配置信息，其保存在`ceph.conf`文件中，这个文件将来会被复制到每个节点的 `/etc/ceph/ceph.conf`

```bash
# 创建集群配置目录
mkdir ceph-cluster && cd ceph-cluster
# 创建 monitor-node
ceph-deploy new i711-ustorage-1 i711-ustorage-2 i711-ustorage-3
# 追加 OSD 副本数量(测试虚拟机总共有3台)
echo "osd pool default size = 3" >> ceph.conf
# 追加时间ceph允许的误差时间范围到ceph.conf
mon_clock_drift_allowed = 5
mon_clock_drift_warn_backoff = 30
```

2.创建集群使用 `ceph-deploy`工具在部署节点上执行即可

```bash
# 安装ceph
$ ceph-deploy install i711-ustorage-1 i711-ustorage-2 i711-ustorage-3
```
**注意：在部署节点部署的时候，可能会因为网络原因导致无法安装ceph和ceph-radosgw，这时候，我们在各个节点上手动安装一下**
```bash
# 添加ceph 源
[Ceph]
name=Ceph packages for $basearch
baseurl=http://download.ceph.com/rpm-jewel/el7/$basearch
enabled=1
gpgcheck=1
type=rpm-md
gpgkey=https://download.ceph.com/keys/release.asc
priority=1

[Ceph-noarch]
name=Ceph noarch packages
baseurl=http://download.ceph.com/rpm-jewel/el7/noarch
enabled=1
gpgcheck=1
type=rpm-md
gpgkey=https://download.ceph.com/keys/release.asc
priority=1

[ceph-source]
name=Ceph source packages
baseurl=http://download.ceph.com/rpm-jewel/el7/SRPMS
enabled=1
gpgcheck=1
type=rpm-md
gpgkey=https://download.ceph.com/keys/release.asc
priority=1


# 执行安装
$ yum install ceph ceph-radosgw -y
```

3.初始化monitor node 和密钥文件

```bash
$ ceph-deploy mon create-initial
```

4.在管理节点上初始化osd

```bash
$ ceph-deploy osd prepare i711-ustorage-1:/dev/sdb i711-ustorage-2:/dev/sdb i711-ustorage-3:/dev/sdb
```

5.在管理节点上激活osd

```bash
$ ceph-deploy osd activate i711-ustorage-1:/dev/sdb1 i711-ustorage-2:/dev/sdb1 i711-ustorage-3:/dev/sdb1
```

6.在管理节点上部署 ceph cli 工具和密钥文件

```bash
$ ceph-deploy admin i711-ustorage-1 i711-ustorage-2 i711-ustorage-3
```

7.确保你对 `ceph.client.admin.keyring`有正确的操作权限，在每个节点上执行

```bash
$ sudo chmod +r /etc/ceph/ceph.client.admin.keyring
```

8.最后检查集群状态

```bash
$ ceph health
HEALTH_OK

$ ceph osd tree
ID WEIGHT  TYPE NAME               UP/DOWN REWEIGHT PRIMARY-AFFINITY
-1 1.44955 root default                                              
-2 0.48318     host i711-ustorage-1                                   
 0 0.48318         osd.0                up  1.00000          1.00000
-3 0.48318     host i711-ustorage-2                                   
 1 0.48318         osd.1                up  1.00000          1.00000
-4 0.48318     host i711-ustorage-3                                   
 2 0.48318         osd.2                up  1.00000          1.00000
```

#### Ceph rados创建

1.创建pool

```bash
$ rados mkpool docker
```

2.创建image

```bash
rbd create docker1 --size 100G -p docker
rbd create docker2 --size 100G -p docker
rbd create docker3 --size 100G -p docker
```

3.关闭不支持的特性

```bash
rbd feature disable docker1 exclusive-lock, object-map, fast-diff, deep-flatten -p docker
rbd feature disable docker2 exclusive-lock, object-map, fast-diff, deep-flatten -p docker
rbd feature disable docker3 exclusive-lock, object-map, fast-diff, deep-flatten -p docker
```

4.隐射image到块设备(依次在每个节点映射)

```bash
# docker1
rbd map docker1 --name client.admin -p docker
# docker2
rbd map docker2 --name client.admin -p docker
# docker3
rbd map docker3 --name client.admin -p docker
```

5.格式化设备

> 这里我们为了满足docker overlay2的需求，格式化的时候需要指定`-n ftype=1`

```bash
mkfs.xfs -n ftype=1 /dev/rbd0
```

6.创建docker root 目录，进行挂载，并添加到fstab中

```bash
# docker1
$ mkdir -p /data_docker/docker1

# 将下面的内容添加到/etc/fstab中
/dev/rbd0       /data_docker/docker1    xfs     noauto  0 0

# docker2
$ mkdir -p /data_docker/docker2

# 将下面的内容添加到/etc/fstab中
/dev/rbd0       /data_docker/docker2    xfs     noauto  0 0

# docker3
$ mkdir -p /data_docker/docker3

# 将下面的内容添加到/etc/fstab中
/dev/rbd0       /data_docker/docker3    xfs     noauto  0 0
```

> 因为ceph在每次重启的时候都需要去重新map，所以这里，我们需要配置下rbdmap.service

7.配置`/etc/ceph/rbdmap`(依次在每个节点上操作)

```bash
# docker1
# RbdDevice             Parameters
#poolname/imagename     id=client,keyring=/etc/ceph/ceph.client.keyring
docker/docker1           id=admin,keyring=/etc/ceph/ceph.client.admin.keyring

# docker2
docker/docker2           id=admin,keyring=/etc/ceph/ceph.client.admin.keyring

# docker3
docker/docker3           id=admin,keyring=/etc/ceph/ceph.client.admin.keyring
# 配置成开机启动
$ systemctl enable rbdmap.service
```

8.修改docker 的service文件，将rbdmap添加到after后面，保证rbdmap先运行挂载成功之后，再启动docker

```bash
vim /usr/lib/systemd/system/docker.service

After=network-online.target firewalld.service rbdmap.service
Wants=network-online.target
```

9.重启下机器，看看重启之后，docker是否启动成功，rbd image时候隐射成功

```bash
docker info
```

#### Ceph FS 创建

1.创建MDS

```bash
$ ceph-deploy mds create i711-ustorage-1 i711-ustorage-2 i711-ustorage-3
```

2.创建pool和fs，创建pool需要指定PG数量

```bash
ceph osd pool create data_data 32
ceph osd pool create data_metadata 32
ceph fs new data data_metadata data_data
```

3.复制密钥到文件中保存，将该文件复制到每个节点上的/etc/ceph下，并保证其权限

```bash
echo "AQCF8QxaBIkrCxAAt12YUP+NzLv0TB5XHeJ4xQ==" > ceph-key
```

4.将挂载添加到每个节点的fstab中

```bash
172.30.33.39:6789,172.30.33.40,172.30.33.41:6789:/      /data_harbor_etcd   ceph name=admin,secretfile=/etc/ceph/ceph-key,noatime,_netdev        0 2
```

5.重启机器后查看挂载

```bash
$ df -Th
Filesystem                                         Type      Size  Used Avail Use% Mounted on
172.30.33.39:6789,172.30.33.40,172.30.33.41:6789:/ ceph      1.5T  436M  1.5T   1% /data_harbor_etcd
```

至此，我们的共享存储就算是创建好了

### etcd 集群搭建

etcd 集群的搭建很简单，我们只需要执行yum 安装即可，重点在后面的配置文件修改

1.安装etcd

```bash
$ yum install etcd -y

# 在data目录下创建etcd目录，并修改其权限
$ mkdir -p /data/etcd
$ chown etcd.etcd  /data/etcd
```

2.修改配置文件,其他几个节点同理

```bash
# 本member名字
ETCD_NAME=etcd1

# 存放数据的位置
ETCD_DATA_DIR="/data_harbor_etcd/etcd/etcd1.etcd"

# 监听其他etcd实例的地址
ETCD_LISTEN_PEER_URLS="http://172.30.33.39:2380"

#监听客户端地址
ETCD_LISTEN_CLIENT_URLS="http://127.0.0.1:2379,http://172.30.33.39:2379"

# 通知其他etcd实例地址
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://172.30.33.39:2380"

# 初始化集群内节点地址
ETCD_INITIAL_CLUSTER="etcd1=http://172.30.33.39:2380,etcd2=http://172.30.33.40:2380,etcd3=http://172.30.33.41:2380"

# 初始化集群状态，new 表示新建
ETCD_INITIAL_CLUSTER_STATE="new"

# 通知客户端地址
ETCD_ADVERTISE_CLIENT_URLS="http://172.30.33.39:2379"
```

3.启动集群，然后查看

```bash
$ systemctl start etcd
$ systemctl enable etcd
$ etcdctl member list

186fb106f678cc55: name=etcd3 peerURLs=http://172.30.33.41:2380 clientURLs=http://172.30.33.41:2379 isLeader=false
bbe26c67e852d6f9: name=etcd1 peerURLs=http://172.30.33.39:2380 clientURLs=http://172.30.33.39:2379 isLeader=true
d4155475d1205f97: name=etcd2 peerURLs=http://172.30.33.40:2380 clientURLs=http://172.30.33.40:2379 isLeader=false
```

至此，etcd集群，也算是搭建完成了，这里我们没有使用域名和https，如果需要使用https的话，则需要证书制作，可参考[k8s-manual](https://kevinguo.me/2017/09/22/manual-deploy-kubernetes/#24-%E5%88%9B%E5%BB%BAetcd%E8%AF%81%E4%B9%A6%E9%85%8D%E7%BD%AE%E7%94%9F%E6%88%90-etcd-%E8%AF%81%E4%B9%A6%E5%92%8C%E7%A7%81%E9%92%A5)

### harbor 集群搭建

1.安装docker-compose

```bash
sudo curl -L https://github.com/docker/compose/releases/download/1.17.0/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
```

2.配置docker网络

```bash
# 安装flannel网络
$ yum install flannel -y

# 配置flannel
FLANNEL_ETCD_ENDPOINTS="http://10.19.65.27:2379,http://10.19.65.28:2379,http://10.19.65.29:2379"
FLANNEL_ETCD_PREFIX="/quarkfinance.com/network"
FLANNEL_OPTIONS="--iface=eth0"

# 在etcd中添加网络记录
etcdctl set /quarkfinance.com/network/config '{ "Network": "10.1.0.0/16" }'

# 在/usr/lib/systemd/system/docker.service中添加 $DOCKER_NETWORK_OPTIONS
ExecStart=/usr/bin/dockerd $DOCKER_NETWORK_OPTIONS

# 配置docker用非root用户启动
groupadd docker
gpasswd -a ${USER} docker
systemctl restart docker

# 启动docker，并将各服务配置成开机启动
systemctl start docker
systemctl enable flanneld
systemctl enable docker
```

3.查看docker网络是否生效

```bash
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN
    link/ether 02:42:90:6e:ac:b3 brd ff:ff:ff:ff:ff:ff
    inet 10.1.82.1/24 scope global docker0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:90ff:fe6e:acb3/64 scope link
       valid_lft forever preferred_lft forever
9: flannel0: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> mtu 1472 qdisc pfifo_fast state UNKNOWN qlen 500
    link/none
    inet 10.1.82.0/16 scope global flannel0
       valid_lft forever preferred_lft forever
    inet6 fe80::ff4f:42ff:391f:2a9d/64 scope link flags 800
       valid_lft forever preferred_lft forever
```

4.万事俱备，只欠harbor，下面，我们就来搭建我们的harbor

> 从git上下载最新的harbor包，git上有online和offlinle两个版，这里我选择online版，image都从外网拉取

```bash
wget https://github.com/vmware/harbor/releases/download/v1.2.2/harbor-online-installer-v1.2.2.tgz
```

5.修改harbor配置文件

> 1.修改挂载位置

```bash
将`docker-compose.yml`和`harbor.cfg`文件中所有的`/data`都修改成`/data_harbor_etcd/harbor`
```

> 2.由于使用外部mysql，所以删除`mysql`service，并且删除掉其他service对mysql的依赖(depends_on)
**如果你的harbor中已经有数据，那么请先导出mysql的数据，然后导入到你外部的mysql中**

```bash
docker exec -ti ${containerID} /bin/bash
mysqldump -u root -proot123 --databases registry > registry.dump

docker cp ${containerID}:/root/registry.dump ./
mysql -h ${your_mysql_host} -u ${your_mysql_user} -p
mysql> source ./registry.dump
```

> 3.在./common/templates/adminserver/env中添加如下内容

```bash
MYSQL_HOST=${your_mysql_host}
MYSQL_PORT=3306
MYSQL_USR=${your_mysql_user}
MYSQL_PWD=${your_mysql_passwd}
```

> 4.修改./common/templates/adminserver/env文件

```bash
将`RESET=false`改为`RESET=true`
```

> 5.由于要使用redis共享session，所以在./common/templates/ui/env中添加如下内容

```bash
_REDIS_URL=redis_ip:6379,100,password,0
```

> 6.关闭其他两个节点上的crt生成功能，留一个生成一套数字证书和私钥

```bash
custom_crt=false
```

> 7.修改common/templates/nginx/nginx.http.conf，找到location /, location /v2/ and location /service/这3个配置块， 将这三个配置块中的proxy_set_header X-Forwarded-Proto $scheme;配置移除

```bash
# When setting up Harbor behind other proxy, such as an Nginx instance, remove the below line if the proxy already has similar settings.
#proxy_set_header X-Forwarded-Proto $$scheme;
```

> 8.修改`common/templates/registry/config.yml`，修改`auth.token.realm`的地址

```bash
auth:
  token:
    issuer: harbor-token-issuer
    realm: https://harbor.quark.com/service/token
    # realm: $ui_url/service/token
    rootcertbundle: /etc/registry/root.crt
    service: harbor-registry
```

> 9.在各个节点上执行prepare脚本生成harbor各容器服务器的配置，然后启动容器

```bash
./prepare
docker-compose up -d
```

**注意：如果你启动之后，提示你无法连接远程数据库，请重启网络和docker daemon**

6.前端nginx LB https配置

> 1.新建一个证书脚本gencert.sh，内容如下

```bash
#!/bin/sh

# create self-signed server certificate:

read -p "Enter your domain [www.example.com]: " DOMAIN

echo "Create server key..."

openssl genrsa -des3 -out $DOMAIN.key 1024

echo "Create server certificate signing request..."

SUBJECT="/C=CN/ST=Hubei/L=Wuhan/O=quark/OU=devops/CN=$DOMAIN"

openssl req -new -subj $SUBJECT -key $DOMAIN.key -out $DOMAIN.csr

echo "Remove password..."

mv $DOMAIN.key $DOMAIN.origin.key
openssl rsa -in $DOMAIN.origin.key -out $DOMAIN.key

echo "Sign SSL certificate..."

openssl x509 -req -days 3650 -in $DOMAIN.csr -signkey $DOMAIN.key -out $DOMAIN.crt

echo "TODO:"
echo "Copy $DOMAIN.crt to /etc/nginx/ssl/$DOMAIN.crt"
echo "Copy $DOMAIN.key to /etc/nginx/ssl/$DOMAIN.key"
echo "Add configuration in nginx:"
echo "server {"
echo "    ..."
echo "    listen 443 ssl;"
echo "    ssl_certificate     /etc/nginx/ssl/$DOMAIN.crt;"
echo "    ssl_certificate_key /etc/nginx/ssl/$DOMAIN.key;"
echo "}"
```

> 2.生成证书，按提示输入

```bash
./gencert.sh
```

> 3.生成内容如下

```bash
[root@i612-devopsyw-1 ssl]# ll
total 20
-rwxr-xr-x 1 root root 949 Nov 22 16:28 gencert.sh
-rw-r--r-- 1 root root 887 Nov 22 17:40 harbor.quark.com.crt
-rw-r--r-- 1 root root 672 Nov 22 16:28 harbor.quark.com.csr
-rw-r--r-- 1 root root 887 Nov 22 17:40 harbor.quark.com.key
-rw-r--r-- 1 root root 963 Nov 22 16:28 harbor.quark.com.origin.key
```

> 4.将`harbor.quark.com.crt` 和 `harbor.quark.com.key` 复制到你自己的nginx的ssl目录下

```bash
cp harbor.quark.com.* /data/bkv2.0.1/common/nginx/ssl
```

> 5.配置nginx的upstream 和 `.conf`文件

**harbor-upstream.conf**
```bash
upstream harbor-server {
        ip_hash;
        server 172.30.33.40:80;
        server 172.30.33.39:80;
        server 172.30.33.41:80;
}
map $http_upgrade $connection_upgrade {
    default Upgrade;
    ''      close;
}
```

**harbor.conf**
```bash
server {
    listen 80;
    server_name harbor.quark.com;
    # Redirect all HTTP requests to HTTPS with a 301 Moved Permanently response.
    rewrite ^(.*)$  https://$host$1 permanent;
}

server {
    listen 443 ssl;
    server_name harbor.quark.com;
    ssl_certificate /data/bkv2.0.1/common/nginx/ssl/harbor.quark.com.crt;
    ssl_certificate_key /data/bkv2.0.1/common/nginx/ssl/harbor.quark.com.key;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    access_log logs/harbor_access.log ;
    error_log logs/harbor_error.log ;
    location / {
        proxy_pass   http://harbor-server;
        proxy_set_header   Host             $host;
        proxy_set_header   X-Real-IP        $remote_addr;
        proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto https;
        client_max_body_size 300M;
    }
}
```

> 6.重新加载nginx

```bash
$ nginx reload
```

> 7.访问试试效果

![harbor-ui](/images/posts/harbor-ui.png)

七.配置docker，让docker可以访问自签名证书的harbor

> 1.在每个docker主机上创建`/etc/docker/certs.d/harbor.quark.com`目录

```bash
$ mkdir -p /etc/docker/certs.d/harbor.quark.com
```

> 2.将`harbor.quark.com.crt` 复制到每个docker主机上的 `/etc/docker/certs.d/harbor.quark.com/`目录下

```bash
$ for IP in seq `39 31`;do
  scp harbor.quark.com.crt root@172.30.33.$IP:/etc/docker/certs.d/habor.quark.com/
done
```

> 3.我们login试试，然后push一个image

```bash
$ docker login harbor.quark.com
Username: ${your_ldap_user}
Password:
Login Succeeded

$ docker tag vmware/harbor-adminserver:v1.2.2 harbor.quark.com/quark/harbor-adminserver:v1.2.2
$ docker push harbor.quark.com/quark/harbor-adminserver:v1.2.2
The push refers to a repository [harbor.quark.com/quark/harbor-adminserver]
4fe250d3c912: Pushed
2202528221a2: Pushed
abf0579c40fd: Pushed
dd60b611baaa: Pushed
v1.2.2: digest: sha256:80bfbc20a1ee2bc6b05dfe31f1e082c08961a0f62e94089ef952800e92a1fc4c size: 1157
```

看看harbor是否已经有了这个image呢

![harbor-image](/images/posts/harbor-image.png)

有了，至此，我们的harbor就算是搭建完成，因为我们是在内网使用，所以使用自签名证书无所谓，如果要上到公网，那么就必须使用可信任的证书机构颁发的证书了
