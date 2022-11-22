---
layout: title
title: BT日志管理
date: 2018-09-12 11:41:49
tags: [运维]
categories: 技术
---

## 前言

日志用来记录用户操作、系统运行状态，虽然不属于核心业务功能，但却是运维、开发定位问题的灵丹妙药，甚至有些日志还能供运营做参考。

BT一直在践行着敏捷开发的开发模式，快速的迭代过程要求我们能够迅速地发现问题、解决问题，查询日志在里面占了重要的一环:

- 刚开始的时候只有一两个后端服务，用户反馈了问题，我们可以直接在日志文件里面找到对应的错误，并且很快就能定位到问题并且解决。
- 后来服务开始增加，一个请求可能经过多个服务处理，用户反馈了问题，我们可能需要查多个服务的日志文件才能定位到问题。与此同时，由于开发人员变多，日志信息没有统一，定位到问题的大概范围后，需要把问题转给对应的开发人员，他再去查询对应的日志。
- 再后来情况变得更加复杂，同一个服务分布式部署在多台机器，比如官网，可能有五六台机器在运行。这个时候用户反馈问题过来，查到对应日志基本是不可能的事情，开发只能尽量在本地复现，这个时候日志在逐渐失去作用。

可以看到，随着业务的逐渐发展，日志管理平台的需求日益迫切，现阶段主要解决以下几个问题:

- 日志规范: 什么情况该打什么类型的日志，日志内容是什么，都需要有一个统一的规范。日志信息过少不能定位到问题，日志信息过多容易淹没重要信息，统一的日志格式有利于做统计分析，统一的日志等级有利于做主动监控。
- 收集与集中展示: 所有日志都应该收集至日志平台，开发人员只需要在一个平台就可以搜索到需要的日志，同时还可以加上权限控制之类的功能。
- 日志监控报警: 之前系统出现问题，都依赖于用户反馈，不够及时，而且用户量大了，收集问题的成本越来越高。实现日志监控报警可以主动发现问题，在问题扩散之前引起重视。

## 日志平台ELK

1. 可以用docker快速搭建，对于没有真正运维的公司来说，简单粗暴。
2. 可以不入侵现有代码，用logstash从文件中直接读取日志。
3. 前期只需要安装logstash + elasticserach + kibana就可以完成日志采集和展示，后期如果日志量变大、对性能要求变高的话，可以增加filebeat作为文件采集，引入kafka消息队列等。

### docker安装elastcisearch

```sh
touch /mnt/config/elk/elasticsearch.yml
echo 'network.host: xxx.xxx.xxx.xxx' > /mnt/config/elk/elasticsearch.yml
# xxx.xxx.xxx.xxx为本机公网ip

docker run -p 9200:9200 -p 9300:9300
\ --name elasticsearch -d -e "discovery.type=single-node"
\ -v /mnt/es-data/:/usr/share/elasticsearch/data/
\ -v /mnt/config/elk/elasticsearch.yml:/config/elasticsearch.yml
\ -e ES_JAVA_OPTS="-Xms12g -Xmx12g"
\ -d elasticsearch
```

> 后面还需安装x-pack，需要定制镜像

### docker安装kibana

```sh
docker run -d -p 5601:5601 
\ --link elasticsearch 
\ -e ELASTICSEARCH_URL=http://elasticsearch:9200
\ --name kibana kibana
```

> 后面还需安装x-pack，需要定制镜像

### docker安装logstash

#### 自定义镜像(安装x-pack)

```sh
mkdir logstash

cd logstash

touch Dockerfile

echo 'FROM logstash:5.6.11-alpine
RUN logstash-plugin install x-pack
ENTRYPOINT ["/docker-entrypoint.sh"]
CMD ["-e", ""]' > Dockerfile

docker build -t bt-logstash .
```

#### logstash部署

