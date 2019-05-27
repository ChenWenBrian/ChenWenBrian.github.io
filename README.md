## 最新文章

---

- 2019-05-27   [漫谈MySQL的Ranking技术](blog/mysql-ranking.md)  

    > 在实际的项目中，我们经常会遇到需要对数据进行排名（非简单排序）的情况，譬如我们有一个学生各个科目的成绩表，要的到一个学生期末考试总成绩的排名表，怎么做呢？简单的按学生分组后计算总分然后再对总分排个序吗？那如何处理并列第一、并列第三的问题呢？这时候我们需要的是按总成绩排名，即ranking，而非order by。

- 2019-05-21   [如何通过JWT Token访问Kubernetes dashboard？](blog/k8s-dashboard-admin.md)  

    > 之前安装[Docker for windows](docker-for-windows.md)一文里提到如何安装免身份验证的dashboard，但是更多的场景是我们需要一个身份验证，那么如何安装一个带身份验证的dashboard，并让我们顺利登陆呢？

- 2019-05-14   [统一管理Windows环境下的OpenSSH](blog/openssh-on-windows.md)  

    > Windows 10从1709开始，默认在系统目录里带了OpenSSH客户端，这样对于开发人员就可以直接在cmd或者powershell上使用ssh命令。但是git-bash上也有一套相对独立的OpenSSH，两者的公钥私钥似乎不能统一管理，怎么办呢？

- 2019-05-13   [如何通过SSH访问Docker Daemon？](blog/docker-daemon-with-ssh.md)  

    > 从Docker 18.09版开始，docker开始支持ssh远程访问docker daemon了。有了ssh，我们远程管理docker daemon时再也不必为开启了tcp端口而可能造成安全隐患而烦恼了。更NICE的是，有了ssh，我们就可以不必本地安装docker for desktop了，docker CE也不用跑在虚拟机环境了，本地电脑有限的资源也不用为开启docker虚拟机而烦恼了；另外云主机的网络速度一般来说都是相当可观的，所以无论是下载镜像还是上传镜像，都会如丝般顺滑；而且如果你有海外云主机的话，各种谷歌镜像对你也不再是问题，简直不要太爽。

- 2019-04-22   [单网卡玩转智能网关——一键自动启停脚本](blog/openwrt-script.md)  

    > 接前文`<<单网卡玩转智能网关——把OpenWRT塞进虚拟机>>`，我们纯手工完成了全局软路由的配置，但是有时候我们并不希望一直开启软路由，于是又得手工改回各项配置；然而有时候又需要软路由了，又要改配置……改来改去的过程那个麻烦那个痛苦啊，还是弄个自动化配置脚本吧。

- 2019-04-18   [单网卡玩转智能网关——把OpenWRT塞进虚拟机](blog/openwrt-for-vm.md)  

    > 玩过智能路由器的同学可能都知道，国内比较火的koolshare论坛提供的梅林固件的那个强大：科学上网、游戏加速、自动签到、VPN服务器、FTP服务器、智能去广告、动态DNS、内网穿透、BT下载、私有云、KMS等众多扩展软件，让你的路由器从此变的强大无比。但是呢，梅林固件支持的路由器毕竟有限，我们不一定刚好拥有，另外如果我们想在工作场所一样拥有强大且不受限的网络访问能力怎么办呢？这里我们就可以考虑软路由了。

    > 软路由是指利用台式机或服务器配合软件形成路由解决方案，主要靠软件的设置，达成路由器的功能；而硬路由则是以特有的硬设备，包括处理器、电源供应、嵌入式软件，提供设定的路由器功能。常见的有海蜘蛛、OpenWRT、DDWRT、Tomato等，这些系统共有的特点是一般对硬件要求较低，甚至只需要一台486电脑，一张软盘，两块网卡就可以安装出一台非常专业的软件防火墙。

