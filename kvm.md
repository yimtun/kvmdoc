



[TOC]











# 一 课程介绍



kvm  虚拟化 实用技能





### 对于开发人员  学习kvm虚拟化有什么用处？



##### 1 

如果对需要运行的应用不是很了解，首次运行最好在虚拟机而不是docker容器中运行，这会为学习某种软件减少很多不必要的麻烦，那么此时我们最好有一个干净的操作系统可以让我们运行测试应用程序，待调整好之后再容器化运行。



##### 2

 开发人员在测试使用 Hadoop  MongoDB  Zookeeper  等分布式服务时，不想基于伪分布式架构进行开发或者学习，此时就有了对虚拟化软件的需求。



##### 3



技术更新快，技术人员总会不断充电学习，平时在公司申请计算资源也有或多或少的限制，相比之下在自己可以

掌控的环境里进行虚拟化的部署以供学习更值得推荐





### 虚拟化软件有很多 为什么学kvm ？



诚然  有许多供开发者 和 企业生成环境 使用的虚拟化软件  但是他们又有各自不适合开发者使用的问题



#####  1

除kvm之外的企业IDC使用的虚拟化软件如xen   esxi  等软件不适合个人开发者使用 架构复杂  不易使用

这些软件大多需要专业的运维人员维护，开发者一般也接触不到这块



##### 2

个人虚拟化软件如 VMware workstation    virtualbox 等软件图像界面简单易用 但是不适合快速创建或销毁多个虚拟机的使用需求





##### 3

公有云的使用一般也是基于图形管理，调用api创建又多了学习成本，且有一定费用。虽然可以按分钟等计费

但也还是需要一定的开销



###  本次kvm 课程的特点

kvm 是成熟的虚拟化技术，就连亚马逊的云计算 最近几年也在使用 kvm 做为其底层技术的基础

也是openstack 等私有云的底层支撑。但是对于开发人员的上述需求点不能够很好的满足



本课程的内容可以让kvm虚拟化技术更易于开发人员使用，不再局限于企业IDC环境的使用场景。



通过多年对kvm使用的经验技能的总结，整合多种kvm虚拟化相关技术在一个脚本里。

达到新建崭新虚拟机操作系统速度不会比docker 容器慢很多，但是得到的是一个完整的操作系统。







环境准备



本文档为markdown 格式 建议在下面网站下载typora  进行 markdown 文档的查看

> https://www.typora.io/







 下载iso  版本 CentOS7.8



> http://mirrors.aliyun.com/centos/7.8.2003/isos/x86_64/CentOS-7-x86_64-DVD-2003.iso



# 二 VMware初始化网络





说明：

运行vmware  的电脑当前 IP 不属于 10.200.100.0/24  网段 可以 进行如下设置

```
1编辑 2虚拟网络编辑器 3 还原默认设置 (个别情况需要先点击更改设置)
```



```
1 编辑 2虚拟网络编辑器 3更改设置 4选择VMnet8  5 子网处填写(子网IP:10.200.100.0子网掩码:255.255.255.0)
```


```
vmnet8   nat 设置 网关 10.200.100.250
```





# 三 kvm宿主机操作系统安装



安装  kvm  宿主机操作系统





vmware 虚拟机cpu 设置

```
处理器 (虚拟化Intel VT-x/EPT 或 AMD-V、RVI，虚拟化CPU性能计数器)
```



```
Install CentOS7 TAB

quiet 后面 输入 net.ifnames=0 biosdevname=0

Enter
```



vmware 登录虚拟机 查看ip



 

xshell  登录 查看到的ip





 初始化kvm宿主机系统



```
sed -i 's/enforcing/disabled/g'  /etc/selinux/config
setenforce 0
sed -i 's/#UseDNS yes/UseDNS no/g'   /etc/ssh/sshd_config
systemctl   restart sshd
systemctl  disable firewalld NetworkManager
systemctl  stop  firewalld  NetworkManager
```



```
echo 'LANG="en_US.UTF-8"'  > /etc/locale.conf
export LANG="en_US.UTF-8"
timedatectl set-timezone Asia/Shanghai
```



```
mv /etc/yum.repos.d/CentOS-Base.repo      /etc/yum.repos.d/CentOS-Base.repo.backup
curl -o /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
curl  -o /etc/yum.repos.d/epel.repo       http://mirrors.aliyun.com/repo/epel-7.repo
```



设置ip

```
cat > /etc/sysconfig/network-scripts/ifcfg-eth0   << EOF 
TYPE=Ethernet
BOOTPROTO=none
NAME=eth0
DEVICE=eth0
ONBOOT=yes
IPADDR=10.200.100.11
NETMASK=255.255.255.0
GATEWAY=10.200.100.250
DNS1=223.5.5.5
EOF
```



