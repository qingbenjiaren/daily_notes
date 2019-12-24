# 概述

配置是整个Sharding-JDBC的核心，是Sharding-JDBC中唯一与应用开发者打交道的模块。配置模块也是Sharding-JDBC的门户，通过它可以快速清晰的理解Sharding-JDBC所提供的功能。

Sharding-JDBC提供了4种配置方式，用于不同的使用场景。通过配置，应用开发者可以灵活的使用分库分表、读写分离以及分库分表+读写分离公用。

# JAVA配置

## 数据分片

以下配置中DataSourceUtils的实现为[DataSourceUtil](https://github.com/geomonlin/incubator-shardingsphere-example/blob/4.0.0-RC2/example-core/example-api/src/main/java/org/apache/shardingsphere/example/core/api/DataSourceUtil.java)

ModuloShardingTableAlgorithm类需要用户自定义实现，详细例子[ModuloShardingTableAlgorithm](https://github.com/geomonlin/incubator-shardingsphere-example/tree/dev/example-core/config-utility/src/main/java/org/apache/shardingsphere/example/algorithm)

```java
DataSource getShardingDataSource() throws SQLException {
         ShardingRuleConfiguration shardingRuleConfig = new ShardingRuleConfiguration();
         shardingRuleConfig.getTableRuleConfigs().add(getOrderTableRuleConfiguration());
         shardingRuleConfig.getTableRuleConfigs().add(getOrderItemTableRuleConfiguration());
         shardingRuleConfig.getBindingTableGroups().add("t_order, t_order_item");
         shardingRuleConfig.getBroadcastTables().add("t_config");
         shardingRuleConfig.setDefaultDatabaseShardingStrategyConfig(new InlineShardingStrategyConfiguration("user_id", "ds${user_id % 2}"));
         shardingRuleConfig.setDefaultTableShardingStrategyConfig(new StandardShardingStrategyConfiguration("order_id", new ModuloShardingTableAlgorithm()));
         return ShardingDataSourceFactory.createDataSource(createDataSourceMap(), shardingRuleConfig, new Properties());
     }
     
     private static KeyGeneratorConfiguration getKeyGeneratorConfiguration() {
         KeyGeneratorConfiguration result = new KeyGeneratorConfiguration("SNOWFLAKE", "order_id");
         return result;
     }
     
     TableRuleConfiguration getOrderTableRuleConfiguration() {
         TableRuleConfiguration result = new TableRuleConfiguration("t_order", "ds${0..1}.t_order${0..1}");
         result.setKeyGeneratorConfig(getKeyGeneratorConfiguration());
         return result;
     }
     
     TableRuleConfiguration getOrderItemTableRuleConfiguration() {
         TableRuleConfiguration result = new TableRuleConfiguration("t_order_item", "ds${0..1}.t_order_item${0..1}");
         return result;
     }
     
     Map<String, DataSource> createDataSourceMap() {
         Map<String, DataSource> result = new HashMap<>();
         result.put("ds0", DataSourceUtil.createDataSource("ds0"));
         result.put("ds1", DataSourceUtil.createDataSource("ds1"));
         return result;
     }
```

## 读写分离

```java
DataSource getMasterSlaveDataSource() throws SQLException {
         MasterSlaveRuleConfiguration masterSlaveRuleConfig = new MasterSlaveRuleConfiguration("ds_master_slave", "ds_master", Arrays.asList("ds_slave0", "ds_slave1"));
         return MasterSlaveDataSourceFactory.createDataSource(createDataSourceMap(), masterSlaveRuleConfig, new Properties());
     }
     
     Map<String, DataSource> createDataSourceMap() {
         Map<String, DataSource> result = new HashMap<>();
         result.put("ds_master", DataSourceUtil.createDataSource("ds_master"));
         result.put("ds_slave0", DataSourceUtil.createDataSource("ds_slave0"));
         result.put("ds_slave1", DataSourceUtil.createDataSource("ds_slave1"));
         return result;
     }
```

## 数据脱敏

```java
DataSource getEncryptDataSource() throws SQLException {
        return EncryptDataSourceFactory.createDataSource(DataSourceUtil.createDataSource("demo_ds"), getEncryptRuleConfiguration(), new Properties());
    }

    private static EncryptRuleConfiguration getEncryptRuleConfiguration() {
        Properties props = new Properties();
        props.setProperty("aes.key.value", "123456");
        EncryptorRuleConfiguration encryptorConfig = new EncryptorRuleConfiguration("AES", props);
        EncryptColumnRuleConfiguration columnConfig = new EncryptColumnRuleConfiguration("plain_pwd", "cipher_pwd", "", "aes");
        EncryptTableRuleConfiguration tableConfig = new EncryptTableRuleConfiguration(Collections.singletonMap("pwd", columnConfig));
        EncryptRuleConfiguration encryptRuleConfig = new EncryptRuleConfiguration();
        encryptRuleConfig.getEncryptors().put("aes", encryptorConfig);
        encryptRuleConfig.getTables().put("t_encrypt", tableConfig);
        return encryptRuleConfig;
    }
```

## 数据分片+读写分离

```java
DataSource getDataSource() throws SQLException {
        ShardingRuleConfiguration shardingRuleConfig = new ShardingRuleConfiguration();
        shardingRuleConfig.getTableRuleConfigs().add(getOrderTableRuleConfiguration());
        shardingRuleConfig.getTableRuleConfigs().add(getOrderItemTableRuleConfiguration());
        shardingRuleConfig.getBindingTableGroups().add("t_order, t_order_item");
        shardingRuleConfig.getBroadcastTables().add("t_config");
        shardingRuleConfig.setDefaultDatabaseShardingStrategyConfig(new StandardShardingStrategyConfiguration("user_id", new PreciseModuloShardingDatabaseAlgorithm()));
        shardingRuleConfig.setDefaultTableShardingStrategyConfig(new StandardShardingStrategyConfiguration("order_id", new PreciseModuloShardingTableAlgorithm()));
        shardingRuleConfig.setMasterSlaveRuleConfigs(getMasterSlaveRuleConfigurations());
        return ShardingDataSourceFactory.createDataSource(createDataSourceMap(), shardingRuleConfig, new Properties());
    }
    
    private static KeyGeneratorConfiguration getKeyGeneratorConfiguration() {
        KeyGeneratorConfiguration result = new KeyGeneratorConfiguration("SNOWFLAKE", "order_id");
        return result;
    }
    
    TableRuleConfiguration getOrderTableRuleConfiguration() {
        TableRuleConfiguration result = new TableRuleConfiguration("t_order", "ds_${0..1}.t_order_${[0, 1]}");
        result.setKeyGeneratorConfig(getKeyGeneratorConfiguration());
        return result;
    }
    
    TableRuleConfiguration getOrderItemTableRuleConfiguration() {
        TableRuleConfiguration result = new TableRuleConfiguration("t_order_item", "ds_${0..1}.t_order_item_${[0, 1]}");
        return result;
    }
    
    List<MasterSlaveRuleConfiguration> getMasterSlaveRuleConfigurations() {
        MasterSlaveRuleConfiguration masterSlaveRuleConfig1 = new MasterSlaveRuleConfiguration("ds_0", "demo_ds_master_0", Arrays.asList("demo_ds_master_0_slave_0", "demo_ds_master_0_slave_1"));
        MasterSlaveRuleConfiguration masterSlaveRuleConfig2 = new MasterSlaveRuleConfiguration("ds_1", "demo_ds_master_1", Arrays.asList("demo_ds_master_1_slave_0", "demo_ds_master_1_slave_1"));
        return Lists.newArrayList(masterSlaveRuleConfig1, masterSlaveRuleConfig2);
    }
    
    Map<String, DataSource> createDataSourceMap() {
        final Map<String, DataSource> result = new HashMap<>();
        result.put("demo_ds_master_0", DataSourceUtil.createDataSource("demo_ds_master_0"));
        result.put("demo_ds_master_0_slave_0", DataSourceUtil.createDataSource("demo_ds_master_0_slave_0"));
        result.put("demo_ds_master_0_slave_1", DataSourceUtil.createDataSource("demo_ds_master_0_slave_1"));
        result.put("demo_ds_master_1", DataSourceUtil.createDataSource("demo_ds_master_1"));
        result.put("demo_ds_master_1_slave_0", DataSourceUtil.createDataSource("demo_ds_master_1_slave_0"));
        result.put("demo_ds_master_1_slave_1", DataSourceUtil.createDataSource("demo_ds_master_1_slave_1"));
        return result;
    }
```

## 数据分片+数据脱敏

```java
public DataSource getDataSource() throws SQLException {
        ShardingRuleConfiguration shardingRuleConfig = new ShardingRuleConfiguration();
        shardingRuleConfig.getTableRuleConfigs().add(getOrderTableRuleConfiguration());
        shardingRuleConfig.getTableRuleConfigs().add(getOrderItemTableRuleConfiguration());
        shardingRuleConfig.getTableRuleConfigs().add(getOrderEncryptTableRuleConfiguration());
        shardingRuleConfig.getBindingTableGroups().add("t_order, t_order_item");
        shardingRuleConfig.setDefaultDatabaseShardingStrategyConfig(new InlineShardingStrategyConfiguration("user_id", "demo_ds_${user_id % 2}"));
        shardingRuleConfig.setDefaultTableShardingStrategyConfig(new StandardShardingStrategyConfiguration("order_id", new PreciseModuloShardingTableAlgorithm()));
        shardingRuleConfig.setEncryptRuleConfig(getEncryptRuleConfiguration());
        return ShardingDataSourceFactory.createDataSource(createDataSourceMap(), shardingRuleConfig, new Properties());
    }
    
    private static TableRuleConfiguration getOrderTableRuleConfiguration() {
        TableRuleConfiguration result = new TableRuleConfiguration("t_order", "demo_ds_${0..1}.t_order_${[0, 1]}");
        result.setKeyGeneratorConfig(getKeyGeneratorConfiguration());
        return result;
    }
    
    private static TableRuleConfiguration getOrderItemTableRuleConfiguration() {
        TableRuleConfiguration result = new TableRuleConfiguration("t_order_item", "demo_ds_${0..1}.t_order_item_${[0, 1]}");
        result.setEncryptorConfig(new EncryptorConfiguration("MD5", "status", new Properties()));
        return result;
    }
    
    private static EncryptRuleConfiguration getEncryptRuleConfiguration() {
        Properties props = new Properties();
        props.setProperty("aes.key.value", "123456");
        EncryptorRuleConfiguration encryptorConfig = new EncryptorRuleConfiguration("AES", props);
        EncryptColumnRuleConfiguration columnConfig = new EncryptColumnRuleConfiguration("plain_order", "cipher_order", "", "aes");
        EncryptTableRuleConfiguration tableConfig = new EncryptTableRuleConfiguration(Collections.singletonMap("order_id", columnConfig));
        EncryptRuleConfiguration encryptRuleConfig = new EncryptRuleConfiguration();
        encryptRuleConfig.getEncryptors().put("aes", encryptorConfig);
        encryptRuleConfig.getTables().put("t_order", tableConfig);
        return encryptRuleConfig;
    }
    
    private static Map<String, DataSource> createDataSourceMap() {
        Map<String, DataSource> result = new HashMap<>();
        result.put("demo_ds_0", DataSourceUtil.createDataSource("demo_ds_0"));
        result.put("demo_ds_1", DataSourceUtil.createDataSource("demo_ds_1"));
        return result;
    }
    
    private static KeyGeneratorConfiguration getKeyGeneratorConfiguration() {
        return new KeyGeneratorConfiguration("SNOWFLAKE", "order_id", new Properties());
    }
```

## 配置项说明

### 数据分片

#### ShardingDataSourceFactory

数据分片的数据源创建工厂

| 名称               | 数据类型                  | 说明             |
| ------------------ | ------------------------- | ---------------- |
| dataSourceMap      | Map<String,DataSource>    | 数据源配置       |
| shardingRuleConfig | ShardingRuleConfiguration | 数据分片配置规则 |
| props(?)           | Properties                | 属性配置         |

#### ShadingRuleConfiguration

分片规则配置对象

| 名称                                      | 数据类型                                 | 说明                                                         |
| ----------------------------------------- | ---------------------------------------- | ------------------------------------------------------------ |
| tableRuleConfigs                          | Collection<TableRuleConfiguration>       | 分片规则列表                                                 |
| bindingTableGroups (?)                    | Collection<String>                       | 绑定表规则列表                                               |
| broadcastTables (?)                       | Collection<String>                       | 广播表规则列表                                               |
| defaultDataSourceName (?)                 | String                                   | 未配置分片规则的表将通过默认数据源定位                       |
| defaultDatabaseShardingStrategyConfig (?) | ShardingStrategyConfiguration            | 默认分库策略                                                 |
| defaultTableShardingStrategyConfig (?)    | ShardingStrategyConfiguration            | 默认分表策略                                                 |
| defaultKeyGeneratorConfig (?)             | KeyGeneratorConfiguration                | 默认自增列值生成器配置，缺省将使用org.apache.shardingsphere.core.keygen.generator.impl.SnowflakeKeyGenerator |
| masterSlaveRuleConfigs (?)                | Collection<MasterSlaveRuleConfiguration> | 读写分离规则，缺省表示不使用读写分离                         |

#### TableRuleConfiguration

表分片规则配置对象

| 名称                               | 数据类型                      | 说明                                                         |
| ---------------------------------- | ----------------------------- | ------------------------------------------------------------ |
| logicTable                         | String                        | 逻辑表名称                                                   |
| actualDataNodes (?)                | String                        | 由数据源名 + 表名组成，以小数点分隔。多个表以逗号分隔，支持inline表达式。缺省表示使用已知数据源与逻辑表名称生成数据节点。用于广播表（即每个库中都需要一个同样的表用于关联查询，多为字典表）或只分库不分表且所有库的表结构完全一致的情况 |
| databaseShardingStrategyConfig (?) | ShardingStrategyConfiguration | 分库策略，缺省表示使用默认分库策略                           |
| tableShardingStrategyConfig (?)    | ShardingStrategyConfiguration | 分表策略，缺省表示使用默认分表策略                           |
| keyGeneratorConfig (?)             | KeyGeneratorConfiguration     | 自增列值生成器配置，缺省表示使用默认自增主键生成器           |
| encryptorConfiguration (?)         | EncryptorConfiguration        | 加解密生成器配置                                             |

#### StandardShardingStrategyConfiguration

ShardingStrategyConfiguration的实现类，用于单分片键的标准分片场景

| 名称                       | 数据类型                 | 说明                      |
| -------------------------- | ------------------------ | ------------------------- |
| shardingColumn             | String                   | 分片列名称                |
| preciseShardingAlgorithm   | PreciseShardingAlgorithm | 精确分片算法，用于=和IN   |
| rangeShardingAlgorithm (?) | RangeShardingAlgorithm   | 范围分片算法，用于BETWEEN |

#### ComplexShardingStrategyConfiguration

ShardingStrategyConfiguration的实现类，用于多分片键的复合分片场景。

| 名称              | 数据类型                     | 说明                         |
| ----------------- | ---------------------------- | ---------------------------- |
| shardingColumns   | String                       | 分片列名称，多个列以逗号分隔 |
| shardingAlgorithm | ComplexKeysShardingAlgorithm | 复合分片算法                 |

#### InlineShardingStrategyConfiguration

| 名称                | 数据类型 | 说明                                                         |
| ------------------- | -------- | ------------------------------------------------------------ |
| shardingColumn      | String   | 分片列名称                                                   |
| algorithmExpression | String   | 分片算法行表达式，需符合groovy语法，详情请参考[行表达式](https://shardingsphere.apache.org/document/current/cn/features/sharding/other-features/inline-expression) |

#### HintShardingStrategyConfiguration

| 名称              | 数据类型              | 说明         |
| ----------------- | --------------------- | ------------ |
| shardingAlgorithm | HintShardingAlgorithm | Hint分片算法 |

#### NoneShardingStrategyConfiguration

ShardingStrategyConfiguration的实现类，用于配置部分片的策略

#### KeyGeneratorConfiguration

| 名称   | 数据类型   | 说明                                                         |
| ------ | ---------- | ------------------------------------------------------------ |
| column | String     | 自增列名称                                                   |
| type   | String     | 自增列值生成器类型，可自定义或选择内置类型：SNOWFLAKE/UUID/LEAF_SEGMENT/LEAF_SNOWFLAKE |
| props  | Properties | 自增列值生成器的相关属性配置                                 |

#### PropertiesConstant

##### SNOWFLAKE

| 名称                                          | 数据类型 | 说明                                                         |
| --------------------------------------------- | -------- | ------------------------------------------------------------ |
| worker.id (?)                                 | long     | 工作机器唯一id，默认为0                                      |
| max.tolerate.time.difference.milliseconds (?) | long     | 最大容忍时钟回退时间，单位：毫秒。默认为10毫秒               |
| max.vibration.offset (?)                      | int      | 最大抖动上限值，范围[0, 4096)，默认为1。注：若使用此算法生成值作分片值，建议配置此属性。此算法在不同毫秒内所生成的key取模2^n (2^n一般为分库或分表数) 之后结果总为0或1。为防止上述分片问题，建议将此属性值配置为(2^n)-1 |

##### LEAF_SEGMENT

| 名称                              | 数据类型 | 说明                                                         |
| --------------------------------- | -------- | ------------------------------------------------------------ |
| server.list                       | String   | 连接注册中心服务器的列表，包括IP地址和端口号，多个地址用逗号分隔。如: host1:2181,host2:2181 |
| leaf.key                          | String   | 用于标识leaf_segment所对应表里面的最大号段id                 |
| leaf.segment.id.initial.value (?) | long     | 号段id初始值，默认为1                                        |
| leaf.segment.step (?)             | long     | 每次分配的号段步长，默认为10000                              |
| registry.center.digest (?)        | String   | 连接注册中心的权限令牌。缺省为不需要权限验证                 |
| registry.center.type (?)          | String   | 注册中心类型，默认为`zookeeper`                              |

##### LEAF_SNOWFLAKE

| 名称                                          | 数据类型 | 说明                                                         |
| --------------------------------------------- | -------- | ------------------------------------------------------------ |
| server.list                                   | String   | 连接注册中心服务器的列表，包括IP地址和端口号，多个地址用逗号分隔。如: host1:2181,host2:2181 |
| service.id                                    | String   | leaf_snowflake位于注册中心上的服务标识                       |
| max.tolerate.time.difference.milliseconds (?) | long     | 最大允许的本机与注册中心的时间误差毫秒数，默认为10000毫秒。如果时间误差超过配置时间则作业启动时将抛异常 |
| registry.center.digest (?)                    | String   | 连接注册中心的权限令牌。缺省为不需要权限验证                 |
| registry.center.type (?)                      | String   | 注册中心类型，默认为`zookeeper`                              |
| max.vibration.offset (?)                      | int      | 最大抖动上限，范围[0, 4096)，默认为1。注：若使用此算法生成值作分片值，建议配置此属性。此算法在不同毫秒内所生成的key取模2^n (2^n一般为分库或分表数) 之后结果总为0或1。为防止上述分片问题，建议将此属性值配置为(2^n)-1 |

#### EncryptRuleConfiguration

| 名称       | 数据类型 | 说明                                              |
| ---------- | -------- | ------------------------------------------------- |
| encryptors | Map      | 加解密器配置列表，可自定义或选择内置类型：MD5/AES |
| tables     | Map      | 加密表配置列表                                    |

#### EncryptorRuleConfiguration

| 名称       | 数据类型   | 说明                                                         |
| ---------- | ---------- | ------------------------------------------------------------ |
| type       | String     | 加解密器类型，可自定义或选择内置类型：MD5/AES                |
| properties | Properties | 属性配置, 注意：使用AES加密器，需要配置AES加密器的KEY属性：aes.key.value |

#### EncryptTableRuleConfiguration

| 名称   | 数据类型 | 说明           |
| ------ | -------- | -------------- |
| tables | Map      | 加密列配置列表 |

#### EncyptColumnRuleConfiguration

| 名称                | 数据类型 | 说明                                                         |
| ------------------- | -------- | ------------------------------------------------------------ |
| plainColumn         | String   | 存储明文的字段                                               |
| cipherColumn        | String   | 存储密文的字段                                               |
| assistedQueryColumn | String   | 辅助查询字段，针对ShardingQueryAssistedEncryptor类型的加解密器进行辅助查询 |
| encryptor           | String   | 加解密器名字                                                 |

#### PropertiesConstant

属性配置项，可以为以下属性

| 名称                               | 数据类型 | 说明                                                  |
| ---------------------------------- | -------- | ----------------------------------------------------- |
| sql.show (?)                       | boolean  | 是否开启SQL显示，默认值: false                        |
| executor.size (?)                  | int      | 工作线程数量，默认值: CPU核数                         |
| max.connections.size.per.query (?) | int      | 每个物理数据库为每次查询分配的最大连接数量。默认值: 1 |
| check.table.metadata.enabled (?)   | boolean  | 是否在启动时检查分表元数据一致性，默认值: false       |
| query.with.cipher.column (?)       | boolean  | 当存在明文列时，是否使用密文列查询，默认值: true      |

### 读写分离

#### MasterSlaveDataSourceFactory

读写分离的数据源创建工厂

| 名称                  | 数据类型                     | 说明                 |
| --------------------- | ---------------------------- | -------------------- |
| dataSourceMap         | Map<String, DataSource>      | 数据源与其名称的映射 |
| masterSlaveRuleConfig | MasterSlaveRuleConfiguration | 读写分离规则         |
| props (?)             | Properties                   | 属性配置             |

#### MasterSlaveRuleConfiguration

读写分离规则配置对象

| 名称                     | 数据类型                        | 说明               |
| ------------------------ | ------------------------------- | ------------------ |
| name                     | String                          | 读写分离数据源名称 |
| masterDataSourceName     | String                          | 主库数据源名称     |
| slaveDataSourceNames     | Collection<String>              | 从库数据源名称列表 |
| loadBalanceAlgorithm (?) | MasterSlaveLoadBalanceAlgorithm | 从库负载均衡算法   |

#### PropertiesConstant

属性配置项，可以为以下属性

| 名称                               | 数据类型 | 说明                                                   |
| ---------------------------------- | -------- | ------------------------------------------------------ |
| sql.show (?)                       | boolean  | 是否打印SQL解析和改写日志，默认值: false               |
| executor.size (?)                  | int      | 用于SQL执行的工作线程数量，为零则表示无限制。默认值: 0 |
| max.connections.size.per.query (?) | int      | 每个物理数据库为每次查询分配的最大连接数量。默认值: 1  |
| check.table.metadata.enabled (?)   | boolean  | 是否在启动时检查分表元数据一致性，默认值: false        |

### 数据脱敏

#### EnctyptDataSourceFactory

| 名称              | 数据类型                 | 说明               |
| ----------------- | ------------------------ | ------------------ |
| dataSource        | DataSource               | 数据源，任意连接池 |
| encryptRuleConfig | EncryptRuleConfiguration | 数据脱敏规则       |
| props (?)         | Properties               | 属性配置           |

#### EncryptRuleConfiguration

| 名称       | 数据类型 | 说明                                              |
| ---------- | -------- | ------------------------------------------------- |
| encryptors | Map      | 加解密器配置列表，可自定义或选择内置类型：MD5/AES |
| tables     | Map      | 加密表配置列表                                    |

#### PropertiesConstant

属性配置项，可以为以下属性

| 名称                         | 数据类型 | 说明                                             |
| ---------------------------- | -------- | ------------------------------------------------ |
| sql.show (?)                 | boolean  | 是否开启SQL显示，默认值: false                   |
| query.with.cipher.column (?) | boolean  | 当存在明文列时，是否使用密文列查询，默认值: true |

### 治理

#### OrchestrationShardingDataSourceFactory

数据分片+治理的数据源工厂

| 名称                | 数据类型                   | 说明                        |
| ------------------- | -------------------------- | --------------------------- |
| dataSourceMap       | Map<String, DataSource>    | 同ShardingDataSourceFactory |
| shardingRuleConfig  | ShardingRuleConfiguration  | 同ShardingDataSourceFactory |
| props (?)           | Properties                 | 同ShardingDataSourceFactory |
| orchestrationConfig | OrchestrationConfiguration | 治理规则配置                |

#### OrchestrationMasterSlaveDataSourceFactory

读写分离+治理的数据源工厂

| 名称                  | 数据类型                     | 说明                           |
| --------------------- | ---------------------------- | ------------------------------ |
| dataSourceMap         | Map<String, DataSource>      | 同MasterSlaveDataSourceFactory |
| masterSlaveRuleConfig | MasterSlaveRuleConfiguration | 同MasterSlaveDataSourceFactory |
| props (?)             | Properties                   | 同ShardingDataSourceFactory    |
| orchestrationConfig   | OrchestrationConfiguration   | 治理规则配置                   |

#### OrchestrationEncryptDataSourceFactory

数据脱敏+治理的数据源工厂

| 名称                | 数据类型                   | 说明                        |
| ------------------- | -------------------------- | --------------------------- |
| dataSource          | DataSource                 | 同EncryptDataSourceFactory  |
| encryptRuleConfig   | EncryptRuleConfiguration   | 同EncryptDataSourceFactory  |
| props (?)           | Properties                 | 同ShardingDataSourceFactory |
| orchestrationConfig | OrchestrationConfiguration | 治理规则配置                |

#### OrchestrationConfiguration

治理规则配置对象

| 名称            | 数据类型                    | 说明                                                         |
| --------------- | --------------------------- | ------------------------------------------------------------ |
| name            | String                      | 治理实例名称                                                 |
| overwrite       | boolean                     | 本地配置是否覆盖注册中心配置，如果可覆盖，每次启动都以本地配置为准 |
| regCenterConfig | RegistryCenterConfiguration | 注册中心配置                                                 |

#### RegistryCenterConfiguration

用于配置注册中心

| 名称                             | 数据类型 | 说明                                                         |
| -------------------------------- | -------- | ------------------------------------------------------------ |
| serverLists                      | String   | 连接注册中心服务器的列表，包括IP地址和端口号，多个地址用逗号分隔。如: host1:2181,host2:2181 |
| namespace (?)                    | String   | 注册中心的命名空间                                           |
| digest (?)                       | String   | 连接注册中心的权限令牌。缺省为不需要权限验证                 |
| operationTimeoutMilliseconds (?) | int      | 操作超时的毫秒数，默认500毫秒                                |
| maxRetries (?)                   | int      | 连接失败后的最大重试次数，默认3次                            |
| retryIntervalMilliseconds (?)    | int      | 重试间隔毫秒数，默认500毫秒                                  |
| timeToLiveSeconds (?)            | int      | 临时节点存活秒数，默认60秒                                   |



# YAML配置

## 配置示例

### 数据分片

```yaml
dataSources:
  ds0: !!org.apache.commons.dbcp.BasicDataSource
    driverClassName: com.mysql.jdbc.Driver
    url: jdbc:mysql://localhost:3306/ds0
    username: root
    password: 
  ds1: !!org.apache.commons.dbcp.BasicDataSource
    driverClassName: com.mysql.jdbc.Driver
    url: jdbc:mysql://localhost:3306/ds1
    username: root
    password: 

shardingRule:  
  tables:
    t_order: 
      actualDataNodes: ds${0..1}.t_order${0..1}
      databaseStrategy:
        inline:
          shardingColumn: user_id
          algorithmExpression: ds${user_id % 2}
      tableStrategy: 
        inline:
          shardingColumn: order_id
          algorithmExpression: t_order${order_id % 2}
      keyGenerator:
        type: SNOWFLAKE
        column: order_id
    t_order_item:
      actualDataNodes: ds${0..1}.t_order_item${0..1}
      databaseStrategy:
        inline:
          shardingColumn: user_id
          algorithmExpression: ds${user_id % 2}
      tableStrategy:
        inline:
          shardingColumn: order_id
          algorithmExpression: t_order_item${order_id % 2}  
  bindingTables:
    - t_order,t_order_item
  broadcastTables:
    - t_config
  
  defaultDataSourceName: ds0
  defaultTableStrategy:
    none:
  defaultKeyGenerator:
    type: SNOWFLAKE
    column: order_id
  
props:
  sql.show: true
```

### 读写分离

```yaml
dataSources:
  ds_master: !!org.apache.commons.dbcp.BasicDataSource
    driverClassName: com.mysql.jdbc.Driver
    url: jdbc:mysql://localhost:3306/ds_master
    username: root
    password: 
  ds_slave0: !!org.apache.commons.dbcp.BasicDataSource
    driverClassName: com.mysql.jdbc.Driver
    url: jdbc:mysql://localhost:3306/ds_slave0
    username: root
    password: 
  ds_slave1: !!org.apache.commons.dbcp.BasicDataSource
    driverClassName: com.mysql.jdbc.Driver
    url: jdbc:mysql://localhost:3306/ds_slave1
    username: root
    password: 

masterSlaveRule:
  name: ds_ms
  masterDataSourceName: ds_master
  slaveDataSourceNames: 
    - ds_slave0
    - ds_slave1

props:
    sql.show: true
```

### 数据脱敏

```yaml
dataSource:  !!org.apache.commons.dbcp2.BasicDataSource
  driverClassName: com.mysql.jdbc.Driver
  url: jdbc:mysql://127.0.0.1:3306/encrypt?serverTimezone=UTC&useSSL=false
  username: root
  password:

encryptRule:
  encryptors:
    encryptor_aes:
      type: aes
      props:
        aes.key.value: 123456abc
    encryptor_md5:
      type: md5
  tables:
    t_encrypt:
      columns:
        user_id:
          plainColumn: user_plain
          cipherColumn: user_cipher
          encryptor: encryptor_aes
        order_id:
          cipherColumn: order_cipher
          encryptor: encryptor_md5
props:
  query.with.cipher.column: true #是否使用密文列查询
```

### 数据分片+读写分离

```yaml
dataSources:
  ds0: !!org.apache.commons.dbcp.BasicDataSource
    driverClassName: com.mysql.jdbc.Driver
    url: jdbc:mysql://localhost:3306/ds0
    username: root
    password: 
  ds0_slave0: !!org.apache.commons.dbcp.BasicDataSource
      driverClassName: com.mysql.jdbc.Driver
      url: jdbc:mysql://localhost:3306/ds0_slave0
      username: root
      password: 
  ds0_slave1: !!org.apache.commons.dbcp.BasicDataSource
      driverClassName: com.mysql.jdbc.Driver
      url: jdbc:mysql://localhost:3306/ds0_slave1
      username: root
      password: 
  ds1: !!org.apache.commons.dbcp.BasicDataSource
    driverClassName: com.mysql.jdbc.Driver
    url: jdbc:mysql://localhost:3306/ds1
    username: root
    password: 
  ds1_slave0: !!org.apache.commons.dbcp.BasicDataSource
        driverClassName: com.mysql.jdbc.Driver
        url: jdbc:mysql://localhost:3306/ds1_slave0
        username: root
        password: 
  ds1_slave1: !!org.apache.commons.dbcp.BasicDataSource
        driverClassName: com.mysql.jdbc.Driver
        url: jdbc:mysql://localhost:3306/ds1_slave1
        username: root
        password: 

shardingRule:  
  tables:
    t_order: 
      actualDataNodes: ms_ds${0..1}.t_order${0..1}
      databaseStrategy:
        inline:
          shardingColumn: user_id
          algorithmExpression: ms_ds${user_id % 2}
      tableStrategy: 
        inline:
          shardingColumn: order_id
          algorithmExpression: t_order${order_id % 2}
      keyGenerator:
        type: SNOWFLAKE
        column: order_id
    t_order_item:
      actualDataNodes: ms_ds${0..1}.t_order_item${0..1}
      databaseStrategy:
        inline:
          shardingColumn: user_id
          algorithmExpression: ms_ds${user_id % 2}
      tableStrategy:
        inline:
          shardingColumn: order_id
          algorithmExpression: t_order_item${order_id % 2}  
  bindingTables:
    - t_order,t_order_item
  broadcastTables:
    - t_config
  
  defaultDataSourceName: ds_0
  defaultTableStrategy:
    none:
  defaultKeyGenerator:
    type: SNOWFLAKE
    column: order_id
  
  masterSlaveRules:
      ms_ds0:
        masterDataSourceName: ds0
        slaveDataSourceNames:
          - ds0_slave0
          - ds0_slave1
        loadBalanceAlgorithmType: ROUND_ROBIN
      ms_ds1:
        masterDataSourceName: ds1
        slaveDataSourceNames: 
          - ds1_slave0
          - ds1_slave1
        loadBalanceAlgorithmType: ROUND_ROBIN
props:
  sql.show: true
```

### 数据分片+数据脱敏

```yaml
dataSources:
  ds_0: !!com.zaxxer.hikari.HikariDataSource
    driverClassName: com.mysql.jdbc.Driver
    jdbcUrl: jdbc:mysql://localhost:3306/demo_ds_0
    username: root
    password:
  ds_1: !!com.zaxxer.hikari.HikariDataSource
    driverClassName: com.mysql.jdbc.Driver
    jdbcUrl: jdbc:mysql://localhost:3306/demo_ds_1
    username: root
    password:

shardingRule:
  tables:
    t_order: 
      actualDataNodes: ds_${0..1}.t_order_${0..1}
      databaseStrategy:
        inline:
          shardingColumn: user_id
          algorithmExpression: ds_${user_id % 2}
      tableStrategy: 
        inline:
          shardingColumn: order_id
          algorithmExpression: t_order_${order_id % 2}
      keyGenerator:
        type: SNOWFLAKE
        column: order_id
    t_order_item:
      actualDataNodes: ds_${0..1}.t_order_item_${0..1}
      databaseStrategy:
        inline:
          shardingColumn: user_id
          algorithmExpression: ds_${user_id % 2}
      tableStrategy:
        inline:
          shardingColumn: order_id
          algorithmExpression: t_order_item_${order_id % 2}
  bindingTables:
    - t_order,t_order_item 
  
  defaultTableStrategy:
    none:
    
  encryptRule:
    encryptors:
      encryptor_aes:
        type: aes
        props:
          aes.key.value: 123456abc
    tables:
      t_order:
        columns:
          order_id:
            plainColumn: order_plain
            cipherColumn: order_cipher
            encryptor: encryptor_aes

props:
  sql.show: true
```

### 治理

```yaml
#省略数据分片、读写分离和数据脱敏配置

orchestration:
  name: orchestration_ds
  overwrite: true
  registry:
    type: zookeeper
    namespace: orchestration
    serverLists: localhost:2181
```

## 配置项说明

### 数据分片

```yaml
dataSources: #数据源配置，可配置多个data_source_name
  <data_source_name>: #<!!数据库连接池实现类> `!!`表示实例化该类
    driverClassName: #数据库驱动类名
    url: #数据库url连接
    username: #数据库用户名
    password: #数据库密码
    # ... 数据库连接池的其它属性

shardingRule:
  tables: #数据分片规则配置，可配置多个logic_table_name
    <logic_table_name>: #逻辑表名称
      actualDataNodes: #由数据源名 + 表名组成，以小数点分隔。多个表以逗号分隔，支持inline表达式。缺省表示使用已知数据源与逻辑表名称生成数据节点。用于广播表（即每个库中都需要一个同样的表用于关联查询，多为字典表）或只分库不分表且所有库的表结构完全一致的情况
        
      databaseStrategy: #分库策略，缺省表示使用默认分库策略，以下的分片策略只能选其一
        standard: #用于单分片键的标准分片场景
          shardingColumn: #分片列名称
          preciseAlgorithmClassName: #精确分片算法类名称，用于=和IN。。该类需实现PreciseShardingAlgorithm接口并提供无参数的构造器
          rangeAlgorithmClassName: #范围分片算法类名称，用于BETWEEN，可选。。该类需实现RangeShardingAlgorithm接口并提供无参数的构造器
        complex: #用于多分片键的复合分片场景
          shardingColumns: #分片列名称，多个列以逗号分隔
          algorithmClassName: #复合分片算法类名称。该类需实现ComplexKeysShardingAlgorithm接口并提供无参数的构造器
        inline: #行表达式分片策略
          shardingColumn: #分片列名称
          algorithmInlineExpression: #分片算法行表达式，需符合groovy语法
        hint: #Hint分片策略
          algorithmClassName: #Hint分片算法类名称。该类需实现HintShardingAlgorithm接口并提供无参数的构造器
        none: #不分片
      tableStrategy: #分表策略，同分库策略
      keyGenerator: 
        column: #自增列名称，缺省表示不使用自增主键生成器
        type: #自增列值生成器类型，缺省表示使用默认自增列值生成器。可使用用户自定义的列值生成器或选择内置类型：SNOWFLAKE/UUID/LEAF_SEGMENT
        props: #属性配置, 注意：使用SNOWFLAKE算法，需要配置worker.id与max.tolerate.time.difference.milliseconds属性。若使用此算法生成值作分片值，建议配置max.vibration.offset属性
          <property-name>: 属性名称
      
  bindingTables: #绑定表规则列表
  - <logic_table_name1, logic_table_name2, ...> 
  - <logic_table_name3, logic_table_name4, ...>
  - <logic_table_name_x, logic_table_name_y, ...>
  broadcastTables: #广播表规则列表
  - table_name1
  - table_name2
  - table_name_x
  
  defaultDataSourceName: #未配置分片规则的表将通过默认数据源定位  
  defaultDatabaseStrategy: #默认数据库分片策略，同分库策略
  defaultTableStrategy: #默认表分片策略，同分库策略
  defaultKeyGenerator: #默认的主键生成算法 如果没有设置,默认为SNOWFLAKE算法
    type: #默认自增列值生成器类型，缺省将使用org.apache.shardingsphere.core.keygen.generator.impl.SnowflakeKeyGenerator。可使用用户自定义的列值生成器或选择内置类型：SNOWFLAKE/UUID/LEAF_SEGMENT
    props:
      <property-name>: #自增列值生成器属性配置, 比如SNOWFLAKE算法的worker.id与max.tolerate.time.difference.milliseconds

  masterSlaveRules: #读写分离规则，详见读写分离部分
    <data_source_name>: #数据源名称，需要与真实数据源匹配，可配置多个data_source_name
      masterDataSourceName: #详见读写分离部分
      slaveDataSourceNames: #详见读写分离部分
      loadBalanceAlgorithmType: #详见读写分离部分
      props: #读写分离负载算法的属性配置
        <property-name>: #属性值
      
props: #属性配置
  sql.show: #是否开启SQL显示，默认值: false
  executor.size: #工作线程数量，默认值: CPU核数
  max.connections.size.per.query: # 每个查询可以打开的最大连接数量,默认为1
  check.table.metadata.enabled: #是否在启动时检查分表元数据一致性，默认值: false
```

### 读写分离

```yaml
dataSources: #省略数据源配置，与数据分片一致

masterSlaveRule:
  name: #读写分离数据源名称
  masterDataSourceName: #主库数据源名称
  slaveDataSourceNames: #从库数据源名称列表
    - <data_source_name1>
    - <data_source_name2>
    - <data_source_name_x>
  loadBalanceAlgorithmType: #从库负载均衡算法类型，可选值：ROUND_ROBIN，RANDOM。若`loadBalanceAlgorithmClassName`存在则忽略该配置
  props: #读写分离负载算法的属性配置
    <property-name>: #属性值
```

### 数据脱敏

```yaml
dataSource: #省略数据源配置

encryptRule:
  encryptors:
    <encryptor-name>:
      type: #加解密器类型，可自定义或选择内置类型：MD5/AES 
      props: #属性配置, 注意：使用AES加密器，需要配置AES加密器的KEY属性：aes.key.value
        aes.key.value: 
  tables:
    <table-name>:
      columns:
        <logic-column-name>:
          plainColumn: #存储明文的字段
          cipherColumn: #存储密文的字段
          assistedQueryColumn: #辅助查询字段，针对ShardingQueryAssistedEncryptor类型的加解密器进行辅助查询
          encryptor: #加密器名字
```

### 治理

```yaml
dataSources: #省略数据源配置
shardingRule: #省略分片规则配置
masterSlaveRule: #省略读写分离规则配置
encryptRule: #省略数据脱敏规则配置

orchestration:
  name: #治理实例名称
  overwrite: #本地配置是否覆盖注册中心配置。如果可覆盖，每次启动都以本地配置为准
  registry: #注册中心配置
    type: #配置中心类型。如：zookeeper
    serverLists: #连接注册中心服务器的列表。包括IP地址和端口号。多个地址用逗号分隔。如: host1:2181,host2:2181
    namespace: #注册中心的命名空间
    digest: #连接注册中心的权限令牌。缺省为不需要权限验证
    operationTimeoutMilliseconds: #操作超时的毫秒数，默认500毫秒
    maxRetries: #连接失败后的最大重试次数，默认3次
    retryIntervalMilliseconds: #重试间隔毫秒数，默认500毫秒
    timeToLiveSeconds: #临时节点存活秒数，默认60秒
```

## Yaml语法说明

`!!` 表示实例化该类

`-` 表示可以包含一个或多个

`[]` 表示数组，可以与减号相互替换使用



# SPRING BOOT配置

## 注意事项

行表达式标识符可以使用`${...}`或`$->{...}`，但前者与Spring本身的属性文件占位符冲突，因此在Spring环境中使用行表达式标识符建议使用`$->{...}`。

## 配置示例

### 数据分片

```properties
spring.shardingsphere.datasource.names=ds0,ds1

spring.shardingsphere.datasource.ds0.type=org.apache.commons.dbcp.BasicDataSource
spring.shardingsphere.datasource.ds0.driver-class-name=com.mysql.jdbc.Driver
spring.shardingsphere.datasource.ds0.url=jdbc:mysql://localhost:3306/ds0
spring.shardingsphere.datasource.ds0.username=root
spring.shardingsphere.datasource.ds0.password=

spring.shardingsphere.datasource.ds1.type=org.apache.commons.dbcp.BasicDataSource
spring.shardingsphere.datasource.ds1.driver-class-name=com.mysql.jdbc.Driver
spring.shardingsphere.datasource.ds1.url=jdbc:mysql://localhost:3306/ds1
spring.shardingsphere.datasource.ds1.username=root
spring.shardingsphere.datasource.ds1.password=

spring.shardingsphere.sharding.tables.t_order.actual-data-nodes=ds$->{0..1}.t_order$->{0..1}
spring.shardingsphere.sharding.tables.t_order.table-strategy.inline.sharding-column=order_id
spring.shardingsphere.sharding.tables.t_order.table-strategy.inline.algorithm-expression=t_order$->{order_id % 2}
spring.shardingsphere.sharding.tables.t_order.key-generator.column=order_id
spring.shardingsphere.sharding.tables.t_order.key-generator.type=SNOWFLAKE
spring.shardingsphere.sharding.tables.t_order_item.actual-data-nodes=ds$->{0..1}.t_order_item$->{0..1}
spring.shardingsphere.sharding.tables.t_order_item.table-strategy.inline.sharding-column=order_id
spring.shardingsphere.sharding.tables.t_order_item.table-strategy.inline.algorithm-expression=t_order_item$->{order_id % 2}
spring.shardingsphere.sharding.tables.t_order_item.key-generator.column=order_item_id
spring.shardingsphere.sharding.tables.t_order_item.key-generator.type=SNOWFLAKE
spring.shardingsphere.sharding.binding-tables=t_order,t_order_item
spring.shardingsphere.sharding.broadcast-tables=t_config

spring.shardingsphere.sharding.default-database-strategy.inline.sharding-column=user_id
spring.shardingsphere.sharding.default-database-strategy.inline.algorithm-expression=ds$->{user_id % 2}
```

### 读写分离

```properties
spring.shardingsphere.datasource.names=master,slave0,slave1

spring.shardingsphere.datasource.master.type=org.apache.commons.dbcp.BasicDataSource
spring.shardingsphere.datasource.master.driver-class-name=com.mysql.jdbc.Driver
spring.shardingsphere.datasource.master.url=jdbc:mysql://localhost:3306/master
spring.shardingsphere.datasource.master.username=root
spring.shardingsphere.datasource.master.password=

spring.shardingsphere.datasource.slave0.type=org.apache.commons.dbcp.BasicDataSource
spring.shardingsphere.datasource.slave0.driver-class-name=com.mysql.jdbc.Driver
spring.shardingsphere.datasource.slave0.url=jdbc:mysql://localhost:3306/slave0
spring.shardingsphere.datasource.slave0.username=root
spring.shardingsphere.datasource.slave0.password=

spring.shardingsphere.datasource.slave1.type=org.apache.commons.dbcp.BasicDataSource
spring.shardingsphere.datasource.slave1.driver-class-name=com.mysql.jdbc.Driver
spring.shardingsphere.datasource.slave1.url=jdbc:mysql://localhost:3306/slave1
spring.shardingsphere.datasource.slave1.username=root
spring.shardingsphere.datasource.slave1.password=

spring.shardingsphere.masterslave.load-balance-algorithm-type=round_robin
spring.shardingsphere.masterslave.name=ms
spring.shardingsphere.masterslave.master-data-source-name=master
spring.shardingsphere.masterslave.slave-data-source-names=slave0,slave1

spring.shardingsphere.props.sql.show=true
```

### 数据脱敏

```properties
spring.shardingsphere.datasource.name=ds

spring.shardingsphere.datasource.ds.type=org.apache.commons.dbcp2.BasicDataSource
spring.shardingsphere.datasource.ds.driver-class-name=com.mysql.jdbc.Driver
spring.shardingsphere.datasource.ds.url=jdbc:mysql://127.0.0.1:3306/encrypt?serverTimezone=UTC&useSSL=false
spring.shardingsphere.datasource.ds.username=root
spring.shardingsphere.datasource.ds.password=
spring.shardingsphere.datasource.ds.max-total=100

spring.shardingsphere.encrypt.encryptors.encryptor_aes.type=aes
spring.shardingsphere.encrypt.encryptors.encryptor_aes.props.aes.key.value=123456
spring.shardingsphere.encrypt.tables.t_order.columns.user_id.plainColumn=user_decrypt
spring.shardingsphere.encrypt.tables.t_order.columns.user_id.cipherColumn=user_encrypt
spring.shardingsphere.encrypt.tables.t_order.columns.user_id.assistedQueryColumn=user_assisted
spring.shardingsphere.encrypt.tables.t_order.columns.user_id.encryptor=encryptor_aes

spring.shardingsphere.props.sql.show=true
spring.shardingsphere.props.query.with.cipher.column=true
```

### 数据分片+读写分离

```properties
spring.shardingsphere.datasource.names=master0,master1,master0slave0,master0slave1,master1slave0,master1slave1

spring.shardingsphere.datasource.master0.type=org.apache.commons.dbcp.BasicDataSource
spring.shardingsphere.datasource.master0.driver-class-name=com.mysql.jdbc.Driver
spring.shardingsphere.datasource.master0.url=jdbc:mysql://localhost:3306/master0
spring.shardingsphere.datasource.master0.username=root
spring.shardingsphere.datasource.master0.password=

spring.shardingsphere.datasource.master0slave0.type=org.apache.commons.dbcp.BasicDataSource
spring.shardingsphere.datasource.master0slave0.driver-class-name=com.mysql.jdbc.Driver
spring.shardingsphere.datasource.master0slave0.url=jdbc:mysql://localhost:3306/master0slave0
spring.shardingsphere.datasource.master0slave0.username=root
spring.shardingsphere.datasource.master0slave0.password=
spring.shardingsphere.datasource.master0slave1.type=org.apache.commons.dbcp.BasicDataSource
spring.shardingsphere.datasource.master0slave1.driver-class-name=com.mysql.jdbc.Driver
spring.shardingsphere.datasource.master0slave1.url=jdbc:mysql://localhost:3306/master0slave1
spring.shardingsphere.datasource.master0slave1.username=root
spring.shardingsphere.datasource.master0slave1.password=

spring.shardingsphere.datasource.master1.type=org.apache.commons.dbcp.BasicDataSource
spring.shardingsphere.datasource.master1.driver-class-name=com.mysql.jdbc.Driver
spring.shardingsphere.datasource.master1.url=jdbc:mysql://localhost:3306/master1
spring.shardingsphere.datasource.master1.username=root
spring.shardingsphere.datasource.master1.password=

spring.shardingsphere.datasource.master1slave0.type=org.apache.commons.dbcp.BasicDataSource
spring.shardingsphere.datasource.master1slave0.driver-class-name=com.mysql.jdbc.Driver
spring.shardingsphere.datasource.master1slave0.url=jdbc:mysql://localhost:3306/master1slave0
spring.shardingsphere.datasource.master1slave0.username=root
spring.shardingsphere.datasource.master1slave0.password=
spring.shardingsphere.datasource.master1slave1.type=org.apache.commons.dbcp.BasicDataSource
spring.shardingsphere.datasource.master1slave1.driver-class-name=com.mysql.jdbc.Driver
spring.shardingsphere.datasource.master1slave1.url=jdbc:mysql://localhost:3306/master1slave1
spring.shardingsphere.datasource.master1slave1.username=root
spring.shardingsphere.datasource.master1slave1.password=

spring.shardingsphere.sharding.tables.t_order.actual-data-nodes=ds$->{0..1}.t_order$->{0..1}
spring.shardingsphere.sharding.tables.t_order.table-strategy.inline.sharding-column=order_id
spring.shardingsphere.sharding.tables.t_order.table-strategy.inline.algorithm-expression=t_order$->{order_id % 2}
spring.shardingsphere.sharding.tables.t_order.key-generator.column=order_id
spring.shardingsphere.sharding.tables.t_order.key-generator.type=SNOWFLAKE
spring.shardingsphere.sharding.tables.t_order_item.actual-data-nodes=ds$->{0..1}.t_order_item$->{0..1}
spring.shardingsphere.sharding.tables.t_order_item.table-strategy.inline.sharding-column=order_id
spring.shardingsphere.sharding.tables.t_order_item.table-strategy.inline.algorithm-expression=t_order_item$->{order_id % 2}
spring.shardingsphere.sharding.tables.t_order_item.key-generator.column=order_item_id
spring.shardingsphere.sharding.tables.t_order_item.key-generator.type=SNOWFLAKE
spring.shardingsphere.sharding.binding-tables=t_order,t_order_item
spring.shardingsphere.sharding.broadcast-tables=t_config

spring.shardingsphere.sharding.default-database-strategy.inline.sharding-column=user_id
spring.shardingsphere.sharding.default-database-strategy.inline.algorithm-expression=master$->{user_id % 2}

spring.shardingsphere.sharding.master-slave-rules.ds0.master-data-source-name=master0
spring.shardingsphere.sharding.master-slave-rules.ds0.slave-data-source-names=master0slave0, master0slave1
spring.shardingsphere.sharding.master-slave-rules.ds1.master-data-source-name=master1
spring.shardingsphere.sharding.master-slave-rules.ds1.slave-data-source-names=master1slave0, master1slave1
```

### 数据分片+数据脱敏

```properties
spring.shardingsphere.datasource.names=ds_0,ds_1

spring.shardingsphere.datasource.ds_0.type=com.zaxxer.hikari.HikariDataSource
spring.shardingsphere.datasource.ds_0.driver-class-name=com.mysql.jdbc.Driver
spring.shardingsphere.datasource.ds_0.jdbc-url=jdbc:mysql://localhost:3306/demo_ds_0
spring.shardingsphere.datasource.ds_0.username=root
spring.shardingsphere.datasource.ds_0.password=

spring.shardingsphere.datasource.ds_1.type=com.zaxxer.hikari.HikariDataSource
spring.shardingsphere.datasource.ds_1.driver-class-name=com.mysql.jdbc.Driver
spring.shardingsphere.datasource.ds_1.jdbc-url=jdbc:mysql://localhost:3306/demo_ds_1
spring.shardingsphere.datasource.ds_1.username=root
spring.shardingsphere.datasource.ds_1.password=

spring.shardingsphere.sharding.default-database-strategy.inline.sharding-column=user_id
spring.shardingsphere.sharding.default-database-strategy.inline.algorithm-expression=ds_$->{user_id % 2}

spring.shardingsphere.sharding.tables.t_order.actual-data-nodes=ds_$->{0..1}.t_order_$->{0..1}
spring.shardingsphere.sharding.tables.t_order.table-strategy.inline.sharding-column=order_id
spring.shardingsphere.sharding.tables.t_order.table-strategy.inline.algorithm-expression=t_order_$->{order_id % 2}
spring.shardingsphere.sharding.tables.t_order.key-generator.column=order_id
spring.shardingsphere.sharding.tables.t_order.key-generator.type=SNOWFLAKE
spring.shardingsphere.sharding.tables.t_order_item.actual-data-nodes=ds_$->{0..1}.t_order_item_$->{0..1}
spring.shardingsphere.sharding.tables.t_order_item.table-strategy.inline.sharding-column=order_id
spring.shardingsphere.sharding.tables.t_order_item.table-strategy.inline.algorithm-expression=t_order_item_$->{order_id % 2}
spring.shardingsphere.sharding.tables.t_order_item.key-generator.column=order_item_id
spring.shardingsphere.sharding.tables.t_order_item.key-generator.type=SNOWFLAKE
spring.shardingsphere.encrypt.encryptors.encryptor_aes.type=aes
spring.shardingsphere.encrypt.encryptors.encryptor_aes.props.aes.key.value=123456
spring.shardingsphere.encrypt.tables.t_order.columns.user_id.plainColumn=user_decrypt
spring.shardingsphere.encrypt.tables.t_order.columns.user_id.cipherColumn=user_encrypt
spring.shardingsphere.encrypt.tables.t_order.columns.user_id.assistedQueryColumn=user_assisted
spring.shardingsphere.encrypt.tables.t_order.columns.user_id.encryptor=encryptor_aes
```

### 治理

```properties
spring.shardingsphere.datasource.names=ds,ds0,ds1
spring.shardingsphere.datasource.ds.type=org.apache.commons.dbcp.BasicDataSource
spring.shardingsphere.datasource.ds.driver-class-name=org.h2.Driver
spring.shardingsphere.datasource.ds.url=jdbc:mysql://localhost:3306/ds
spring.shardingsphere.datasource.ds.username=root
spring.shardingsphere.datasource.ds.password=

spring.shardingsphere.datasource.ds0.type=org.apache.commons.dbcp.BasicDataSource
spring.shardingsphere.datasource.ds0.driver-class-name=com.mysql.jdbc.Driver
spring.shardingsphere.datasource.ds0.url=jdbc:mysql://localhost:3306/ds0
spring.shardingsphere.datasource.ds0.username=root
spring.shardingsphere.datasource.ds0.password=

spring.shardingsphere.datasource.ds1.type=org.apache.commons.dbcp.BasicDataSource
spring.shardingsphere.datasource.ds1.driver-class-name=com.mysql.jdbc.Driver
spring.shardingsphere.datasource.ds1.url=jdbc:mysql://localhost:3306/ds1
spring.shardingsphere.datasource.ds1.username=root
spring.shardingsphere.datasource.ds1.password=

spring.shardingsphere.sharding.default-data-source-name=ds
spring.shardingsphere.sharding.default-database-strategy.inline.sharding-column=user_id
spring.shardingsphere.sharding.default-database-strategy.inline.algorithm-expression=ds$->{user_id % 2}
spring.shardingsphere.sharding.tables.t_order.actual-data-nodes=ds$->{0..1}.t_order$->{0..1}
spring.shardingsphere.sharding.tables.t_order.table-strategy.inline.sharding-column=order_id
spring.shardingsphere.sharding.tables.t_order.table-strategy.inline.algorithm-expression=t_order$->{order_id % 2}
spring.shardingsphere.sharding.tables.t_order.key-generator.column=order_id
spring.shardingsphere.sharding.tables.t_order_item.actual-data-nodes=ds$->{0..1}.t_order_item$->{0..1}
spring.shardingsphere.sharding.tables.t_order_item.table-strategy.inline.sharding-column=order_id
spring.shardingsphere.sharding.tables.t_order_item.table-strategy.inline.algorithm-expression=t_order_item$->{order_id % 2}
spring.shardingsphere.sharding.tables.t_order_item.key-generator.column=order_item_id
spring.shardingsphere.sharding.binding-tables=t_order,t_order_item
spring.shardingsphere.sharding.broadcast-tables=t_config

spring.shardingsphere.orchestration.name=spring_boot_ds_sharding
spring.shardingsphere.orchestration.overwrite=true
spring.shardingsphere.orchestration.registry.type=zookeeper
spring.shardingsphere.orchestration.registry.namespace=orchestration-spring-boot-sharding-test
spring.shardingsphere.orchestration.registry.server-lists=localhost:2181
```

### JNDI

以上配置示例中，所有数据源配置均可使用JNDI代替，例如`数据分片`

```properties
spring.shardingsphere.datasource.names=ds0,ds1

spring.shardingsphere.datasource.ds0.jndi-name=java:comp/env/jdbc/ds0
spring.shardingsphere.datasource.ds1.jndi-name=jdbc/ds1

spring.shardingsphere.sharding.tables.t_order.actual-data-nodes=ds$->{0..1}.t_order$->{0..1}
spring.shardingsphere.sharding.tables.t_order.table-strategy.inline.sharding-column=order_id
spring.shardingsphere.sharding.tables.t_order.table-strategy.inline.algorithm-expression=t_order$->{order_id % 2}
spring.shardingsphere.sharding.tables.t_order.key-generator.column=order_id
spring.shardingsphere.sharding.tables.t_order.key-generator.type=SNOWFLAKE
spring.shardingsphere.sharding.tables.t_order_item.actual-data-nodes=ds$->{0..1}.t_order_item$->{0..1}
spring.shardingsphere.sharding.tables.t_order_item.table-strategy.inline.sharding-column=order_id
spring.shardingsphere.sharding.tables.t_order_item.table-strategy.inline.algorithm-expression=t_order_item$->{order_id % 2}
spring.shardingsphere.sharding.tables.t_order_item.key-generator.column=order_item_id
spring.shardingsphere.sharding.tables.t_order_item.key-generator.type=SNOWFLAKE
spring.shardingsphere.sharding.binding-tables=t_order,t_order_item
spring.shardingsphere.sharding.broadcast-tables=t_config

spring.shardingsphere.sharding.default-database-strategy.inline.sharding-column=user_id
spring.shardingsphere.sharding.default-database-strategy.inline.algorithm-expression=ds$->{user_id % 2}
```

## 配置项说明

### 数据分片

```properties
spring.shardingsphere.datasource.names= #数据源名称，多数据源以逗号分隔

spring.shardingsphere.datasource.<data-source-name>.type= #数据库连接池类名称
spring.shardingsphere.datasource.<data-source-name>.driver-class-name= #数据库驱动类名
spring.shardingsphere.datasource.<data-source-name>.url= #数据库url连接
spring.shardingsphere.datasource.<data-source-name>.username= #数据库用户名
spring.shardingsphere.datasource.<data-source-name>.password= #数据库密码
spring.shardingsphere.datasource.<data-source-name>.xxx= #数据库连接池的其它属性

spring.shardingsphere.sharding.tables.<logic-table-name>.actual-data-nodes= #由数据源名 + 表名组成，以小数点分隔。多个表以逗号分隔，支持inline表达式。缺省表示使用已知数据源与逻辑表名称生成数据节点。用于广播表（即每个库中都需要一个同样的表用于关联查询，多为字典表）或只分库不分表且所有库的表结构完全一致的情况

#分库策略，缺省表示使用默认分库策略，以下的分片策略只能选其一

#用于单分片键的标准分片场景
spring.shardingsphere.sharding.tables.<logic-table-name>.database-strategy.standard.sharding-column= #分片列名称
spring.shardingsphere.sharding.tables.<logic-table-name>.database-strategy.standard.precise-algorithm-class-name= #精确分片算法类名称，用于=和IN。该类需实现PreciseShardingAlgorithm接口并提供无参数的构造器
spring.shardingsphere.sharding.tables.<logic-table-name>.database-strategy.standard.range-algorithm-class-name= #范围分片算法类名称，用于BETWEEN，可选。该类需实现RangeShardingAlgorithm接口并提供无参数的构造器

#用于多分片键的复合分片场景
spring.shardingsphere.sharding.tables.<logic-table-name>.database-strategy.complex.sharding-columns= #分片列名称，多个列以逗号分隔
spring.shardingsphere.sharding.tables.<logic-table-name>.database-strategy.complex.algorithm-class-name= #复合分片算法类名称。该类需实现ComplexKeysShardingAlgorithm接口并提供无参数的构造器

#行表达式分片策略
spring.shardingsphere.sharding.tables.<logic-table-name>.database-strategy.inline.sharding-column= #分片列名称
spring.shardingsphere.sharding.tables.<logic-table-name>.database-strategy.inline.algorithm-expression= #分片算法行表达式，需符合groovy语法

#Hint分片策略
spring.shardingsphere.sharding.tables.<logic-table-name>.database-strategy.hint.algorithm-class-name= #Hint分片算法类名称。该类需实现HintShardingAlgorithm接口并提供无参数的构造器

#分表策略，同分库策略
spring.shardingsphere.sharding.tables.<logic-table-name>.table-strategy.xxx= #省略

spring.shardingsphere.sharding.tables.<logic-table-name>.key-generator.column= #自增列名称，缺省表示不使用自增主键生成器
spring.shardingsphere.sharding.tables.<logic-table-name>.key-generator.type= #自增列值生成器类型，缺省表示使用默认自增列值生成器。可使用用户自定义的列值生成器或选择内置类型：SNOWFLAKE/UUID/LEAF_SEGMENT
spring.shardingsphere.sharding.tables.<logic-table-name>.key-generator.props.<property-name>= #属性配置, 注意：使用SNOWFLAKE算法，需要配置worker.id与max.tolerate.time.difference.milliseconds属性。若使用此算法生成值作分片值，建议配置max.vibration.offset属性

spring.shardingsphere.sharding.binding-tables[0]= #绑定表规则列表
spring.shardingsphere.sharding.binding-tables[1]= #绑定表规则列表
spring.shardingsphere.sharding.binding-tables[x]= #绑定表规则列表

spring.shardingsphere.sharding.broadcast-tables[0]= #广播表规则列表
spring.shardingsphere.sharding.broadcast-tables[1]= #广播表规则列表
spring.shardingsphere.sharding.broadcast-tables[x]= #广播表规则列表

spring.shardingsphere.sharding.default-data-source-name= #未配置分片规则的表将通过默认数据源定位
spring.shardingsphere.sharding.default-database-strategy.xxx= #默认数据库分片策略，同分库策略
spring.shardingsphere.sharding.default-table-strategy.xxx= #默认表分片策略，同分表策略
spring.shardingsphere.sharding.default-key-generator.type= #默认自增列值生成器类型，缺省将使用org.apache.shardingsphere.core.keygen.generator.impl.SnowflakeKeyGenerator。可使用用户自定义的列值生成器或选择内置类型：SNOWFLAKE/UUID/LEAF_SEGMENT
spring.shardingsphere.sharding.default-key-generator.props.<property-name>= #自增列值生成器属性配置, 比如SNOWFLAKE算法的worker.id与max.tolerate.time.difference.milliseconds

spring.shardingsphere.sharding.master-slave-rules.<master-slave-data-source-name>.master-data-source-name= #详见读写分离部分
spring.shardingsphere.sharding.master-slave-rules.<master-slave-data-source-name>.slave-data-source-names[0]= #详见读写分离部分
spring.shardingsphere.sharding.master-slave-rules.<master-slave-data-source-name>.slave-data-source-names[1]= #详见读写分离部分
spring.shardingsphere.sharding.master-slave-rules.<master-slave-data-source-name>.slave-data-source-names[x]= #详见读写分离部分
spring.shardingsphere.sharding.master-slave-rules.<master-slave-data-source-name>.load-balance-algorithm-class-name= #详见读写分离部分
spring.shardingsphere.sharding.master-slave-rules.<master-slave-data-source-name>.load-balance-algorithm-type= #详见读写分离部分

spring.shardingsphere.props.sql.show= #是否开启SQL显示，默认值: false
spring.shardingsphere.props.executor.size= #工作线程数量，默认值: CPU核数
```

### 读写分离

```properties
#省略数据源配置，与数据分片一致

spring.shardingsphere.sharding.master-slave-rules.<master-slave-data-source-name>.master-data-source-name= #主库数据源名称
spring.shardingsphere.sharding.master-slave-rules.<master-slave-data-source-name>.slave-data-source-names[0]= #从库数据源名称列表
spring.shardingsphere.sharding.master-slave-rules.<master-slave-data-source-name>.slave-data-source-names[1]= #从库数据源名称列表
spring.shardingsphere.sharding.master-slave-rules.<master-slave-data-source-name>.slave-data-source-names[x]= #从库数据源名称列表
spring.shardingsphere.sharding.master-slave-rules.<master-slave-data-source-name>.load-balance-algorithm-class-name= #从库负载均衡算法类名称。该类需实现MasterSlaveLoadBalanceAlgorithm接口且提供无参数构造器
spring.shardingsphere.sharding.master-slave-rules.<master-slave-data-source-name>.load-balance-algorithm-type= #从库负载均衡算法类型，可选值：ROUND_ROBIN，RANDOM。若`load-balance-algorithm-class-name`存在则忽略该配置

spring.shardingsphere.props.sql.show= #是否开启SQL显示，默认值: false
spring.shardingsphere.props.executor.size= #工作线程数量，默认值: CPU核数
spring.shardingsphere.props.check.table.metadata.enabled= #是否在启动时检查分表元数据一致性，默认值: false
```

### 数据脱敏

```properties
#省略数据源配置，与数据分片一致

spring.shardingsphere.encrypt.encryptors.<encryptor-name>.type= #加解密器类型，可自定义或选择内置类型：MD5/AES 
spring.shardingsphere.encrypt.encryptors.<encryptor-name>.props.<property-name>= #属性配置, 注意：使用AES加密器，需要配置AES加密器的KEY属性：aes.key.value
spring.shardingsphere.encrypt.tables.<table-name>.columns.<logic-column-name>.plainColumn= #存储明文的字段
spring.shardingsphere.encrypt.tables.<table-name>.columns.<logic-column-name>.cipherColumn= #存储密文的字段
spring.shardingsphere.encrypt.tables.<table-name>.columns.<logic-column-name>.assistedQueryColumn= #辅助查询字段，针对ShardingQueryAssistedEncryptor类型的加解密器进行辅助查询
spring.shardingsphere.encrypt.tables.<table-name>.columns.<logic-column-name>.encryptor= #加密器名字
```

### 治理

```properties
#省略数据源、数据分片、读写分离和数据脱敏配置

spring.shardingsphere.orchestration.name= #治理实例名称
spring.shardingsphere.orchestration.overwrite= #本地配置是否覆盖注册中心配置。如果可覆盖，每次启动都以本地配置为准
spring.shardingsphere.orchestration.registry.type= #配置中心类型。如：zookeeper
spring.shardingsphere.orchestration.registry.server-lists= #连接注册中心服务器的列表。包括IP地址和端口号。多个地址用逗号分隔。如: host1:2181,host2:2181
spring.shardingsphere.orchestration.registry.namespace= #注册中心的命名空间
spring.shardingsphere.orchestration.registry.digest= #连接注册中心的权限令牌。缺省为不需要权限验证
spring.shardingsphere.orchestration.registry.operation-timeout-milliseconds= #操作超时的毫秒数，默认500毫秒
spring.shardingsphere.orchestration.registry.max-retries= #连接失败后的最大重试次数，默认3次
spring.shardingsphere.orchestration.registry.retry-interval-milliseconds= #重试间隔毫秒数，默认500毫秒
spring.shardingsphere.orchestration.registry.time-to-live-seconds= #临时节点存活秒数，默认60秒
spring.shardingsphere.orchestration.registry.props= #配置中心其它属性
```

# SPRING命名空间配置

## 注意事项

行表达式标识符可以使用`${...}`或`$->{...}`，但前者与Spring本身的属性文件占位符冲突，因此在Spring环境中使用行表达式标识符建议使用`$->{...}`。

## 配置示例

详细example: [shardingsphere-example](https://github.com/apache/incubator-shardingsphere-example/tree/dev/sharding-jdbc-example/sharding-example/sharding-spring-namespace-jpa-example/src/main/resources/META-INF)

### 数据分片

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:p="http://www.springframework.org/schema/p"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:tx="http://www.springframework.org/schema/tx"
       xmlns:sharding="http://shardingsphere.apache.org/schema/shardingsphere/sharding"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
                        http://www.springframework.org/schema/beans/spring-beans.xsd
                        http://shardingsphere.apache.org/schema/shardingsphere/sharding
                        http://shardingsphere.apache.org/schema/shardingsphere/sharding/sharding.xsd
                        http://www.springframework.org/schema/context
                        http://www.springframework.org/schema/context/spring-context.xsd
                        http://www.springframework.org/schema/tx
                        http://www.springframework.org/schema/tx/spring-tx.xsd">
    <context:annotation-config />

    <bean id="entityManagerFactory" class="org.springframework.orm.jpa.LocalContainerEntityManagerFactoryBean">
        <property name="dataSource" ref="shardingDataSource" />
        <property name="jpaVendorAdapter">
            <bean class="org.springframework.orm.jpa.vendor.HibernateJpaVendorAdapter" p:database="MYSQL" />
        </property>
        <property name="packagesToScan" value="org.apache.shardingsphere.example.core.jpa.entity" />
        <property name="jpaProperties">
            <props>
                <prop key="hibernate.dialect">org.hibernate.dialect.MySQLDialect</prop>
                <prop key="hibernate.hbm2ddl.auto">create</prop>
                <prop key="hibernate.show_sql">true</prop>
            </props>
        </property>
    </bean>
    <bean id="transactionManager" class="org.springframework.orm.jpa.JpaTransactionManager" p:entityManagerFactory-ref="entityManagerFactory" />
    <tx:annotation-driven />

    <bean id="ds0" class="org.apache.commons.dbcp.BasicDataSource" destroy-method="close">
        <property name="driverClassName" value="com.mysql.jdbc.Driver" />
        <property name="url" value="jdbc:mysql://localhost:3306/ds0" />
        <property name="username" value="root" />
        <property name="password" value="" />
    </bean>

    <bean id="ds1" class="org.apache.commons.dbcp.BasicDataSource" destroy-method="close">
        <property name="driverClassName" value="com.mysql.jdbc.Driver" />
        <property name="url" value="jdbc:mysql://localhost:3306/ds1" />
        <property name="username" value="root" />
        <property name="password" value="" />
    </bean>

    <bean id="preciseModuloDatabaseShardingAlgorithm" class="org.apache.shardingsphere.example.algorithm.PreciseModuloShardingDatabaseAlgorithm" />
    <bean id="preciseModuloTableShardingAlgorithm" class="org.apache.shardingsphere.example.algorithm.PreciseModuloShardingTableAlgorithm" />

    <sharding:standard-strategy id="databaseShardingStrategy" sharding-column="user_id" precise-algorithm-ref="preciseModuloDatabaseShardingAlgorithm" />
    <sharding:standard-strategy id="tableShardingStrategy" sharding-column="order_id" precise-algorithm-ref="preciseModuloTableShardingAlgorithm" />

    <sharding:key-generator id="orderKeyGenerator" type="SNOWFLAKE" column="order_id" />
    <sharding:key-generator id="itemKeyGenerator" type="SNOWFLAKE" column="order_item_id" />

    <sharding:data-source id="shardingDataSource">
        <sharding:sharding-rule data-source-names="ds0,ds1">
            <sharding:table-rules>
                <sharding:table-rule logic-table="t_order" actual-data-nodes="ds$->{0..1}.t_order$->{0..1}" database-strategy-ref="databaseShardingStrategy" table-strategy-ref="tableShardingStrategy" key-generator-ref="orderKeyGenerator" />
                <sharding:table-rule logic-table="t_order_item" actual-data-nodes="ds$->{0..1}.t_order_item$->{0..1}" database-strategy-ref="databaseShardingStrategy" table-strategy-ref="tableShardingStrategy" key-generator-ref="itemKeyGenerator" />
            </sharding:table-rules>
            <sharding:binding-table-rules>
                <sharding:binding-table-rule logic-tables="t_order, t_order_item" />
            </sharding:binding-table-rules>
            <sharding:broadcast-table-rules>
                <sharding:broadcast-table-rule table="t_config" />
            </sharding:broadcast-table-rules>
        </sharding:sharding-rule>
    </sharding:data-source>
</beans>
```

### 读写分离

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:p="http://www.springframework.org/schema/p"
       xmlns:tx="http://www.springframework.org/schema/tx"
       xmlns:master-slave="http://shardingsphere.apache.org/schema/shardingsphere/masterslave"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
                        http://www.springframework.org/schema/beans/spring-beans.xsd
                        http://www.springframework.org/schema/context
                        http://www.springframework.org/schema/context/spring-context.xsd
                        http://www.springframework.org/schema/tx
                        http://www.springframework.org/schema/tx/spring-tx.xsd
                        http://shardingsphere.apache.org/schema/shardingsphere/masterslave
                        http://shardingsphere.apache.org/schema/shardingsphere/masterslave/master-slave.xsd">
    <context:annotation-config />
    <context:component-scan base-package="org.apache.shardingsphere.example.core.jpa" />

    <bean id="entityManagerFactory" class="org.springframework.orm.jpa.LocalContainerEntityManagerFactoryBean">
        <property name="dataSource" ref="masterSlaveDataSource" />
        <property name="jpaVendorAdapter">
            <bean class="org.springframework.orm.jpa.vendor.HibernateJpaVendorAdapter" p:database="MYSQL" />
        </property>
        <property name="packagesToScan" value="org.apache.shardingsphere.example.core.jpa.entity" />
        <property name="jpaProperties">
            <props>
                <prop key="hibernate.dialect">org.hibernate.dialect.MySQLDialect</prop>
                <prop key="hibernate.hbm2ddl.auto">create</prop>
                <prop key="hibernate.show_sql">true</prop>
            </props>
        </property>
    </bean>
    <bean id="transactionManager" class="org.springframework.orm.jpa.JpaTransactionManager" p:entityManagerFactory-ref="entityManagerFactory" />
    <tx:annotation-driven />

    <bean id="ds_master" class="org.apache.commons.dbcp.BasicDataSource" destroy-method="close">
        <property name="driverClassName" value="com.mysql.jdbc.Driver" />
        <property name="url" value="jdbc:mysql://localhost:3306/ds_master" />
        <property name="username" value="root" />
        <property name="password" value="" />
    </bean>

    <bean id="ds_slave0" class="org.apache.commons.dbcp.BasicDataSource" destroy-method="close">
        <property name="driverClassName" value="com.mysql.jdbc.Driver" />
        <property name="url" value="jdbc:mysql://localhost:3306/ds_slave0" />
        <property name="username" value="root" />
        <property name="password" value="" />
    </bean>

    <bean id="ds_slave1" class="org.apache.commons.dbcp.BasicDataSource" destroy-method="close">
        <property name="driverClassName" value="com.mysql.jdbc.Driver" />
        <property name="url" value="jdbc:mysql://localhost:3306/ds_slave1" />
        <property name="username" value="root" />
        <property name="password" value="" />
    </bean>

    <!-- 4.0.0-RC1 版本 负载均衡策略配置方式 -->
    <!-- <bean id="randomStrategy" class="org.apache.shardingsphere.example.spring.namespace.algorithm.masterslave.RandomMasterSlaveLoadBalanceAlgorithm" /> -->

    <!-- 4.0.0-RC2 之后版本 负载均衡策略配置方式 -->
    <master-slave:load-balance-algorithm id="randomStrategy" type="RANDOM" />

    <master-slave:data-source id="masterSlaveDataSource" master-data-source-name="ds_master" slave-data-source-names="ds_slave0, ds_slave1" strategy-ref="randomStrategy">
        <master-slave:props>
            <prop key="sql.show">true</prop>
            <prop key="executor.size">10</prop>
            <prop key="foo">bar</prop>
        </master-slave:props>
    </master-slave:data-source>
</beans>
```

### 数据脱敏

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:encrypt="http://shardingsphere.apache.org/schema/shardingsphere/encrypt"
       xmlns:bean="http://www.springframework.org/schema/util"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
                        http://www.springframework.org/schema/beans/spring-beans.xsd 
                        http://shardingsphere.apache.org/schema/shardingsphere/encrypt
                        http://shardingsphere.apache.org/schema/shardingsphere/encrypt/encrypt.xsd 
                        http://www.springframework.org/schema/util 
                        http://www.springframework.org/schema/util/spring-util.xsd">
   
    <bean id="ds" class="com.zaxxer.hikari.HikariDataSource" destroy-method="close">
        <property name="driverClassName" value="com.mysql.jdbc.Driver"/>
        <property name="jdbcUrl" value="jdbc:mysql://localhost:3306/demo_ds?useSSL=false&amp;useUnicode=true&amp;characterEncoding=UTF-8"/>
        <property name="username" value="root"/>
        <property name="password" value=""/>
    </bean>
    
    <bean:properties id="props">
        <prop key="aes.key.value">123456</prop>
    </bean:properties>
    
    <encrypt:data-source id="encryptDataSource" data-source-name="ds" >
        <encrypt:encrypt-rule>
            <encrypt:tables>
                <encrypt:table name="t_order">
                    <encrypt:column logic-column="user_id" plain-column="user_decrypt" cipher-column="user_encrypt" assisted-query-column="user_assisted" encryptor-ref="encryptor_aes" />
                    <encrypt:column logic-column="order_id" plain-column="order_decrypt" cipher-column="order_encrypt" assisted-query-column="order_assisted" encryptor-ref="encryptor_md5"/>
                </encrypt:table>
            </encrypt:tables>
            <encrypt:encryptors>
                <encrypt:encryptor id="encryptor_aes" type="AES" props-ref="props"/>
                <encrypt:encryptor id="encryptor_md5" type="MD5" />
            </encrypt:encryptors>
        </encrypt:encrypt-rule>
        <encrypt:props>
            <prop key="sql.show">true</prop>
            <prop key="query.with.cipher.column">true</prop>
        </encrypt:props>
    </encrypt:data-source>
</beans>
```

### 数据分片 + 读写分离

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:p="http://www.springframework.org/schema/p"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:tx="http://www.springframework.org/schema/tx"
       xmlns:sharding="http://shardingsphere.apache.org/schema/shardingsphere/sharding"
       xmlns:master-slave="http://shardingsphere.apache.org/schema/shardingsphere/masterslave"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
                        http://www.springframework.org/schema/beans/spring-beans.xsd
                        http://www.springframework.org/schema/context
                        http://www.springframework.org/schema/context/spring-context.xsd
                        http://www.springframework.org/schema/tx
                        http://www.springframework.org/schema/tx/spring-tx.xsd
                        http://shardingsphere.apache.org/schema/shardingsphere/sharding
                        http://shardingsphere.apache.org/schema/shardingsphere/sharding/sharding.xsd
                        http://shardingsphere.apache.org/schema/shardingsphere/masterslave
                        http://shardingsphere.apache.org/schema/shardingsphere/masterslave/master-slave.xsd">
    <context:annotation-config />
    <context:component-scan base-package="org.apache.shardingsphere.example.core.jpa" />

    <bean id="entityManagerFactory" class="org.springframework.orm.jpa.LocalContainerEntityManagerFactoryBean">
        <property name="dataSource" ref="shardingDataSource" />
        <property name="jpaVendorAdapter">
            <bean class="org.springframework.orm.jpa.vendor.HibernateJpaVendorAdapter" p:database="MYSQL" />
        </property>
        <property name="packagesToScan" value="org.apache.shardingsphere.example.core.jpa.entity" />
        <property name="jpaProperties">
            <props>
                <prop key="hibernate.dialect">org.hibernate.dialect.MySQLDialect</prop>
                <prop key="hibernate.hbm2ddl.auto">create</prop>
                <prop key="hibernate.show_sql">true</prop>
            </props>
        </property>
    </bean>
    <bean id="transactionManager" class="org.springframework.orm.jpa.JpaTransactionManager" p:entityManagerFactory-ref="entityManagerFactory" />
    <tx:annotation-driven />

    <bean id="ds_master0" class="org.apache.commons.dbcp.BasicDataSource" destroy-method="close">
        <property name="driverClassName" value="com.mysql.jdbc.Driver" />
        <property name="url" value="jdbc:mysql://localhost:3306/ds_master0" />
        <property name="username" value="root" />
        <property name="password" value="" />
    </bean>

    <bean id="ds_master0_slave0" class="org.apache.commons.dbcp.BasicDataSource" destroy-method="close">
        <property name="driverClassName" value="com.mysql.jdbc.Driver" />
        <property name="url" value="jdbc:mysql://localhost:3306/ds_master0_slave0" />
        <property name="username" value="root" />
        <property name="password" value="" />
    </bean>

    <bean id="ds_master0_slave1" class="org.apache.commons.dbcp.BasicDataSource" destroy-method="close">
        <property name="driverClassName" value="com.mysql.jdbc.Driver" />
        <property name="url" value="jdbc:mysql://localhost:3306/ds_master0_slave1" />
        <property name="username" value="root" />
        <property name="password" value="" />
    </bean>

    <bean id="ds_master1" class="org.apache.commons.dbcp.BasicDataSource" destroy-method="close">
        <property name="driverClassName" value="com.mysql.jdbc.Driver" />
        <property name="url" value="jdbc:mysql://localhost:3306/ds_master1" />
        <property name="username" value="root" />
        <property name="password" value="" />
    </bean>

    <bean id="ds_master1_slave0" class="org.apache.commons.dbcp.BasicDataSource" destroy-method="close">
        <property name="driverClassName" value="com.mysql.jdbc.Driver" />
        <property name="url" value="jdbc:mysql://localhost:3306/ds_master1_slave0" />
        <property name="username" value="root" />
        <property name="password" value="" />
    </bean>

    <bean id="ds_master1_slave1" class="org.apache.commons.dbcp.BasicDataSource" destroy-method="close">
        <property name="driverClassName" value="com.mysql.jdbc.Driver" />
        <property name="url" value="jdbc:mysql://localhost:3306/ds_master1_slave1" />
        <property name="username" value="root" />
        <property name="password" value="" />
    </bean>

    <!-- 4.0.0-RC1 版本 负载均衡策略配置方式 -->
    <!-- <bean id="randomStrategy" class="org.apache.shardingsphere.example.spring.namespace.algorithm.masterslave.RandomMasterSlaveLoadBalanceAlgorithm" /> -->

    <!-- 4.0.0-RC2 之后版本 负载均衡策略配置方式 -->
    <master-slave:load-balance-algorithm id="randomStrategy" type="RANDOM" />

    <sharding:inline-strategy id="databaseStrategy" sharding-column="user_id" algorithm-expression="ds_ms$->{user_id % 2}" />
    <sharding:inline-strategy id="orderTableStrategy" sharding-column="order_id" algorithm-expression="t_order$->{order_id % 2}" />
    <sharding:inline-strategy id="orderItemTableStrategy" sharding-column="order_id" algorithm-expression="t_order_item$->{order_id % 2}" />

    <sharding:key-generator id="orderKeyGenerator" type="SNOWFLAKE" column="order_id" />
    <sharding:key-generator id="itemKeyGenerator" type="SNOWFLAKE" column="order_item_id" />

    <sharding:data-source id="shardingDataSource">
        <sharding:sharding-rule data-source-names="ds_master0,ds_master0_slave0,ds_master0_slave1,ds_master1,ds_master1_slave0,ds_master1_slave1">
            <sharding:master-slave-rules>
                <sharding:master-slave-rule id="ds_ms0" master-data-source-name="ds_master0" slave-data-source-names="ds_master0_slave0, ds_master0_slave1" strategy-ref="randomStrategy" />
                <sharding:master-slave-rule id="ds_ms1" master-data-source-name="ds_master1" slave-data-source-names="ds_master1_slave0, ds_master1_slave1" strategy-ref="randomStrategy" />
            </sharding:master-slave-rules>
            <sharding:table-rules>
                <sharding:table-rule logic-table="t_order" actual-data-nodes="ds_ms$->{0..1}.t_order$->{0..1}" database-strategy-ref="databaseStrategy" table-strategy-ref="orderTableStrategy" key-generator-ref="orderKeyGenerator" />
                <sharding:table-rule logic-table="t_order_item" actual-data-nodes="ds_ms$->{0..1}.t_order_item$->{0..1}" database-strategy-ref="databaseStrategy" table-strategy-ref="orderItemTableStrategy" key-generator-ref="itemKeyGenerator" />
            </sharding:table-rules>
            <sharding:binding-table-rules>
                <sharding:binding-table-rule logic-tables="t_order, t_order_item" />
            </sharding:binding-table-rules>
            <sharding:broadcast-table-rules>
                <sharding:broadcast-table-rule table="t_config" />
            </sharding:broadcast-table-rules>
        </sharding:sharding-rule>
    </sharding:data-source>
</beans>
```

### 数据分片 + 数据脱敏

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:p="http://www.springframework.org/schema/p"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:encrypt="http://shardingsphere.apache.org/schema/shardingsphere/encrypt"
       xmlns:tx="http://www.springframework.org/schema/tx"
       xmlns:bean="http://www.springframework.org/schema/util"
       xmlns:sharding="http://shardingsphere.apache.org/schema/shardingsphere/sharding"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
                        http://www.springframework.org/schema/beans/spring-beans.xsd
                        http://shardingsphere.apache.org/schema/shardingsphere/sharding
                        http://shardingsphere.apache.org/schema/shardingsphere/sharding/sharding.xsd
                        http://www.springframework.org/schema/context
                        http://www.springframework.org/schema/context/spring-context.xsd
                        http://shardingsphere.apache.org/schema/shardingsphere/encrypt
                        http://shardingsphere.apache.org/schema/shardingsphere/encrypt/encrypt.xsd
                        http://www.springframework.org/schema/tx
                        http://www.springframework.org/schema/tx/spring-tx.xsd
                        http://www.springframework.org/schema/util http://www.springframework.org/schema/util/spring-util.xsd">
    <import resource="classpath:META-INF/shardingTransaction.xml"/>
    <context:annotation-config />
    <context:component-scan base-package="org.apache.shardingsphere.example.core.jpa"/>

    <bean id="entityManagerFactory" class="org.springframework.orm.jpa.LocalContainerEntityManagerFactoryBean">
        <property name="dataSource" ref="shardingDataSource" />
        <property name="jpaVendorAdapter">
            <bean class="org.springframework.orm.jpa.vendor.HibernateJpaVendorAdapter" p:database="MYSQL" />
        </property>
        <property name="packagesToScan" value="org.apache.shardingsphere.example.core.jpa.entity" />
        <property name="jpaProperties">
            <props>
                <prop key="hibernate.dialect">org.hibernate.dialect.MySQLDialect</prop>
                <prop key="hibernate.hbm2ddl.auto">create-drop</prop>
                <prop key="hibernate.show_sql">true</prop>
            </props>
        </property>
    </bean>
    <bean id="transactionManager" class="org.springframework.orm.jpa.JpaTransactionManager" p:entityManagerFactory-ref="entityManagerFactory" />
    <tx:annotation-driven />

    <bean id="demo_ds_0" class="com.zaxxer.hikari.HikariDataSource" destroy-method="close">
        <property name="driverClassName" value="com.mysql.jdbc.Driver"/>
        <property name="jdbcUrl" value="jdbc:mysql://localhost:3306/demo_ds_0"/>
        <property name="username" value="root"/>
        <property name="password" value=""/>
        <property name="maximumPoolSize" value="16"/>
    </bean>
    
    <bean id="demo_ds_1" class="com.zaxxer.hikari.HikariDataSource" destroy-method="close">
        <property name="driverClassName" value="com.mysql.jdbc.Driver"/>
        <property name="jdbcUrl" value="jdbc:mysql://localhost:3306/demo_ds_1"/>
        <property name="username" value="root"/>
        <property name="password" value=""/>
        <property name="maximumPoolSize" value="16"/>
    </bean>

    <sharding:inline-strategy id="databaseStrategy" sharding-column="user_id" algorithm-expression="demo_ds_${user_id % 2}" />
    <sharding:inline-strategy id="orderTableStrategy" sharding-column="order_id" algorithm-expression="t_order_${order_id % 2}" />
    <sharding:inline-strategy id="orderItemTableStrategy" sharding-column="order_id" algorithm-expression="t_order_item_${order_id % 2}" />
    <sharding:inline-strategy id="orderEncryptTableStrategy" sharding-column="order_id" algorithm-expression="t_order_encrypt_${order_id % 2}" />

    <sharding:key-generator id="orderKeyGenerator" type="SNOWFLAKE" column="order_id" />
    <sharding:key-generator id="itemKeyGenerator" type="SNOWFLAKE" column="order_item_id" />

    <bean:properties id="dataProtectorProps">
        <prop key="appToken">business</prop>
    </bean:properties>

    <sharding:data-source id="shardingDataSource">
        <sharding:sharding-rule data-source-names="demo_ds_0, demo_ds_1">
            <sharding:table-rules>
                <sharding:table-rule logic-table="t_order" actual-data-nodes="demo_ds_${0..1}.t_order_${0..1}" database-strategy-ref="databaseStrategy" table-strategy-ref="orderTableStrategy" key-generator-ref="orderKeyGenerator" />
                <sharding:table-rule logic-table="t_order_item" actual-data-nodes="demo_ds_${0..1}.t_order_item_${0..1}" database-strategy-ref="databaseStrategy" table-strategy-ref="orderItemTableStrategy" key-generator-ref="itemKeyGenerator" />
            </sharding:table-rules>
            <sharding:encrypt-rule>
                <encrypt:tables>
                    <encrypt:table name="t_order">
                        <encrypt:column logic-column="user_id" plain-column="user_decrypt" cipher-column="user_encrypt" assisted-query-column="user_assisted" encryptor-ref="encryptor_aes" />
                        <encrypt:column logic-column="order_id" plain-column="order_decrypt" cipher-column="order_encrypt" assisted-query-column="order_assisted" encryptor-ref="encryptor_md5"/>
                    </encrypt:table>
                </encrypt:tables>
                <encrypt:encryptors>
                    <encrypt:encryptor id="encryptor_aes" type="AES"  props-ref="props"/>
                    <encrypt:encryptor id="encryptor_md5" type="MD5" />
                </encrypt:encryptors>
            </sharding:encrypt-rule>

        </sharding:sharding-rule>

        <sharding:props>
            <prop key="sql.show">true</prop>
        </sharding:props>
    </sharding:data-source>
     <bean:properties id="props">
        <prop key="aes.key.value">123456</prop>
    </bean:properties>
</beans>
```

### 治理

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:orchestration="http://shardingsphere.apache.org/schema/shardingsphere/orchestration"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
                           http://www.springframework.org/schema/beans/spring-beans.xsd
                           http://shardingsphere.apache.org/schema/shardingsphere/orchestration
                           http://shardingsphere.apache.org/schema/shardingsphere/orchestration/orchestration.xsd">
    <orchestration:registry-center id="regCenter" type="zookeeper" server-lists="localhost:2181" namespace="orchestration-spring-namespace-demo" operation-timeout-milliseconds="1000" 
    max-retries="3" />
</beans>
```

## 配置项说明

### 分库分表

命名空间：http://shardingsphere.apache.org/schema/shardingsphere/sharding/sharding.xsd

#### <sharding:data-source />

| *名称*        | *类型* | *说明*           |
| :------------ | :----- | :--------------- |
| id            | 属性   | Spring Bean Id   |
| sharding-rule | 标签   | 数据分片配置规则 |
| props (?)     | 标签   | 属性配置         |

#### <sharding:sharding-rule />

| *名称*                            | *类型* | *说明*                                                       |
| :-------------------------------- | :----- | :----------------------------------------------------------- |
| data-source-names                 | 属性   | 数据源Bean列表，多个Bean以逗号分隔                           |
| table-rules                       | 标签   | 表分片规则配置对象                                           |
| binding-table-rules (?)           | 标签   | 绑定表规则列表                                               |
| broadcast-table-rules (?)         | 标签   | 广播表规则列表                                               |
| default-data-source-name (?)      | 属性   | 未配置分片规则的表将通过默认数据源定位                       |
| default-database-strategy-ref (?) | 属性   | 默认数据库分片策略，对应<sharding:xxx-strategy>中的策略Id，缺省表示不分库 |
| default-table-strategy-ref (?)    | 属性   | 默认表分片策略，对应<sharding:xxx-strategy>中的策略Id，缺省表示不分表 |
| default-key-generator-ref (?)     | 属性   | 默认自增列值生成器引用，缺省使用`org.apache.shardingsphere.core.keygen.generator.impl.SnowflakeKeyGenerator` |
| encrypt-rule (?)                  | 标签   | 脱敏规则                                                     |

#### <sharding:table-rules />

| *名称*         | *类型* | *说明*             |
| :------------- | :----- | :----------------- |
| table-rule (+) | 标签   | 表分片规则配置对象 |

#### <sharding:table-rule />

| *名称*                    | *类型* | *说明*                                                       |
| :------------------------ | :----- | :----------------------------------------------------------- |
| logic-table               | 属性   | 逻辑表名称                                                   |
| actual-data-nodes (?)     | 属性   | 由数据源名 + 表名组成，以小数点分隔。多个表以逗号分隔，支持inline表达式。缺省表示使用已知数据源与逻辑表名称生成数据节点。用于广播表（即每个库中都需要一个同样的表用于关联查询，多为字典表）或只分库不分表且所有库的表结构完全一致的情况 |
| database-strategy-ref (?) | 属性   | 数据库分片策略，对应<sharding:xxx-strategy>中的策略Id，缺省表示使用<sharding:sharding-rule />配置的默认数据库分片策略 |
| table-strategy-ref (?)    | 属性   | 表分片策略，对应<sharding:xxx-strategy>中的策略Id，缺省表示使用<sharding:sharding-rule />配置的默认表分片策略 |
| key-generator-ref (?)     | 属性   | 自增列值生成器引用，缺省表示使用默认自增列值生成器           |

#### <sharding:binding-table-rules />

| *名称*                 | *类型* | *说明*     |
| :--------------------- | :----- | :--------- |
| binding-table-rule (+) | 标签   | 绑定表规则 |

#### <sharding:binding-table-rule />

| *名称*       | *类型* | *说明*                             |
| :----------- | :----- | :--------------------------------- |
| logic-tables | 属性   | 绑定规则的逻辑表名，多表以逗号分隔 |

#### <sharding:broadcast-table-rules />

| *名称*                   | *类型* | *说明*     |
| :----------------------- | :----- | :--------- |
| broadcast-table-rule (+) | 标签   | 广播表规则 |

#### <sharding:broadcast-table-rule />

| *名称* | *类型* | *说明          |
| :----- | :----- | :------------- |
| table  | 属性   | 广播规则的表名 |

#### <sharding:standard-strategy />

| *名称*                  | *类型* | *说明*                                                       |
| :---------------------- | :----- | :----------------------------------------------------------- |
| id                      | 属性   | Spring Bean Id                                               |
| sharding-column         | 属性   | 分片列名称                                                   |
| precise-algorithm-ref   | 属性   | 精确分片算法引用，用于=和IN。该类需实现PreciseShardingAlgorithm接口 |
| range-algorithm-ref (?) | 属性   | 范围分片算法引用，用于BETWEEN。该类需实现RangeShardingAlgorithm接口 |

#### <sharding:complex-strategy />

| *名称*           | *类型* | *说明*                                                       |
| :--------------- | :----- | :----------------------------------------------------------- |
| id               | 属性   | Spring Bean Id                                               |
| sharding-columns | 属性   | 分片列名称，多个列以逗号分隔                                 |
| algorithm-ref    | 属性   | 复合分片算法引用。该类需实现ComplexKeysShardingAlgorithm接口 |

#### <sharding:inline-strategy />

| *名称*               | *类型* | *说明*                             |
| :------------------- | :----- | :--------------------------------- |
| id                   | 属性   | Spring Bean Id                     |
| sharding-column      | 属性   | 分片列名称                         |
| algorithm-expression | 属性   | 分片算法行表达式，需符合groovy语法 |

#### <sharding:hint-database-strategy />

| *名称*        | *类型* | *说明*                                            |
| :------------ | :----- | :------------------------------------------------ |
| id            | 属性   | Spring Bean Id                                    |
| algorithm-ref | 属性   | Hint分片算法。该类需实现HintShardingAlgorithm接口 |

#### <sharding:none-strategy />

| *名称* | *类型* | *说明*         |
| :----- | :----- | :------------- |
| id     | 属性   | Spring Bean Id |

#### <sharding:key-generator />

| *名称*    | *类型* | *说明*                                                       |
| :-------- | :----- | :----------------------------------------------------------- |
| column    | 属性   | 自增列名称                                                   |
| type      | 属性   | 自增列值生成器类型，可自定义或选择内置类型：SNOWFLAKE/UUID/LEAF_SEGMENT/LEAF_SNOWFLAKE |
| props-ref | 属性   | 自增列值生成器的属性配置引用                                 |

#### PropertiesConstant

属性配置项，可以为以下自增列值生成器的属性。

##### SNOWFLAKE

| *名称*                                        | *数据类型* | *说明*                                                       |
| :-------------------------------------------- | :--------- | :----------------------------------------------------------- |
| worker.id (?)                                 | long       | 工作机器唯一id，默认为0                                      |
| max.tolerate.time.difference.milliseconds (?) | long       | 最大容忍时钟回退时间，单位：毫秒。默认为10毫秒               |
| max.vibration.offset (?)                      | int        | 最大抖动上限值，范围[0, 4096)，默认为1。注：若使用此算法生成值作分片值，建议配置此属性。此算法在不同毫秒内所生成的key取模2^n (2^n一般为分库或分表数) 之后结果总为0或1。为防止上述分片问题，建议将此属性值配置为(2^n)-1 |

##### LEAF_SEGMENT

| *名称*                            | *数据类型* | *说明*                                                       |
| :-------------------------------- | :--------- | :----------------------------------------------------------- |
| server.list                       | String     | 连接注册中心服务器的列表，包括IP地址和端口号，多个地址用逗号分隔。如: host1:2181,host2:2181 |
| leaf.key                          | String     | 用于标识leaf_segment所对应表里面的最大号段id                 |
| leaf.segment.id.initial.value (?) | long       | 号段id初始值，默认为1                                        |
| leaf.segment.step (?)             | long       | 每次分配的号段步长，默认为10000                              |
| registry.center.digest (?)        | String     | 连接注册中心的权限令牌。缺省为不需要权限验证                 |
| registry.center.type (?)          | String     | 注册中心类型，默认为`zookeeper`                              |

##### LEAF_SNOWFLAKE

| *名称*                                        | *数据类型* | *说明*                                                       |
| :-------------------------------------------- | :--------- | :----------------------------------------------------------- |
| server.list                                   | String     | 连接注册中心服务器的列表，包括IP地址和端口号，多个地址用逗号分隔。如: host1:2181,host2:2181 |
| service.id                                    | String     | leaf_snowflake位于注册中心上的服务标识                       |
| max.tolerate.time.difference.milliseconds (?) | long       | 最大允许的本机与注册中心的时间误差毫秒数，默认为10000毫秒。如果时间误差超过配置时间则作业启动时将抛异常 |
| registry.center.digest (?)                    | String     | 连接注册中心的权限令牌。缺省为不需要权限验证                 |
| registry.center.type (?)                      | String     | 注册中心类型，默认为`zookeeper`                              |
| max.vibration.offset (?)                      | int        | 最大抖动上限，范围[0, 4096)，默认为1。注：若使用此算法生成值作分片值，建议配置此属性。此算法在不同毫秒内所生成的key取模2^n (2^n一般为分库或分表数) 之后结果总为0或1。为防止上述分片问题，建议将此属性值配置为(2^n)-1 |

#### <sharding:encrypt-rule />

| *名称*                  | *类型* | *说明*     |
| :---------------------- | :----- | :--------- |
| encrypt:encrypt-rule(?) | 标签   | 加解密规则 |

#### <sharding:props />

| *名称*                             | *类型* | *说明*                                                |
| :--------------------------------- | :----- | :---------------------------------------------------- |
| sql.show (?)                       | 属性   | 是否开启SQL显示，默认值: false                        |
| executor.size (?)                  | 属性   | 工作线程数量，默认值: CPU核数                         |
| max.connections.size.per.query (?) | 属性   | 每个物理数据库为每次查询分配的最大连接数量。默认值: 1 |
| check.table.metadata.enabled (?)   | 属性   | 是否在启动时检查分表元数据一致性，默认值: false       |
| query.with.cipher.column (?)       | 属性   | 当存在明文列时，是否使用密文列查询，默认值: true      |

### 读写分离

命名空间：http://shardingsphere.apache.org/schema/shardingsphere/masterslave/master-slave.xsd

#### <master-slave:data-source />

| *名称*                  | *类型* | *说明*                                                       |
| :---------------------- | :----- | :----------------------------------------------------------- |
| id                      | 属性   | Spring Bean Id                                               |
| master-data-source-name | 属性   | 主库数据源Bean Id                                            |
| slave-data-source-names | 属性   | 从库数据源Bean Id列表，多个Bean以逗号分隔                    |
| strategy-ref (?)        | 属性   | 从库负载均衡算法引用。该类需实现MasterSlaveLoadBalanceAlgorithm接口 |
| strategy-type (?)       | 属性   | 从库负载均衡算法类型，可选值：ROUND_ROBIN，RANDOM。若`strategy-ref`存在则忽略该配置 |
| props (?)               | 标签   | 属性配置                                                     |

#### <master-slave:props />

| *名称*                             | *类型* | *说明*                                                |
| :--------------------------------- | :----- | :---------------------------------------------------- |
| sql.show (?)                       | 属性   | 是否开启SQL显示，默认值: false                        |
| executor.size (?)                  | 属性   | 工作线程数量，默认值: CPU核数                         |
| max.connections.size.per.query (?) | 属性   | 每个物理数据库为每次查询分配的最大连接数量。默认值: 1 |
| check.table.metadata.enabled (?)   | 属性   | 是否在启动时检查分表元数据一致性，默认值: false       |

#### <master-slave:load-balance-algorithm />

4.0.0-RC2 版本 添加

| *名称*        | *类型* | *说明*                                                    |
| :------------ | :----- | :-------------------------------------------------------- |
| id            | 属性   | Spring Bean Id                                            |
| type          | 属性   | 负载均衡算法类型，’RANDOM’或’ROUND_ROBIN’，支持自定义拓展 |
| props-ref (?) | 属性   | 负载均衡算法配置参数                                      |

### 数据脱敏

命名空间：http://shardingsphere.apache.org/schema/shardingsphere/encrypt/encrypt.xsd

#### <encrypt:data-source />

| *名称*           | *类型* | *说明*            |
| :--------------- | :----- | :---------------- |
| id               | 属性   | Spring Bean Id    |
| data-source-name | 属性   | 加密数据源Bean Id |
| props (?)        | 标签   | 属性配置          |

#### <encrypt:encryptors />

| *名称*       | *类型* | *说明*     |
| :----------- | :----- | :--------- |
| encryptor(+) | 标签   | 加密器配置 |

#### <encrypt:encryptor />

| *名称*    | *类型* | *说明*                                                       |
| :-------- | :----- | :----------------------------------------------------------- |
| id        | 属性   | 加密器的名称                                                 |
| type      | 属性   | 加解密器类型，可自定义或选择内置类型：MD5/AES                |
| props-ref | 属性   | 属性配置, 注意：使用AES加密器，需要配置AES加密器的KEY属性：aes.key.value |

#### <encrypt:tables />

| *名称*   | *类型* | *说明*     |
| :------- | :----- | :--------- |
| table(+) | 标签   | 加密表配置 |

#### <encrypt:table />

| *名称*    | *类型* | *说明*     |
| :-------- | :----- | :--------- |
| column(+) | 标签   | 加密列配置 |

#### <encrypt:column />

| *名称*                 | *类型* | *说明*                                                       |
| :--------------------- | :----- | :----------------------------------------------------------- |
| logic-column           | 属性   | 逻辑列名                                                     |
| plain-column           | 属性   | 存储明文的字段                                               |
| cipher-column          | 属性   | 存储密文的字段                                               |
| assisted-query-columns | 属性   | 辅助查询字段，针对ShardingQueryAssistedEncryptor类型的加解密器进行辅助查询 |

#### <encrypt:props />

| *名称*                       | *类型* | *说明*                                           |
| :--------------------------- | :----- | :----------------------------------------------- |
| sql.show (?)                 | 属性   | 是否开启SQL显示，默认值: false                   |
| query.with.cipher.column (?) | 属性   | 当存在明文列时，是否使用密文列查询，默认值: true |

### 数据分片 + 治理

命名空间：http://shardingsphere.apache.org/schema/shardingsphere/orchestration/orchestration.xsd

#### <orchestration:sharding-data-source />

| *名称*              | *类型* | *说明*                                                       |
| :------------------ | :----- | :----------------------------------------------------------- |
| id                  | 属性   | ID                                                           |
| data-source-ref (?) | 属性   | 被治理的数据库id                                             |
| registry-center-ref | 属性   | 注册中心id                                                   |
| overwrite           | 属性   | 本地配置是否覆盖注册中心配置。如果可覆盖，每次启动都以本地配置为准。缺省为不覆盖 |

### 读写分离 + 治理

命名空间：http://shardingsphere.apache.org/schema/shardingsphere/orchestration/orchestration.xsd

#### <orchestration:master-slave-data-source />

| *名称*              | *类型* | *说明*                                                       |
| :------------------ | :----- | :----------------------------------------------------------- |
| id                  | 属性   | ID                                                           |
| data-source-ref (?) | 属性   | 被治理的数据库id                                             |
| registry-center-ref | 属性   | 注册中心id                                                   |
| overwrite           | 属性   | 本地配置是否覆盖注册中心配置。如果可覆盖，每次启动都以本地配置为准。缺省为不覆盖 |

### 数据脱敏 + 治理

命名空间：http://shardingsphere.apache.org/schema/shardingsphere/orchestration/orchestration.xsd

#### <orchestration:encrypt-data-source />

| *名称*              | *类型* | *说明*                                                       |
| :------------------ | :----- | :----------------------------------------------------------- |
| id                  | 属性   | ID                                                           |
| data-source-ref (?) | 属性   | 被治理的数据库id                                             |
| registry-center-ref | 属性   | 注册中心id                                                   |
| overwrite           | 属性   | 本地配置是否覆盖注册中心配置。如果可覆盖，每次启动都以本地配置为准。缺省为不覆盖 |

### 治理注册中心

命名空间：http://shardingsphere.apache.org/schema/shardingsphere/orchestration/orchestration.xsd

#### <orchestration:registry-center />

| *名称*                             | *类型* | *说明*                                                       |
| :--------------------------------- | :----- | :----------------------------------------------------------- |
| id                                 | 属性   | 注册中心的Spring Bean Id                                     |
| type                               | 属性   | 注册中心类型。如：zookeeper                                  |
| server-lists                       | 属性   | 连接注册中心服务器的列表，包括IP地址和端口号，多个地址用逗号分隔。如: host1:2181,host2:2181 |
| namespace (?)                      | 属性   | 注册中心的命名空间                                           |
| digest (?)                         | 属性   | 连接注册中心的权限令牌。缺省为不需要权限验证                 |
| operation-timeout-milliseconds (?) | 属性   | 操作超时的毫秒数，默认500毫秒                                |
| max-retries (?)                    | 属性   | 连接失败后的最大重试次数，默认3次                            |
| retry-interval-milliseconds (?)    | 属性   | 重试间隔毫秒数，默认500毫秒                                  |
| time-to-live-seconds (?)           | 属性   | 临时节点存活秒数，默认60秒                                   |
| props-ref (?)                      | 属性   | 配置中心其它属性                                             |