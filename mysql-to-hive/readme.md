StreamSets 实现 MySQL 增量更新到 Hive
===
过程包括三个 SDC 阶段：Hive Metadata processor、Hive Metastore destination、Hadoop FS or MapR FS destinations

![image_0](https://user-images.githubusercontent.com/32835617/57267584-327cb000-70b3-11e9-8541-9ef1307692b9.png)
![image_1](https://user-images.githubusercontent.com/32835617/57267586-33154680-70b3-11e9-86a3-6f8ffef4c006.png)

Hive Metadate 元数据处理器和 Hive Metastore 目标协同工作，以协调传入记录结构与Hive中相应表模式之间的任何差异。如果该表尚不存在，则创建该表。如果传入记录中的字段不作为Hive表中的列存在，则更新Hive架构以匹配新记录结构。

创建源表
---
```
创建数据库表，如果已经存在就可以跳过此过程
CREATE TABLE `mysql2hive` (
  `id` int(10) NOT NULL AUTO_INCREMENT,
  `station_code` varchar(20) DEFAULT NULL COMMENT '加油站编码',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```
创建管道
---
在浏览器中，登录SDC并创建新管道。添加JDBC查询使用者源并对其进行配置：
#### JDBC 选项卡
* **JDBC Connection String:** 连接数据库的URL，因环境而异 `jdbc:mysql://192.168.8.234:3306/cqxs`
* **SQL Query:** `select * from mysql2hive where id > ${OFFSET} order by id`
* **Initial Offset:** `0`
* **Offset Column:** `id`

Note - 在此以及所有其他选项卡中，将未列出的属性保留为其默认配置。
#### Credentials 选项卡
* **Username:** 您的MySQL用户名
* **Password:** 您的MySQL密码
#### Legacy Drivers
* **JDBC Driver Class Name:** `com.mysql.jdbc.Driver`
#### Advanced 选项卡
* **Create JDBC Namespace Headers:** 已启用

**注意：** 启用“`高级`”选项卡上的“`创建JDBC命名空间标题`”属性 - 如果没有它，Hive元数据处理器将无法使用Decimal类型！此模式使用传入记录中的字段类型信息在Hive中创建相应的模式。从JDBC读取的数据库记录包含此信息，从Avro或SDC数据格式的文件或消息队列中读取的记录也是如此，但分隔的数据格式则不包含。在提取分隔数据文件时，该解决方案仍会创建Hive表，但默认情况下，所有列都将具有STRING类型。生产环境建议配置管道 Error Records (错误日志)输出至文件/队列中。
具体如下：

![1](https://user-images.githubusercontent.com/32835617/57267868-41179700-70b4-11e9-8f7f-51b4053a6902.png)

![2](https://user-images.githubusercontent.com/32835617/57267871-41b02d80-70b4-11e9-8fca-27da05d5c477.png)

配置Hive Metadata处理器
---
分析传入数据的结构，将其与Hive Metastore架构进行比较，并创建捕获Hive表结构中所需更改的元数据记录。<br>
**注意：** SDC要求可以在单个目录中访问Hadoop和Hive的配置文件。在添加Hive和Hadoop pipeline阶段，在SDC的下resources目录下创建一个名为hadoop-conf的文件夹，链接`core-site.xml`，`hdfs-site.xml`以及`hive-site.xml`配置文件到hadoop-conf目录下。具体命令如下：
```
cd <your SDC resources directory>
mkdir hadoop-conf
cd hadoop-conf
ln -s <path to hadoop config>/core-site.xml
ln -s <path to hadoop config>/hdfs-site.xml
ln -s <path to hive config>/hive-site.xml
cd ..
# Omit the following step if you did not create an sdc system user and group
chown -R sdc:sdc hadoop-conf
```
**Note：** 测试环境中 SDC reources directory 为平台中StreamSets配置 Resources directory 路径，path to hadoop config 和 path to hive config 位于 /etc/hadoop/conf 和 /etc/hive/conf
#### Hive选项卡
* **JDBC URL:** 它具有表单，`jdbc:hive2://localhost:10000/default`但可能会因您的环境而异。<br>特别是:如果您使用具有默认配置的MapR，则需要在URL中指定用户名和密码，因此:`jdbc:hive2://localhost:10000/default;user=<username>;password=<password>`。<br>如果使用Kerberos，则可能需要添加主体参数以指定Hive Kerberos用户。<br>测试环境:`jdbc:hive2://dcc-nn2.dcc.com:10001/cqxs;principal=hive/dcc-nn2.dcc.com@DCC.COM`
  
* **JDBC Driver Name:** 对于Apache Hadoop环境，这将是`org.apache.hive.jdbc.HiveDriver`，否则您应该为您的分发指定特定的驱动程序类。
* **Hadoop Configuration Directory:** `hadoop-conf`
#### Table标签
* **Database Expression:** `default`- 如果您希望使用不同的Hive数据库名称，请更改此项。
* **Table Name:** `${record:attribute('jdbc.tables')}`
* **分区配置:** 点击“ - ”按钮删除dt条目。没有使用。

如果使用分区，则需建立分区表，配置如下：
#### Partition Configuration标签
* **Partition Column Name:** `day_batch_date` - 分区的名称
* **Partition Value Type:** `STRING` - 分区字段的类型
* **Partition Value Expression:** `${record:value('/trade_date')}` - obtain partition value from record(注意分区名称和 record 的字段名称不能相同)

![3](https://user-images.githubusercontent.com/32835617/61018067-13a6dc80-a3c8-11e9-935c-950c34c4e49a.png)

注意使用`${record:attribute('jdbc.tables')}`作为表名 - 这将通过管道将MySQL表名传递给Hive。
具体如下：

![4](https://user-images.githubusercontent.com/32835617/57267872-4248c400-70b4-11e9-948c-2cd613ca9ac8.png)

Hive元数据处理器在其＃1输出流和＃2上的元数据上发出数据记录。

Hadoop FS
---
连接到Hive元数据处理器的＃1输出
#### Hadoop FS选项卡
* **Hadoop FS URI:** 如果SDC使用hdfs-site.xml配置文件中的地址连接，则可以将其保留为空。否则，Hadoop的FS指定形式的URI `hdfs://hostname/`和MAPR FS指定为 `maprfs:///mapr/my.cluster.com/`。
* **HDFS用户:** 适当的HDFS用户名
* **Kerberos身份验证:** 如果您的Hadoop环境受到保护，您应该启用此功能
* **Hadoop FS配置目录:** 您可能需要更改此设置以适合您的环境。该目录必须包含`core-site.xml`和`hdfs-site.xml`配置文件。
#### Output Files选项卡
* **Directory in Header:** 已启用
* **Max Records in File:** 1 
* **Use Roll Attribute:** 已启用
* **Roll Attribute Name:** roll
#### Data Format标签
* **Data Format:** Avro
* **Avro Schema Location:** In Record Header

**注意：** 目标将继续写入文件，直到满足这五个条件中的第一个：
* 已写入“文件中的最大记录”中指定的记录数（零表示没有最大值）
* 已达到指定的“最大文件大小”（同样，零表示没有最大值）
* 没有为指定的“空闲超时”写入记录
* 处理具有指定卷标头属性的记录
* 管道停止了

当Hive元数据处理器检测到架构更改时，它会设置roll header属性以向目标发出数据文件应“滚动”的信号 - 也就是说，当前文件已关闭且新文件已打开。
我们将文件中的Max Records设置为1，因此目标在写入每条记录后立即关闭文件，因为我们希望立即查看数据。如果我们保留了默认值，我们可能会在Hive写入后一小时内看不到Hive中的某些数据。这可能适用于生产部署，但是这将是一个非常耗时的教程！
具体如下：

![5](https://user-images.githubusercontent.com/32835617/57267873-4379f100-70b4-11e9-9487-cd24c772375a.png)

![6](https://user-images.githubusercontent.com/32835617/57267874-4379f100-70b4-11e9-9b2d-250d3ea2215c.png)

![7](https://user-images.githubusercontent.com/32835617/57267875-44128780-70b4-11e9-8fd2-c24479ad85ef.png)

Hive Metastore
---
连接到Hive元数据处理器的＃2输出
#### Hive选项卡
* **JDBC URL:** 将其设置为与Hive元数据处理器中的值相同。
* **JDBC Driver Name:** 将其设置为与Hive元数据处理器中的值相同。
* **Hadoop配置目录:** hadoop-conf
#### Advanced选项卡
* **存储为Avro:** 已启用

注意：当前不会在Hive元数据处理器的＃2输出上看到元数据记录，但可以在Hive Metastore的输入上看到。

运行
---
![8](https://user-images.githubusercontent.com/32835617/57267876-44ab1e00-70b4-11e9-98d8-c39f2705f78b.png)

输入2条，输出3条，为什么多一条输出呢？
2个数据记录被发送到Hadoop或MapR FS目的地，而1个元数据记录被发送到Hive Metastore，其中包含创建表的指令。
SDC没有通知Impala元数据更改，因此在我们第一次查询表之前需要使用`invalidate metadata`命令。
添加2条新纪录，过几秒记录计数会增加

![9](https://user-images.githubusercontent.com/32835617/57267878-4543b480-70b4-11e9-8ed3-c4eedd458ee3.png)

还有三个数据记录输出，但由于架构未更改，因此不再生成元数据记录。转到Impala Shell并查询表 - 可以使用refresh，而不是更昂贵的invalidate，因为表结构没有改变 - 只是添加了新的数据文件。
如果修改表结构，添加两个字段`ALTER TABLE mysql2hive ADD COLUMN latitude DECIMAL(8,6), ADD COLUMN longitude DECIMAL(9,6);`<br>
查看SDC监视面板，将看不到任何更改。虽然我们在MySQL中更改了表，但没有添加任何记录，因此SDC不知道这一变化。通过添加纬度和经度字段的新记录来解决这个问题：
`INSERT INTO mysql2hive(station_code,latitude,longitude) VALUES('NA06',29.538607,106.473115);
INSERT INTO mysql2hive(station_code,latitude,longitude) VALUES('NB02',29.652442,107.353779);`
几秒钟后，SDC监控面板将显示6条输入记录和8条输出记录 - 另外2条数据记录和4条以前的元数据记录：

![10](https://user-images.githubusercontent.com/32835617/57267879-4543b480-70b4-11e9-8014-93304f4cbe25.png)

在Impala Shell中，您需要再次刷新Impala的缓存，然后才能看到表中的其他列。

出现的问题
---
* 使用Kerberos认证在连接Hive的JDBC URL需要添加主体参数以指定Hive Kerberos用户 <br> 例如：`jdbc:hive2://dcc-nn2.dcc.com:10001/cqxs;principal=hive/dcc-nn2.dcc.com@DCC.COM`<br> 
* No valid privileges User sdc does not have privileges for DESCDATABASE The required privileges: Server=server1->Db=cqxs->action=select;Server=server1->Db=cqxs->action=insert;' Mismatch: Actual: {} -- 权限问题：StreamSets 默认的用户是 sdc，sdc 对 Hive 没有读写的权限。解决办法：在 Hive 中创建 sdc 角色，并且赋予读写的权限。

Hive创建sdc用户
---
* 在所有集群节点执行下面命令<CDH集群默认创建><br>
	添加组：`groupadd sdc`<br>
	添加用户：`useradd sdc -G sdc`<br>
	添加密码：`echo 'Chongqing2018!' | passwd --stdin sdc`<br>
* 创建角色和权限<br>
	1、登录 HiveServer2 所在节点<br>
	执行 `kinit -kt dccbdp.keytab  dccbdp`<br>
	2、通过beeline链接，命令如下：<br>
	`beeline -u "jdbc:hive2://dcc-nn2.dcc.com:10001/default;principal=hive/dcc-nn2.dcc.com@DCC.COM;hive.server2.proxy.user=hive"` <br>
	3、创建角色并赋予hive用户权限 <br>
	赋予 sdc 角色所有的权限：<br>
	`CREATE ROLE sdc;`<br>
	`GRANT ALL ON SERVER server1 TO ROLE sdc;`<br>
	`GRANT ROLE sdc TO GROUP hive; ` <br>
	`GRANT ROLE sdc TO GROUP dccbdp_g; <dccbdp_g 组拥有 Hive 的所有权限>`<br>
	赋予 cqxs_role 角色 select 的权限：<br>
	`grant select on database cqxs to role cqxs_role;`<br>
	`grant role cqxs_role to group cqxs_g;`<br>
* 在 Hue 中创建和 linux 系统对应的用户和组，并赋予组需要的权限

补充
---
#### configure a Hive / Impala JDBC driver for Data Collector
StreamSets comes bundled with the open-source Hive JDBC driver. Using the default driver, URLs will look like the following:
* Unsecured: jdbc:hive2://hive-server2-host.company.com:10000/dbName
* LDAP Auth: jdbc:hive2://hive-server2-host.company.com:10000/dbName;user=username;password=*
* Kerberos: jdbc:hive2://hive-server2-host.company.com:10000/dbName;principal=hive/hive-server2-host.company.com@COMPANY.COM
* SSL + Kerberos: jdbc:hive2://hive-server2-host.company.com:10000/dbName;principal=hive/hive-server2-host.company.com@COMPANY.COM;ssl=true;sslTrustStore=/path/to/truststore.jks
#### Cloudera also provides a Hive driver.  Using the Cloudera Hive driver:
* Unsecured: jdbc:hive2://hive-server2-host.company.com:10000/dbName
* LDAP Auth: jdbc:hive2://hive-server2-host.company.com:10000/dbName;AuthMech=3;UID=username;PWD=*
* Kerberos: jdbc:hive2://hive-server2-host.company.com:10000/dbName;AuthMech=1;KrbRealm=COMPANY.COM;KrbHostFQDN=hive-server2-host.company.com;KrbServiceName=hive
* SSL + Kerberos: jdbc:hive2://hive-server2-host.company.com:10000/dbName;AuthMech=1;KrbRealm=COMPANY.COM;KrbHostFQDN=hive-server2-host.company.com;KrbServiceName=hive;SSL=1;SSLKeyStore=/path/to/truststore.jks
#### Cloudera also has an Impala driver. Using the Cloudera Impala driver:
* Unsecured: jdbc:impala://impala-daemon-host.company.com:21050/dbName
* LDAP Auth: jdbc:impala://impala-daemon-host.company.com:21050/dbName;AuthMech=3;UID=username;PWD=*
* Kerberos: jdbc:impala://impala-daemon-host.company.com:21050/dbName;AuthMech=1;KrbRealm=COMPANY.COM;KrbHostFQDN=impala-daemon-host.company.com;KrbServiceName=impala
* SSL + Kerberos: jdbc:impala://impala-daemon-host.company.com:21050/dbName;AuthMech=1;KrbRealm=COMPANY.COM;KrbHostFQDN=impala-daemon-host.company.com;KrbServiceName=impala;SSL=1;SSLKeyStore=/path/to/truststore.jks