```sh
touch /mnt/config/elk/logstash.conf

echo '
input {
    file {
        path => ["/mnt/log/*/app-error.log", "/mnt/log/*/app-out.log"]
        codec => multiline {
            pattern => "^\d{4}\-\d{2}\-\d{2} \d{2}:\d{2}(:\d{2})?"
            negate => true
            what => "previous"
        }
    }
}
input {
    file {
        path => ["/mnt/log/nginx/*.log"]
        codec => multiline {
            pattern => "^%{IPV4}|^\d{4}\/\d{2}\/\d{2} \d{2}:\d{2}:\d{2}"
            negate => true
            what => "previous"
        }
    }
}
filter {
    grok {
        match => [
            "message", "(?<logdate>%{YEAR}/%{MONTHNUM}/%{MONTHDAY} %{HOUR}:%{MINUTE}(:%{SECOND})?)",
            "message", "(?<logdate>%{YEAR}-%{MONTHNUM}-%{MONTHDAY} %{HOUR}:%{MINUTE}(:%{SECOND})?)",
            "message", "(?<logdate>%{MONTHDAY}/%{MONTH}/%{YEAR}:%{HOUR}:%{MINUTE}(:%{SECOND})?)",
            "message", "(?<logdate>%{MONTHDAY}-%{MONTH}-%{YEAR} %{HOUR}:%{MINUTE}(:%{SECOND})?)"
        ]
    }
    grok {
        match => [
            "path", "\/mnt\/log\/(?<soft>[\S\s]+)\/(?<level>\D+?)[\d\-\_]*\.log"
        ]
    }
    mutate {
        replace => { "logdate" => "%{logdate} +08:00" }
    }
    date {
        match => ["logdate", "yy/M/d H:m:s", "y-M-d H:m", "y-M-d H:m:s", "d/MMM/y:H:m:s", "y/M/d H:m:s", "d-MMM-y H:m:s"]
    }
    mutate {
        lowercase => [ "soft", "level" ]
        add_tag => ["%{soft}", "%{level}"]
    }
}
output {
    elasticsearch {
        user => logstash
        password => logstash
        hosts => ["120.78.249.137:9200"]
        index => "prod-%{soft}-%{level}-%{+YYYY.MM.dd}"
    }
}
' > /mnt/config/elk/logstash.conf

touch /mnt/config/elk/logstash.yml

echo 'xpack.monitoring.elasticsearch.username: logstash_system
xpack.monitoring.elasticsearch.password: logstash_system
xpack.monitoring.elasticsearch.url: http://120.78.249.137:9200' > /mnt/config/elk/logstash.yml

docker run -d -it --name logstash
\ -v /mnt/config/elk/logstash.conf:/usr/share/logstash/pipeline/logstash.conf
-v /mnt/config/elk/logstash.yml:/usr/share/logstash/config/logstash.yml
\ -v /mnt/log:/mnt/log 
\ registry.btclass.net/bt-logstash:lastest
\ -f /usr/share/logstash/pipeline/logstash.conf
```
## 日志平台splunk

### splunk安装

安装splunk 6.4.0

```
groupadd splunk

useradd -d /opt/splunk -m -g splunk splunk

wget -O splunk-6.4.0-f2c836328108-Linux-x86_64.tgz 'https://www.splunk.com/page/download_track?file=6.4.0/linux/splunk-6.4.0-f2c836328108-Linux-x86_64.tgz&ac=&wget=true&name=wget&platform=Linux&architecture=x86_64&version=6.4.0&product=splunk&typed=release'

tar -xvf splunk-6.4.0-f2c836328108-Linux-x86_64.tgz

cp -rp splunk/* /opt/splunk/

chown -R splunk: /opt/splunk/
```

破解splunk

```
su - splunk

cd ./bin

mv ./splunkd splunkd.bak

wget -O splunkd 'https://tmp.btclass.cn/software/splunkd'

./splunk start --accept-license
```

访问网页管理界面，填以下授权码

