---

layout: post
title: openshift镜像管理--删除重建(三)
date: 2018-03-01
tags: [ openshift ]

---

# 镜像管理（三）
# 重新安装系统自带docker-registry仓库解决证书问题,结合对照镜像管理二安装，这里主要是删除，安装的方法可参考官方文档


## docker仓库
 系统默认安装docker仓库，但是未使用HTTPS加密，这里删除重装
### 删除
 ```
  oc delete route docker-registry
  oc delete sa registry
  oc delete dc docker-registry
  oc delete svc docker-registry
  oc delete clusterrolebinding.authorization.openshift.io registry-registry-role
 ```
### 安装
  ```
  oadm registry --service-account=registry
  ```
### 授权
```
  oadm policy add-role-to-user system:registry dev
  oadm policy add-role-to-user admin admin -n openshift
  oadm policy add-role-to-user system:image-builder dev
  oc adm policy add-role-to-user system:image-puller system:anonymous -n openshift

oadm policy add-scc-to-user privileged -z default

oadm policy add-scc-to-user anyuid -z default
```

## 更改为HTTPS加密
```
  oc adm ca create-server-cert \
      --signer-cert=/etc/origin/master/ca.crt \
      --signer-key=/etc/origin/master/ca.key \
      --signer-serial=/etc/origin/master/ca.serial.txt \
      --hostnames='docker-registry.default.svc.cluster.local,172.30.212.246,registry.oc.downtown8.cn' \
      --cert=/etc/secrets/registry.crt \
      --key=/etc/secrets/registry.key
  
  
  oc delete secrets registry-secret
  oc secrets new registry-secret     /etc/secrets/registry.crt     /etc/secrets/registry.key
  oc secrets link registry registry-secret
  oc secrets link default  registry-secret
  oc volume dc/docker-registry --add --type=secret --secret-name=registry-secret -m /etc/secrets
  oc env dc/docker-registry     REGISTRY_HTTP_TLS_CERTIFICATE=/etc/secrets/registry.crt     REGISTRY_HTTP_TLS_KEY=/etc/secrets/registry.key



 oc patch dc/docker-registry --api-version=v1 -p '{"spec": {"template": {"spec": {"containers":[{
      "name":"registry",
      "livenessProbe":  {"httpGet": {"scheme":"HTTPS"}}
    }]}}}}'
  
  
  oc patch dc/docker-registry --api-version=v1 -p '{"spec": {"template": {"spec": {"containers":[{
      "name":"registry",
      "readinessProbe":  {"httpGet": {"scheme":"HTTPS"}}
    }]}}}}'
 ```
 
### 生成route。注意域名需要解析到router上
 ```
  oc create route passthrough    \
      --service=docker-registry    \
      --hostname=registry.oc.downtown8.cn
  ``` 
### 将生成的证书copy到各个node上
 ```
  ansible nodes -m shell -a 'mkdir -p /etc/docker/certs.d/docker-registry.default.svc.cluster.local:5000'
  ansible nodes -m shell -a 'mkdir -p /etc/docker/certs.d/registry.oc.downtown8.cn'
ansible nodes -m shell -a 'mkdir -p /etc/docker/certs.d/registry.oc.downtown8.cn:443'


 ansible nodes -m copy -a 'src=/etc/origin/master/ca.crt dest=/etc/docker/certs.d/docker-registry.default.svc.cluster.local:5000/ca.crt'
 
  ansible nodes -m copy -a 'src=/etc/origin/master/ca.crt dest=/etc/docker/certs.d/registry.oc.downtown8.cn/ca.crt'
  ansible nodes -m copy -a 'src=/etc/origin/master/ca.crt dest=/etc/docker/certs.d/registry.oc.downtown8.cn:443/ca.crt'
  ``` 
## 重启各个node节点的docker进程
## 登录测试