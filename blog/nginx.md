# 如何在CentOS 7.2上使用nginx 1.15.0

SSL证书的签发与更新请参考[免费的服务端证书与客户端证书](ssl-certs.md)。

nginx官方版本未提供CentOS 7.2的nginx 1.15.0，只有7.4以上版本才有官方版本，这里简单介绍下如何本地编译一份自己需要的nginx。

  - [版本说明](#%E7%89%88%E6%9C%AC%E8%AF%B4%E6%98%8E)
  - [安装说明](#%E5%AE%89%E8%A3%85%E8%AF%B4%E6%98%8E)
    - [编译说明](#%E7%BC%96%E8%AF%91%E8%AF%B4%E6%98%8E)
    - [编译参数](#%E7%BC%96%E8%AF%91%E5%8F%82%E6%95%B0)
  - [如何配置logrotate日志自动归档](#%E5%A6%82%E4%BD%95%E9%85%8D%E7%BD%AElogrotate%E6%97%A5%E5%BF%97%E8%87%AA%E5%8A%A8%E5%BD%92%E6%A1%A3)

## 版本说明

| 名称         | 值                                 |
| ------------ | ---------------------------------- |
| 操作系统     | CentOS 7.2                         |
| 编译版本     | [nginx 1.15.0](nginx-1.15.0.zip)   |
| 附带插件     | nginScript                         |
| 依赖库       | pcre zlib openssl                  |
| 安装目录     | /usr/local/nginx                   |
| 日志目录     | /usr/local/nginx/logs/             |
| 启动命令     | `/usr/local/nginx/nginx`           |
| 停止命令     | `/usr/local/nginx/nginx -s stop`   |
| 动态加载配置 | `/usr/local/nginx/nginx -s reload` |

## 安装说明

CentOS上使用`wget`从Git上下载[nginx 1.15.0](nginx-1.15.0.zip)，然后通过unzip解压并移动到安装目录，使用`chmod 755 /usr/local/nginx/sbin/nginx`修改执行权限。

PS：如需增加模块或重新编译新版本，请参照编译说明。

### 编译说明

NGINX编译需依赖`pcre`库（8.32-15.el7）、`zlib`库（1.2.7-15.el7）、`openssl`库（1:1.0.1e-42.el7.9），请务必直接`yum install`上述三个库。NGINX部分插件如njs模块调用的时候会直接调用系统安装的依赖库，而不是编译时指定的库。另外，该编译版本添加了`nginScript`模块，需单独下载源码与nginx一起编译。

### 编译参数

```
--with-http_stub_status_module --with-http_gzip_static_module --with-http_realip_module --with-http_sub_module --with-http_ssl_module --with-stream --add-module=../njs/nginx
```


## 如何配置logrotate日志自动归档

Linux系统有提供logrotate可以自动归档日志，若需按天自动归档nginx的日志，可参考：

- 创建日志归档配置文件：`vim /etc/logrotate.d/nginx`

- 在配置文件内输入：

```vim
/usr/local/nginx/logs/*.log {
    daily
    rotate 14
    olddir archive
    missingok
    notifempty
    dateext
    copytruncate
    compress
    delaycompress
}
```

- 测试归档能否正常进行，查看是否报错并根据错误提示修复：`logrotate -f -d /etc/logrotate.d/nginx`