```
<?xml version="1.0" encoding="UTF-8"?>
<license>
  <signature>aOkSwn0ofj0eWrO5ZnHKfAGl2lxo/uT6g5Bqz6YtWZm41WxWq9tjDzjZ0CvsrwhaE6kK+PV+1lIiG59fmsYpMyorA7ADB3kNlwtvcouJ3VFYfpnVP62X8cIe/nzquWl6FqH+SJcyTtcWJ/U/G+u9Dqn1C0WdzrFsO90rQdZmotAfaydR86u9BNyFQz9U+1uNYPJCxM1Wwkvu+FwtiRtMJSta5V0Zt1OmH8L9M/jhb7OVAyRnERUtD8ov5gTBOtQlf5Hi+xBOJS5XP7xsSWWHiQSnzSRdbzKcmNh9k6jDF9t5otdBLIDi/2p7uQ22oF2rkxu6/U0j5CuctFjGUHHQkw==</signature>
  <payload>
      <type>enterprise</type>
      <group_id>Enterprise</group_id>
      <quota>109951199256769</quota>
      <max_violations>5</max_violations>
      <window_period>30</window_period>
      <creation_time>1408986780</creation_time>
      <absolute_expiration_time>4133350357</absolute_expiration_time>
      <relative_expiration_interval>4133350357</relative_expiration_interval>
      <label>Splunk Enterprise</label>
      <features>
          <feature>Auth</feature>
          <feature>FwdData</feature>
          <feature>RcvData</feature>
          <feature>LocalSearch</feature>
          <feature>DistSearch</feature>
          <feature>RcvSearch</feature>
          <feature>ScheduledSearch</feature>
          <feature>Alerting</feature>
          <feature>DeployClient</feature>
          <feature>DeployServer</feature>
          <feature>SplunkWeb</feature>
          <feature>SigningProcessor</feature>
          <feature>SyslogOutputProcessor</feature>
          <feature>AllowDuplicateKeys</feature>
      </features>
      <sourcetypes/>
 <guid>553A0D4F-3B7B-4AD5-B241-89B94386A07F</guid></payload>
</license>
```

### forwarder安装

```
<!--wget -O splunkforwarder-6.4.0-f2c836328108-Linux-x86_64.tgz 'https://www.splunk.com/page/download_track?file=6.4.0/linux/splunkforwarder-6.4.0-f2c836328108-Linux-x86_64.tgz&ac=&wget=true&name=wget&platform=Linux&architecture=x86_64&version=6.4.0&product=universalforwarder&typed=release'-->

wget -O splunkforwarder-6.4.0-f2c836328108-Linux-x86_64.tgz 'https://tmp.btclass.cn/software/splunkforwarder-6.4.0-f2c836328108-Linux-x86_64.tgz'

tar zxvf splunkforwarder-6.4.0-f2c836328108-Linux-x86_64.tgz -C /opt
```

```
touch /opt/splunkforwarder/etc/system/local/outputs.conf

echo '[tcpout]
defaultGroup = default-autolb-group

[tcpout:default-autolb-group]
server = 120.78.249.137:9997

[tcpout-server://120.78.249.137:9997]' > /opt/splunkforwarder/etc/system/local/outputs.conf
```

```
touch /opt/splunkforwarder/etc/system/local/inputs.conf

echo '[default]
host = 改成当前服务器IP

[monitor:///mnt/log/*/*out.log]
disabled = false
index = main

[monitor:///mnt/log/*/*error.log]
disabled = false
index = main

[monitor:///mnt/log/*/*access.log]
disabled = false
index = main

[monitor:///mnt/log/*/*/*out.log]
disabled = false
index = main

[monitor:///mnt/log/*/*/*error.log]
disabled = false
index = main

[monitor:///mnt/log/*/*/*access.log]
disabled = false
index = main' > /opt/splunkforwarder/etc/system/local/inputs.conf
```

```
cd /opt/splunkforwarder/bin

./splunk start --accept-license
```

### splunk使用

splunk地址: xxxxx
账号: xxxx
密码: xxxx

