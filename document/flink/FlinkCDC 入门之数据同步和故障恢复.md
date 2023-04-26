## 前言

FlinkCDC 是一款基于 Change Data Capture（CDC）技术的数据同步工具，可以用于将关系型数据库中的数据实时同步到 Flink 流处理中进行实时计算和分析，下图来自[官网](https://ververica.github.io/flink-cdc-connectors/)的介绍。![img](https://raw.githubusercontent.com/zhangjiwei1221/note/master/img/20230319130416.png)

下图<sup>[1](#参考文献)</sup>是 FlinkCDC 与其它常见 开源 CDC 方案的对比：

![2](https://raw.githubusercontent.com/zhangjiwei1221/note/master/img/20230319130409.png)

可以看见的是相比于其它开源产品，FlinkCDC 不仅支持增量同步，还支持全量/全量+增量的同步，同时 FlinkCDC 还支持故障恢复（基于检查点机制实现），能够快速恢复数据同步的进度，并且支持的数据源也很丰富<sup>[2](#参考文献)</sup>（在 2.3 版本已支持 MongoDB、MySQL、OceanBase、Oracle、PostgressSQL、SQLServer、TiDB、Db2 等数据源）。

本文将介绍 FlinkCDC 在数据同步和故障恢复等方面的内容（以 MySQL 和 Oracle 为例），同时完整代码也已上传到[GitHub](https://github.com/zhangjiwei1221/blog/tree/master/flinkcdc)。



## 效果展示

### MySQL

![动画](https://raw.githubusercontent.com/zhangjiwei1221/note/master/img/20230319170826.gif)



### Oracle（相比 MySQL 延迟会稍高）

![动画](https://raw.githubusercontent.com/zhangjiwei1221/note/master/img/20230319171613.gif)



## 数据库配置

### MySQL(5.7)

修改`my.cnf`配置文件（Windows 下是 my.ini 文件），**增加**以下配置内容：

```conf
[mysqld]
# 开启 binlog
log-bin=mysql-bin
# 选择 ROW 模式
binlog-format=ROW
# 对于 MySQL 集群, 不同节点的 server_id 必须不同
server_id=1
# 过期时间
expire_logs_days=30
```

> Tips: 修改完成后需要重启 MySQL 服务

建库建表：

```mysql
# 建库
create database flink;
# 建表
create table flink.`user` (
	`id` bigint(20) not null,
	`username` varchar(20) default null,
	`password` varchar(63) default null,
	`status` int(2) default null,
	`create_time` datetime default null,
	primary key (`id`)
) ENGINE = InnoDB default CHARSET = utf8mb4;
```

创建用户并授权：

```mysql
# 创建用户 flink
CREATE USER flink IDENTIFIED BY 'flink';
# 授权
GRANT SELECT, RELOAD, SHOW DATABASES, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'flink'@'%';
# 将 flink 库的所有权限授权给 flink 用户
GRANT ALL PRIVILEGES ON flink.* TO 'flink'@'%';
# 刷新权限
FLUSH PRIVILEGES;
```



### Oracle(11g)

以 DBA 身份连接：

```shell
# SID 需要根据实际情况进行设置, 比如: XE.
export ORACLE_SID=SID
sqlplus /nolog
CONNECT sys/manager AS SYSDBA
```

配置日志：

```shell
alter system set db_recovery_file_dest_size = 20G;
# 日志文件的地址可以根据自己的情况进行设置
alter system set db_recovery_file_dest = '/opt/oracle/oradata/recovery_area' scope=spfile;
shutdown immediate;
startup mount;
alter database archivelog;
alter database open;
```

确认是否配置成功：

```shell
archive log list;
```

![image-20230319172813540](https://raw.githubusercontent.com/zhangjiwei1221/note/master/img/20230319172813.png)

创建用户并授权：

```sql
CREATE USER flink IDENTIFIED BY flink;
GRANT CREATE SESSION TO flink;
GRANT FLASHBACK ANY TABLE TO flink;
GRANT SELECT ANY TABLE TO flink;
GRANT SELECT_CATALOG_ROLE TO flink;
GRANT EXECUTE_CATALOG_ROLE TO flink;
GRANT SELECT ANY TRANSACTION TO flink;
GRANT CREATE TABLE TO flink;
```

建表并增加日志记录：

```sql
# 建表
CREATE TABLE flink."user" (
	id NUMBER NOT NULL,
    username VARCHAR2(20),
    password VARCHAR2(63),
    status INTEGER,
    create_time TIMESTAMP,
    PRIMARY KEY(id)
);
# 日志配置
ALTER TABLE flink."user" ADD SUPPLEMENTAL LOG DATA (ALL) COLUMNS;
```



## 代码配置

### 运行环境

|      依赖       |  版本  |
| :-------------: | :----: |
|      Java       |   17   |
| flink-connector | 2.1.0  |
|      flink      | 1.13.0 |
|      maven      | 3.6.2  |



### 连接配置

```yaml
flinkcdc:
  data-source:
    # 默认类型为 MySQL
    addr: localhost:3306
    database: flink
    username: flink
    password: flink
    table-list:
      - user
```

> Tips: 关于数据源的连接完整配置属性可参考 DataSourceProperties.java 文件，关于检查点的配置可参考 CheckPointProperties.java 文件



### 恢复点配置

为了实现故障恢复（应用停止运行过程中数据库有增删改操作的情况）的情况，需要在代码中进行恢复点的相关配置：

```java
// 获取配置的恢复点路径, 首次运行不存在会默认进行创建
var saveDir = checkPointProperties.getSaveDir();
var folder = new File(saveDir);
if (!folder.exists() && !folder.isDirectory()) {
    if (!folder.mkdirs()) {
        throw new IllegalStateException("文件夹创建失败");
    }
}
var dataSourceType = dataSourceProperties.getType().name().toLowerCase();
var dataSourceSaveDir = saveDir + File.separator + dataSourceType;
var savepointDir = SavepointUtils.getSavepointRestore(dataSourceSaveDir);
var configuration = new Configuration();
if (savepointDir != null) {
    // 设置恢复点路径
    var savepointRestoreSettings = SavepointRestoreSettings.forPath(savepointDir);
    SavepointRestoreSettings.toConfiguration(savepointRestoreSettings, configuration);
}
// 启用检查点并设置检查点的保存路径
var env = StreamExecutionEnvironment.getExecutionEnvironment(configuration);
env.enableCheckpointing(checkPointProperties.getInterval(), CheckpointingMode.EXACTLY_ONCE);
var checkpointConfig = env.getCheckpointConfig();
checkpointConfig.setCheckpointStorage(checkPointProperties.getStorageType().getPrefix() + dataSourceSaveDir);
```



### 通用注意点

为了避免数值类型显示是一堆字符串，需要增加以下配置：

```java
// 详见 https://github.com/ververica/flink-cdc-connectors/wiki/FAQ(ZH)#%E9%80%9A%E7%94%A8-faq Q5
prop.setProperty("bigint.unsigned.handling.mode","long");
prop.setProperty("decimal.handling.mode","double");
```



### ORACLE 配置注意点

为了避免日志增长过快以及读取日志满的问题，需要增加以下配置：

```java
// 详见 https://github.com/ververica/flink-cdc-connectors/wiki/FAQ(ZH)#oracle-cdc-faq Q1
prop.setProperty("log.mining.strategy", "online_catalog");
prop.setProperty("log.mining.continuous.mine", "true");
```

对于 Oracle 11g，连接配置中需要增加：

```java
// 详见 https://github.com/ververica/flink-cdc-connectors/wiki/FAQ(ZH)#oracle-cdc-faq Q2
prop.setProperty("database.tablename.case.insensitive", "false");
```



## 项目运行及使用介绍

### 下载代码

由于本人将博客相关的示例代码都集中到了一个仓库，因此如果不想拉取整个仓库，推荐使用`GitZip for github`这个插件，就可以只下载部分的文件（选中指定文件后点击右下角的下载按钮）：

![image-20230319183238608](https://raw.githubusercontent.com/zhangjiwei1221/note/master/img/20230319183238.png)



### 使用介绍

对于需要监控的表，只需要创建相应的实体类，并新建一个类继承`AbstractMessageListener`（可重写其中的 create、delete、update、read等方法处理相应的事件）即可，其中 FlickCdcMessageListener 注解内的参数填相应的表名即可监听相应的表变更事件（同时需要在 yaml 文件中 tableList 中增加要监听的表，如果是 Oracle 数据库还需要增加日志配置）：

```java
import cn.butterfly.flinkcdc.annotation.FlickCdcMessageListener;
import cn.butterfly.flinkcdc.pojo.User;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;

/**
 * 用户表消息监听器
 *
 * @author zjw
 * @date 2023-03-14
 */
@Slf4j
@Component
@FlickCdcMessageListener("user")
public class UserMessageListener extends AbstractMessageListener<User> {

    @Override
    public void create(User user) {
        log.info("新增用户: {}", user);
    }

}
```



### 其它注意点

1. FlinkCDC 默认的同步策略是第一次运行先进行全量同步，后续即可进行增量读取，因此表数据量比较大的时候，重写 AbstractMessageListener#read 方法时需要特别注意处理大量数据的情况。
2. 由于 Flink CDC 是根据数据库的事务日志来获取数据更改的，如果恢复点之后发生了数据更改，那么在恢复点之后的数据将被重复读取，因此需要考虑重复读取的情况。



## 总结

本文简单介绍了 FlinkCDC 的数据同步和故障恢复方面的内容，对相关基础知识进行了省略（例如检查点），如果是第一次接触和使用 FlinkCDC，建议先结合官网的[示例](https://ververica.github.io/flink-cdc-connectors/master/content/connectors/index.html)进行学习，同时建议先通读一篇官方的[FAQ](https://github.com/ververica/flink-cdc-connectors/wiki/FAQ(ZH))。



## 参考文献

1. [基于 Flink CDC 实现海量数据的实时同步和转换](https://flink-learning.org.cn/article/detail/eed4549f80e80cc30c69c406cb08b59a)
2. [https://github.com/ververica/flink-cdc-connectors#supported-tested-databases](https://github.com/ververica/flink-cdc-connectors#supported-tested-databases)