- 2019-03-28   [如何在Docker for windows上安装kubernetes？](blog/docker-for-windows.md)  

    > Dockers这些年是越来越火，连微软家的Windows也不干寂寞了，在Windows 10 pro以及Windows 2016以上的版本添加了对docker的官方支持。这下子Windows环境的开发人员一下子方便多了，再也不需要额外的Linux服务器或者虚拟机了（好吧，本质还是虚拟机hyper-v）。更NICE的是从Docker for Windows 18.02 EDGE开始，增加了kubernetes的支持，并且kubectl也被加入到本地path里并设置好了context（如果里用kubectl操作其他contexts，记得切换）。这下更酷了，容器编排以及本地集群部署问题统统都给你解决了。

- 2018-09-18   [Kafka中的消息是否会丢失和重复消费](blog/kafka.md)  

    > Kafka是由Apache软件基金会开发的一个开源流处理平台，由Scala和Java编写。Kafka是一种高吞吐量的分布式发布订阅消息系统，它可以处理消费者规模的网站中的所有动作流数据。 这种动作（网页浏览，搜索和其他用户的行动）是在现代网络上的许多社会功能的一个关键因素。 这些数据通常是由于吞吐量的要求而通过处理日志和日志聚合来解决。 对于像Hadoop的一样的日志数据和离线分析系统，但又要求实时处理的限制，这是一个可行的解决方案。Kafka的目的是通过Hadoop的并行加载机制来统一线上和离线的消息处理，也是为了通过集群来提供实时的消息。

- 2018-09-11   [如何使用nginx为influxdb提供API网关服务](blog/influxdb.md)  

    > InfluxDB是一个开源的时序数据库，使用GO语言开发，特别适合用于处理和分析资源监控数据这种时序相关数据。而InfluxDB自带的各种特殊函数如求标准差，随机取样数据，统计数据变化比等，使数据统计和实时分析变得十分方便。关于influxdb的基础知识本文就不做详细介绍了，这里主要是针对influxdb在开发中的痛点提供一个API的转换渠道。

    > Influxdb本身自带有HTTP接口，但是语法可能比较冷门，使用的是[Line Protocol](https://docs.influxdata.com/influxdb/v1.6/write_protocols/line_protocol_tutorial/#special-characters-and-keywords)。这种协议在实际开发中对开发人员可能相当不友好，一个不小心就会出现数据或字段的转义错误。本文提供一种基于JSON格式的语法转换渠道，基于nginx的njs模块提供JSON到Line Protocol的转换。


- 2018-08-20   [认识Hystrix](blog/hystrix.md)  

    > 在一个分布式系统里，许多依赖不可避免的会调用失败，比如超时、异常等，如何能够保证在一个依赖出问题的情况下，不会导致整体服务失败，这个就是Hystrix需要做的事情。Hystrix提供了熔断、隔离、Fallback、cache、监控等功能，能够在一个、或多个依赖同时出现问题时保证系统依然可用。

- 2018-06-03   [如何在CentOS 7.2上编译使用nginx 1.15.0](blog/nginx.md)  

    > nginx官方版本未提供CentOS 7.2的nginx 1.15.0，只有7.4以上版本才有官方版本，这里简单介绍下如何本地编译一份自己需要的nginx。

- 2018-04-10   [全民HTTPS：免费的服务端证书与客户端证书](blog/ssl-certs.md)  

    > 在开发测试过程中，我们经常要对客户端与服务器端的HTTP通讯加密，也就是HTTPS。而HTTPS往往依赖于证书，能公网使用的证书往往都是收费的，且费用不低。国外一家公司[Let's Encrypt](https://letsencrypt.org/getting-started/)为我们提供了另一种选择，免费的服务端证书，并且可以通过各种浏览器、安卓与IOS的验证。

    > 但是Let's Encrypt的免费证书有效期只有90天，本文将介绍如何（通过[certbot](https://certbot.eff.org/lets-encrypt/centosrhel7-other)自动刷新有效期）。
    
    > 在需要双向HTTPS校验的场景下，我们往往还需要客户端证书。客户端证书我会另外介绍openssl自签发证书，因为客户端证书实际是由我们自己的服务器端去校验，所以自签发证书即可解决大部分问题。
