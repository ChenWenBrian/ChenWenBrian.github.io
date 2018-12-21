# 如何使用nginx为influxdb提供API网关服务

目录：
  - [1 如何通过POST一个简单的JSON报文往influxdb里插入数据？](#1-%E5%A6%82%E4%BD%95%E9%80%9A%E8%BF%87post%E4%B8%80%E4%B8%AA%E7%AE%80%E5%8D%95%E7%9A%84json%E6%8A%A5%E6%96%87%E5%BE%80influxdb%E9%87%8C%E6%8F%92%E5%85%A5%E6%95%B0%E6%8D%AE)
    - [示例：插入数据的请求报文协议](#%E7%A4%BA%E4%BE%8B%E6%8F%92%E5%85%A5%E6%95%B0%E6%8D%AE%E7%9A%84%E8%AF%B7%E6%B1%82%E6%8A%A5%E6%96%87%E5%8D%8F%E8%AE%AE)
  - [2 如何通过简单URL构成的GET请求去查询influxdb？](#2-%E5%A6%82%E4%BD%95%E9%80%9A%E8%BF%87%E7%AE%80%E5%8D%95url%E6%9E%84%E6%88%90%E7%9A%84get%E8%AF%B7%E6%B1%82%E5%8E%BB%E6%9F%A5%E8%AF%A2influxdb)
    - [示例：查询用户最近一次写入记录的时间](#%E7%A4%BA%E4%BE%8B%E6%9F%A5%E8%AF%A2%E7%94%A8%E6%88%B7%E6%9C%80%E8%BF%91%E4%B8%80%E6%AC%A1%E5%86%99%E5%85%A5%E8%AE%B0%E5%BD%95%E7%9A%84%E6%97%B6%E9%97%B4)
  - [3 nginx配置说明与njs模块代码](#3-nginx%E9%85%8D%E7%BD%AE%E8%AF%B4%E6%98%8E%E4%B8%8Enjs%E6%A8%A1%E5%9D%97%E4%BB%A3%E7%A0%81)
    - [示例：nginx配置文件](#%E7%A4%BA%E4%BE%8Bnginx%E9%85%8D%E7%BD%AE%E6%96%87%E4%BB%B6)
    - [示例：njs模块配置文件](#%E7%A4%BA%E4%BE%8Bnjs%E6%A8%A1%E5%9D%97%E9%85%8D%E7%BD%AE%E6%96%87%E4%BB%B6)
  - [附录：Influxdb与Chronograf的安装说明](#%E9%99%84%E5%BD%95influxdb%E4%B8%8Echronograf%E7%9A%84%E5%AE%89%E8%A3%85%E8%AF%B4%E6%98%8E)
    - [Influxdb的安装方法](#influxdb%E7%9A%84%E5%AE%89%E8%A3%85%E6%96%B9%E6%B3%95)
    - [Influxdb安装后的基本信息](#influxdb%E5%AE%89%E8%A3%85%E5%90%8E%E7%9A%84%E5%9F%BA%E6%9C%AC%E4%BF%A1%E6%81%AF)
    - [Chronograf的安装方法](#chronograf%E7%9A%84%E5%AE%89%E8%A3%85%E6%96%B9%E6%B3%95)
    - [Chronograf安装后的基本信息](#chronograf%E5%AE%89%E8%A3%85%E5%90%8E%E7%9A%84%E5%9F%BA%E6%9C%AC%E4%BF%A1%E6%81%AF)

InfluxDB是一个开源的时序数据库，使用GO语言开发，特别适合用于处理和分析资源监控数据这种时序相关数据。而InfluxDB自带的各种特殊函数如求标准差，随机取样数据，统计数据变化比等，使数据统计和实时分析变得十分方便。关于influxdb的基础知识本文就不做详细介绍了，这里主要是针对influxdb在开发中的痛点提供一个API的转换渠道。

Influxdb本身自带有HTTP接口，但是语法可能比较冷门，使用的是[Line Protocol](https://docs.influxdata.com/influxdb/v1.6/write_protocols/line_protocol_tutorial/#special-characters-and-keywords)。这种协议在实际开发中对开发人员可能相当不友好，一个不小心就会出现数据或字段的转义错误。本文提供一种基于JSON格式的语法转换渠道，基于nginx的njs模块提供JSON到Line Protocol的转换。

> 基本思路：**APP/Server** `----json---->` **nginx** `----line protocols---->` **influxdb**

关于如何编译带njs模块的nginx，请参考[如何在CentOS 7.2上使用nginx 1.15.0](nginx.md)

## 1 如何通过POST一个简单的JSON报文往influxdb里插入数据？

influxdb的line protocol支持单条插入与批量插入，本文以插入单条数据为例。例如我们现在有一个用户APP端的行为监控需求，需要从APP端不断发送用户在APP上的操作记录或者收集用户资料。

### 示例：插入数据的请求报文协议

**HTTP协议Header与URL**（PS:这个URL格式需根据自身的业务场景自行定制）

| 名称                | 格式                                          | 示例                                       | 备注                                                                                                                 |
| ------------------- | --------------------------------------------- | ------------------------------------------ | -------------------------------------------------------------------------------------------------------------------- |
| HTTP方法            | 固定值                                        | POST                                       |
| URL                 | /analysis/`${business-type}`/`${measurement}` | /analysis/action-monitor/start_apply_event |
| ├─ ${business-type} | `/a-zA-Z0-9\-/`                               | action-monitor                             | 业务类型：与influxdb的数据库一一映射。譬如我们有action-monitor库，balabala库，foobarfoobar库等等分别对应不同的业务。 |
| └─ ${measurement}   | `/a-zA-Z0-9_\-/`                              | start_apply_event                          | 监控行为，即influxdb的measurement。譬如我们有一张表叫start_apply_event……                                             |
| Content-Type        | 固定值                                        | application/json;charset=UTF-8             | HTTP header                                                                                                          |

**写入接口消息体JSON格式**（PS:这个消息体格式可以通用）

| 名称              | 类型   | 必选 | 备注                                                                   |
| ----------------- | ------ | ---- | ---------------------------------------------------------------------- |
| tags              | Array  | 否   | 要监控上报的tag数组，tag用于对记录分类/分组                            |
| └─ tags[`name`]   | Object |      | 键值对，对应influxdb的tag字段名和值；支持string、bool、number等类型    |
| fields            | Array  | 是   | 要监控上报的field数组，field用于存储各种记录值                         |
| └─ fields[`name`] | Object |      | 键值对，对应influxdb的field字段名和值；支持string、bool、number等类型  |
| time              | Date   | 否   | 时间戳：该记录的写入时间，默认值为当前influxdb的系统时间，精确到毫秒。 |

**请求JSON示例**

```json
{
    "tags":{
        "role":"Sales Agent",
    },
    "fields":{
        "user-id":"SA20163389",
        "age":32,
        "married":true,
        "address":"广东省深圳市福田区新浩E都3楼"
    },
    "time":"2018-07-07T11:12:54.111817682Z" //可选
}
```

**成功返回JSON示例**

```json
{
    "ret":0,
    "msg":"success",
    "status":204
}
```

> 注：请求的JSON报文如果不合法，nginx会直接返回400；其他错误则返回influxdb原始JSON错误消息。注：influxdb的HTTP协议本身在处理错误请求时返回的也是JSON报文。

---

## 2 如何通过简单URL构成的GET请求去查询influxdb？

查询influxdb的语法本身是通过在URL的query string中构建sql语句实现的，但是现实中我们往往不能直接将sql接口暴露给前端使用。所以这里提供的了一种从URL到sql的转换方法，其中sql中的各种参数，需在URL中用正则式限定，防止sql注入。

### 示例：查询用户最近一次写入记录的时间

根据指定的用户ID与行为，查询一段时间内，该用户最近一次上报/写入数据的时间。

**查询接口说明**，见下表

| 名称                | 格式                                                                  | 示例                                                        | 备注                                                                                                                 |
| ------------------- | --------------------------------------------------------------------- | ----------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------- |
| HTTP方法            | 固定值                                                                | GET                                                         |
| URL                 | /analysis-last-`${time}`/`${business-type}`/`${measurement}`/`${uid}` | /analysis-last-30d/action-monitor/start_apply_event/6h20031 |
| ├─ ${time}          | `/\d+[wdhms]/`                                                        | 30d                                                         | 要查询的时间范围，如30天内则为30d，24小时则为24h，支持周w、天d、时h、分m、秒s。                                           |
| ├─ ${business-type} | `/a-zA-Z0-9\-/`                                                       | action-monitor                                              | 业务类型：与influxdb的数据库一一映射。譬如我们有action-monitor库，balabala库，foobarfoobar库等等分别对应不同的业务。 |
| ├─ ${measurement}   | `/a-zA-Z0-9_\-/`                                                      | start_apply_event                                           | 监控行为，及influxdb的measurement。譬如我们有一张表叫start_apply_event……                                             |
| └─ ${uid}           | `/a-zA-Z0-9_\-/`                                                      | 6h20031                                                     | 用户ID                                                                                                               |

**返回的JSON示例**： 返回的JSON消息nginx不做任何转换处理，直接返回influxdb原始JSON串。[influxdb API参考链接](https://docs.influxdata.com/influxdb/v1.6/tools/api/#query)

```json
{
    "results": [
        {
            "statement_id": 0,
            "series": [
                {
                    "name": "start_apply_event",
                    "columns": [
                        "time",
                        "uid"
                    ],
                    "values": [
                        [
                            "2018-07-07T11:12:54.111817682Z",
                            "6h20031"
                        ]
                    ]
                }
            ]
        }
    ]
}
```

---



## 3 nginx配置说明与njs模块代码

### 示例：nginx配置文件

> 注意：该配置文件中，数据库名称采用映射模式可以对前端隐藏库名；而表名没有做映射，而是简单的采用白名单模式；

```ini
#允许访问的influxdb库&映射关系，在该map里列出允许访问的库（凡是没有映射的库，nginx会拦截相应的请求）
map $analysis_biz_type $influx_db {
    default                 "NotFound";
    "action-monitor"        "action_monitor_db";
    "data-statistics"       "data_statistics_db";
    "cs-monitor"            "cs_monitor_db";
}

#允许访问的influxdb表，在该map里列出允许访问的表（凡是没有列出的表，nginx会拦截相应的请求）
map $analysis_table $is_valid_influx_table {
    default                        0;
    "start_apply_event"            1;
    "submit_apply_event"           1;
    "modify_pwd_event"             1;
    "activate_event"               1;
    "launch_event"                 1;
    "contact_event"                1;
    "cs_login"                     1;
    "cs_record_commit"             1;
    "dau_event"                    1;
}

# influxdb HTTP转发接口：提供权限控制、数据库过滤、表过滤、数据过滤以及数据格式转换等
server {
    listen  80 default_server;
    server_name  localhost;
    
    access_log  logs/influxdb.access.log  main;

    # 行为监控-提交数据API
    location ~ ^/analysis/(?<analysis_biz_type>[a-zA-Z0-9\-]+)/(?<analysis_table>[a-zA-Z0-9_]+)$ {
        #拦截不符合报文规范的请求
        limit_except POST {
            deny all;
        }
        if ( $influx_db = "NotFound" ) {
            return 400 "Invalid business type";
        }
        if ( $is_valid_influx_table = 0 ) {
            return 400 "Invalid measurement";
        }
        
        #2MB buffer for large JSON
        client_body_buffer_size 2048k;

        #执行njs模块的json转换函数
        js_content insertInfluxDB;
    }

    # 行为监控-查询最近一次提交数据时间API
    location ~ ^/analysis-last-(?<analysis_time>[0-9]+[wdhms])/(?<analysis_biz_type>[a-zA-Z0-9\-]+)/(?<analysis_table>[a-zA-Z0-9_]+)/(?<analysis_uid>[a-zA-Z0-9_\-]+)$ {
        limit_except GET {
            deny all;
        }
        if ( $influx_db = "NotFound" ) {
            return 400 "Invalid business type";
        }
        if ( $is_valid_influx_table = 0 ) {
            return 400 "Invalid measurement";
        }
        js_content queryInfluxDB;
    }

    # influxdb nginScript内部转发接口，用来接收从njs模块处理后的请求
    location /influxdb {
        internal;
        rewrite ^\/influxdb(\/.*)$ $1 break;
        proxy_set_header X-Real-Ip $real_ip_or_remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_pass http://192.168.10.100:8086; #这里是你的influxdb的服务器地址，可以用upstream去定义集群
    }
}

```

### 示例：njs模块配置文件

> 注意：该配置文件中，数据库名称采用映射模式可以对前端隐藏库名；而表名没有做映射，而是简单的采用白名单模式；

```js
//==============================================InfluxDB==============================================

//字符串辅助函数（注：njs暂不支持arguments以及prototype扩展）
function format(template, args) {
    return template.replace(/{(\d+)}/g, function (match, number) {
        return typeof args[number] != 'undefined'
            ? args[number].toString()
            : match;
    });
};

//njs函数，提供json到line protocal的转换
function insertInfluxDB(r) {
    try {
        var s = r.variables.analysis_table, data = JSON.parse(r.requestBody);
        for (var t in data.tags) {
            s += format(",{0}={1}", [t, data.tags[t].replace(/[,=\s]/g, "\\$&")]);
        }
        s += " ";

        for (var f in data.fields) {
            if (typeof data.fields[f] == "string") {
                s += format('{0}="{1}",', [f, data.fields[f].replace(/[\\"]/g,'\\$&')]);
            } else {
                s += format('{0}={1},', [f, data.fields[f]]);
            }
        }
        s = s.slice(0, -1);

        //如果时间是字符串格式，则转换成influxdb使用的数值（UTC时间）
        if (typeof data.time == "string") {
            s += " " + (new Date(data.time)).getTime() * 1000000;
        }

        r.subrequest("/influxdb/write", { method: 'POST', body: s, args: "db=" + encodeURIComponent(r.variables.influx_db) }, function (res) {
            if (res.status == 204) {
                r.headersOut["Content-Type"] = "application/json;charset=UTF-8";
                r.return(200, JSON.stringify({
                    "ret": 0,
                    "msg": "success",
                    "status": res.status
                }));
            } else {
                r.return(res.status, res.responseBody);
            }
        });
    } catch (error) {
        r.error("insertInfluxDB exception: " + error);
        r.return(400, "Invalid JSON format");
    }
}

//njs函数，提供URL中的参数到sql的转换（注：如果参数没有在nginx的conf文件中限定格式，这里就需要防止sql注入）
function queryInfluxDB(r) {
    try {
        var sql = 'SELECT last("uid") AS "uid"  FROM "{0}" WHERE time > now() - {1} AND "uid"=\'{2}\'';
        sql = format(sql, [r.variables.analysis_table, r.variables.analysis_time, r.variables.analysis_uid]);
        var queryString = 'pretty=true&db=' + encodeURIComponent(r.variables.influx_db) + '&q=' + encodeURIComponent(sql);
        r.internalRedirect("/influxdb/query?" + queryString);
    } catch (error) {
        r.error("queryInfluxDB exception: " + error);
        r.return(400, "Invalid query format");
    }
}
```

---

## 附录：Influxdb与Chronograf的安装说明

### Influxdb的安装方法

CentOS上可使用`wget`从官网上下载[influxdb 1.6.0](https://dl.influxdata.com/influxdb/releases/influxdb-1.6.0.x86_64.rpm)，然后通过`yum localinstall`本地安装RPM包。安装完毕后会自动注册服务，由systemd监管。更多细节可以参考[官方文档](https://docs.influxdata.com/influxdb/v1.6/introduction/installation/)。

```bash
wget https://dl.influxdata.com/influxdb/releases/influxdb-1.6.0.x86_64.rpm
sudo yum localinstall influxdb-1.6.0.x86_64.rpm
```

### Influxdb安装后的基本信息

| 名称     | 值                                 |
| -------- | ---------------------------------- |
| 操作系统 | CentOS 7.2                         |
| 应用版本 | Influxdb 1.6.0                     |
| 数据目录 | /var/lib/influxdb                  |
| 配置文件 | /etc/influxdb/influxdb.conf        |
| 日志文件 | /var/log/influxdb/influxd.log      |
| 启动命令 | `systemctl start influxdb.service` |
| 停止命令 | `systemctl stop influxdb.service`  |

### Chronograf的安装方法

CentOS上可使用`wget`从官网上下载[Chronograf 1.6.0](https://dl.influxdata.com/chronograf/releases/chronograf-1.6.0.x86_64.rpm)，然后通过`yum localinstall`本地安装RPM包。安装完毕后会自动注册服务，由systemd监管。更多细节可以参考[官方文档](https://docs.influxdata.com/chronograf/v1.6/introduction/installation/)。

```bash
wget https://dl.influxdata.com/chronograf/releases/chronograf-1.6.0.x86_64.rpm
sudo yum localinstall chronograf-1.6.0.x86_64.rpm
```

### Chronograf安装后的基本信息

| 名称     | 值                                   |
| -------- | ------------------------------------ |
| 操作系统 | CentOS 7.2                           |
| 应用版本 | Chronograf 1.6.0                     |
| 数据目录 | /var/lib/chronograf                  |
| 配置文件 | /etc/chronograf/chronograf.conf      |
| 日志文件 | /var/log/chronograf/chronograf.log   |
| 启动命令 | `systemctl start chronograf.service` |
| 停止命令 | `systemctl stop chronograf.service`  |

