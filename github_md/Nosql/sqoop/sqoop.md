# RDBMS -> Hadoop 使用
> 大数据时代，怎能不去看看大象呢。数据的使用场景一般分为：OLTP && OLAP类型。 对于OLTP类型的场景，RDBMS中的佼佼者Oracle，Mysql 非常适合，尤其是在电商，移动互联网领域，Mysql在国内更是崭露头角。但是在OLAP类型中，Mysql则表现的不那么尽如人意，（又或者我没有使用好它）。所以，并不是所有场景都适用于RDBMS。这里，我们来聊聊数据库的使用。

## 分类
-----------------------
* **RDBMS**
	
	1. 对于查询实时性很强的应用，我们用Mysql
	
	2. 对于响应要求很高，低延迟的，我们用Mysql 	 
	
* **NoSQL**

	1. Key-value 场景，数据量大，使用Hbase，MongoDB
	
* **cache**

	1. Redis
	
	2. Memcache

* **Hive**

	1. 非实时性查询，离线查询，统计。 使用Hive
	
	2. 离线archive备份 。  我们会使用hdfs 或者 hive来离线备份数据，安全，使用方便。
	
	3. online 保留半年RDBMS数据，满足线上业务需求。超过半年的archive到hdfs，便于统计，审计。
	
以上分类，只是简单粗暴的一种，详细的请看各自的官方文档。
	
## sqoop
----------------

我们今天的主题是，讨论第四种场景。将在线关系数据库离线备份到hdfs，既满足archive需求，又可随时满足开发进行高性能的审计，统计需求。
 
* **sqoop简介**

	1. sqoop 是一个轻量级，可以将RDBMS表 直接导入到 hdfs，hive，hbase的一种工具。
	
	2. 特点：简单，方便。 
	
	3. 缺点：坑比较多，还不是很完善。
	
* **sqoop测试计划**

	1. 在使用的前，由于官方文档的缺陷和不完善，我们需要自己去测一些坑，并将其填好。
	
	2. 我们只需要测试三点
		
		a）表导入到hive中，是否数据和RDBMS一致？
		
		b）表导入到hdfs中，是否数据和RDBMS一致？
  
		c）RDBMS中表的数据类型 和 hive表 类型是否一致？哪里不一致？
		
* **sqoop测试项目**

	* 数值类型：测试均以RDBMS的datatype临界值为基准测试。
	
	类型  |  大小 | 范围（有符号）| 范围 （无符号）|
-----|-------|-------------|------------
TINYINT|1 字节|(-128，127) |(0，255) |
SMALLINT|2 字节|(-32 768，32 767) |(0，65 535) |
MEDIUMINT|3 字节|(-8 388 608，8 388 607) |(0，16 777 215)|
INT|4 字节|(-2 147 483 648，2 147 483 647)|(0，4 294 967 295) |
BIGINT|8 字节|(-9 233 372 036 854 775 808，9 223 372 036 854 775 807) |(0，18 446 744 073 709 551 615) |
FLOAT|4 字节|(-3.402 823 466 E+38，1.175 494 351 E-38)，0，(1.175 494 351 E-38，3.402 823 466 351 E+38) |0，(1.175 494 351 E-38，3.402 823 466 E+38) |
DOUBLE|8 字节|(1.797 693 134 862 315 7 E+308，2.225 073 858 507 201 4 E-308)，0，(2.225 073 858 507 201 4 E-308，1.797 693 134 862 315 7 E+308) |0，(2.225 073 858 507 201 4 E-308，1.797 693 134 862 315 7 E+308)|


	* 字符串类型： 测试主要以换行符，空格，特殊字符为主。

	* 日期和时间类型

	类型  |  大小 | 范围（有符号）| 
-----|-------|-------------|
DATE|3 字节|1000-01-01/9999-12-31 |
DATETIME|8 字节|1000-01-01 00:00:00/9999-12-31 23:59:59 |
DATE|8 字节|1970-01-01 00:00:00/2037 年某时 |


* **sqoop测试结论**

```
1）数值类型
	a）数据类型转换的变化
	TINYINT（mysql）  								 		-> TINYINT(hive)
		注意：
			1) tinyint(1)  ：注意用tinyInt1isBit=false 避免
			2）tinyint(非1) ： tinyint unsigned ， 由于hive中只支持signed， 所以超过最大值127会变成null。 
				解决方法：--map-column-hive f_unsigned=Int
	SMALLINT（mysql）  							     		-> SMALLINT(hive)
		注意：数据没问题。 因为hive会自动将SMALLINT转换成INT 
	MEDIUMINT（mysql）  										-> MEDIUMINT(hive)
		注意：数据没问题。因为hive会MEDIUMINT unsigned 自动转换成bigint，MEDIUMINT signed 自动转换成int signed。
	INT（mysql）      									 	-> INT(hive)
		注意：数据没问题。 因为hive会自动将int unsigned 转换成 bigint signed。
	BIGINT（mysql）    										-> BIGINT(hive)
		注意：
			1）bigint unsigned 有问题。  hive不支持bigint unsigned。
				解决方法：--map-column-java f_unsigned=String --map-column-hive f_unsigned=String
	FLOAT（mysql）	  										-> FLOAT(hive)
		注意：数据没问题。自动转换成double
	DOUBLE（mysql）     										-> DOUBLE(hive)
		注意：数据没问题。
	DECIMAL（mysql）   										-> DECIMAL(hive)
		注意：数据会丢失精度
			解决办法：--map-column-hive field_column=String
2) 字符串类型（mysql）： char，varchar，text
		注意：没问题。但是需要替换掉换行符，字段间隔符号 --fields-terminated-by '\0x01'  --hive-delims-replacement ' '
3）时间类型（mysql）： date，datetime，timestamp
		注意：
			1）没问题。默认会自动全部替换成String。  
			2）0000-00-00 00：00：00 会默认转换成 NULL
4）其他类型（mysql） 
	SET，ENUM 禁止
	BLOB 禁止
	BIT 禁止
	BINARY、VARBINARY 禁止

```

## 总结

------------

* **使用规范**

	* 表必须是按月或者按日分表，且格式必须是： $tablename_YYYYMM or $tablename_YYYYMMDD
	
	* bigint unsigned 必须转换成  bigint
	
	* 字符串类型最好不要有换行符，如果有，也会被我们的脚本自动替换成空格。
	
	* Decimal && tinyint unsigned，时间类型（date，datetime，timestamp） 会被转换成String，所以hive查询时，请不要对字符串进行运算操作，可再程序中完成。
	
	* 禁止使用SET，ENUM ，BLOB，BIT，BINARY，VARBINARY

* **如何使用**

	*  [官方 hive 使用手册](https://cwiki.apache.org/confluence/display/Hive/LanguageManual+Select)
	
	*  [Anjuke hive 平台](http://hue.corp.anjuke.com/metastore/tables/)




