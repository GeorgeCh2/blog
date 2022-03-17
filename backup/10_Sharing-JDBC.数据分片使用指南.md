# [Sharing-JDBC 数据分片使用指南](https://github.com/GeorgeCh2/blog/issues/10)

## 0. 引入 Maven 依赖
``` xml
<!-- for spring boot -->
<dependency>
    <groupId>org.apache.shardingsphere</groupId>
    <artifactId>sharding-jdbc-spring-boot-starter</artifactId>
    <version>${sharding.sphere.version}</version>
</dependency>
<!-- for spring namespace -->
<dependency>
    <groupId>org.apache.shardingsphere</groupId>
    <artifactId>sharding-jdbc-spring-namespace</artifactId>
    <version>${sharding.sphere.version}</version>
</dependency>
```
删除 druid SpringBoot 依赖
```xml
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>druid-spring-boot-starter</artifactId>
    <version>1.1.4</version>
</dependency>
```
## 1. 基于 Spring Boot 的规则配置
``` YAML
# 数据源命名
spring.shardingsphere.datasource.names = data_source_0,data_source_1,data_source_2,data_source_3
# 每个查询可以打开的最大连接数（项目初始化时，sharding-jdbc 会加载每个数据源所有表的 meta 信息，所以需要将该连接数进行调整，提升启动速度）
spring.shardingsphere.props.max.connections.size.per.query = 10
 
# 数据源配置
# 数据库连接池名称
spring.shardingsphere.datasource.data_source_0.type=com.alibaba.druid.pool.DruidDataSource
# 数据库驱动类名
spring.shardingsphere.datasource.data_source_0.driver-class-name=com.mysql.jdbc.Driver
# 数据库 url
spring.shardingsphere.datasource.data_source_0.url=jdbc:mysql://xxx
# 数据库用户名
spring.shardingsphere.datasource.data_source_0.username = xxx
# 数据库密码
spring.shardingsphere.datasource.data_source_0.password = xxx
# 数据库连接池其他属性
spring.shardingsphere.datasource.data_source_0.filters = mergeStat,com.btime.webser.monitor.druid.falconFilter
spring.shardingsphere.datasource.data_source_0.initial-size = 5
spring.shardingsphere.datasource.data_source_0.min-idle = 5
spring.shardingsphere.datasource.data_source_0.max-active = 100
spring.shardingsphere.datasource.data_source_0.validation-query = SELECT 1
spring.shardingsphere.datasource.data_source_0.test-while-idle = true
spring.shardingsphere.datasource.data_source_0.test-on-borrow = false
spring.shardingsphere.datasource.data_source_0.test-on-return = false
spring.shardingsphere.datasource.data_source_0.time-between-eviction-runs-millis = 60000
spring.shardingsphere.datasource.data_source_0.min-evictable-idle-time-millis = 300000
 
spring.shardingsphere.datasource.data_source_1.type = com.alibaba.druid.pool.DruidDataSource
spring.shardingsphere.datasource.data_source_1.driver-class-name = com.mysql.jdbc.Driver
spring.shardingsphere.datasource.data_source_1.url = jdbc:mysql://xxx
spring.shardingsphere.datasource.data_source_1.username = xxx
spring.shardingsphere.datasource.data_source_1.password = xxx
spring.shardingsphere.datasource.data_source_1.initial-size = 5
spring.shardingsphere.datasource.data_source_1.min-idle = 5
spring.shardingsphere.datasource.data_source_1.max-active = 100
spring.shardingsphere.datasource.data_source_1.validation-query = SELECT 1
spring.shardingsphere.datasource.data_source_1.test-while-idle = true
spring.shardingsphere.datasource.data_source_1.test-on-borrow = false
spring.shardingsphere.datasource.data_source_1.test-on-return = false
spring.shardingsphere.datasource.data_source_1.time-between-eviction-runs-millis = 60000
spring.shardingsphere.datasource.data_source_1.min-evictable-idle-time-millis = 300000
 
spring.shardingsphere.datasource.data_source_2.type = com.alibaba.druid.pool.DruidDataSource
spring.shardingsphere.datasource.data_source_2.driver-class-name = com.mysql.jdbc.Driver
spring.shardingsphere.datasource.data_source_2.url = jdbc:mysql://xxx
spring.shardingsphere.datasource.data_source_2.username = xxxx
spring.shardingsphere.datasource.data_source_2.password = xxx
spring.shardingsphere.datasource.data_source_2.initial-size = 5
spring.shardingsphere.datasource.data_source_2.min-idle = 5
spring.shardingsphere.datasource.data_source_2.max-active = 100
spring.shardingsphere.datasource.data_source_2.validation-query = SELECT 1
spring.shardingsphere.datasource.data_source_2.test-while-idle = true
spring.shardingsphere.datasource.data_source_2.test-on-borrow = false
spring.shardingsphere.datasource.data_source_2.test-on-return = false
spring.shardingsphere.datasource.data_source_2.time-between-eviction-runs-millis = 60000
spring.shardingsphere.datasource.data_source_2.min-evictable-idle-time-millis = 300000
 
spring.shardingsphere.datasource.data_source_3.type = com.alibaba.druid.pool.DruidDataSource
spring.shardingsphere.datasource.data_source_3.driver-class-name = com.mysql.jdbc.Driver
spring.shardingsphere.datasource.data_source_3.url = jdbc:mysql://xxx
spring.shardingsphere.datasource.data_source_3.username = xxx
spring.shardingsphere.datasource.data_source_3.password = xxx
spring.shardingsphere.datasource.data_source_3.initial-size = 5
spring.shardingsphere.datasource.data_source_3.min-idle = 5
spring.shardingsphere.datasource.data_source_3.max-active = 100
spring.shardingsphere.datasource.data_source_3.validation-query = SELECT 1
spring.shardingsphere.datasource.data_source_3.test-while-idle = true
spring.shardingsphere.datasource.data_source_3.test-on-borrow = false
spring.shardingsphere.datasource.data_source_3.test-on-return = false
spring.shardingsphere.datasource.data_source_3.time-between-eviction-runs-millis = 60000
spring.shardingsphere.datasource.data_source_3.min-evictable-idle-time-millis = 300000
 
# 默认数据源
spring.shardingsphere.sharding.default-data-source-name = data_source_0
 
# 数据源分片配置
spring.shardingsphere.sharding.tables.data_source.actual-data-nodes = data_source_$->{0..3}.data_source_$->{0..1023}
spring.shardingsphere.sharding.tables.data_source.database-strategy.inline.sharding-column = fid
spring.shardingsphere.sharding.tables.data_source.database-strategy.inline.algorithm-expression = data_source_$->{fid % 4}
spring.shardingsphere.sharding.tables.data_source.table-strategy.inline.sharding-column = fid
spring.shardingsphere.sharding.tables.data_source.table-strategy.inline.algorithm-expression = data_source_$->{(fid.intdiv(4)) % 1024}
spring.shardingsphere.sharding.tables.data_source_ext.actual-data-nodes = file_ext_$->{0..3}.data_source_ext_$->{0..1023}
spring.shardingsphere.sharding.tables.data_source_ext.database-strategy.inline.sharding-column = fid
spring.shardingsphere.sharding.tables.data_source_ext.database-strategy.inline.algorithm-expression = file_ext_$->{fid % 4}
spring.shardingsphere.sharding.tables.data_source_ext.table-strategy.inline.sharding-column = fid
spring.shardingsphere.sharding.tables.data_source_ext.table-strategy.inline.algorithm-expression = data_source_ext_$->{(fid.intdiv(4)) % 1024}

```
## 2. 注意事项
* 需要配置 mybatis mapper.xml 和 config.xml 所在的位置
   ``` YAML
   #mybatis
   mybatis.config-location = classpath:mybatis/config.xml
   mybatis.mapper-locations = classpath:mybatis/mapper/*.xml
   ```
* groovy 表达式除法得到的结果不是整数，需要使用 intdiv() 来取整
* on duplicate key update 不支持 #{name} 的格式，需要替换成 values(name)
   MySQL 官方文档说明：
   `In assignment value expressions in the ON DUPLICATE KEY UPDATE clause, you can use the VALUES(col_name) function to refer to column values from the INSERT portion of the INSERT ... ON DUPLICATE KEY UPDATE statement.`
　