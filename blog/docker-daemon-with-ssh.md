# 如何通过SSH访问Docker Daemon？

从Dockers 18.09版开始，docker开始支持ssh远程访问docker daemon了。有了ssh，我们远程管理docker daemon时再也不必为开启了tcp端口而造成的可能安全隐患而烦恼了。更nice的是，有了ssh，我们就可以不必本地安装docker for desktop了，docker CE也不用跑在虚拟机环境了，本地电脑有限的资源也不用为开启docker虚拟机而烦恼了；云主机的网络速度一般来说都是相当可观的，所以无论是下载镜像还是上传镜像，都会如丝般顺滑；而且如果你有海外云主机的话，各种谷歌镜像对你也不再是问题，简直不要太爽。

## 在远程服务器上安装docker daemon
以centos 7为例，ssh登陆到服务器，并确保当前用户有root权限或者sudo权限。
```shell
#卸载所有旧版组件（可选）
$ sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine

#设置yum的仓库地址（推荐）
$ sudo yum install -y yum-utils \
  device-mapper-persistent-data \
  lvm2
$ sudo yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo

#安装docker CE
$ sudo yum install docker-ce docker-ce-cli containerd.io

#启动docker
$ sudo systemctl start docker
```

## docker daemon如何开启tcp

默认情况下，docker安装完毕后会开启unit socket进行本地通讯，我们可以查看docker的systemd配置，在`/usr/lib/systemd/system/`目录下。
```shell
$ sudo cat /usr/lib/systemd/system/docker.socket 
[Unit]
Description=Docker Socket for the API
PartOf=docker.service

[Socket]
ListenStream=/var/run/docker.sock
SocketMode=0660
SocketUser=root
SocketGroup=docker

[Install]
WantedBy=sockets.target
```

要开启TCP端口，我们也可以使用systemd的方式，新建一个`docker-tcp.socket`的文件如下：
```shell
$ sudo vi /usr/lib/systemd/system/docker.socket 
[Unit]
Description=Docker Socket for the API
PartOf=docker.service

[Socket]
ListenStream=/var/run/docker.sock
SocketMode=0660
SocketUser=root
SocketGroup=docker

[Install]
WantedBy=sockets.target
[wildfox@cw-proxy ~]$ cat /usr/lib/systemd/system/docker-tcp.socket 
[Unit]
Description=Docker TCP Socket for the API

[Socket]
ListenStream=2375
BindIPv6Only=both
Service=docker.service

[Install]
WantedBy=sockets.target
```
保存退出后，依次执行以下命令让systemd生效（PS:必要情况下可以尝试重启服务器）。
```sh
sudo systemctl daemon-reload

sudo systemctl stop docker.service
sudo systemctl enable docker-tcp.socket
sudo systemctl start docker-tcp.socket
sudo systemctl start docker.service
```
通过`sudo netstat -ntlp` 查看2375端口是否已开启，或者`docker -H 127.0.0.1 ps`查看返回结果是否正常，也可判断TCP 2375端口是否已成功开启。

## docker客户端与服务端的通讯方式
docker客户端与服务端通讯，通常使用unix socket /var/run/docker.sock或者TCP socket。以下为docker daemon的一个启动示例：
```shell
$ ps aux | grep dockerd
root 2900 0.1 4.4 388008 45424 ? Sl 09:28 0:01 /usr/local/bin/dockerd -g /var/lib/docker 
-H unix:// 
-H tcp://0.0.0.0:2376 
--label provider=virtualbox 
--tlsverify 
--tlscacert=/var/lib/boot2docker/ca.pem
--tlscert=/var/lib/boot2docker/server.pem
--tlskey=/var/lib/boot2docker/server-key.pem
--storage-driver aufs
```
这里有2个重要的参数：
- -H unix://, 表示本地unix socket /var/run/docker.sock. 本地访问时，Docker client使用这个unix的socket与daemon通讯；
- -H tcp://0.0.0.0:2376 使docker daemon通过2376端口，有了可以暴露在任何网络的能力。出于安全考虑，这个端口一般需要通过安全组打开，或者严格限制访问的IP。这个sample里使用了https认证。

而我们知道，在Linux环境里，ssh是被广泛使用且一般都会默认开放的端口，它支持多种证书访问方式，且可以给很多应用提供网络通讯的通道。而从docker 18.09开始，docker原生支持了ssh，这将为我们远程管理docker daemon带来极大的便利。

## 开启ssh远程访问docker daemon

首先可以本地生成一对RSA秘钥，并在本地ssh客户端上将私钥导入。注意Windows 10 1809以上版本系统有带openssh，需启动ssh-agent服务才能管理私钥。
```sh
$ ssh-add -k ~/.ssh/private_key
```
然后在远程服务器上将公钥与账号绑定。
账号绑定后务必将账号加入到docker的组里（root不需要），不然客户端ssh会出错，提示找不到http://docker什么的。
```sh
$ sudo usermod -aG docker your-account
```
完成后最好重启一次服务器

本地验证效果：
```sh
$ docker -H ssh://your-account@your-host-ip-or-domain ps
```