---
displayed_sidebar: docs
keywords: ['fenqu','fentong', 'lengre']
toc_max_heading_level: 4
description: 分区与分桶
sidebar_position: 30
---

# 数据分布

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

建表时，您可以通过设置合理的分区和分桶，实现数据均匀分布和查询性能提升。数据均匀分布是指数据按照一定规则划分为子集，并且均衡地分布在不同节点上。查询时能够有效裁剪数据扫描量，最大限度地利用集群的并发性能，从而提升查询性能。

> **说明**
>
> - 您在建表时设置数据分布后，如果业务场景查询模式和数据特点发生变化，则自 3.2 版本起，StarRocks 支持您[在建表后修改一些数据分布相关属性](#建表后优化数据分布自-32)，以满足最新业务场景对查询性能的要求。
> - 自 3.1 版本起，您在建表和新增分区时可以不设置分桶键（即 DISTRIBUTED BY 子句）。StarRocks 默认使用随机分桶，将数据随机地分布在分区的所有分桶中。更多信息，请参见[随机分桶](#随机分桶自-v31)。
> - 自 2.5.7 版本起，您在建表和新增分区时可以不设置分桶数量 (BUCKETS)。StarRocks 默认自动设置分桶数量，如果自动设置分桶数量后性能未能达到预期，并且您比较熟悉分桶机制，则您也可以[手动设置分桶数量](#设置分桶数量)。

## 功能简介

### 常见的数据分布方式

现代分布式数据库中，常见的数据分布方式有如下几种：Round-Robin、Range、List 和 Hash。如下图所示：

![数据分布方式](../../_assets/3.3.2-1.png)

- Round-Robin：以轮询的方式把数据逐个放置在相邻节点上。

- Range：按区间进行数据分布。如上图所示，区间 [1-3]、[4-6] 分别对应不同的范围 (Range)。

- List：直接基于离散的各个取值做数据分布，性别、省份等数据就满足这种离散的特性。每个离散值会映射到一个节点上，多个不同的取值可能也会映射到相同节点上。

- Hash：通过哈希函数把数据映射到不同节点上。

为了更灵活地划分数据，除了单独采用上述数据分布方式之一以外，您还可以根据具体的业务场景需求组合使用这些数据分布方式。常见的组合方式有 Hash+Hash、Range+Hash、Hash+List。

### StarRocks 的数据分布方式

StarRocks 支持单独和组合使用数据分布方式。
> **说明**
>
> 除了常见的分布方式外， StarRocks 还支持了 Random 分布，可以简化分桶设置。

并且 StarRocks 通过设置分区 + 分桶的方式来实现数据分布。

- 第一层为分区：在一张表中，可以进行分区，支持的分区方式有表达式分区、Range 分区和 List 分区，或者不分区（即全表只有一个分区）。
- 第二层为分桶：在一个分区中，必须进行分桶。支持的分桶方式有哈希分桶和随机分桶。

| 数据分布方式      | 分区和分桶方式                                               | 说明                                                         |
| ----------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| Random 分布       | 随机分桶                                                     | 一张表为一个分区，表中数据随机分布至不同分桶。该方式为默认数据分布方式。 |
| Hash 分布         | 哈希分桶                                                     | 一张表为一个分区，对表中数据的分桶键值使用哈希函数进行计算后，得出其哈希值，分布到对应分桶。 |
| Range+Random 分布 | <ol><li>表达式分区或者 Range 分区</li><li>随机分桶</li></ol> | <ol><li>表的数据根据分区列值所属范围，分布至对应分区。</li><li>同一分区的数据随机分布至不同分桶。</li></ol> |
| Range+Hash 分布   | <ol><li>表达式分区或者 Range 分区</li><li>哈希分桶</li></ol> | <ol><li>表的数据根据分区列值所属范围，分布至对应分区。</li><li>对同一分区的数据的分桶键值使用哈希函数进行计算，得出其哈希值，分布到对应分桶。</li></ol> |
| List+Random 分布  | <ol><li>表达式分区或者 List 分区</li><li>随机分桶</li></ol>  | <ol><li>表的数据根据分区列值所属枚举值列表，分布至对应分区。</li><li>同一分区的数据随机分布至不同分桶。</li></ol> |
| List+ Hash 分布   | <ol><li>表达式分区或者 List 分区</li><li>哈希分桶</li></ol>  | <ol><li>表的数据根据分区列值所属枚举值列表，分布至对应分区。</li><li>对同一分区的数据的分桶键值使用哈希函数进行计算，得出其哈希值，分布到对应分桶。</li></ol> |

- Random 分布

  建表时不设置分区和分桶方式，则默认使用 Random 分布

    ```SQL
    CREATE TABLE site_access1 (
        event_day DATE,
        site_id INT DEFAULT '10', 
        pv BIGINT DEFAULT '0' ,
        city_code VARCHAR(100),
        user_name VARCHAR(32) DEFAULT ''
    )
    DUPLICATE KEY (event_day,site_id,pv);
    -- 没有设置任何分区和分桶方式，默认为 Random 分布（目前仅支持明细表）
    ```

- Hash 分布

    ```SQL
    CREATE TABLE site_access2 (
        event_day DATE,
        site_id INT DEFAULT '10',
        city_code SMALLINT,
        user_name VARCHAR(32) DEFAULT '',
        pv BIGINT SUM DEFAULT '0'
    )
    AGGREGATE KEY (event_day, site_id, city_code, user_name)
    -- 设置分桶方式为哈希分桶，并且必须指定分桶键
    DISTRIBUTED BY HASH(event_day,site_id); 
    ```

- Range + Random 分布

    ```SQL
    CREATE TABLE site_access3 (
        event_day DATE,
        site_id INT DEFAULT '10', 
        pv BIGINT DEFAULT '0' ,
        city_code VARCHAR(100),
        user_name VARCHAR(32) DEFAULT ''
    )
    DUPLICATE KEY(event_day,site_id,pv)
    -- 设为分区方式为表达式分区，并且使用时间函数的分区表达式（当然您也可以设置分区方式为 Range 分区）
    PARTITION BY date_trunc('day', event_day);
    -- 没有设置分桶方式，默认为随机分桶（目前仅支持明细表）
    ```

- Range + Hash 分布

    ```SQL
    CREATE TABLE site_access4 (
        event_day DATE,
        site_id INT DEFAULT '10',
        city_code VARCHAR(100),
        user_name VARCHAR(32) DEFAULT '',
        pv BIGINT SUM DEFAULT '0'
    )
    AGGREGATE KEY(event_day, site_id, city_code, user_name)
    -- 设为分区方式为表达式分区，并且使用时间函数的分区表达式（当然您也可以设置分区方式为 Range 分区）
    PARTITION BY date_trunc('day', event_day)
    -- 设置分桶方式为哈希分桶，必须指定分桶键
    DISTRIBUTED BY HASH(event_day, site_id);
    ```

- List + Random 分布

    ```SQL
    CREATE TABLE t_recharge_detail1 (
        id bigint,
        user_id bigint,
        recharge_money decimal(32,2), 
        city varchar(20) not null,
        dt date not null
    )
    DUPLICATE KEY(id)
    -- 设为分区方式为表达式分区，并且使用列分区表达式（当然您也可以设置分区方式为 List 分区）
    PARTITION BY (city);
    -- 没有设置分桶方式，默认为随机分桶（目前仅支持明细表）
    ```

- List + Hash 分布

    ```SQL
    CREATE TABLE t_recharge_detail2 (
        id bigint,
        user_id bigint,
        recharge_money decimal(32,2), 
        city varchar(20) not null,
        dt date not null
    )
    DUPLICATE KEY(id)
    -- 设为分区方式为表达式分区，并且使用列分区表达式（当然您也可以设置分区方式为 List 分区）
    PARTITION BY (city)
    -- 设置分桶方式为哈希分桶，并且必须指定分桶键
    DISTRIBUTED BY HASH(city,id); 
    ```

#### 分区

分区用于将数据划分成不同的区间。分区的主要作用是将一张表按照分区键拆分成不同的管理单元，针对每一个管理单元选择相应的存储策略，比如分桶数、冷热策略、存储介质、副本数等。StarRocks 支持在一个集群内使用多种存储介质，您可以将新数据所在分区放在 SSD 盘上，利用 SSD 优秀的随机读写性能来提高查询性能，将旧数据存放在 SATA 盘上，以节省数据存储的成本。

| **分区方式**       | **适用场景**                                                     | **分区创建方式**                                  |
| ------------------ | ------------------------------------------------------------ | --------------------------------------------- |
| 表达式分区（推荐） | 原称自动创建分区，适用大多数场景，并且灵活易用。适用于按照连续日期范围或者枚举值来查询和管理数据。 | 导入时自动创建                                |
| Range 分区（不推荐）| 典型的场景是数据简单有序，并且通常按照连续日期/数值范围来查询和管理数据。再如一些特殊场景，比如历史数据需要按月划分分区，而最近数据需要按天划分分区。 | 动态、批量或者手动创建 |
| List  分区（不推荐）| 典型的场景是按照枚举值来查询和管理数据，并且一个分区中需要包含各分区列的多值。比如经常按照国家和城市来查询和管理数据，则可以使用该方式，选择分区列为 `city`，一个分区包含属于一个国家的多个城市的数据。 | 手动创建                              |

**选择分区列和分区粒度**

- 分区键由一个或者多个分区列组成。选择合理的分区列可以有效的裁剪查询数据时扫描的数据量。业务系统中⼀般会选择根据时间进行分区，以优化大量删除过期数据带来的性能问题，同时也方便冷热数据分级存储，此时可以使用时间列作为分区列进行表达式分区或者 Range 分区。此外，如果经常按照枚举值查询数据和管理数据，则可以选择枚举值的列作为分区列进行表达式分区或者 List 分区。
- 选择分区单位时需要综合考虑数据量、查询特点、数据管理粒度等因素。
  - 示例 1：表单月数据量很小，可以按月分区，相比于按天分区，可以减少元数据数量，从而减少元数据管理和调度的资源消耗。
  - 示例 2：表单月数据量很大，而大部分查询条件精确到天，如果按天分区，可以做有效的分区裁剪，减少查询扫描的数据量。
  - 示例 3：数据要求按天过期，可以按天分区。

#### 分桶

一个分区按分桶方式被分成了多个桶 bucket，每个桶的数据称之为一个 tablet。

分桶方式：StarRocks 支持[随机分桶](#随机分桶自-v31)（自 v3.1）和[哈希分桶](#哈希分桶)。

- 随机分桶，建表和新增分区时无需设置分桶键。在同一分区内，数据随机分布到不同的分桶中。
- 哈希分桶，建表和新增分区时需要指定分桶键。在同一分区内，数据按照分桶键划分分桶后，所有分桶键的值相同的行会唯一分配到对应的一个分桶。

分桶数量：默认由 StarRocks 自动设置分桶数量（自 v2.5.7）。同时也支持您手动设置分桶数量。更多信息，请参见[设置分桶数量](#设置分桶数量)。

## 创建和管理分区

### 创建分区

#### 表达式分区（推荐）

> **注意**
>
> StarRocks [存算分离模式](../../deployment/shared_data/shared_data.mdx)自 v3.1.0 起支持时间函数表达式分区，自 v3.1.1 起支持列表达式分区。

[表达式分区](expression_partitioning.md)，原称自动创建分区，更加灵活易用，适用于大多数场景，比如按照连续日期范围或者枚举值来查询和管理数据。

您仅需要在建表时使用分区表达式，即可实现导入数据时自动创建分区，不需要预先创建出分区或者配置动态分区属性。

从 v3.4 开始，表达式分区方式进一步得到优化，统一所有分区策略，并支持更复杂的解决方案。在大多数情况下，建议您使用表达式分区。表达式分区将在未来版本中逐渐取代其他分区策略。

示例一：基于 DATETIME 列使用简单时间函数表达式分区。

```SQL
CREATE TABLE site_access(
    event_day DATETIME NOT NULL,
    site_id INT DEFAULT '10',
    city_code VARCHAR(100),
    user_name VARCHAR(32) DEFAULT '',
    pv BIGINT DEFAULT '0'
)
DUPLICATE KEY(event_day, site_id, city_code, user_name)
PARTITION BY time_slice(event_day, INTERVAL 7 day)
DISTRIBUTED BY HASH(event_day, site_id)
```

示例二：基于多列使用列表达式分区。

```SQL
CREATE TABLE t_recharge_detail1 (
    id bigint,
    user_id bigint,
    recharge_money decimal(32,2), 
    city varchar(20) not null,
    dt varchar(20) not null
)
DUPLICATE KEY(id)
PARTITION BY dt,city
DISTRIBUTED BY HASH(`id`);
```

示例三：基于 Unix 时间戳列使用复杂时间函数表达式分区。

```SQL
CREATE TABLE orders (
    ts BIGINT NOT NULL,
    id BIGINT NOT NULL,
    city STRING NOT NULL
)
PARTITION BY from_unixtime(ts,'%Y%m%d');
```

示例四：使用时间函数表达式和列表达式的混合表达式分区。

```SQL
CREATE TABLE orders (
    ts BIGINT NOT NULL,
    id BIGINT NOT NULL,
    city STRING NOT NULL
)
PARTITION BY from_unixtime(ts,'%Y%m%d'), city;
```

#### Range 分区

Range 分区适用于简单且具有连续性的数据，如时间序列数据或连续的数值数据。并且经常按照连续日期/数值范围，来查询和管理数据。以及一些特殊场景，比如一张表的分区粒度不一致，历史数据需要按月划分分区，而最近数据需要按天划分分区。

您需要显示定义数据分区列，以及分区和分区列值范围的映射关系。数据导入时 StarRocks 会根据数据分区列值所属范围，将数据分配到相应的分区中。

在分区列的数据类型方面，在 3.3.0 之前，Range 分区仅支持分区列为日期和整数类型。自 3.3.0 起，新增支持三个特定时间函数作为分区列。在显式定义分区与分区列值范围的映射关系时，您需要使用特定时间函数将时间戳或字符串的分区列值转成日期，然后按转换后的日期划分分区。

  :::info

  - 如果分区列值是时间戳，则划分分区时，需要使用 from_unixtime 或者 from_unixtime_ms 函数将时间戳转换为日期。并且使用 from_unixtime 函数时，分区列仅支持 INT 和 BIGINT 类型，使用 from_unixtime_ms 函数时，分区列仅支持为 BIGINT 类型。
  - 如果分区列是字符串（STRING、VARCHAR 或者 CHAR类型），则划分分区时，需要使用 str2date 函数将字符串转换为日期。

  :::

##### 手动创建分区

定义每个分区和分区列值范围的映射关系。

- **分区列为日期类型**

    ```SQL
    CREATE TABLE site_access(
        event_day DATE,
        site_id INT,
        city_code VARCHAR(100),
        user_name VARCHAR(32),
        pv BIGINT SUM DEFAULT '0'
    )
    AGGREGATE KEY(event_day, site_id, city_code, user_name)
    PARTITION BY RANGE(event_day)(
        PARTITION p1 VALUES LESS THAN ("2020-01-31"),
        PARTITION p2 VALUES LESS THAN ("2020-02-29"),
        PARTITION p3 VALUES LESS THAN ("2020-03-31")
    )
    DISTRIBUTED BY HASH(site_id);
    ```

- **分区列为整数类型**

    ```SQL
    CREATE TABLE site_access(
        datekey INT,
        site_id INT,
        city_code SMALLINT,
        user_name VARCHAR(32),
        pv BIGINT SUM DEFAULT '0'
    )
    AGGREGATE KEY(datekey, site_id, city_code, user_name)
    PARTITION BY RANGE (datekey) (
        PARTITION p1 VALUES LESS THAN ("20200131"),
        PARTITION p2 VALUES LESS THAN ("20200229"),
        PARTITION p3 VALUES LESS THAN ("20200331")
    )
    DISTRIBUTED BY HASH(site_id);
    ```

- **三个特定时间函数作为分区列（自 3.3.0 起支持）**
  
  在显式定义分区与分区列值范围的映射关系时，您需要先使用特定时间函数将时间戳或字符串的分区列值转成日期，然后按转换后的日期划分分区。

  <Tabs groupId="manual partitioning">
  <TabItem value="example1" label="分区列值为时间戳" default>

  ```SQL
  -- 精确到秒的 10 位时间戳，例如：1703832553
  CREATE TABLE site_access(
      event_time bigint,
      site_id INT,
      city_code SMALLINT,
      user_name VARCHAR(32),
      pv BIGINT SUM DEFAULT '0'
    )
  AGGREGATE KEY(event_time, site_id, city_code, user_name)
  PARTITION BY RANGE(from_unixtime(event_time)) (
      PARTITION p1 VALUES LESS THAN ("2021-01-01"),
      PARTITION p2 VALUES LESS THAN ("2021-01-02"),
      PARTITION p3 VALUES LESS THAN ("2021-01-03")
  )
  DISTRIBUTED BY HASH(site_id)
  ;
  
  -- 精确到毫秒的 13 位时间戳，例如：1703832553219
  CREATE TABLE site_access(
      event_time bigint,
      site_id INT,
      city_code SMALLINT,
      user_name VARCHAR(32),
      pv BIGINT SUM DEFAULT '0'
    )
  AGGREGATE KEY(event_time, site_id, city_code, user_name)
  PARTITION BY RANGE(from_unixtime_ms(event_time))(
      PARTITION p1 VALUES LESS THAN ("2021-01-01"),
      PARTITION p2 VALUES LESS THAN ("2021-01-02"),
      PARTITION p3 VALUES LESS THAN ("2021-01-03")
  )
  DISTRIBUTED BY HASH(site_id);
  ```

  </TabItem>
  <TabItem value="example2" label="分区列值为字符串">

    ```SQL
    CREATE TABLE site_access (
         event_time  varchar(100),
         site_id INT,
         city_code SMALLINT,
         user_name VARCHAR(32),
         pv BIGINT SUM DEFAULT '0'
    )
    AGGREGATE KEY(event_time, site_id, city_code, user_name)
    PARTITION BY RANGE(str2date(event_time, '%Y-%m-%d'))(
        PARTITION p1 VALUES LESS THAN ("2021-01-01"),
        PARTITION p2 VALUES LESS THAN ("2021-01-02"),
        PARTITION p3 VALUES LESS THAN ("2021-01-03")
    )
    DISTRIBUTED BY HASH(site_id);
    ```

  </TabItem>
  </Tabs>

##### 动态分区

建表时[配置动态分区属性](dynamic_partitioning.md)，StarRocks 会⾃动提前创建新的分区，删除过期分区，从而确保数据的时效性，实现对分区的⽣命周期管理（Time to Life，简称 “TTL”）。

区别于表达式分区中自动创建分区功能，动态创建分区只是根据您配置的动态分区属性，定期提前创建一些分区。如果导入的新数据不属于这些提前创建的分区，则导入任务会报错。而表达式分区中自动创建分区功能会根据导入数据创建对应的新分区。

##### 批量创建分区

建表时和建表后，支持批量创建分区，通过 START、END 指定批量分区的开始和结束，EVERY 子句指定分区增量值。其中，批量分区包含 START 的值，但是不包含 END 的值。分区的命名规则同动态分区一样。

- **分区列为日期类型**

    当分区列为日期类型时，建表时通过 START、END 指定批量分区的开始日期和结束日期，EVERY 子句指定分区增量值。并且 EVERY 子句中用 INTERVAL 关键字表示日期间隔，目前支持日期间隔的单位为 HOUR（自 3.0 版本起）、DAY、WEEK、MONTH、YEAR。

  <Tabs groupId="batch partitioning(date)">
  <TabItem value="example1" label="日期间隔相同" default>

    如下示例中，批量分区的开始日期为 `2021-01-01` 和结束日期为 `2021-01-04`，增量值为一天：

    ```SQL
    CREATE TABLE site_access (
        datekey DATE,
        site_id INT,
        city_code SMALLINT,
        user_name VARCHAR(32),
        pv BIGINT DEFAULT '0'
    )
    DUPLICATE KEY(datekey, site_id, city_code, user_name)
    PARTITION BY RANGE (datekey) (
        START ("2021-01-01") END ("2021-01-04") EVERY (INTERVAL 1 DAY)
    )
    DISTRIBUTED BY HASH(site_id);
    ```

    则相当于在建表语句中使用如下 PARTITION BY 子句：

    ```SQL
    PARTITION BY RANGE (datekey) (
        PARTITION p20210101 VALUES [('2021-01-01'), ('2021-01-02')),
        PARTITION p20210102 VALUES [('2021-01-02'), ('2021-01-03')),
        PARTITION p20210103 VALUES [('2021-01-03'), ('2021-01-04'))
    )
    ```

  </TabItem>
  <TabItem value="example2" label="日期间隔不同">

    支持针对不同的日期分区区间（日期分区区间不能相重合），使用不同的 EVERY 子句指定日期间隔。一个日期分区区间，按照对应 EVERY 子句定义的日期间隔，批量创建分区，例如：

    ```SQL
    CREATE TABLE site_access (
        datekey DATE,
        site_id INT,
        city_code SMALLINT,
        user_name VARCHAR(32),
        pv BIGINT DEFAULT '0'
    )
    DUPLICATE KEY(datekey, site_id, city_code, user_name)
    PARTITION BY RANGE (datekey) (
        START ("2019-01-01") END ("2021-01-01") EVERY (INTERVAL 1 YEAR),
        START ("2021-01-01") END ("2021-05-01") EVERY (INTERVAL 1 MONTH),
        START ("2021-05-01") END ("2021-05-04") EVERY (INTERVAL 1 DAY)
    )
    DISTRIBUTED BY HASH(site_id);
    ```

    则相当于在建表语句中使用如下 PARTITION BY 子句：

    ```SQL
    PARTITION BY RANGE (datekey) (
        PARTITION p2019 VALUES [('2019-01-01'), ('2020-01-01')),
        PARTITION p2020 VALUES [('2020-01-01'), ('2021-01-01')),
        PARTITION p202101 VALUES [('2021-01-01'), ('2021-02-01')),
        PARTITION p202102 VALUES [('2021-02-01'), ('2021-03-01')),
        PARTITION p202103 VALUES [('2021-03-01'), ('2021-04-01')),
        PARTITION p202104 VALUES [('2021-04-01'), ('2021-05-01')),
        PARTITION p20210501 VALUES [('2021-05-01'), ('2021-05-02')),
        PARTITION p20210502 VALUES [('2021-05-02'), ('2021-05-03')),
        PARTITION p20210503 VALUES [('2021-05-03'), ('2021-05-04'))
    )
    ```

  </TabItem>
  </Tabs>

- **分区列为数值类型**

    当分区列为整数类型时，建表时通过 START、END 指定批量分区的开始值和结束值，EVERY 子句指定分区增量值。

    > **说明**
    >
    > START、END 所指定的分区列的值需要使用英文引号包裹，而 EVERY 子句中的分区增量值不用英文引号包裹。

  <Tabs groupId="batch partitioning(integer)">
  <TabItem value="example1" label="数值间隔相同" default>

    如下示例中，批量分区的开始值为 `1` 和结束值为 `5`，分区增量值为 `1`：

    ```SQL
    CREATE TABLE site_access (
        datekey INT,
        site_id INT,
        city_code SMALLINT,
        user_name VARCHAR(32),
        pv BIGINT DEFAULT '0'
    )
    DUPLICATE KEY(datekey, site_id, city_code, user_name)
    PARTITION BY RANGE (datekey) (
        START ("1") END ("5") EVERY (1)
    )
    DISTRIBUTED BY HASH(site_id);
    ```

    则相当于在建表语句中使用如下 PARTITION BY 子句：

    ```SQL
    PARTITION BY RANGE (datekey) (
        PARTITION p1 VALUES [("1"), ("2")),
        PARTITION p2 VALUES [("2"), ("3")),
        PARTITION p3 VALUES [("3"), ("4")),
        PARTITION p4 VALUES [("4"), ("5"))
    )
    ```

  </TabItem>
  <TabItem value="example2" label="数值间隔不同">

    支持针对不同的数值分区区间（数值分区区间不能相重合），使用不同的 EVERY 子句指定数值间隔。一个数值分区区间，按照对应 EVERY 子句定义的数值间隔，批量创建分区，例如：

    ```SQL
    CREATE TABLE site_access (
        datekey INT,
        site_id INT,
        city_code SMALLINT,
        user_name VARCHAR(32),
        pv BIGINT DEFAULT '0'
    )
    DUPLICATE KEY(datekey, site_id, city_code, user_name)
    PARTITION BY RANGE (datekey) (
        START ("1") END ("10") EVERY (1),
        START ("10") END ("100") EVERY (10)
    )
    DISTRIBUTED BY HASH(site_id);
    ```

  </TabItem>
  </Tabs>

- **三个特定时间函数作为分区列（自 3.3.0 起支持）**

  在显式定义分区与分区列值范围的映射关系时，您需要先使用特定时间函数将时间戳或字符串的分区列值转成日期，然后按转换后的日期划分分区。

  <Tabs groupId="batch partitioning(timestamp and string)">
  <TabItem value="example1" label="分区列值为时间戳" default>

  ```SQL
  -- 精确到秒的 10 位时间戳，例如：1703832553
  CREATE TABLE site_access(
      event_time bigint,
      site_id INT,
      city_code SMALLINT,
      user_name VARCHAR(32),
      pv BIGINT DEFAULT '0'
    )
  PARTITION BY RANGE(from_unixtime(event_time)) (
      START ("2021-01-01") END ("2021-01-10") EVERY (INTERVAL 1 DAY)
  )
  DISTRIBUTED BY HASH(site_id);
  -- 精确到毫秒的 13 位时间戳，例如：1703832553219
  CREATE TABLE site_access(
      event_time bigint,
      site_id INT,
      city_code SMALLINT,
      user_name VARCHAR(32),
      pv BIGINT DEFAULT '0'
  )
  PARTITION BY RANGE(from_unixtime_ms(event_time))(
      START ("2021-01-01") END ("2021-01-10") EVERY (INTERVAL 1 DAY)
  )
  DISTRIBUTED BY HASH(site_id);
  ```

  </TabItem>
  <TabItem value="example2" label="分区列为字符串类型">

    ```SQL
    CREATE TABLE site_access (
         event_time  varchar(100),
         site_id INT,
         city_code SMALLINT,
         user_name VARCHAR(32),
         pv BIGINT DEFAULT '0'
  )
    PARTITION BY RANGE(str2date(event_time, '%Y-%m-%d'))(
        START ("2021-01-01") END ("2021-01-10") EVERY (INTERVAL 1 DAY)
    )
    DISTRIBUTED BY HASH(site_id);
    ```

  </TabItem>
  </Tabs>

#### List 分区（自 v3.1）

[List 分区](list_partitioning.md)适用于按照枚举值来查询和管理数据。尤其适用于一个分区中需要包含各分区列的多个值。比如经常按照国家和城市来查询和管理数据，则可以使用该方式，选择分区列为 `city`，一个分区包含属于一个国家的多个城市的数据。

StarRocks 会按照您显式定义的枚举值列表与分区的映射关系将数据分配到相应的分区中。

### 管理分区

#### 增加分区

对于 Range 分区和 List 分区，您可以手动增加新的分区，用于存储新的数据，而表达式分区可以实现导入新数据时自动创建分区，您无需手动新增分区。
新增分区的默认分桶数量和原分区相同。您也可以根据新分区的数据规模调整分桶数量。

如下示例中，在 `site_access` 表添加新的分区，用于存储新月份的数据：

```SQL
ALTER TABLE site_access
ADD PARTITION p4 VALUES LESS THAN ("2020-04-30")
DISTRIBUTED BY HASH(site_id);
```

#### 删除分区

执行如下语句，删除 `site_access` 表中分区 p1 及数据：

> 说明：分区中的数据不会立即删除，会在 Trash 中保留一段时间（默认为一天）。如果误删分区，可以通过 [RECOVER 命令](../../sql-reference/sql-statements/backup_restore/RECOVER.md)恢复分区及数据。

```SQL
ALTER TABLE site_access
DROP PARTITION p1;
```

#### 恢复分区

执行如下语句，恢复 `site_access` 表中分区 `p1` 及数据：

```SQL
RECOVER PARTITION p1 FROM site_access;
```

#### 查看分区

执行如下语句，查看 `site_access` 表中分区情况：

```SQL
SHOW PARTITIONS FROM site_access;
```

## 设置分桶

### 随机分桶（自 v3.1）

对每个分区的数据，StarRocks 将数据随机地分布在所有分桶中，适用于数据量不大，对查询性能要求不高的场景。如果您不设置分桶方式，则默认由 StarRocks 使用随机分桶，并且自动设置分桶数量。

不过值得注意的是，如果查询海量数据且查询时经常使用一些列会作为条件列，随机分桶提供的查询性能可能不够理想。在该场景下建议您使用[哈希分桶](#哈希分桶)，当查询时经常使用这些列作为条件列时，只需要扫描和计算查询命中的少量分桶，则可以显著提高查询性能。

**使用限制**

- 仅支持明细表。
- 不支持指定 [Colocation Group](../../using_starrocks/Colocate_join.md)。
- 不支持 [Spark Load](../../loading/SparkLoad.md)。

如下建表示例中，没有使用 `DISTRIBUTED BY xxx` 语句，即表示默认由 StarRocks 使用随机分桶，并且由 StarRocks 自动设置分桶数量。

```SQL
CREATE TABLE site_access1(
    event_day DATE,
    site_id INT DEFAULT '10', 
    pv BIGINT DEFAULT '0' ,
    city_code VARCHAR(100),
    user_name VARCHAR(32) DEFAULT ''
)
DUPLICATE KEY(event_day,site_id,pv);
```

当然，如果您比较熟悉 StarRocks 的分桶机制，使用随机分桶建表时，也可以手动设置分桶数量。

```SQL
CREATE TABLE site_access2(
    event_day DATE,
    site_id INT DEFAULT '10', 
    pv BIGINT DEFAULT '0' ,
    city_code VARCHAR(100),
    user_name VARCHAR(32) DEFAULT ''
)
DUPLICATE KEY(event_day,site_id,pv)
DISTRIBUTED BY RANDOM BUCKETS 8; -- 手动设置分桶数量为 8
```

### 哈希分桶

对每个分区的数据，StarRocks 会根据分桶键和[分桶数量](#设置分桶数量)进行哈希分桶。在哈希分桶中，使用特定的列值作为输入，通过哈希函数计算出一个哈希值，然后将数据根据该哈希值分配到相应的桶中。

**优点**

- 提高查询性能。相同分桶键值的行会被分配到一个分桶中，在查询时能减少扫描数据量。
- 均匀分布数据。通过选取较高基数（唯一值的数量较多）的列作为分桶键，能更均匀的分布数据到每一个分桶中。

**如何选择分桶键**

假设存在列同时满足高基数和经常作为查询条件，则建议您选择其为分桶键，进行哈希分桶。 如果不存在这些同时满足两个条件的列，则需要根据查询进行判断。

- 如果查询比较复杂，则建议选择高基数的列为分桶键，保证数据在各个分桶中尽量均衡，提高集群资源利用率。
- 如果查询比较简单，则建议选择经常作为查询条件的列为分桶键，提高查询效率。

并且，如果数据倾斜情况严重，您还可以使用多个列作为数据的分桶键，但是建议不超过 3 个列。

**注意事项**

- **建表时，如果使用哈希分桶，则必须指定分桶键**。
- 组成分桶键的列仅支持整型、DECIMAL、DATE/DATETIME、CHAR/VARCHAR/STRING 数据类型。
- 自 3.2 起，建表后支持通过 ALTER TABLE 修改分桶键。

如下示例中，`site_access` 表采用 `site_id` 作为分桶键，其原因在于 `site_id` 为高基数列。此外，针对 `site_access` 表的查询请求，基本上都以站点作为查询过滤条件，采用 `site_id` 作为分桶键，还可以在查询时裁剪掉大量无关分桶。

```SQL
CREATE TABLE site_access(
    event_day DATE,
    site_id INT DEFAULT '10',
    city_code VARCHAR(100),
    user_name VARCHAR(32) DEFAULT '',
    pv BIGINT SUM DEFAULT '0'
)
AGGREGATE KEY(event_day, site_id, city_code, user_name)
PARTITION BY RANGE(event_day) (
    PARTITION p1 VALUES LESS THAN ("2020-01-31"),
    PARTITION p2 VALUES LESS THAN ("2020-02-29"),
    PARTITION p3 VALUES LESS THAN ("2020-03-31")
)
DISTRIBUTED BY HASH(site_id);
```

如下查询中，假设每个分区有 10 个分桶，则其中 9 个分桶被裁减，因而系统只需要扫描 `site_access` 表中 1/10 的数据：

```SQL
select sum(pv)
from site_access
where site_id = 54321;
```

但是如果 `site_id` 分布十分不均匀，大量的访问数据是关于少数网站的（幂律分布，二八规则），那么采用上述分桶方式会造成数据分布出现严重的倾斜，进而导致系统局部的性能瓶颈。此时，您需要适当调整分桶的字段，以将数据打散，避免性能问题。例如，可以采用 `site_id`、`city_code` 组合作为分桶键，将数据划分得更加均匀。相关建表语句如下：

```SQL
CREATE TABLE site_access
(
    site_id INT DEFAULT '10',
    city_code SMALLINT,
    user_name VARCHAR(32) DEFAULT '',
    pv BIGINT SUM DEFAULT '0'
)
AGGREGATE KEY(site_id, city_code, user_name)
DISTRIBUTED BY HASH(site_id,city_code);
```

在实际使用中，您可以依据自身的业务特点选择以上两种分桶方式。采用 `site_id` 的分桶方式对于短查询十分有利，能够减少节点之间的数据交换，提高集群整体性能；采用 `site_id`、`city_code` 的组合分桶方式对于长查询有利，能够利用分布式集群的整体并发性能。

> 说明：
>
> - 短查询是指扫描数据量不大、单机就能完成扫描的查询。
>
> - 长查询是指扫描数据量大、多机并行扫描能显著提升性能的查询。

### 设置分桶数量

在 StarRocks 中，分桶是实际物理文件组织的单元。

#### 建表时

- 自动设置（推荐）

  自 2.5.7 版本起，StarRocks 支持根据机器资源和数据量自动设置分区中分桶数量。

  :::tip

  如果表单个分区原始数据规模预计超过 100 GB，建议您手动设置分区中分桶数量。

  :::

  <Tabs groupId="automaticexamples1">
  <TabItem value="example1" label="哈希分桶表" default>
  建表示例：

  ```sql
  CREATE TABLE site_access (
      site_id INT DEFAULT '10',
      city_code SMALLINT,
      user_name VARCHAR(32) DEFAULT '',
      event_day DATE,
      pv BIGINT SUM DEFAULT '0')
  AGGREGATE KEY(site_id, city_code, user_name,event_day)
  PARTITION BY date_trunc('day', event_day)
  DISTRIBUTED BY HASH(site_id,city_code); -- 无需手动设置分区中分桶数量
  ```

  </TabItem>
  <TabItem value="example2" label="随机分桶表">

  针对随机分桶表，StarRocks 在支持自动设置分区中分桶数量的基础上，自 3.2 版本起，还进一步优化了自动设置分桶数量的逻辑，支持了**在导入数据至分区的过程中**根据集群能力和导入数据量等**按需动态增加**分区中分桶数量。在提高建表易用性的同时，还能提升大数据集的导入性能。

  :::warning

  - 如果需要启用按需动态增加分桶数量，您需要设置表属性 `PROPERTIES("bucket_size"="xxx")`，指定单个分桶的大小。如果分区的数据量不大，则可以设置 `bucket_size` 为 1 GB，如果分区的数据量大，则可以设置 `bucket_size` 为 4 GB。
  - 启用后，如果需要回滚至 3.1 版本，则需要删除启用按需动态增加分桶数量的表，并且手动执行元数据 checkpoint [ALTER SYSTEM CREATE IMAGE](../../sql-reference/sql-statements/cluster-management/nodes_processes/ALTER_SYSTEM.md) 成功后才能回滚。

  :::

  建表示例：

  ```sql
  CREATE TABLE details1 (
      event_day DATE,
      site_id INT DEFAULT '10', 
      pv BIGINT DEFAULT '0',
      city_code VARCHAR(100),
      user_name VARCHAR(32) DEFAULT '')
  DUPLICATE KEY (event_day,site_id,pv)
  PARTITION BY date_trunc('day', event_day)
  -- 该表分区中的分桶数量由 StarRocks 自动设置，并且因为指定单个分桶的大小为 1 GB，分桶数量会按需动态增加。
  PROPERTIES("bucket_size"="1073741824")
  ;
  
  CREATE TABLE details2 (
      event_day DATE,
      site_id INT DEFAULT '10',
      pv BIGINT DEFAULT '0' ,
      city_code VARCHAR(100),
      user_name VARCHAR(32) DEFAULT '')
  DUPLICATE KEY (event_day,site_id,pv)
  PARTITION BY date_trunc('day', event_day)
  -- 该表分区中的分桶数量由 StarRocks 自动设置，并且由于没有指定单个分桶的大小，分桶数量不会按需动态增加。
  ;
  ```

  </TabItem>
  </Tabs>

- 手动设置

  自 2.4 版本起，StarRocks 提供了自适应的 Tablet 并行扫描能力，即一个查询中涉及到的任意一个 Tablet 可能是由多个线程并行地分段扫描，减少了 Tablet 数量对查询能力的限制，从而可以简化对分区中分桶数量的设置。简化后，确定分区中分桶数量方式可以是：首先预估每个分区的数据量，然后按照每 10 GB 原始数据一个 Tablet 计算，从而确定分区中分桶数量。

  如果需要开启并行扫描 Tablet，则您需要确保系统变量 `enable_tablet_internal_parallel` 全局生效 `SET GLOBAL enable_tablet_internal_parallel = true;`。

  <Tabs groupId="manualexamples1">
  <TabItem value="example1" label="哈希分桶表" default>

  ```sql
  CREATE TABLE site_access (
      site_id INT DEFAULT '10',
      city_code SMALLINT,
      user_name VARCHAR(32) DEFAULT '',
      event_day DATE,
      pv BIGINT SUM DEFAULT '0')
  AGGREGATE KEY(site_id, city_code, user_name,event_day)
  PARTITION BY date_trunc('day', event_day)
  DISTRIBUTED BY HASH(site_id,city_code) BUCKETS 30; -- 假设导入一个分区的原始数据量为 300 GB，则按照每 10 GB 原始数据一个 Tablet，则分区中分桶数量可以设置为 30。
  ```

  </TabItem>
  <TabItem value="example2" label="随机分桶表">

    ```sql
  CREATE TABLE details (
      site_id INT DEFAULT '10', 
      city_code VARCHAR(100),
      user_name VARCHAR(32) DEFAULT '',
      event_day DATE,
      pv BIGINT DEFAULT '0'
  )
  DUPLICATE KEY (site_id,city_code)
  PARTITION BY date_trunc('day', event_day)
  DISTRIBUTED BY RANDOM BUCKETS 30
  ; 
    ```

    </TabItem>
    </Tabs>

#### 建表后

- 自动设置（推荐）

  自 2.5.7 版本起，StarRocks 支持根据机器资源和数据量自动设置分区中分桶数量。

  :::tip

  如果表单个分区原始数据规模预计超过 100 GB，建议您手动设置分区中分桶数量。

  :::

  <Tabs groupId="automaticexamples2">
  <TabItem value="example1" label="哈希分桶表" default>

    ```sql
  -- 自动设置所有分区的分桶数量
  ALTER TABLE site_access DISTRIBUTED BY HASH(site_id,city_code);
  
  -- 自动设置指定分区的分桶数量
  ALTER TABLE site_access PARTITIONS (p20230101, p20230102)
  DISTRIBUTED BY HASH(site_id,city_code);
  
  -- 自动设置新增分区中分桶数量
  ALTER TABLE site_access ADD PARTITION p20230106 VALUES [('2023-01-06'), ('2023-01-07'))
  DISTRIBUTED BY HASH(site_id,city_code);
    ```

    </TabItem>
    <TabItem value="example2" label="随机分桶表">

  针对随机分桶表，StarRocks 在支持自动设置分区中分桶数量的基础上，自 3.2 版本起，还进一步优化了自动设置分桶数量的逻辑，支持了**在导入数据至分区的过程中**根据集群能力和导入数据量等**按需动态增加**分区中分桶数量。

  :::warning

  - 如果需要启用按需动态增加分桶数量，您需要设置表属性 `PROPERTIES("bucket_size"="xxx")`，指定单个分桶的大小。如果分区的数据量不大，则可以设置 `bucket_size` 为 1 GB，如果分区的数据量大，则可以设置 `bucket_size` 为 4 GB。
  - 启用后，如果需要回滚至 3.1 版本，则需要删除启用按需动态增加分桶数量的表，并且手动执行元数据 checkpoint [ALTER SYSTEM CREATE IMAGE](../../sql-reference/sql-statements/cluster-management/nodes_processes/ALTER_SYSTEM.md) 成功后才能回滚。

  :::

    ```sql
  -- 自动设置所有分区的分桶数量，并且不开启按需动态增加分桶数量，分桶数量固定。
  ALTER TABLE details DISTRIBUTED BY RANDOM;
  -- 自动设置所有分区的分桶数量，并且开启按需动态增加分桶数量。
  ALTER TABLE details SET("bucket_size"="1073741824");
  
  -- 自动设置指定分区的分桶数量
  ALTER TABLE details PARTITIONS (p20230103, p20230104)
  DISTRIBUTED BY RANDOM;
  
  -- 自动设置新增分区中分桶数量
  ALTER TABLE details ADD PARTITION  p20230106 VALUES [('2023-01-06'), ('2023-01-07'))
  DISTRIBUTED BY RANDOM;
    ```

    </TabItem>
    </Tabs>

- 手动设置

  手动指定分区中分桶数量。分区的分桶数量的计算方式可以参考如上[建表时手动设置分区中分桶数量](#建表时)。

  <Tabs groupId="manualexamples2">
  <TabItem value="example1" label="哈希分桶表" default>

  ```sql
  -- 手动指定所有分区中分桶数量
  ALTER TABLE site_access
  DISTRIBUTED BY HASH(site_id,city_code) BUCKETS 30;
  -- 手动指定部分分区中分桶数量
  ALTER TABLE site_access
  partitions p20230104
  DISTRIBUTED BY HASH(site_id,city_code)  BUCKETS 30;
  -- 手动指定新增分区中分桶数量
  ALTER TABLE site_access
  ADD PARTITION p20230106 VALUES [('2023-01-06'), ('2023-01-07'))
  DISTRIBUTED BY HASH(site_id,city_code) BUCKETS 30;
  ```

  </TabItem>
  <TabItem value="example2" label="随机分桶表">

  ```sql
  -- 手动指定所有分区中分桶数量
  ALTER TABLE details
  DISTRIBUTED BY RANDOM BUCKETS 30;
  -- 手动指定部分分区中分桶数量
  ALTER TABLE details
  partitions p20230104
  DISTRIBUTED BY RANDOM BUCKETS 30;
  -- 手动指定新增分区中分桶数量
  ALTER TABLE details
  ADD PARTITION p20230106 VALUES [('2023-01-06'), ('2023-01-07'))
  DISTRIBUTED BY RANDOM BUCKETS 30;
  ```

  手动设置动态分区的默认分桶数量。

  ```sql
  ALTER TABLE details_dynamic
  SET ("dynamic_partition.buckets"="xxx");
  ```

  </TabItem>
  </Tabs>

#### 查看分桶数量

如果查看分区中分桶数量，您可以执行 [SHOW PARTITIONS](../../sql-reference/sql-statements/table_bucket_part_index/SHOW_PARTITIONS.md)。

:::info

- 如果是随机分桶表并且开启按需动态增加分桶数量，建表后在导入过程中，分区的分桶数量会**动态增加**，返回结果显示的是分区**当前**的分桶数量。

- 如果是随机分桶表，分区内部实际的划分层次为：分区 > 子分区 > 分桶，为了增加分桶数量，StarRocks 实际上是新增一个子分区，子分区包括一定数量的分桶，因此  [SHOW PARTITIONS](../../sql-reference/sql-statements/table_bucket_part_index/SHOW_PARTITIONS.md) 返回结果中会显示分区名称相同的多条数据行，表示同一分区中子分区的情况。

:::

## 建表后优化数据分布（自 3.2）

> **注意**
>
> StarRocks [存算分离模式](../../deployment/shared_data/shared_data.mdx)暂不支持该特性。

随着业务场景中查询模式和数据量变化，建表时设置的分桶方式和分桶数量，以及排序键可能不再能适应新的业务场景，导致查询性能下降，此时可以通过 `ALTER TABLE` 调整分桶方式和分桶数量，以及排序键，优化数据分布。比如：

- **分区中数据量增多，增加分桶数量**

  当按天分区的分区数据量相比原来变大很多，原本的分桶数量不再合适时，可以加大分桶数量，以让每个 Tablet 的大小一般控制在 1 GB ~ 10 GB。

- **通过调整分桶键，来避免数据倾斜**

  当发现原有分桶键会导致数据倾斜（比如原来的分桶键只有 `k1` 一列），可以设置更合适的列、或者加入更多一些列到分桶键中。如下：

    ```SQL
    ALTER TABLE t DISTRIBUTED BY HASH(k1, k2) BUCKETS 20;
    -- 如果是 StarRocks 的版本是 3.1及以上，并且使用的是明细表，则建议直接改成默认分桶设置，即随机分桶并且由 StarRocks 自动设置分桶数量
    ALTER TABLE t DISTRIBUTED BY RANDOM;
    ```

- 如果表为主键表，当业务的查询模式有较大变化，经常需要用到表中另外几个列作为条件列时，则可以调整排序键。如下：

    ```SQL
    ALTER TABLE t ORDER BY k2, k1;
    ```

更多信息，参见 [ALTER TABLE](../../sql-reference/sql-statements/table_bucket_part_index/ALTER_TABLE.md) 。

## 最佳实践

对于 StarRocks 而言，分区和分桶的选择是非常关键的。在建表时选择合理的分区键和分桶键，可以有效提高集群整体性能。因此建议在选择分区键和分桶键时，根据业务情况进行调整。

- **数据倾斜**
  
  如果业务场景中单独采用倾斜度大的列做分桶，很大程度会导致访问数据倾斜，那么建议采用多列组合的方式进行数据分桶。

- **高并发**
  
  分区和分桶应该尽量覆盖查询语句所带的条件，这样可以有效减少扫描数据，提高并发。

- **高吞吐**
  
  尽量把数据打散，让集群以更高的并发扫描数据，完成相应计算。

- **元数据管理**

  Tablet 过多会增加 FE/BE 的元数据管理和调度的资源消耗。
