---

layout: post
title: openshift镜像管理(一)
date: 2018-03-01
tags: [ openshift ]

---


#  openshift 镜像管理 （一）

opeshift 可以自己对接第三方仓库，也可以使用自己自带的docker register

申请两个域名，用于内部访问：

域名 | 对应ip |
----|-----|
registry-console-default.idc.yst.com.cn | 10.213.3.177、10.213.3.178
docker-registry-default.idc.yst.com.cn | 10.213.3.177、10.213.3.178



## 一、 registry-console，内置的镜像仓库的web-console

openshift用ansible安装完成后，用拥有admin权限的账号dev，在default项目下可以查看到两个pod：docker-registry和registry-console。这两个分别表示openshift内置的镜像仓库和openshift本身提供的一个web console的页面用于查看内置仓库的镜像。先说下registry-console。


### 手动部署registry-console

- 默认通过ansible安装openshift 后，会有自带这个pod。
- 如果没有可以通过手动安装registry-console。

[手动安装registry-console的方法：](https://docs.openshift.org/latest/install_config/registry/deploy_registry_existing_clusters.html#deploying-the-registry-console)

> 需要着重注意的是：router名字要跟  OAuth client 的跳转链接一致，否则无法访问。如果发现不一致就重新修改部署。

如图

![](https://ws1.sinaimg.cn/large/006tNc79gy1foxf02r0ijj31gu0i2gqs.jpg)

> 实际中，重新部署的后的router他的host是自动生成的，无法直接访问。所以需要将registy-console的router修改为可以访问的域名。重新部署的步骤：

1、删除dc、svc、is、oauthclients 等，否则重新部署会失败
``` bash
oc delete dc registry-console
oc delete svc registry-console
oc delete route registry-console
oc delete is registry-console
oc delete oauthclients cockpit-oauth-client
``` 

2、参考上述官方文档，但在定义router的时候要指定hostname（域名）。当然在web页面新建router也是可以的。

``` bash
oc create route passthrough --service registry-console \
    --port registry-console \
    --hostname registry-console-default.idc.yst.com.cn \
    -n default 
```


3、重新部署registry-console

``` bash
oc new-app -n default --template=registry-console \
    -p OPENSHIFT_OAUTH_PROVIDER_URL="https://master-openshift.idc.yst.com.cn:8443" \
    -p REGISTRY_HOST=$(oc get route docker-registry -n default --template='{{ .spec.host }}') \
    -p COCKPIT_KUBE_URL=$(oc get route registry-console -n default --template='https://{{ .spec.host }}')
``` 

如图：
![](https://ws3.sinaimg.cn/large/006tNc79gy1foxf0pevpcj30ul0d60vp.jpg)


### 通过router 域名访问 registy-console 的web页面

[docker registry web-console 访问地址](https://registry-console-default.idc.yst.com.cn/registry)

直接登录router域名，跳转登录后进入内置仓库镜像管理页面，界面如图：

![](https://ws3.sinaimg.cn/large/006tNc79gy1foxf11f67nj30xo0cwacf.jpg)





## 二、docker-registry 内置镜像仓库

openshift通过ansible 安装后自动安装了docker-registry用于存储镜像。用户可以直接推送镜像到内部参考，推送的镜像可以在registry-console页面看到，也可以openshift web页面内通过选择

### 安装配置内置镜像仓库
默认安装，如果自己手动安装，请参考文档
镜像仓库文档

### 开放镜像仓库
[参考官方文档](https://docs.openshift.org/latest/install_config/registry/securing_and_exposing_registry.html#Manually%20Exposing%20a%20Secure%20Registry)

- 这里需要注意的是第3步:
host ： docker-register 中router的hostname
ca_certificate_file ： master 主机上的ca证书，/etc/origin/master/ca.crt on the master

``` bash
$ sudo mkdir -p /etc/docker/certs.d/<host>
$ sudo cp <ca_certificate_file> /etc/docker/certs.d/<host>
$ sudo systemctl restart docker
```

实际master主机上的操作：

``` bash
[root@master-openshift certs.d]# mkdir /etc/docker/certs.d/docker-registry-default.idc.yst.com.cn
[root@master-openshift docker-registry-default.idc.yst.com.cn]# pwd
/etc/docker/certs.d/docker-registry-default.idc.yst.com.cn
[root@master-openshift docker-registry-default.idc.yst.com.cn]# cp /etc/origin/master/ca.crt /etc/docker/certs.d/docker-registry-default.idc.yst.com.cn/
[root@master-openshift docker-registry-default.idc.yst.com.cn]# ls
ca.crt
```


- 另外如果还是无法登录，参考docker 文档 

[Troubleshoot insecure registry](https://docs.docker.com/registry/insecure/#docker-still-complains-about-the-certificate-when-using-authentication)

本次操作如下，操作系统为centos7.4

``` bash
root@master-openshift docker-registry-default.idc.yst.com.cn]# cd /etc/pki/ca-trust/source/anchors/
[root@master-openshift anchors]# ls
openshift-ca.crt
[root@master-openshift anchors]#  
[root@master-openshift anchors]# cp /etc/origin/master/ca.crt /etc/pki/ca-trust/source/anchors/docker-registry-default.idc.yst.com.cn.crt
[root@master-openshift anchors]# ls
docker-registry-default.idc.yst.com.cn.crt  openshift-ca.crt
[root@master-openshift anchors]# update-ca-trust enable
[root@master-openshift anchors]# 
[root@master-openshift anchors]# systemctl daemon-reload
[root@master-openshift anchors]# systemctl restart docker
```
