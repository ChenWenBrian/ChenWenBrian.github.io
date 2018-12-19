## 最新文章

---

- 2018-12-19   [如何在CentOS 7.2上编译使用nginx 1.15.0](blog/nginx.md)  

    > nginx官方版本未提供CentOS 7.2的nginx 1.15.0，只有7.4以上版本才有官方版本，这里简单介绍下如何本地编译一份自己需要的nginx。

- 2018-12-19   [免费的服务端证书与客户端证书](blog/ssl-certs.md)  

    > 在开发测试过程中，我们经常要对客户端与服务器端的HTTP通讯加密，也就是HTTPS。而HTTPS往往依赖于证书，能公网使用的证书往往都是收费的，且费用不低。国外一家公司[Let's Encrypt](https://letsencrypt.org/getting-started/)为我们提供了另一种选择，免费的服务端证书，并且可以通过各种浏览器、安卓与IOS的验证。
    > 但是Let's Encrypt的免费证书有效期只有90天，本文将介绍如何（通过[certbot](https://certbot.eff.org/lets-encrypt/centosrhel7-other)自动刷新有效期）。
    > 在需要双向HTTPS校验的场景下，我们往往还需要客户端证书。客户端证书我会另外介绍openssl自签发证书，因为客户端证书实际是由我们自己的服务器端去校验，所以自签发证书即可解决大部分问题。
