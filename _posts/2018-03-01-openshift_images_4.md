---

layout: post
title: openshift镜像管理--持久化(四)
date: 2018-03-01
tags: [ openshift ]

---

# 镜像仓库必须做持久化，否则当集群重启或者镜像仓库重启会导致推送进去的镜像丢失

# 镜像仓库持久化

## 1、通过nfs 的方式挂载持久化卷
 先申请了一台nfs的服务并设置好挂载点：10.213.3.60:/data
 

## 2、检查挂在点（这是挂载后的）

``` 
[root@master-openshift ~]# oc volumes dc/docker-registry --all
deploymentconfigs/docker-registry
  pvc/docker-registry-claim (allocated 80GiB) as registry-storage
    mounted at /registry
  secret/registry-certificates as volume-gtsfz
    mounted at /etc/secrets
```
### 已经挂载上了，查看挂载点的已使用容量

``` 
[root@master-openshift ~]# oc rsh docker-registry-3-20h6k 'du' '-sh' '/registry'
96M	/registry
```


## 3、创建持久化卷 pv images-pv.json

``` 
[root@master-openshift ~]# cat images-pv.json 
{
  "apiVersion": "v1",
  "kind": "PersistentVolume",
  "metadata": {
     "name": "images"
  },
  "spec": {
     "capacity": {
        "storage": "80Gi"
     },
     "accessModes": [ "ReadWriteOnce" ],
     "nfs": {
        "path": "/data",
        "server": "10.213.3.60"
     },
     "persistentVolumeReclaimPolicy": "Retain"
  }
}

```

``` 
[root@master-openshift ~]# oc create -f images-pv.json
persistentvolume "images" created
```

``` 
[root@master-openshift ~]# oc get pv
NAME      CAPACITY   ACCESSMODES   RECLAIMPOLICY   STATUS    CLAIM                           STORAGECLASS   REASON    AGE
images    80Gi       RWO           Retain          Bound     default/docker-registry-claim                            2m
```

## 4、创建持久化卷请求 images-pvc.json

``` 
[root@master-openshift ~]# cat images-pvc.json 
{
  "apiVersion": "v1",
  "kind": "PersistentVolumeClaim",
  "metadata": {
    "name": "docker-registry-claim"
  },
  "spec": {
    "accessModes": [  
      "ReadWriteOnce"    
    ],
    "resources": {
      "requests": {
        "storage": "80Gi"
      }
    }
  }
}
```
``` 
[root@master-openshift ~]# oc create -f images-pvc.json
persistentvolumeclaim "docker-registry-claim" created
```

``` 
[root@master-openshift ~]# oc get pvc
NAME                    STATUS    VOLUME    CAPACITY   ACCESSMODES   STORAGECLASS   AGE
docker-registry-claim   Bound     images    80Gi       RWO                          9s
```

### 关联持久化卷请求

```
[root@master-openshift ~]# oc volume dc/docker-registry --add --name=registry-storage -t pvc --claim-name=docker-registry-claim --overwrite
deploymentconfig "docker-registry" updated
```


## 5、过程中的问题

### 问题1:master链接不到node节点

``` 
[root@master-openshift ~]# oc rsh docker-registry-3-20h6k 'du' '-sh' '/registry'
Error from server: error dialing backend: dial tcp 10.213.3.178:10250: getsockopt: no route to host
```  
这个是因为 服务端口未开启导致的，解决办法执行下面iptable的命令：

```
 iptables -I INPUT -p tcp -m tcp --dport 10250 -j ACCEPT
```
[参考文章](https://ieevee.com/tech/2017/02/20/k8s-deploy.html)
