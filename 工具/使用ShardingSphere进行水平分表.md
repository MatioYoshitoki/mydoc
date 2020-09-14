### 需求
1. 需要通过传入的参数来自动判断目标表
2. 并不是所有表都需要分表，需要可配置哪些表需要分表，哪些表不需要分表
3. 当不需要分表时，如果目标表不存在，则自动根据参数字段创建对应的目标表。
4. 需要自定义分表规则。(因为部分大表每日新增数据就超过百万，所以目前使用按天分表方案)
5. 需要支持多数据源

### 环境
JDK8
SpringBoot 2.2.2.RELEASE
reactor-kafka 1.2.2.RELEASE
sharding-jdbc-core 4.1.1
mysql8

### 方案
为了提高自由度，采用了JAVA API的方式来实现ShardingSphere DataSource对象的创建。
使用自定义的配置文件来提供对ShardingSphere JDBC的参数配置

### 开始
ps. 假设SpringBoot以及kafka环境均无问题。并与需要分表的t_order 与不需要分表的t_user为例。
引入依赖
```xml
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>8.0.19</version>
</dependency>
<dependency>
    <groupId>org.apache.shardingsphere</groupId>
    <artifactId>sharding-jdbc-core</artifactId>
    <version>4.1.1</version>
</dependency>
```
添加相应的配置
```yaml
event-store:
   datasource:
     name: test
     type: shardingSphere #指定type，代表使用shardingSphere进行数据源管理
     url: jdbc:mysql://127.0.0.1:3306/test?useSSL=false&useUnicode=true&characterEncoding=utf-8&allowPublicKeyRetrieval=true
     username: root
     password: your_password
     rules:
       t_order: # 逻辑表名
         type: standard # 使用standard策略进行分表管理
         actual-data-nodes: test.t_order # 节点
         sharding-column: create_time #根据create_time进行分表
         algorithm: com.example.testsharding.config.TimeShardingAlgorithm #分片规则类
```
分片策略介绍，以下为官网内容：
> ### 分片策略
包含分片键和分片算法，由于分片算法的独立性，将其独立抽离。真正可用于分片操作的是分片键 + 分片算法，也就是分片策略。目前提供 5 种分片策略。
> ###### 标准分片策略
对应 StandardShardingStrategy。提供对 SQ L语句中的 =, >, <, >=, <=, IN 和 BETWEEN AND 的分片操作支持。 StandardShardingStrategy 只支持单分片键，提供 PreciseShardingAlgorithm 和 RangeShardingAlgorithm 两个分片算法。 PreciseShardingAlgorithm 是必选的，用于处理 = 和 IN 的分片。 RangeShardingAlgorithm 是可选的，用于处理 BETWEEN AND, >, <, >=, <=分片，如果不配置 RangeShardingAlgorithm，SQL 中的 BETWEEN AND 将按照全库路由处理。
> ###### 复合分片策略
对应 ComplexShardingStrategy。复合分片策略。提供对 SQL 语句中的 =, >, <, >=, <=, IN 和 BETWEEN AND 的分片操作支持。 ComplexShardingStrategy 支持多分片键，由于多分片键之间的关系复杂，因此并未进行过多的封装，而是直接将分片键值组合以及分片操作符透传至分片算法，完全由应用开发者实现，提供最大的灵活度。
> ###### 行表达式分片策略
对应 InlineShardingStrategy。使用 Groovy 的表达式，提供对 SQL 语句中的 = 和 IN 的分片操作支持，只支持单分片键。 对于简单的分片算法，可以通过简单的配置使用，从而避免繁琐的 Java 代码开发，如: t_user_$->{u_id % 8} 表示 t_user 表根据 u_id 模 8，而分成 8 张表，表名称为 t_user_0 到 t_user_7。 详情请参见行表达式。
> ###### Hint分片策略
对应 HintShardingStrategy。通过 Hint 指定分片值而非从 SQL 中提取分片值的方式进行分片的策略。
> ###### 不分片策略
对应 NoneShardingStrategy。不分片的策略。

我们使用的是标准分片侧率，也就是standard。
使用standard策略的时候需要提供分片规则类，也就是 com.example.testsharding.config.TimeShardingAlgorithm
以下为com.example.testsharding.config.TimeShardingAlgorithm的代码
```java
package com.example.testsharding.config;


import com.example.testshardingsphere.util.ParaseShardingKeyTool;
import org.apache.shardingsphere.api.sharding.standard.PreciseShardingAlgorithm;
import org.apache.shardingsphere.api.sharding.standard.PreciseShardingValue;
import cn.hutool.core.date.DateUtil;

import java.util.Collection;


public class TimeShardingTableAlgorithm implements PreciseShardingAlgorithm<Long> {
    @Override
    public String doSharding(Collection<String> availableTargetNames, PreciseShardingValue<Long> shardingValue) {
        return shardingValue.getLogicTableName() +
                "_" + DateUtil.format(new Date(shardingValue.getValue()), "yyyyMMdd");
    }
}
```

通过自定义配置获取DataSource实例
```Java
@Bean
public DataSource getDataSource(JDBCConfig config) {
  Map<String, DataSource> dataSourceMap = Maps.newHashMap();
  BasicDataSource dataSource = new BasicDataSource();
  dataSource.setDriverClassName("com.mysql.cj.jdbc.Driver");
  dataSource.setUrl(config.getUrl());
  dataSource.setUsername(config.getUsername());
  dataSource.setPassword(config.getPassword());
  dataSourceMap.put(config.getName(), dataSource);

  ShardingRuleConfiguration shardingRuleConfiguration = new ShardingRuleConfiguration();
  List<TableRuleConfiguration> tableRules = new ArrayList<>(Collections.emptyList());

  config.getRules().forEach((key, value) -> {
      TableRuleConfiguration tableRuleConfig = new TableRuleConfiguration(key, value.getActualDataNodes());
      StandardShardingStrategyConfiguration tableStrategy;
      try {
          tableStrategy = new StandardShardingStrategyConfiguration(value.getShardingColumn(), (PreciseShardingAlgorithm) Class.forName(value.getAlgorithm()).newInstance());
      }catch (Exception e){
          e.printStackTrace();
          return;
      }
      tableRuleConfig.setTableShardingStrategyConfig(tableStrategy);
      tableRules.add(tableRuleConfig);
  });
  shardingRuleConfiguration.setTableRuleConfigs(tableRules);

  return ShardingDataSourceFactory.createDataSource(dataSourceMap, shardingRuleConfiguration, new Properties());
}
```