```
systemctl  restart network
```





使用xhell 连接 设置的固定ip 



```
ssh  10.200.100.11
poweroff
```



```
关闭虚拟机 在vmware  创建克隆 
创建完整克隆  克隆名字  kvm1-bak
```





# 四 libvirt 安装



```
ssh  10.200.100.11
```



## 创建网桥 名字 br0



```
yum -y install bridge-utils
```



```
cat >  /etc/sysconfig/network-scripts/ifcfg-br0  << EOF
DEVICE=br0
ONBOOT=yes
BOOTPROTO=static
NM_CONTROLLED=no
IPADDR=10.200.100.11
NETMASK=255.255.255.0
GATEWAY=10.200.100.250
DNS1=223.5.5.5
USERCTL=no
TYPE=Bridge
EOF
```





```
cat >  /etc/sysconfig/network-scripts/ifcfg-eth0  << EOF
DEVICE=eth0
ONBOOT=yes
BRIDGE=br0
EOF
```



```
systemctl  restart network
```



##   安装软件包



```
egrep '(vmx|svm)' /proc/cpuinfo
```



```
yum -y install libvirt
systemctl   enable --now libvirtd
```





## 创建kvm 网络

```
echo "<network><name>br0</name><uuid>`uuidgen`</uuid><forward mode='bridge'/><bridge name='br0'/></network>"  > /etc/libvirt/qemu/networks/br0.xml
```



```
virsh net-define    /etc/libvirt/qemu/networks/br0.xml
virsh net-start   br0
virsh  net-autostart  br0
```



## 创建kvm 存储池



```
mkdir -p /kvm/image
virsh pool-define-as  kvmimage  dir --target  "/kvm/image"
virsh pool-start  kvmimage
virsh  pool-autostart   kvmimage
```







# 五 安装 qemu



```
yum -y install qemu-kvm
```



# 六 获取模板镜像和脚本等资源



获取虚拟磁盘模板



```
yum -y install docker
```



```
cat > /etc/docker/daemon.json  << EOF
{
  "registry-mirrors": ["https://2xdz2l32.mirror.aliyuncs.com"]
}
EOF
```



```
systemctl   start docker
```



```
docker pull docker.io/yimtune/kvmfile:v1
```



```
docker run -d -p 80:80 --name kvm  docker.io/yimtune/kvmfile:v1
```



```
yum -y install wget
```



```
wget  http://127.0.0.1/CentOS7.6.1810.x86_64.qcow2   -P  /kvm/image/
```


```
virsh  vol-info CentOS7.6.1810.x86_64.qcow2  --pool kvmimage 
```


```
systemctl  restart libvirtd
```



```
virsh  vol-info CentOS7.6.1810.x86_64.qcow2  --pool kvmimage 
```



`````sdfg
Name:           CentOS7.6.1810.x86_64.qcow2
Type:           file
Capacity:       3.00 GiB
Allocation:     641.88 MiB
`````

```
rm -rf /root/create_vm.sh*
wget  http://127.0.0.1/create_vm.sh   -P /root/
wget  http://127.0.0.1/template.xml  -P /etc/libvirt/
```



```
chmod +x /root/create_vm.sh
```



```
qemu-img info /kvm/image/CentOS7.6.1810.x86_64.qcow2 
```



```
image: /kvm/image/CentOS7.6.1810.x86_64.qcow2
file format: qcow2
virtual size: 3.0G (3221225472 bytes)
disk size: 642M
cluster_size: 65536
Format specific information:
    compat: 1.1
    lazy refcounts: false
```



清理docker 环境

```
docker  stop  kvm
docker  rm  kvm
systemctl  stop docker
yum -y remove docker
```



```
systemctl  restart network
```





# 七 使用镜像模板快速生成虚拟机



```
yum -y install libguestfs-tools-c
```



```
/root/create_vm.sh
```



# 八 删除虚拟机


查看当前正在运行的虚拟机

```
virsh  list
```

强制关机


```
virsh  destroy 10.200.100.101
```



```
virsh  list --all
```







查看虚拟机的虚拟磁盘

```
virsh  domblklist 10.200.100.101
```



根据虚拟磁盘的路径 找到 虚拟磁盘所在的pool 



```
virsh  vol-pool /kvm/image/10.200.100.101.qcow2
```



查看pool  的名字为 kvmimage   中的 vol

```
virsh  vol-list --pool kvmimage
```



根据 pool 和 vol  删除 虚拟机的虚拟磁盘



```
virsh  vol-delete 10.200.100.101.qcow2  --pool kvmimage
```

删除虚拟机

```
virsh  undefine 10.200.100.101
```







```
virsh  list --all
```























































































