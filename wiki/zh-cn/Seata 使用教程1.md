---
title: Seata笔记-1
author: vikenlove（大菲.Fei）
date: 2019/07/25
keywords: seata、分布式事务、高可用、seata教程
---
# Seata 的起源
### 起源
关于Seata 或许有很多人已经并不陌生，但是也许还有很多人不了解SEATA到底是神马？
Seata 是阿里巴巴基于多年分布式事务处理经验并开源的分布式事务框架。
ben'wen
Seata当前版本0.7.1， github[https://github.com/seata/seata] 欢迎大家试用。
# Seata 入门
### 第一章：服务端&客户端
Seata 包含服务端和客户端两部分，所谓服务端 也就是我们需要单独运行，所有客户端就是我们本地微服务需要纳入到分布式事务中的微服务段。

Seata服务端：[https://github.com/seata/seata/releases]

通过上面路径我们可以下载到Seata-server，解压后包含3部分：bin、conf、lib  
>bin：为服务启动目录：分别为windows启动脚本：seata-server.bat linux启动脚本：seata-server.sh  
>lib：相关运行jar包  
>conf：服务器配置&初始化脚本文件  
>>db_store.sql：基于HA高可用模式即：db模式。（seata-server启动包含2种模式：db、file 两种模式）  
>>file.conf：基于file运行模式涉及到的相关配置  
>>logback.xml：服务端日志文件  
>>nacos-config.py：nacos注册中心脚本脚本（python版）  
>>nacos-config.sh：nacos注册中心初始化脚本文件（linux or windows版本）  
>>nacos-config.txt：初始化脚本文件执行的相关初始化数据  
>>registry.conf：注册服务中心 & 配置中心 服务器配置文件  
 
以上是seata-server0.7.1版本解压后对应的相关文件。 对于以上文件实际上在我们运用当中分为三个部分：    
1.基于db模式，也就是所谓高可用集群模式需要用到的sql文件  
2.基于file模式，服务端本地需要依赖的数据。  
3.基于nacos进行配置管理所需要的初始化脚本。nacos-config.txt 内容其实和file.conf是一样的，使用方式不同。  

客户端：在这里就不多去介绍了，即我们本地工程所使用的微服务项目，但是这里需要注意的是，我们的客户端中需要用到两个文件，分别是  
file.conf  
registry.conf    
为什么我们的客户端也需要这两个文件呢？原因很简单。服务端的registry.conf是当seata-server 以微服务形势启动后，需要让注册服务中心、即：nacos、zookeeper、知道我的服务的provider是谁。而客户端上registry.conf 同样是让当前客户端知道我对应注册的地址是哪  
而file.conf则是相关配置信息文件，后面具体提到。

### 第二章：配置详细
>registry.conf其中包含两部分：注册服务配置 & 配置中心配置
>>registry：注册服务配置目前支持：file 、nacos 、eureka、redis、zk、consul、etcd3、sofa  
>>config：配置中心服务目前支持：file、nacos 、apollo、zk、consul、etcd3  
```Java
registry {
  # file 、nacos 、eureka、redis、zk、consul、etcd3、sofa
  type = "nacos"

  nacos {
    serverAddr = "localhost"
    namespace = "public"
    cluster = "default"
  }
  eureka {
    serviceUrl = "http://localhost:1001/eureka"
    application = "default"
    weight = "1"
  }
  redis {
    serverAddr = "localhost:6379"
    db = "0"
  }
  zk {
    cluster = "default"
    serverAddr = "127.0.0.1:2181"
    session.timeout = 6000
    connect.timeout = 2000
  }
  consul {
    cluster = "default"
    serverAddr = "127.0.0.1:8500"
  }
  etcd3 {
    cluster = "default"
    serverAddr = "http://localhost:2379"
  }
  sofa {
    serverAddr = "127.0.0.1:9603"
    application = "default"
    region = "DEFAULT_ZONE"
    datacenter = "DefaultDataCenter"
    cluster = "default"
    group = "SEATA_GROUP"
    addressWaitTime = "3000"
  }
  file {
    name = "file.conf"
  }
}

config {
  # file、nacos 、apollo、zk、consul、etcd3
  type = "nacos"

  nacos {
    serverAddr = "localhost"
    namespace = "public"
    cluster = "default"
  }
  consul {
    serverAddr = "127.0.0.1:8500"
  }
  apollo {
    app.id = "seata-server"
    apollo.meta = "http://192.168.1.204:8801"
  }
  zk {
    serverAddr = "127.0.0.1:2181"
    session.timeout = 6000
    connect.timeout = 2000
  }
  etcd3 {
    serverAddr = "http://localhost:2379"
  }
  file {
    name = "file.conf"
  }
}

```
所有配置的指定方式均通过：  type = "nacos" type=指定值，进行响应服务的指定。这里的file只的是以本地file文件形式进行注册存储，也就是我们seata-server中需要用到的file.conf中的内容

>file.conf
```Java
transport {
  # tcp udt unix-domain-socket
  type = "TCP"
  #NIO NATIVE
  server = "NIO"
  #enable heartbeat
  heartbeat = true
  #thread factory for netty
  thread-factory {
    boss-thread-prefix = "NettyBoss"
    worker-thread-prefix = "NettyServerNIOWorker"
    server-executor-thread-prefix = "NettyServerBizHandler"
    share-boss-worker = false
    client-selector-thread-prefix = "NettyClientSelector"
    client-selector-thread-size = 1
    client-worker-thread-prefix = "NettyClientWorkerThread"
    # netty boss thread size,will not be used for UDT
    boss-thread-size = 1
    #auto default pin or 8
    worker-thread-size = 8
  }
  shutdown {
    # when destroy server, wait seconds
    wait = 3
  }
  serialization = "seata"
  compressor = "none"
}
service {
  #vgroup->rgroup
  vgroup_mapping.my_test_tx_group = "default"
  #only support single node
  default.grouplist = "127.0.0.1:8091"
  #degrade current not support
  enableDegrade = false
  #disable
  disable = false
  #unit ms,s,m,h,d represents milliseconds, seconds, minutes, hours, days, default permanent
  max.commit.retry.timeout = "-1"
  max.rollback.retry.timeout = "-1"
}

client {
  async.commit.buffer.limit = 10000
  lock {
    retry.internal = 10
    retry.times = 30
  }
  report.retry.count = 5
}

## transaction log store
store {
  ## store mode: file、db
  mode = "db"

  ## file store
  file {
    dir = "sessionStore"

    # branch session size , if exceeded first try compress lockkey, still exceeded throws exceptions
    max-branch-session-size = 16384
    # globe session size , if exceeded throws exceptions
    max-global-session-size = 512
    # file buffer size , if exceeded allocate new buffer
    file-write-buffer-cache-size = 16384
    # when recover batch read size
    session.reload.read_size = 100
    # async, sync
    flush-disk-mode = async
  }

  ## database store
  db {
    ## the implement of javax.sql.DataSource, such as DruidDataSource(druid)/BasicDataSource(dbcp) etc.
    datasource = "dbcp"
    ## mysql/oracle/h2/oceanbase etc.
    db-type = "mysql"
    url = "jdbc:mysql://127.0.0.1:3306/seata-server"
    user = "root"
    password = "123456"
    min-conn = 5
    max-conn = 200
    global.table = "global_table"
    branch.table = "branch_table"
    lock-table = "lock_table"
    query-limit = 100
  }
}
lock {
  ## the lock store mode: local、remote
  mode = "remote"

  local {
    ## store locks in user's database
  }

  remote {
    ## store locks in the seata's server
  }
}
recovery {
  committing-retry-delay = 30
  asyn-committing-retry-delay = 30
  rollbacking-retry-delay = 30
  timeout-retry-delay = 30
}

transaction {
  undo.data.validation = true
  undo.log.serialization = "jackson"
}

## metrics settings
metrics {
  enabled = false
  registry-type = "compact"
  # multi exporters use comma divided
  exporter-list = "prometheus"
  exporter-prometheus-port = 9898
}
```
