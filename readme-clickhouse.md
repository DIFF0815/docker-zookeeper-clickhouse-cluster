# clickhouse 基础知识
[clickhouse中文文档](https://clickhouse.com/docs/zh)

## clickhouse的连接方式

### http方式
* 默认端口：8123
* 以HTTP接口方式访问时，需注意使用GET方法请求时是默认readonly的。换句话说，若要作修改数据的查询，只能使用POST方法
* 在config.yml中配置
```shell
<http_port>8123</http_port> 
```
### tcp方式
* 默认端口：9000
* 命令行中的clickHouse-client连接是基于TCP方式的
* 在config.yml中配置
```shell
<tcp_port>9000</tcp_port> 
```

### mysql协议方式
* 默认端口：9004
* 在config.yml中配置
```shell
<mysql_port>9004</mysql_port> 
```

## clickhouse数据类型
[clickhouse数据类型](https://clickhouse.com/docs/zh/sql-reference/data-types)

## 数据库引擎
[clickhouse数据库引擎](https://clickhouse.com/docs/zh/engines/database-engines)
### 主要了解mysql、Atomic引擎

## 表引擎
[clickhouse表引擎](https://clickhouse.com/docs/zh/engines/table-engines)
### 主要了解MergeTree系列引擎、Mysql引擎、分布式引擎、视图

## 其他clickhouse文档
[clickhouse基础](https://www.cnblogs.com/dflmg/p/11464748.html)