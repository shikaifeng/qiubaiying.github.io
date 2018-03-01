---

layout: post
title: openshift镜像管理(二)
date: 2018-03-01
tags: [ openshift ]

---


# openshift 镜像管理（二）

## 三、登录仓库

### 前置条件
配置docker service 的配置文件，增加指向仓库的地址（对外就是配置的router地址，还可以是svc域名或者ip）

``` bash
vi /etc/sysconfig/docker
--insecure-registry=docker-registry-default.idc.yst.com.cn
```

配置如图：
![](https://ws1.sinaimg.cn/large/006tNc79gy1foxf5tdg22j31ha0b0jts.jpg)

### 登录内置的镜像仓库
[登录镜像仓库官方文档](https://docs.openshift.org/latest/install_config/registry/accessing_registry.html#access)

操作的例子如下，用的 docker-registy的ip+port的方式进行访问。
step1: 登录内部镜像仓库
``` bash
[root@master-openshift master]# docker login -u $(oc whoami) -p $(oc whoami -t) <registry_ip>:<port>
Login Succeeded
```

step2: 对镜像打tag
``` bash
docker tag docker.io/busybox 172.30.241.232:5000/openshift/busybox
```

step3:  push镜像

``` bash
docker push 172.30.241.232:5000/openshift/busybox:v2

``` 
step4 : web-consolse 查看镜像及版本
![](https://ws3.sinaimg.cn/large/006tNc79gy1foxf66ohkgj30ug04ojrw.jpg)

``` bash
docker login -u $(oc whoami) -p $(oc whoami -t) 
```

### 通过 web-console的提示进行登录

![](https://ws3.sinaimg.cn/large/006tNc79gy1foxf6j3gl0j30sh0dbjtf.jpg)

> 域名确保可以访问，并在任何需要访问的docker 客户端里面配置--insecure-registry 启动参数
-e unused 那个参数可以不用带
-p 后面token 会变，失效后可以到这里获取

- push 的镜像在部署时可通过tag进行区分和选择
![](https://ws1.sinaimg.cn/large/006tNc79gy1foxf7118ppj311o0gggn2.jpg)

[docker镜像仓库无法登录问题](https://github.com/moby/moby/issues/19112)

### 局域网的其他机器登录的例子

![](https://ws3.sinaimg.cn/large/006tNc79gy1foxf7imfifj31300r41kx.jpg)


## 四、遇到的问题
### 1、router 无法访问的问题排查
1）router查看主机的 80 和 443 端口，是否被监听：
``` bash
	ss -ltn|egrep -w "80|443"
``` 
2)  防火墙设置 iptables 和 firewalld

centos 7有两个防火墙设置，openshift使用了iptables，可以关闭掉firewalld。


### 2、docker service 配置文件修改
有两个地方可以修改docker service的启动参数
- 文件1：
/usr/lib/systemd/system/docker.service

``` bash
ExecStart=/usr/bin/dockerd  --insecure-registry=10.213.3.187   --registry-mirror=https://rofea0rs.mirror.aliyuncs.com
```

- 文件2:
/etc/sysconfig/docker

``` bash
[root@master ~]# cat /etc/sysconfig/docker
# /etc/sysconfig/docker

# Modify these options if you want to change the way the docker daemon runs
OPTIONS='--selinux-enabled --log-driver=journald --signature-verification=false'
if [ -z "${DOCKER_CERT_PATH}" ]; then
    DOCKER_CERT_PATH=/etc/docker
fi

# Do not add registries in this file anymore. Use /etc/containers/registries.conf
#other_args="--insecure-registry 172.30.0.0/16"

INSECURE_REGISIRY="--insecure-registry 172.30.0.0/16"
```

配置如图：
![](https://ws1.sinaimg.cn/large/006tNc79gy1foxf7vu2kqj31ha0b0jts.jpg)

> 两个配置文件只需要配置1个就可以了，一般只需要配置文件2。文件1会引用文件2的配置参数。

- 修改配置后重启docker：

``` bash
$  systemctl daemon-reload
$  systemctl restart docker
```

- 文件3:
```
  [root@localhost ~]# cat /etc/docker/daemon.json
  { "insecure-registries":["10.10.239.222:5000"] }
```
在使用daocloud进行加速时，会有个bug，需要去掉最后一个方括号后面的","

### 3、将本地镜像导入openshift 项目

oc import-image 10.213.3.187/test/nfsq-common-eureka:nfsq-common-eureka_22 -n openshift --confirm --insecure 

这个方式导入不能带版本号，同一个项目登录时

### 4、 权限问题

- 项目部署的权限问题
openshift内部启动容器，默认不使用权限，会有权限问题，所以需要修改openshift启动容器的权限，下面是比较粗暴的修改，具体要参考openshift的安全设置。

修改anyuid 的 SELINUX 为 RunAsAny，并赋权给default账户。

``` bash
oc edit scc anyuid
oadm policy add-scc-to-user anyuid -z default
```

