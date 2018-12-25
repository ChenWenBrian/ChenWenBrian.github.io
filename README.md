## 最新文章

---

- 2018-09-11   [如何使用nginx为influxdb提供API网关服务](blog/influxdb.md)  

    > InfluxDB是一个开源的时序数据库，使用GO语言开发，特别适合用于处理和分析资源监控数据这种时序相关数据。而InfluxDB自带的各种特殊函数如求标准差，随机取样数据，统计数据变化比等，使数据统计和实时分析变得十分方便。关于influxdb的基础知识本文就不做详细介绍了，这里主要是针对influxdb在开发中的痛点提供一个API的转换渠道。
    > Influxdb本身自带有HTTP接口，但是语法可能比较冷门，使用的是[Line Protocol](https://docs.influxdata.com/influxdb/v1.6/write_protocols/line_protocol_tutorial/#special-characters-and-keywords)。这种协议在实际开发中对开发人员可能相当不友好，一个不小心就会出现数据或字段的转义错误。本文提供一种基于JSON格式的语法转换渠道，基于nginx的njs模块提供JSON到Line Protocol的转换。


- 2018-08-20   [认识Hystrix](blog/hystrix.md)  

    > 在一个分布式系统里，许多依赖不可避免的会调用失败，比如超时、异常等，如何能够保证在一个依赖出问题的情况下，不会导致整体服务失败，这个就是Hystrix需要做的事情。Hystrix提供了熔断、隔离、Fallback、cache、监控等功能，能够在一个、或多个依赖同时出现问题时保证系统依然可用。

- 2018-06-03   [如何在CentOS 7.2上编译使用nginx 1.15.0](blog/nginx.md)  

    > nginx官方版本未提供CentOS 7.2的nginx 1.15.0，只有7.4以上版本才有官方版本，这里简单介绍下如何本地编译一份自己需要的nginx。

- 2018-04-10   [免费的服务端证书与客户端证书](blog/ssl-certs.md)  

    > 在开发测试过程中，我们经常要对客户端与服务器端的HTTP通讯加密，也就是HTTPS。而HTTPS往往依赖于证书，能公网使用的证书往往都是收费的，且费用不低。国外一家公司[Let's Encrypt](https://letsencrypt.org/getting-started/)为我们提供了另一种选择，免费的服务端证书，并且可以通过各种浏览器、安卓与IOS的验证。
    > 但是Let's Encrypt的免费证书有效期只有90天，本文将介绍如何（通过[certbot](https://certbot.eff.org/lets-encrypt/centosrhel7-other)自动刷新有效期）。
    > 在需要双向HTTPS校验的场景下，我们往往还需要客户端证书。客户端证书我会另外介绍openssl自签发证书，因为客户端证书实际是由我们自己的服务器端去校验，所以自签发证书即可解决大部分问题。
