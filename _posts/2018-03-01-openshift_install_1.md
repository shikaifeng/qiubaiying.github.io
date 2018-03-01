---

layout: post
title: openshift集群安装和部署(一)
date: 2018-03-01
tags: [ openshift ]

---

# **openshift 集群安装部署（一）**

# 一、主机准备

## 1、申请主机

类型       | cpu |内存 |  主机名     |  IP地址|
----------|-----|-----|------------|---------|
master节点 |  4核| 16G | master-openshift.idc.yst.com.cn| 10.213.3.176 |
node节点-1 |  4核| 8G  | node01-openshift.idc.yst.com.cn| 10.213.3.177 |
node节点-2 |  4核| 8G  | node02-openshift.idc.yst.com.cn| 10.213.3.178 |
jenkins服务器 | 4核| 4G | jk-dc.idc.yst.com.cn | 10.213.3.184 |
nfs服务器 | 4核 | 4G | 10.213.3.176/177/178已挂载nfs服务器目录，挂载地址为：10.213.3.60:/data；挂载点为：/home/data | 10.213.3.60 |

>注，操作系统版本：CentOS 7.3 以上
>[官方系统要求参考文档](https://docs.openshift.org/3.6/install_config/install/prerequisites.html)

		[root@master ~]# cat /etc/redhat-release 
		CentOS Linux release 7.4.1708 (Core) 


## 2 、 配置主机名（每个节点都需要操作）
### master、node节点分别配置相应的主机名，以master为例

	hostnamectl set-hostname master-openshift.idc.yst.com.cn
	
hostname -f的结果要跟hostname一样
hostnamectl --transient、hostnamectl --static、 hostnamectl --pretty
三个命令得出的结果要一致，如果不一致，则要自己设置下

	hostnamectl set-hostname master-openshift.idc.yst.com.cn
	hostnamectl --pretty set-hostname master-openshift.idc.yst.com.cn
	hostnamectl --static set-hostname master-openshift.idc.yst.com.cn
	hostnamectl --transient set-hostname master-openshift.idc.yst.com.cn


>注，要为服务申请dns域名解析。提交变更单向运维申请域名


# 二、安装前预置	
## 1、激活网络（每个节点都需要操作）
   centos 的网络默认是没有激活，需要手动进行激活。如果网络未激活会导致后续安装失败。
### 1.1、打开NetworkManager
- 查看网络状态
```  bash
 [root@master ~]# systemctl show NetworkManager | grep ActiveState
 ActiveState=inactive
```
- 设置开机启用 NetworkManager：
```  bash
systemctl enable NetworkManager
```
- 立即启动 NetworkManager：
```  bash
systemctl start NetworkManager
```

### 1.2、激活网络
```  bash
[root@master-openshift ~]# nmcli con show
名称     UUID                                  类型            设备    
docker0  0aea51a3-613d-4468-ac3a-92cc10fd22a8  bridge          docker0 
ens160   ea74cf24-c2a2-ecee-3747-a2d76d46f93b  802-3-ethernet  ens160 
```

```  bash
nmcli con up ens160
nmcli con mod ens160 connection.autoconnect yes
systemctl restart NetworkManager
```




## 2、安装及配置软件包（每个节点都需要操作）
### 2.1 在所有节点上都要安装配置依赖的软件包
	yum install -y wget git net-tools bind-utils iptables-services bridge-utils bash-completion
	
安装完成后如图：
![](https://i.loli.net/2018/03/01/5a97973b57ce6.png)


### 2.2 在所有节点上都要docker	
	yum install -y docker
	
验证docker是否安装成功，如下：
![](https://ws4.sinaimg.cn/large/006tNc79gy1foxbvpxmcbj30jo094gnx.jpg)

注意：安装时遇到 Docker daemon未启动的，执行启动命令：

	service docker start
	
 Docker daemon未启动如图：
![](https://ws2.sinaimg.cn/large/006tNc79gy1foxbwdwtd0j30k4051ta9.jpg)


### 2.3 在所有节点上配置Docker 镜像服务器
选择中国科技大学的镜像服务器进行加速（也可以选择诸如daocloud的加速器）。修改/etc/sysconfig/docker文件，在OPTIONS变量中追加参数
```vim
--registry-mirror=https://docker.mirrors.ustc.edu.cn --insecure-registry=172.30.0.0/16
```  
配置文件如图：
![](https://ws4.sinaimg.cn/large/006tNc79gy1foxbwlz7mdj30yt0b278n.jpg)

设置开机启动并启动Docker服务
```  bash
systemctl enable docker
systemctl start docker
``` 



## 3、Master节点上相关配置

### 3.1 启用EPEL仓库以安装Ansible

	yum -y install https://dl.fedoraproject.org/pub/epel/7/x86_64/e/epel-release-7-10.noarch.rpm


确认 epel 仓库已经安装

![](https://ws3.sinaimg.cn/large/006tNc79gy1foxbwx2wlgj30xs074tb1.jpg)

执行：
						
	sed -i -e "s/^enabled=1/enabled=0/" /etc/yum.repos.d/epel.repo
	yum -y --enablerepo=epel install ansible pyOpenSSL

### 3.2 配置master节点和node节点之间的互信

 ansible 是基于agentles架构实现的，即不需要在远程的目标主机上预先安装agent程序。
ansible 对远程主机命令的执行依赖ssh等远程控制协议。因此将在master上执行ansible playbook 安装openshift，所以需要配置mater节点到哥哥node节点的互信，包括master到master的互信。

- master节点上生成ssh密钥：
``` bash								
	ssh-keygen -f /root/.ssh/id_rsa -N ''
``` 
执行结果如下：
![](https://ws1.sinaimg.cn/large/006tNc79gy1foxbx4f4y8j30lh0b0q5n.jpg)

- 执行脚本：
``` bash
for host in master-openshift.idc.yst.com.cn \
	node01-openshift.idc.yst.com.cn \
	node02-openshift.idc.yst.com.cn; \
do ssh-copy-id -i ~/.ssh/id_rsa.pub $host; \
done
``` 
执行过程：
![](https://ws3.sinaimg.cn/large/006tNc79gy1foxbxdwxabj30t80o7wke.jpg)

验证下是否添加成功（以node1为例）：
![](https://ws2.sinaimg.cn/large/006tNc79gy1foxbxm7ncij30xn052gow.jpg)

### 3.3 下载安装openshift 的ansible playbook

- 下载
``` bash
 wget https://github.com/openshift/openshift-ansible/archive/openshift-ansible-3.7.0-0.126.0.tar.gz
``` 
- 解压
``` bash
 tar zxvf openshift-ansible-3.7.0-0.126.0.tar.gz
``` 

### 3.4 单独安装etcd集群
安装单Master的Openshift集群可以不单独安装etcd。这里选择单独安装一个节点的etcd集群。
> 在实际的生产环境中，推荐配置含有3个或以上成员的etcd集群，保证高可用性。

		yum -y install etcd
		systemctl enable etcd # 使etcd自动启动
		systemctl start etcd


## 4、配置Ansible
- 备份原有的hosts文件	
												
				mv -f /etc/ansible/hosts /etc/ansible/hosts.org

- 配置hosts文件，文件在/etc/ansible/hosts
  参考[openshift 官方配置文档](https://docs.openshift.org/latest/install_config/install/advanced_install.html#adv-install-example-inventory-files)
- 配置文件如下：

``` bash
# Create an OSEv3 group that contains the masters and nodes groups
[OSEv3:children]
masters
nodes
etcd

# Set variables common for all OSEv3 hosts
[OSEv3:vars]
# SSH user, this user should allow ssh based auth without requiring a password
ansible_ssh_user=root
openshift_deployment_type=origin
openshift_release=3.6.0
openshift_disable_check=disk_availability,docker_storage,memory_availability,docker_image_availability

# uncomment the following to enable htpasswd authentication; defaults to DenyAllPasswordIdentityProvider
openshift_master_identity_providers=[{'name':'htpasswd_auth','login':'true','challenge':'true','kind':'HTPasswdPasswordIdentityProvider','filename':'/etc/origin/master/htpasswd'}]

# host group for masters
[masters]
master-openshift.idc.yst.com.cn

# host group for nodes, includes region info
[nodes]
master-openshift.idc.yst.com.cn
node01-openshift.idc.yst.com.cn
node02-openshift.idc.yst.com.cn
node01-openshift.idc.yst.com.cn openshift_node_labels="{'region': 'infra', 'zone': 'east'}"
node02-openshift.idc.yst.com.cn openshift_node_labels="{'region': 'infra', 'zone': 'west'}"

[etcd]
master-openshift.idc.yst.com.cn

``` 






