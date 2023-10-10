# ClickHouse同步MySQL数据
[文档](https://blog.csdn.net/ZGL_cyy/article/details/130302767)

## 说明
* MySQL 的用户群体很大，为了能够增强数据的实时性，很多解决方案会利用binlog将数据写入到ClickHouse。为了能够监听binlog事件，我们需要用到类似canal这样的第三方中间件，这无疑增加了系统的复杂度。
* ClickHouse 20.8.2.3版本新增加了MaterializeMySQL的database引擎，该database能映射到MySQL中的某个database，并自动在ClickHouse中创建对应的ReplacingMergeTree。ClickHouse服务做为MySQL副本，读取Binlog并执行DDL和DML请求，实现了基于MySQL Binlog机制的业务数据库实时同步功能

## MaterializeMySQL数据库引擎
* MaterializeMySQL同时支持全量和增量同步，在database创建之初会全量同步MySQL中的表和数据，之后则会通过binlog进行增量同步。
* MaterializeMySQL的database为其所创建的每张ReplacingMergeTree自动增加了_sign 和 _version字段
  * _version：用作 ReplacingMergeTree的ver版本参数，每当监听到 insert、update和delete事件时，在databse内全局自增
  * _sign：则用于标记是否被删除，取值 1 或者 -1

## MaterializeMySQL 支持如下几种 binlog 事件:
  * MYSQL_WRITE_ROWS_EVENT: _sign = 1，_version ++
  * MYSQL_DELETE_ROWS_EVENT: _sign = -1，_version ++
  * MYSQL_UPDATE_ROWS_EVENT: 新数据 _sign = 1
  * MYSQL_QUERY_EVENT: 支持 CREATE TABLE 、DROP TABLE 、RENAME TABLE 等。

## 使用原理
* DDL 查询
  * MySQL DDL 查询被转换成相应的 ClickHouse DDL 查询（ALTER, CREATE, DROP, RENAME）。如果 ClickHouse 不能解析某些 DDL 查询，该查询将被忽略。
* 数据复制
  * MaterializeMySQL 不支持直接插入、删除和更新查询，而是将 DDL 语句进行相应转换：
  * MySQL INSERT 查询被转换为 INSERT with _sign=1。
  * MySQL DELETE 查询被转换为 INSERT with _sign=-1。
  * MySQL UPDATE 查询被转换成 INSERT with _sign=1 和 INSERT with _sign=-1。
* SELECT 查询
  * 如果在 SELECT 查询中没有指定_version，则使用 FINAL 修饰符，返回_version 的最大值对应的数据，即最新版本的数据。
  * 如果在 SELECT 查询中没有指定_sign，则默认使用 WHERE _sign=1，即返回未删除状态（_sign=1)的数据。
* 索引转换
  * ClickHouse 数据库表会自动将 MySQL 主键和索引子句转换为ORDER BY 元组。
  * ClickHouse 只有一个物理顺序，由 ORDER BY 子句决定。如果需要创建新的物理顺序，请使用物化视图。

## 测试
### mysql准备工作
* 打开docker-compose.yml里面的mysql容器
* 执行docker-compose up mysql
* 用数据库管理工具测试连接mysql能不能正常

* 确保 MySQL 开启了 binlog 功能，且格式为 ROW 打开/etc/my.cnf,在[mysqld]下添加：
    ```text
    server-id=1
    log-bin=mysql-bin
    binlog_format=ROW
    ```
* 开启GTID模式,如果如果 clickhouse使用的是20.8prestable之后发布的版本，那么MySQL还需要配置开启GTID模式, 这种方式在mysql主从模式下可以确保数据同步的一致性(主从切换时)。
    ```text
    gtid-mode=on
    enforce-gtid-consistency=1 # 设置为主从强一致性
    log-slave-updates=1 # 记录日志
    ```
* GTID是MySQL复制增强版，从MySQL5.6版本开始支持，目前已经是MySQL主流复制模式。它为每个event分配一个全局唯一ID和序号，我们可以不用关心MySQL集群主从拓扑结构，直接告知MySQL这个GTID即可。
* 修改完配置后，重启容器












