# 全民HTTPS：免费的服务端证书与客户端证书

在开发测试过程中，我们经常要对客户端与服务器端的HTTP通讯加密，也就是HTTPS。而HTTPS往往依赖于证书，能公网使用的证书往往都是收费的，且费用不低。国外一家公司[Let's Encrypt](https://letsencrypt.org/getting-started/)为我们提供了另一种选择，免费的服务端证书，并且可以通过各种浏览器、安卓与IOS的验证。

但是Let's Encrypt的免费证书有效期只有90天，本文将介绍如何（通过[certbot](https://certbot.eff.org/lets-encrypt/centosrhel7-other)自动刷新有效期）。

在需要双向HTTPS校验的场景下，我们往往还需要客户端证书。客户端证书我会另外介绍openssl自签发证书，因为客户端证书实际是由我们自己的服务器端去校验，所以自签发证书即可解决大部分问题。

---

  - [获取免费的Let's Encrypt服务器证书并自动续期](#%E8%8E%B7%E5%8F%96%E5%85%8D%E8%B4%B9%E7%9A%84lets-encrypt%E6%9C%8D%E5%8A%A1%E5%99%A8%E8%AF%81%E4%B9%A6%E5%B9%B6%E8%87%AA%E5%8A%A8%E7%BB%AD%E6%9C%9F)
    - [获取证书](#%E8%8E%B7%E5%8F%96%E8%AF%81%E4%B9%A6)
    - [自动更新证书](#%E8%87%AA%E5%8A%A8%E6%9B%B4%E6%96%B0%E8%AF%81%E4%B9%A6)
  - [如何采用openssl签发客户端的证书](#%e5%a6%82%e4%bd%95%e9%87%87%e7%94%a8openssl%e7%ad%be%e5%8f%91%e5%ae%a2%e6%88%b7%e7%ab%af%e7%9a%84%e8%af%81%e4%b9%a6)
    - [签发CA证书](#%E7%AD%BE%E5%8F%91ca%E8%AF%81%E4%B9%A6)
    - [签发Server证书](#%E7%AD%BE%E5%8F%91server%E8%AF%81%E4%B9%A6)
    - [签发客户端证书](#%E7%AD%BE%E5%8F%91%E5%AE%A2%E6%88%B7%E7%AB%AF%E8%AF%81%E4%B9%A6)
    - [附录CA环境的基本配置：](#%E9%99%84%E5%BD%95ca%E7%8E%AF%E5%A2%83%E7%9A%84%E5%9F%BA%E6%9C%AC%E9%85%8D%E7%BD%AE)

---

## 获取免费的Let's Encrypt服务器证书并自动续期

服务器端证书采用Let's Encrypt的90天免费证书，为方便部署与自动更新，所有配置均采用certbot去自动完成。

### 获取证书

各开发测试环境的SSL端口可以由nginx模拟，需在80端口上配置Let's Encrypt自动获取/更新证书的验证路径，nginx示例如下：

```ini  
#Let's Encrypt自动获取/更新证书的验证路径
location /.well-known/acme-challenge/ {	
    alias    html/.well-known/acme-challenge/;
    autoindex off;
}
```

nginx的80端口服务器添加以上配置后，才可以在申请证书时完成域名的验证（需外网能访问到该nginx）。nginx配置完成后需安装certbot，在CentOS上安装certbot需EPEL库的支持。若没有启用EPEL，请先安装。安装命令：`yum install epel-release`

EPEL启用后即可安装Certbot，安装命令：`yum install certbot`

为方便获取证书并完成自动验证，我们使用Certbot的webroot插件去获取证书，以DEV环境为例，执行以下命令：

```bash
certbot certonly --webroot -w /usr/local/nginx/html -d aio.dev.xnph66.com
```

参数说明：

- `-certonly`：表示使用certbot来获取证书；
- `--webroot`：表示使用该插件，将域名验证的txt文件直接放入到指定的web服务器根路径下；
- `-w /usr/local/nginx/html`: 指定nginx的web根路径，该路径即前文提到的nginx的配置路径；
- `-d aio.dev.xnph66.com`：指定要获取的证书域名，支持多域名，如果有多个，则直接带上多个该参数即可；

顺利完成证书获取后，会输出证书与私钥的路径，部分输出如下：
```shell
IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at:
   /etc/letsencrypt/live/aio.dev.xnph66.com/fullchain.pem
   Your key file has been saved at:
   /etc/letsencrypt/live/aio.dev.xnph66.com/privkey.pem
   Your cert will expire on 2018-10-21. To obtain a new or tweaked
   version of this certificate in the future, simply run certbot
   again. To non-interactively renew *all* of your certificates, run
   "certbot renew"
```

得到证书与私钥后，将证书和私钥的路径配置到nginx的443端口的服务器配置中，然后重启nginx后，可在浏览器中验证证书是否生效。

### 自动更新证书

Certbot在获取到证书后，支持在证书过期前自动更新证书。使用命令`certbot renew --dry-run`测试自动更新，如果命令输出正常，则可以将自动更新命令`certbot renew`加入到`crontab`计划任务中。参考示例如下（每天12点执行一次更新检查）：

```sh
$ sudo crontab -e
# m h dom mon dow command
0 12 * * * /usr/bin/certbot renew --quiet
```

---

## 如何采用openssl签发客户端的证书

平时我们通过各类证书颁发机构申请的各种证书，本质上是单向证书，其作用是让客户端能够验证服务器，同时加密HTTP通道。那如何让服务器也能验证客户端，确保客户端是自己的客户，而非来自黑客的攻击呢？

我们可以使用双向HTTPS加密。首先服务器必须是自己的，在需要使用双向HTTPS加密时，可以使用openssl签发客户端证书，自己的服务器充当CA。
完整过程可以参考[OpenSSL Certificate Authority](https://jamielinux.com/docs/openssl-certificate-authority/introduction.html)

### 签发CA证书

在各开发测试环境的nginx服务器上以创建了完整的CA签发环境，如想了解，可以参考[Create the root pair](https://jamielinux.com/docs/openssl-certificate-authority/create-the-root-pair.html)

CA环境的私钥密码为brian，在签发证书时需要输入。

### 签发Server证书

由于postman对自签发的服务器证书支持相当不友好，为方便使用postman测试API，服务器证书不再使用openssl签发，而是由Let's Encrypt获取。

附Server端证书签名命令：

```bash
#生成秘钥和证书请求文件
openssl req -config openssl.cnf -newkey rsa:2048 -nodes -sha256 -keyout private/test.xnph66.com.key -out csr/test.xnph66.com.csr
#签发证书文件
openssl ca -config openssl.cnf -extensions server_cert -days 375 -notext -md sha256 -in csr/test.xnph66.com.csr -out certs/test.xnph66.com.crt
```

### 签发客户端证书

首先root用户登陆CA服务器（需准备一台Linux服务器做CA），进入CA目录`cd /root/ca`，使用如下命令获取客户端证书私钥和证书请求文件：

```shell
openssl req -config openssl.cnf -newkey rsa:2048 -nodes -sha256 -keyout private/api-test-client.key -out csr/api-test-client.csr
```

该命令会输出私钥api-test-client.key以及证书申请文件api-test-client.csr，并将输出文件放入到对应目录。

接下来使用CA签发该客户端证书，命令如下：

```shell
openssl ca -config openssl.cnf -extensions user_cert -days 300 -notext -md sha256 -in csr/api-test-client.csr -out certs/api-test-client.crt
```

签发完成后，证书会放置在certs目录下。将certs目录下的`api-test-client.crt`和private目录下的`api-test-client.key`拷贝出来，提供给开发测试使用即可。

如需将PEM格式的证书和私钥转换层PFX/P12格式，可以使用如下命令转换：

```shell
openssl pkcs12 -export -out certs/api-test-client.pfx -inkey private/api-test-client.key -in certs/api-test-client.crt -certfile certs/ca.cert.pem
```

### 附录CA环境的基本配置：

```ini
# OpenSSL root CA configuration file.
# Copy to `/root/ca/openssl.cnf`.

[ ca ]
# `man ca`
default_ca = CA_default

[ CA_default ]
# Directory and file locations.
dir               = /root/ca
certs             = $dir/certs
crl_dir           = $dir/crl
new_certs_dir     = $dir/newcerts
database          = $dir/index.txt
serial            = $dir/serial
RANDFILE          = $dir/private/.rand

# The root key and root certificate.
private_key       = $dir/private/ca.key.pem
certificate       = $dir/certs/ca.cert.pem

# For certificate revocation lists.
crlnumber         = $dir/crlnumber
crl               = $dir/crl/ca.crl.pem
crl_extensions    = crl_ext
default_crl_days  = 30

# SHA-1 is deprecated, so use SHA-2 instead.
default_md        = sha256

name_opt          = ca_default
cert_opt          = ca_default
default_days      = 730
preserve          = no
policy            = policy_strict

[ policy_strict ]
# The root CA should only sign intermediate certificates that match.
# See the POLICY FORMAT section of `man ca`.
countryName             = match
stateOrProvinceName     = match
organizationName        = match
organizationalUnitName  = optional
commonName              = supplied
emailAddress            = optional
subjectAltName         = optional

[ policy_loose ]
# Allow the intermediate CA to sign a more diverse range of certificates.
# See the POLICY FORMAT section of the `ca` man page.
countryName             = optional
stateOrProvinceName     = optional
localityName            = optional
organizationName        = optional
organizationalUnitName  = optional
commonName              = supplied
emailAddress            = optional
subjectAltName         = optional

[ req ]
# Options for the `req` tool (`man req`).
default_bits        = 2048
distinguished_name  = req_distinguished_name
string_mask         = utf8only

# SHA-1 is deprecated, so use SHA-2 instead.
default_md          = sha256

# Extension to add when the -x509 option is used.
x509_extensions     = v3_ca

[ req_distinguished_name ]
# See <https://en.wikipedia.org/wiki/Certificate_signing_request>.
countryName                     = Country Name (2 letter code)
stateOrProvinceName             = State or Province Name
localityName                    = Locality Name
0.organizationName              = Organization Name
organizationalUnitName          = Organizational Unit Name
commonName                      = Common Name
emailAddress                    = Email Address

# Optionally, specify some defaults.
countryName_default             = CN
stateOrProvinceName_default     = Guang Dong
localityName_default            = Shen Zhen
0.organizationName_default      = Shenzhen XXX Co., Ltd
#organizationalUnitName_default  = Information Technology Department
#emailAddress_default            =

[ v3_ca ]
# Extensions for a typical CA (`man x509v3_config`).
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints = critical, CA:true
keyUsage = critical, digitalSignature, cRLSign, keyCertSign

[ v3_intermediate_ca ]
# Extensions for a typical intermediate CA (`man x509v3_config`).
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints = critical, CA:true, pathlen:0
keyUsage = critical, digitalSignature, cRLSign, keyCertSign

[ user_cert ]
# Extensions for client certificates (`man x509v3_config`).
basicConstraints = CA:FALSE
nsCertType = client, email
nsComment = "OpenSSL Generated Client Certificate"
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer
keyUsage = critical, nonRepudiation, digitalSignature, keyEncipherment
extendedKeyUsage = clientAuth, emailProtection

[ server_cert ]
# Extensions for server certificates (`man x509v3_config`).
basicConstraints = CA:FALSE
nsCertType = server
nsComment = "OpenSSL Generated Server Certificate"
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer:always
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth

subjectAltName = @alt_names

[alt_names]
# DO REMEMBER TO Modify before CA sign=============
DNS.1       = test.xnph66.com
#DNS.2       = aio.sit.xnph66.com
#DNS.3       = aio.uat.xnph66.com

[ crl_ext ]
# Extension for CRLs (`man x509v3_config`).
authorityKeyIdentifier=keyid:always

[ ocsp ]
# Extension for OCSP signing certificates (`man ocsp`).
basicConstraints = CA:FALSE
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer
keyUsage = critical, digitalSignature
extendedKeyUsage = critical, OCSPSigning
```