1. 点击 `Search&Reporting` 进入应用
2. 可以直接在搜索框内进行搜索，也可以点击数据摘要，根据主机或者来源直接进行搜索
3. 如果想要搜索错误日志，在搜索框输入error即可，在右侧可以选择对应的时间段
4. 如果想忽略掉某个错误，则可搜索 error NOT (ENOTFOUND ms-distribution.btclass.cn ms-distribution.btclass.cn:443)

## 日志规范

### 存在的问题

日志平台搭建完，就已经可以看到现有的日志存在的问题:

1. 单条日志行数过多，超出logstash的限制: 把日志输出成一行，牺牲一定的可读性，好处是不再需要根据不同的日志制定多行分条策略；
2. 日志内容过多: 比如题库日志里包含了sequelize对象，或者很大的一包数据，需要确定这些数据是否真的有打印到日志里的必要，能否只打印关键信息。
3. 存在意义不明的数据: 比如有一条的日志message是`======== 55`，意义不明。
4. 时间格式没有统一: logstash配置里使用了大量正则表达式，从日志里提取出时间，如果时间格式可以统一，将可以提高一定效率。
5. 不能看到日志上下文: 需要有一个requestId，可以方便地查到某个请求的相关日志。
6. 需要明确的日志等级。
7. 敏感数据，比如用户密码、数据库密码等可能出现在日志里。

### 解决问题

#### 日志框架选择(仅针对node)

对比了`winston`,`log4js`,`bunyan`,`tracer`几个日志框架，都能实现基本的日志功能，最终选择了`bunyan`。

主要以下几点:
1. bunyan使用json字符串格式打印字符串，没有多行问题。
2. bunyan利用log.child可以方便地给请求加上requestId。
3. 其他诸如，日志文件切分、日志等级定义、日志代码位置等，bunyan也都支持。

#### 定义日志等级

> 日志等级主要根据网上的观点、日志框架的建议和平常的经验进行定义

1. FATAL: 系统级别错误，需要管理员马上查看并修复。一般是进程遇到无法修复的错误时而退出时记录的日志，此时服务已经完全挂掉了。这种日志在我们目前的node服务中基本没有，可以在www文件中增加`process.on('uncaughtException', (err) => { log.error(err) });
process.on('unhandledRejection', (err) => { log.error(err) });`来捕获服异常并打日志。
2. ERROR: 接口级别的错误，紧急程度低于`FATAL`，但是系统对于用户而言已经出现一定程度的不可用了。
3. WARN: 表示系统可能会出现问题，比如请求处理时间过长、数据库慢查询、系统存在异常数据且无法自动修复但是又不会影响当前请求。该级别的日志，虽然不需要马上处理，但是管理员需要定时查看并修复，说明系统存在隐患。
4. INFO: 该日志记录系统正常运行状态，服务的启动日志、请求成功执行、某些操作的关键信息(比如某条推送的内容和接收者)，通过查询`INFO`日志可以帮助定位前面三种错误。`INFO`日志不宜过多，可以只打关键信息，比如前面说的推送接受者，如果群发人数过多可以选择只打印总人数和整体推送情况，同时还要注意敏感信息的保护。
5. DEBUG 和 TRACE: 这两个日志主要是对系统的每一步运行进行精确的记录，通过该日志可以查看某一个操作每一步的执行过程，可以准确定位是何种操作，何种参数，何种顺序导致了某种错误的发生。对于这两个日志没有找到具体的规范，个人理解，`INFO`用于记录关键节点的状态，而`DEBUG`和`TRACE`用于记录每个节点间的过程，这种日志比较详细，日志量也比较大，考虑定时删除或者仅部分服务输出。

#### bunyan使用

对bunyan进行简单的封装，增加了requestId，便于追踪请求链路日志，[详见bt-logger](https://gitlab.btclass.cn/lib/bt-logger)。

如果所有请求日志都要输出requestId的话，对旧代码有比较大的入侵，故只在controller层及以上部分输出requestId，service及以下层级不输出。

![](http://cdn.jsblog.site/15406263745123.jpg)
使用bunyan在splunk上的输出


