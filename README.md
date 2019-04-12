# Elastic stack (ELK) on Docker

[![Join the chat at https://gitter.im/deviantony/docker-elk](https://badges.gitter.im/Join%20Chat.svg)](https://gitter.im/deviantony/docker-elk?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)
[![Elastic Stack 版本](https://img.shields.io/badge/ELK-6.7.0-blue.svg?style=flat)](https://github.com/deviantony/docker-elk/issues/376)
[![构建状态](https://api.travis-ci.org/deviantony/docker-elk.svg?branch=master)](https://travis-ci.org/deviantony/docker-elk)

使用Docker和Docker Compose运行最新版本的[Elastic stack](https://www.elastic.co/elk-stack)。

它将使您能够使用Elasticsearch的搜索/聚合功能分析任何数据集
和Kibana的可视化力量。

基于Elastic的官方Docker镜像:

* [elasticsearch](https://github.com/elastic/elasticsearch-docker)
* [logstash](https://github.com/elastic/logstash-docker)
* [kibana](https://github.com/elastic/kibana-docker)

**注意**: 该项目的其他分支机构可用:

* [`x-pack`](https://github.com/deviantony/docker-elk/tree/x-pack): X-Pack support
* [`searchguard`](https://github.com/deviantony/docker-elk/tree/searchguard): Search Guard support
* [`vagrant`](https://github.com/deviantony/docker-elk/tree/vagrant): run Docker inside Vagrant

## 内容

1. [要求](#要求)
     * [主机设置](#主机设置)
     * [SELinux](#selinux)
     * [Docker for Windows](#docker-for-windows)
2. [用法](#用法)
     * [构建stack](#构建stack)
     * [初始设置](#initial-setup)
3. [配置](＃配置)
     * [我如何调整Kibana配置？](#how-can-i-tune-the-kibana-configuration)
     * [如何调整Logstash配置？](#how-can-i-tune-the-logstash-configuration)
     * [我如何调整Elasticsearch配置？](#how-can-i-tune-the-elasticsearch-configuration)
     * [如何扩展Elasticsearch集群？](#how-can-i-scale-the-elasticsearch-cluster)
4. [存储](#存储)
     * [如何保存Elasticsearch数据？](#how-can-i-persist-elasticsearch-data)
5. [可扩展性](#extensibility)
     * [如何添加插件？](#how-can-i-add-plugins)
     * [如何启用提供的扩展？](#how-can-i-enable-the-provided-extensions)
6. [JVM调整](#jvm-tuning)
     * [如何指定服务使用的内存量？](#how-can-i-specified-of-memory-of-of-a-service)
     * [如何启用与服务的远程JMX连接？](#how-can-i-enable-a-remote-jmx-connection-a-service)
7. [更进一步](#going-further)
     * [使用较新的堆栈版本](#using-a-newer-stack-version)
     * [插件和集成](#plugins-and-integraterations)
     * [Docker Swarm](#docker-swarm)

## 要求

### 主机设置

1.安装[Docker](https://www.docker.com/community-edition#/download)版本** 17.05 + **
2.安装[Docker Compose](https://docs.docker.com/compose/install/)版本** 1.6.0 + **
3.克隆此存储库

### SELinux

在开箱即用的SELinux的发行版上，你需要重新上传文件或设置SELinux
进入许可模式，以便docker-elk正常启动。 例如，在Redhat和CentOS上，以下是
应用适当的上下文:

```console
$ chcon -R system_u:object_r:admin_home_t:s0 docker-elk/
```

### Docker for Windows

如果您使用的是Docker for Windows，请确保为`C:`驱动器(Docker for Windows>设置>共享驱动器)启用了“共享驱动器”功能。 请参阅[为Windows共享驱动器配置Docker](https://blogs.msdn.microsoft.com/stevelasker/2016/06/14/configuring-docker-for-windows-volumes/)(MSDN博客)。

## 用法

### 构建stack

**注意**:如果您切换分支或更新基本映像 - 您可能需要先运行`docker-compose build`

使用`docker-compose`启动docker:

```console
$ docker-compose up
```

您还可以通过在上面的命令中添加`-d`标志来在后台运行所有服务(分离模式)。

给Kibana初始化几秒钟，然后通过点击访问Kibana Web UI
[http://localhost:5601](http://localhost:5601) ，带有Web浏览器。

默认情况下，堆栈公开以下端口:
* 5000:Logstash TCP输入。
* 9200:Elasticsearch HTTP
* 9300:Elasticsearch TCP传输
* 5601:Kibana

**警告**:如果您使用`boot2docker`，则必须通过`boot2docker` IP地址而不是`localhost`访问它。

**警告**:如果您使用的是* Docker Toolbox *，则必须通过`docker-machine` IP地址访问它，而不是
`localhost`。

现在堆栈正在运行，您将需要注入一些日志条目。 随附的Logstash配置允许您
通过TCP发送内容:

```console
$ nc localhost 5000 < /path/to/logfile.log
```

## 初始设置

### 默认Kibana索引模式创建

当Kibana第一次启动时，它没有配置任何索引模式。

#### 通过Kibana Web UI

**注意**:您需要先将数据注入Logstash，然后才能通过Kibana Web配置Logstash索引模式
UI。 然后你要做的就是点击* Create *按钮。

请参阅[连接KibanaElasticsearch](https://www.elastic.co/guide/en/kibana/current/connect-to-elasticsearch.html)获取详细说明
关于索引模式配置。

####在命令行上

通过Kibana API创建索引模式:

```console
$ curl -XPOST -D- 'http://localhost:5601/api/saved_objects/index-pattern' \
    -H 'Content-Type: application/json' \
    -H 'kbn-version: 6.7.0' \
    -d '{"attributes":{"title":"logstash-*","timeFieldName":"@timestamp"}}'
```

只要首次打开Kibana UI，创建的模式将自动标记为默认索引模式。

## 配置

**注意**:配置不会动态重新加载，您需要在更改后重新启动堆栈
组件的配置。

### 如何调整Kibana配置？

Kibana默认配置存储在`kibana/config/kibana.yml`中。

也可以映射整个`config`目录而不是单个文件。

### 如何调整Logstash配置？

Logstash配置存储在`logstash / config / logstash.yml`中。

也可以映射整个`config`目录而不是单个文件，但是你必须知道
Logstash将期待一个
[`log4j2.properties`](https://github.com/elastic/logstash-docker/tree/master/build/logstash/config)文件为自己
日志记录。

### 如何调整Elasticsearch配置？

Elasticsearch配置存储在`elasticsearch/config/elasticsearch.yml`中。

您还可以通过环境变量指定要直接覆盖的选项:

```yml
elasticsearch:

  environment:
    network.host: "_non_loopback_"
    cluster.name: "my-cluster"
```

### 如何扩展Elasticsearch集群？

按照Wiki的说明:[缩小Elasticsearch](https://github.com/deviantony/docker-elk/wiki/Elasticsearch-cluster)

## 存储

### 如何保存Elasticsearch数据？

存储在Elasticsearch中的数据将在容器重启后保留，但在容器删除后不会保留。

为了在删除Elasticsearch容器后保留Elasticsearch数据，您必须安装卷
你的Docker主机。 将`elasticsearch`服务声明更新为:

```yml
elasticsearch:

  volumes:
    - /path/to/storage:/usr/share/elasticsearch/data
```

这会将Elasticsearch数据存储在`/path/to/storage`中。

**注意:**注意这些特定于操作系统的注意事项:
* ** Linux:** [非特权`elasticsearch`用户] [esuser]在Elasticsearch图像中使用，因此
   安装的数据目录必须由uid`1000`拥有。
* ** macOS:**默认的Docker for Mac配置允许从`/Users/`，`/Volumes/`，`/private/`安装文件，
   和`/ tmp`独家。 按照[文档] [macmounts]中的说明添加更多位置。

[esuser]:https://github.com/elastic/elasticsearch-docker/blob/016bcc9db1dd97ecd0ff60c1290e7fa9142f8ddd/templates/Dockerfile.j2#L22
[macmounts]:https://docs.docker.com/docker-for-mac/osxfs/

## 可扩展性

### 如何添加插件？

要将插件添加到任何ELK组件，您必须:

1. 将`RUN`语句添加到相应的`Dockerfile`(例如``RUN logstash-plugin install logstash-filter-json`)
2. 将关联的插件代码配置添加到服务配置(例如Logstash输入/输出)
3. 使用`docker-compose build`命令重建图像

### 如何启用提供的扩展程序？
[`extensions`](扩展)目录中有一些扩展。 这些扩展提供了哪些功能
不是标准弹性堆栈的一部分，但可以用于通过额外的集成来丰富它。

这些扩展的文档在每个单独的子目录中提供，基于每个扩展。 一些
其中需要手动更改默认的ELK配置。

## JVM调优

###如何指定服务使用的内存量？

默认情况下，Elasticsearch和Logstash都以[总主机的1/4开头
内存](https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/parallel.html#default_heap_size)分配给
JVM堆大小。

Elasticsearch和Logstash的启动脚本可以根据环境的值附加额外的JVM选项
变量，允许用户调整每个组件可以使用的内存量:

| Service       | Environment variable |
|---------------|----------------------|
| Elasticsearch | ES_JAVA_OPTS         |
| Logstash      | LS_JAVA_OPTS         |

为了适应内存不足的环境(默认情况下Docker for Mac只有2 GB可用)，堆大小
在 `docker-compose.yml`文件中，每个服务的分配默认为256MB。 如果你想覆盖
默认的JVM配置，编辑 `docker-compose.yml`文件中的匹配环境变量。

例如，要增加Logstash的最大JVM堆大小:

```yml
logstash:

  environment:
    LS_JAVA_OPTS: "-Xmx1g -Xms1g"
```

### 如何启用与服务的远程JMX连接？

对于Java Heap内存(见上文)，您可以指定JVM选项以启用JMX并映射Docker上的JMX端口
主办。

使用以下内容更新`{ES，LS} _JAVA_OPTS`环境变量(我在端口上映射了JMX服务)
18080，你可以改变)。 不要忘记使用您的IP地址更新`-Djava.rmi.server.hostname`选项
Docker主机(替换** DOCKER_HOST_IP **):

```yml
logstash:

  environment:
    LS_JAVA_OPTS: "-Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.port=18080 -Dcom.sun.management.jmxremote.rmi.port=18080 -Djava.rmi.server.hostname=DOCKER_HOST_IP -Dcom.sun.management.jmxremote.local.only=false"
```

## 走得更远

### 使用更新的堆栈版本

要使用与存储库中当前可用的Elastic Stack版本不同的Elastic Stack版本，只需更改版本即可
`.env`文件中的数字，并使用以下代码重建堆栈:

```console
$ docker-compose build
$ docker-compose up
```

**注意**:始终注意[升级说明](https://www.elastic.co/guide/en/elasticsearch/reference/current/setup-upgrade.html)
在执行堆栈升级之前为每个单独的组件。

### 插件和集成

请参阅以下Wiki页面:

* [外部应用程序](https://github.com/deviantony/docker-elk/wiki/External-applications)
* [热门整合](https://github.com/deviantony/docker-elk/wiki/Popular-integrations)

### Docker Swarm

Docker Swarm的实验支持以`docker-stack.yml`文件的形式提供，可以部署在
使用以下命令的现有Swarm集群:

```console
$ docker stack deploy -c docker-stack.yml elk
```

如果部署所有组件时没有任何错误，则以下命令将显示3个正在运行的服务:

```console
$ docker stack services elk
```

**注意:**要在Swarm模式下扩展Elasticsearch，请配置* zen *以使用DNS名称`tasks.elasticsearch`而不是
`elasticsearch`。
