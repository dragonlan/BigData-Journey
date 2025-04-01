<H2>一、ClickHouse的搭建
Clickhouse的多集群搭建网上文章很多，我主要参考https://blog.csdn.net/qq_42873554/article/details/143368665搭建。但这文章里面也不完全对这里简述一下过程。
尚硅谷的课程文档：https://pan.baidu.com/s/1kQque0SxFTkYXs_xjjJafw#list/path=%2Fsharelink1102835136004-325135947614055%2F%E5%B0%9A%E7%A1%85%E8%B0%B7%E5%A4%A7%E6%95%B0%E6%8D%AE%E6%8A%80%E6%9C%AF%E4%B9%8BClickHouse%2F1.%E7%AC%94%E8%AE%B0&parentPath=%2Fsharelink1102835136004-325135947614055
但这文章里面也不完全对这里简述一下过程。
<H3>1、安装必要的环境工具和配置，包括JDK,防火墙、主机HOST
<H3>2、安装zookeeper
  zookeeper只需要3个节点（方便容灾和选举），生产集群建议独立3个节点，不要和ck集群混用。测试集群可以和ck节点混用。
  zookeeper的启动和停止上面文章中有。

  zookeeper的访问命令：
  ./zkCli.sh -server ip:port
  查看节点命令：help
  ls / 
  ls -R /
  获取节点的值：
  get /clickhouse/tables/diyi/test_table_local/replicas/replica_diyi_I/host
  
<H3>3、搭建clickhouse节点集群
下载文件：https://packages.clickhouse.com/rpm/
需要4个文件：
clickhouse-client、clickhouse-server、clickhouse-common-static、clickhouse-common-static-dbg
四个文件找相同版本的。

<H3>配置集群
安装后，修改/etc/clickhouse/config.xml文件，通常集群放在/etc/clickhouse-server/config.d/metrika.xml中
  
<H2>二、ClickHouse的配套
<H2>三、ClickHouse遇到的问题
