# CentOS 7 开启Google BBR

## BBR介绍

Google BBR (Bottleneck Bandwidth and RTT) 是一种新的TCP拥塞控制算法,它可以高效增加吞吐和降低网络延迟，并且Linux Kernel4.9+已经集成该算法。开启BBR也非常简单，因为它只需要在发送端开启，网络其他节点和接收端不需要任何改变。

## 升级内核

### 1. 查看当然内核版本

打开Terminal，输入`uname -r`查看：

```shell
# uname -r
3.10.0-957.21.2.el7.x86_64
```

我们看到输出的内核版本小于4.9，则表示内核需要升级。如果大于4.9，则跳过升级，可以直接开启BBR。

### 2. 升级内核

**安装 ELRepo 仓库**

```shell
# rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
# yum install https://www.elrepo.org/elrepo-release-7.0-3.el7.elrepo.noarch.rpm
```

这里以centos 7为例，如果你不清楚最新的版本RPM，可以打开[http://elrepo.org](http://elrepo.org/tiki/tiki-index.php)查看对应系统的最新版本。

**安装最新版kernel**

```shell
# yum --enablerepo=elrepo-kernel install kernel-ml -y
```

**确认是否安装成功**

```shell
# rpm -qa | grep kernel
```

如果输出类似如下，包含kernel-ml-5.1.9-1.el7.elrepo.x86_64，则表示安装成功
>kernel-ml-5.1.9-1.el7.elrepo.x86_64           
>kernel-3.10.0-957.21.2.el7.x86_64                
>kernel-3.10.0-862.11.6.el7.x86_64                
>abrt-addon-kerneloops-2.1.11-52.el7.centos.x86_64              
>kernel-tools-3.10.0-957.21.2.el7.x86_64             
>kernel-tools-libs-3.10.0-957.21.2.el7.x86_64              

**设置开机默认启动项**

查询可选的内核：

```shell
# egrep ^menuentry /etc/grub2.cfg | cut -f 2 -d \'
```

输出结果类似如下：

>CentOS Linux (5.1.9-1.el7.elrepo.x86_64) 7 (Core)               
>CentOS Linux (3.10.0-957.21.2.el7.x86_64) 7 (Core)               
>CentOS Linux (3.10.0-862.11.6.el7.x86_64) 7 (Core)               

该列表从0开始索引，所以5.19内核索引为0，我们将该索引值设为启动项，并重启系统

```shell
# grub2-set-default 0
# reboot
```

重启完成后再执行`uname -r`检查下当前系统内核，如果输出如下则表示内核更新成功。

>5.1.9-1.el7.elrepo.x86_64


**删除旧内核**

说明：删除旧内核的目的是为了防止 yum 更新旧版内核之后覆盖了 grub 默认启动项

```shell
# yum -y remove kernel kernel-tools
```


## 开启BBR

### 1. 修改sysctl配置

```shell
# echo 'net.core.default_qdisc=fq' | tee -a /etc/sysctl.conf
# echo 'net.ipv4.tcp_congestion_control=bbr' |  tee -a /etc/sysctl.conf
# sysctl -p
```

以上变更，亦可以直接通过`vim /etc/sysctl.conf`命令打开配置文件，并手工追加以下两行内容：
```conf
net.core.default_qdisc=fq
net.ipv4.tcp_congestion_control=bbr
```
然后输入`:wq`保存并退出，然后输入`sysctl -p`应用生效

### 2. 检查是否加载BBR

```shell
# lsmod | grep bbr
```

如果输出结果包含tcp_bbr，则表示开启成功

>tcp_bbr 20480 3