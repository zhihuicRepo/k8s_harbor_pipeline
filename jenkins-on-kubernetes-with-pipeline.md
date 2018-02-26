---
layout: post
title: jenkins with pipeline on kubernetes
categories: [kubernetes,jenkins]
description: jenkins with pipeline on kubernetes
keywords: kubernetes,jenkins
---

> jenkins CI/CD用了有很长一段时间了，包括现公司的docker container deployment也是通过写pipeline workflow来实现的，但是当我在将jenkins迁往kubernetes的过程中，还是踩了不少的坑，这里记录下来。

该流程包含了 `checkout scm` --> `build artifacts` --> `build image` --> `deploy to k8s`

流程相对简单，而且并没有涉及到代码分支，集中测试，蓝绿部署等等

### 一、集群以及必要组件的搭建

请参考[手动搭建kubernetes HA集群](https://kevinguo.me/categories/#kubernetes),[kubernetes ceph笔记](https://kevinguo.me/categories/#ceph)

### 二、jenkins各个yaml文件

所有文件都放在[这里](https://github.com/chinakevinguo/kubernetes-jenkins.git)，我们搭建的时候只需将对应的位置修改成自己的即可

### 三、配置jenkins

> jenkins 部署成功之后，我们需要安装对应的插件，配置和kubernetes的关联，这里除了必要的插件之外，我们额外需要安装一个kubernetes Plugin

kubernetes cloud的配置相对简单，我们只需要指定`Kubernetes URL`以及`Jenkins URL`即可，因为jenkins在kubernetes中，所以`Kubernetes URL`和`Jenkins URL`均为内部service就行了，如下图

![](/images/posts/kubernetes-cloud.png)

### 四、新建pipeline job 测试

> jenkins kubernetes cloud配置成功之后，我们就需要来新建一个pipeline job测试一下，这里我新建了一个`learn-groovy`的job

新建job更加的简单，只需要指定你的`Jenkinsfile`的地址即可，如下图

![](/images/posts/jenkins-pipeline.png)


> 所有的工作都在`Jenkinsfile`中定义完成，这就是pipeline了

关于当前这个example项目的对应配置文件有如下几个
* app-deploy.yaml 当前项目部署所需的yaml文件
* Jenkinsfile 当前项目部署流程所需文件
* Jenkinsfile.yaml 当前项目构建部署过程中可变参数的变量文件
* Dockerfile 构建image所需文件

以上所有文件在[这里](https://github.com/chinakevinguo/learn-groovy.git)

我们的job构建成功后，最终的结果如下

![](/images/posts/jenkins-kubernetes-result.png)

![](/images/posts/jenkins-kubernetes-result-2.png)
