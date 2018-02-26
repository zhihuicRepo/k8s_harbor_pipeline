---
layout: post
title: kubespray容器化部署kubernetes高可用集群(3)
categories: [kubernetes, docker]
description: kargo容器化部署kubernetes高可用集群
keywords: kargo,kubernetes,docker
---

> 上一篇我们详细的剥析了通过kargo生成的各类服务的配置文件，学会了，如何生成证书，如何配置etcd,calico,kubelet，学会了如何配置一个kubernetes的高可用集群。既然集群已经配好了，那么这一章，我们就来学学如何配置一些常用的插件。

<!--more-->
目前尚不确定kubernetes各个插件版本之间的兼容性，我使用kubespray部署的各个IMAGE在[第一章]( https://kevinguo.me/2017/07/06/kargo%E5%AE%B9%E5%99%A8%E5%8C%96%E9%83%A8%E7%BD%B2kubernetes%E9%AB%98%E5%8F%AF%E7%94%A8%E9%9B%86%E7%BE%A41)中已经列出来了，中途在部署heapster和kibana的时候出了点问题，其他还未发现什么问题

目前，我所用到的插件如下：

* ingress
* kubedns
* dashboard
* efk
* heapster

### kubernetes-dashboard

1.从`kubernetes`官方git下载[最新源码](https://github.com/kubernetes/kubernetes.git)

```bash
git clone https://github.com/kubernetes/kubernetes.git
```

2.进入dashboard所在的目录，执行yml文件

```bash
cd $(pwd)/kubernetes/cluster/addons/dashboard
kubectl create -f .
```

3.确认dashboard是否成功创建

```bash
# 确认deployment
kubectl get deployment -n kube-system |grep dashboard

# 确认svc
kubectl get svc -n kube-system |grep dashboard

# 确认ep
kubectl get ep -n kube-system |grep dashboard

# 确认pods
kubectl get pods -n kube-system |grep dashboard

# 查看日志是否有报错
kubectl logs kubernetes-dashboard-1041558748-sppxt -n kube-system
```

初步确认dashboard启动成功，最终确认，需要等ingress创建后访问再看

### EFK

1.从`kubernetes`官方git下载[最新源码](https://github.com/kubernetes/kubernetes.git)

```bash
git clone https://github.com/kubernetes/kubernetes.git
```

2.为你要运行fluentd的节点添加label，因为`fluentd-es-ds.yml`文件中有`beta.kubernetes.io/fluentd-ds-ready: "true"`标签

```bash
kubectl label node k8s-node01 beta.kubernetes.io/fluentd-ds-ready=true
kubectl label node k8s-node02 beta.kubernetes.io/fluentd-ds-ready=true
kubectl label node k8s-registry beta.kubernetes.io/fluentd-ds-ready=true
```

3.确认标签添加是否成功

```bash
[root@k8s-master01 ~]# kubectl get nodes -l beta.kubernetes.io/fluentd-ds-ready
NAME           STATUS    AGE       VERSION
k8s-node01     Ready     19h       v1.6.7+coreos.0
k8s-node02     Ready     19h       v1.6.7+coreos.0
k8s-registry   Ready     19h       v1.6.7+coreos.0
```

4.进入fluentd-elasticsearch所在的目录，执行yml文件

```bash
cd $(pwd)/kubernetes/cluster/addons/fluentd-elasticsearch
kubectl create -f .
```

5.确认EFK是否创建成功

```bash
# 确认elasticsearch是否创建成功
kubectl get statefulset -n kube-system |grep elasticsearch
kubectl get svc -n kube-system|grep elasticsearch
kubectl get ep -n kube-system |grep elasticsearch
kubectl get pods -n kube-system |grep elasticsearch

# 确认fluentd是否创建成功
kubectl get ds -n kube-system |grep fluentd
kubectl get pods -n kube-system -o wide |grep fluentd

# 确认kibana是否创建成功
kubectl get deployment -n kube-system |grep kibana
kubectl get svc -n kube-system|grep kibana
kubectl get ep -n kube-system |grep kibana
kubectl get pods -n kube-system |grep kibana

# 查看日志是否有报错
kubectl logs elasticsearch-logging-0
kubectl logs fluentd-es-v2.0.1-kn2h2 -n kube-system
kubectl logs kibana-logging-3636197754-tnwjh -n kube-system
```

初步确认EFK启动成功，最终确认，需要等ingress创建后访问再看

### heapster

heapster比较简单

1.从`kubernetes`官方git下载[最新源码](https://github.com/kubernetes/heapster.git)

```bash
git clone https://github.com/kubernetes/heapster.git
```

2.进入heapster的deploy目录，执行kube.sh

```bash
cd $(pwd)/heapster/deploy
sh kube.sh start
```

3.确认heapster是否创建成功

```bash
# 确认heapster是否创建成功
kubectl get deployment -n kube-system |grep heapster
kubectl get svc -n kube-system |grep heapster
kubectl get ep -n kube-system |grep heapster
kubectl get pods -n kube-system |grep heapster

# 确认influxdb是否创建成功
kubectl get deployment -n kube-system |grep influxdb
kubectl get svc -n kube-system |grep influxdb
kubectl get ep -n kube-system |grep influxdb
kubectl get pods -n kube-system |grep influxdb

# 确认grafana是否创建成功
kubectl get deployment -n kube-system |grep grafana
kubectl get svc -n kube-system |grep grafana
kubectl get ep -n kube-system |grep grafana
kubectl get pods -n kube-system |grep grafana

# 查看日志是否有报错
kubectl logs heapster-1528902802-wcf24 -n kube-system
kubectl logs monitoring-grafana-2527507788-jsw1n -n kube-system
kubectl logs monitoring-influxdb-3480804314-31h4w -n kube-system
```

初步确认heapster启动成功，最终确认，需要等ingress创建后访问再看

### ingress

我们知道kubernetes暴露服务的方式目前有三种: LoadBlancer Service、NodePort Service、Ingress，前两个这里暂时不讲，这里主要解释下什么是Ingress

#### 什么是 ingress

Ingress Controller 实质上可以理解为一个监视器，Ingress Controller 通过不断跟kubernetes API打交道，实时感知后端service、pod等变化，比如新增和减少pod，service增加与减少等；当得到这些变化信息后，Ingress Controller 再结合Ingress 生成配置，然后更新反向代理负载均衡器，并刷新配置，达到服务发现的作用

下面的图说明一切问题
![](/images/posts/Ingress.png)

**注意：** 如果你进入 Ingress Controller里面看过它的 nginx 配置，你会发现实际上，Ingress是直接将请求转发到了服务的 endpoint IP，并没有转发给 Service Cluster IP，据说是为了提高性能，那么这个 Service Cluster IP 存在的意义在哪呢，如果你也在问这个问题，那么你得重新学习下 Service Cluster IP在 kubernetes 集群中的作用了

这里简单说一下，Cluster IP是kubernetes 集群中的一个虚拟IP，仅仅是为了方便集群内部通信和服务发现的

概念说了这么多，下面就开始实际操作吧

1.从`kubernetes`官方git下载[最新源码](https://github.com/kubernetes/ingress.git)

```bash
git clone https://github.com/kubernetes/ingress.git
```

2.为你要运行 Ingress 的节点添加label标签：role=frontal

```bash
kubectl label node k8s-node01 role=frontal
kubectl label node k8s-node02 role=frontal
```

3.进入ingress的`examples/daemonset/nginx`目录，修改yml文件

**注意：官方给出的 daemonset 版本的yml文件中，没有绑定宿主机的80端口，也就是说前端 Nginx 没有监听宿主机的80端口（尚且不知为何），所以需要自己加一下`hostNetwork`；同时因为我是将ingress部署到我指定的前端，所以还需要在文件末尾加上`nodeSelector`,截图如下**

![](/images/posts/ingress-daemonset.png)

![](/images/posts/ingress-nodeselector.png)

执行创建

```bash
cd $(pwd)/ingress/examples/daemonset/nginx

kubectl create -f nginx-ingress-daemonset.yaml
```

4.确认 Ingress controller 是否创建成功

```bash
# 确认 ingress 是否创建成功
kubectl get ds -n kube-system |grep ing
kubectl get pods -n kube-system |grep ingress

# 查看日志是否有报错
kubectl logs nginx-ingress-lb-mfdh5 -n kube-system
```

初步确认ingress controller启动成功，最终确认，需要等ingress创建后访问再看

5.Ingress controller已经创建成功了，那么现在我们来创建ingress 检验一下前面创建的服务是否成功

dashboard-ingress.yml

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: dashboard-ingress
  namespace: kube-system
spec:
  rules:
  - host: dashboard.kevinguo.me
    http:
      paths:
      - backend:
          serviceName: kubernetes-dashboard
          servicePort: 80
        path: /
```

kibana-ingress.yml

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: kibana-ingress
  namespace: kube-system
spec:
  rules:
  - host: kibana.kevinguo.me
    http:
      paths:
      - backend:
          serviceName: kibana-logging
          servicePort: 5601
```

elasticsearch-ingress.yml

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: elasticsearch-ingress
  namespace: kube-system
spec:
  rules:
  - host: elasticsearch.kevinguo.me
    http:
      paths:
      - backend:
          serviceName: elasticsearch-logging
          servicePort: 9200
        path: /
```

执行创建

```bash
kubectl create -f elasticsearch-ingress.yml
kubectl create -f dashboard-ingress.yml
kubectl create -f kibana-ingress.yml
```

创建完成后，我们来看看据诶过如何呢

Dashboard and heapster
![](/images/posts/dashboard-heapster.png)

EFK
![](/images/posts/kibana.png)

后期再研究新的东西了再加吧，头疼，下班了
