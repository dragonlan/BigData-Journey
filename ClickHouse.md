<H2>一、ClickHouse的搭建</H2>
Clickhouse的多集群搭建网上文章很多，我主要参考https://blog.csdn.net/qq_42873554/article/details/143368665搭建。但这文章里面也不完全对这里简述一下过程。  
尚硅谷的课程文档：https://pan.baidu.com/s/1kQque0SxFTkYXs_xjjJafw#list/path=%2Fsharelink1102835136004-325135947614055%2F%E5%B0%9A%E7%A1%85%E8%B0%B7%E5%A4%A7%E6%95%B0%E6%8D%AE%E6%8A%80%E6%9C%AF%E4%B9%8BClickHouse%2F1.%E7%AC%94%E8%AE%B0&parentPath=%2Fsharelink1102835136004-325135947614055
但这文章里面也不完全对这里简述一下过程。  
<H3>1、安装必要的环境工具和配置，包括JDK,防火墙、主机HOST</H3>  
<H3>2、安装zookeeper</H3> 
  zookeeper只需要3个节点（方便容灾和选举），生产集群中建议占用独立3个节点，不要和ck集群混用。测试集群可以和ck节点混用。  
  zookeeper的启动和停止上面文章中有。  

  zookeeper的访问命令：  
  ./zkCli.sh -server ip:port  
  查看节点命令：help  
  ls /   
  ls -R /  
  获取节点的值：  
  get /clickhouse/tables/diyi/test_table_local/replicas/replica_diyi_I/host  
  
<H3>3、搭建clickhouse节点集群</H3>
下载文件：https://packages.clickhouse.com/rpm/  
需要4个文件：  
a、clickhouse-client、clickhouse-server、clickhouse-common-static、clickhouse-common-static-dbg  
四个文件找相同版本的。  
b、配置集群  
安装后，修改/etc/clickhouse/config.xml文件，通常集群放在/etc/clickhouse-server/config.d/metrika.xml中  
c、创建本地表和分布式表，并插入数据  
  以创建分布式表语句为例  
创建各机器的本地表：  
```CREATE TABLE default.test_table_local ON CLUSTER ck_cluster  
(
    `id`        UInt64 ,  
    `name`         String,  
    `create_time`    Datetime,   
) ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/test_table_local', '{replica}')  
partition by toYYYYMMDD(create_time)  
primary key(id)  
order by (id,name);```
创建分布式表：  <br>
```CREATE TABLE test_table_all on cluster ck_cluster(  
    `id`        UInt64 ,
    `name`         String,
    `create_time`    Datetime  
)ENGINE=Distributed(ck_cluster, default, test_table_local, hiveHash(id));  ```


在/etc/clickhouse-server/config.d/metrika.xml的配置中，通常会配置两个宏  
```<macros>  
        <shard>02</shard>  
        <replica>replica_208</replica>  
</macros> ```
是在创建本地表时使用的，用来标注本节点表格代表的第几个分片的第几个副本。  
副本的地址在zookeeper中可以看到，路径为  
 /clickhouse/tables/｛shard｝/test_table_local/replicas/｛replica｝/host  
 所以每个节点的shard和replica不能同时一样。  

 基于这点，所以不建议在一个节点上配置不同分片的副本。曾经尝试过利用3台机器，配置3分片2副本，配置完启动的时候clickhouse报在节点上已经存在实例而无法启动第二副本的错误。
 网上有各种办法来绕过，有说在节点上启动两个clickhouse，但这样存在端口冲突、配置文件共用等各种问题，也有说把表放在同一个ck的不同数据库中，但总很怪异，所以不建议如此做。

<H2>二、ClickHouse的配套</H2>
1、zookeeper  
clickhouse使用zookeepr来发现有更新，需要同步，但数据的同步不需要经过zookeeper。  
zookeeper中的路径一般有2个，clickhouse和zookeeper
clickhouse对应的路径是创建表格时填写进入的。
/clickhouse/tables/｛shard｝/test_table_local/replicas/｛replica｝/host
clickhouse下有三个节点：sessions, tables, task_queue
/clickhouse/sessions/zookeeper/ 下的uuid应该是对应连接该zookeeper的ck实例
tables是创建表指定的table名称，一次是tables下的分片/表的各种信息，字段可以网上查
replicas：分片下表的副本信息，
log：分片下表的log，副本都是订阅该log，发现该log有变化时，将log加入自己副本的queue中，再执行队列中的任务


<H2>三、ClickHouse遇到的问题</H2>
1、host的问题
网上搭建手册均没提到host问题。
我在搭建时，直接用ip作为config.xml或者config.d/metria.xml中host节点，本以为配置了ip，访问都用ip，实际不是，造成的现象是副本数据不能同步。
经过查询，有些文章说到是因为zookeeper是通过host通讯的，也就是/clickhouse/tables/｛shard｝/test_table_local/replicas/｛replica｝/host中的值是zookeeper自己采集的主机host
所以ck的每个节点需要在/etc/hosts中配置其他副本的host信息，这样节点才能解析host得到对方ip地址，从而建连同步数据。

实际中，我们团队的同事用了另外一种解决办法，就是把每个节点的host改为其ip，也就是/etc/hostname 中存储ip，这样即使不配置/etc/hosts也能连接成功。
我想是gethostbyname这个函数遇到ip形式的host直接返回了输入的ip吧，但是给文本host的时候，不配置/etc/hosts就无法解析到其他副本地址。
