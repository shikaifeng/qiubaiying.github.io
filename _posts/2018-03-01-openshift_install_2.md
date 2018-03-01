---

layout: post
title: openshift集群安装和部署(二)
date: 2018-03-01
tags: [ openshift ]

---

# **openshift 集群安装部署（二）**

# 三、执行安装
## 1、在线安装
- 执行安装命令：

``` bash
ansible-playbook ~/openshift-ansible-openshift-ansible-3.7.0-0.126.0/playbooks/byo/config.yml
```

# 四、安装成功后的校验

## 1、安装完成后，常规状态校验
- 安装成功后，如图，Ansible会输出一个结果汇总信息，从汇总信息可以判断安装的执行结果。 
![](https://ws1.sinaimg.cn/large/006tNc79gy1foxdrsljf7j30m003d0td.jpg)

- 查看当前用户

		[root@master-openshift ~]# oc whoami
		system:admin

- 校验node节点是否正常工作

		oc get nodes -o wide
		oc version
- 查看节点信息：
![](https://ws3.sinaimg.cn/large/006tNc79gy1foxds9gfvij312s02omxx.jpg)

- 查看版本信息：

 ![](https://ws4.sinaimg.cn/large/006tNc79gy1foxdskd2k3j30dt0410t6.jpg)

- 查看资源列表

	oc get all -o wide
	
各个pod都启动正常，如果有异常的pod，可通过oc describe 命令查看状态，或者通过logs命令查看日志。下图为启动成功的状态：
![](https://ws4.sinaimg.cn/large/006tNc79gy1foxdtqspjij31h20ep782.jpg)

- 网页登陆

>注意，必须要用https。地址：https://master-openshift.idc.yst.com.cn:8443

[openshift master web console 登陆地址](https://master-openshift.idc.yst.com.cn:8443)

![](https://ws1.sinaimg.cn/large/006tNc79gy1foxdum9qv8j310a0i0q55.jpg)

## 2、组件校验
### 2.1、默认安装的 Image Stream（默认安装社区提供的is）

	oc get is -n openshift

![](https://ws4.sinaimg.cn/large/006tNc79gy1foxdvht69fj30sd07ogny.jpg)

	oc get istag -n openshift

![](https://ws4.sinaimg.cn/large/006tNc79gy1foxdvru17oj31h00p1amn.jpg)

### 2.2、默认导入的 Template（默认安装社区提供的模版）

	oc get template -n openshift
		
![](https://ws2.sinaimg.cn/large/006tNc79gy1foxdw21ujmj311j0gmdm1.jpg)

### 2.3、部署Router (3.6版本的Openshift Ansible自动安装)
	oc get pod -n default | grep "router"
	
![](https://ws3.sinaimg.cn/large/006tNc79gy1foxdwbk102j30hv01pt8t.jpg)

### 2.4、部署Registry (3.6版本的Openshift Ansible自动安装)   

![](https://ws2.sinaimg.cn/large/006tNc79gy1foxdwjw671j30i601taa9.jpg)

> 后续要为registry组件配置持久化的后端




# 五、安装过程中的问题
## 问题1: 网络未激活，安装报错
我在安装过程中就遇到下面的一个问题，这个网络的问题应该在准备阶段就做好。参考之前的网络激活步骤。


安装过程中的问题：
![](https://ws4.sinaimg.cn/large/006tNc79gy1foxdxhofr5j31h906l0ut.jpg)

### 1） NetworkManager 没打开，所有节点都要执行

- 确认是否为network 没打开的问题

```bash
[root@master ~]# systemctl show NetworkManager | grep ActiveState
ActiveState=inactive
```

- 设置开机启用 NetworkManager：

		systemctl enable NetworkManager
- 立即启动 NetworkManager：

		systemctl start NetworkManager

- 再次查看network的状态：
```
[root@master-openshift ~]# systemctl show NetworkManager | grep ActiveState
ActiveState=active
```

###  2）激活网络
```  bash
[root@master-openshift ~]# nmcli con show
名称     UUID                                  类型            设备    
docker0  0aea51a3-613d-4468-ac3a-92cc10fd22a8  bridge          docker0 
ens160   ea74cf24-c2a2-ecee-3747-a2d76d46f93b  802-3-ethernet  ens160 
```
```bash
nmcli con up ens160
```
```bash
nmcli con mod ens160 connection.autoconnect yes
```
```bash
systemctl restart NetworkManager
```

## 问题2:安装完成后，相关pod启动不正常

查看po的状态：
![](https://ws3.sinaimg.cn/large/006tNc79gy1foxdy52vnlj31h00fmgpr.jpg)

- 我这里的解决办法是删除报错的pod 或者 rc，会重新自动部署。
- 具体问题需要通过describe 查看pod的具体问题，然后进行排查。
- 也有可能是image由于网络原因没有下载下来



# 五 、安装后配置
## 1、账号问题
安装时，我们在ansible的hosts文件中定义了htpasswd文件作为后端的用户身份信息库。

- ansible 的hosts 文件配置信息如下：

```bash
# uncomment the following to enable htpasswd authentication; defaults to DenyAllPasswordIdentityProvider
openshift_master_identity_providers=[{'name':'htpasswd_auth','login':'true','challenge':'true','kind':'HTPasswdPasswordIdentityProvider','filename':'/etc/origin/master/htpasswd'}]

```

- 通过htpasswd 命令来创建dev 账户：

``` bash
[root@master-openshift master]# htpasswd -b /etc/origin/master/htpasswd dev dev
Adding password for user dev

[root@master-openshift master]# cat htpasswd 
dev:$apr1$LTowVr7v$.FZAM4OySihVKzR.0An1X.
```


## 2、账号权限问题

部署镜像时，使用openshift启动镜像时，有时会遇到权限不足的问题，导致容器无法启动
这时简单的方法就是给default（默认账号）赋予较高的权限，使得容器能够正常运行

``` bash

oadm policy add-scc-to-user anyuid -z default

```