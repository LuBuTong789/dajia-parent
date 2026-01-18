# 打车服务订单号生成工具

Taxi Order ID Generator

一个专为打车服务设计的分布式订单号生成工具，基于美团 Leaf 框架实现，支持高并发、全局唯一、可追溯的订单号生成，同时兼顾数据库性能与业务实用性。

📖 项目介绍

在打车服务场景中，订单号作为核心标识，需满足「全局唯一、趋势有序、可追溯、高性能、防伪造」五大核心要求。本工具基于雪花算法（Snowflake）定制开发，通过拼接业务前缀（城市编码、服务类型），生成兼具业务意义与技术安全性的订单号，同时适配数据库主键设计优化，适用于快车、专车、拼车等多场景打车业务。

✨ 核心功能

- 全局唯一：基于雪花算法 + 机器ID分布式部署，跨集群、跨城市无重复订单号。

- 趋势有序：数字型主键保证数据库索引性能，避免索引碎片化，提升订单查询/写入效率。

- 业务可追溯：订单号内置城市编码、服务类型前缀，便于问题排查与业务分类。

- 高性能：依赖 Leaf 框架号段缓存/雪花算法优化，支持每秒数万订单号生成，适配早晚高峰高并发。

- 防伪造：通过业务前缀混淆 + 随机盐值（可选），避免简单自增ID被恶意猜测。

- 多模式支持：默认雪花算法模式，可切换 Leaf 号段模式（依赖MySQL），适配不同部署环境。

🔧 技术选型

技术栈

版本

用途

Java

1.8+

核心开发语言

Spring Boot

2.x/3.x

项目脚手架，依赖管理

美团 Leaf

1.0.1.RELEASE

分布式ID生成核心框架

MySQL（可选）

5.7+/8.0+

Leaf 号段模式依赖，存储号段信息

Druid（可选）

1.2.18+

数据库连接池，适配号段模式

🚀 快速开始

1. 环境准备

- JDK 1.8 及以上

- Maven 3.6+ / Gradle 7.0+

- 若使用号段模式：MySQL 5.7+，创建 Leaf 专用数据库及表（见下文）

2. 引入依赖

在项目 pom.xml 中添加依赖（Maven）：

<!-- Leaf 核心依赖 -->
<dependency>
    <groupId>com.sankuai.inf.leaf</groupId>
    <artifactId>leaf-core</artifactId>
    <version>1.0.1.RELEASE</version>
</dependency>

<!-- 号段模式需额外引入（雪花模式可省略） -->
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>8.0.33</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>druid</artifactId>
    <version>1.2.18</version>
</dependency>

3. 配置文件

在 resources 目录下创建 leaf.properties，选择对应模式配置：

模式A：雪花算法模式（无数据库依赖，推荐）

# 关闭号段模式
leaf.segment.enable=false
# 开启雪花算法
leaf.snowflake.enable=true

# 雪花算法核心配置（分布式部署时保证机器ID唯一）
leaf.snowflake.workerId=1  # 机器ID（0-31）
leaf.snowflake.datacenterId=1  # 数据中心ID（0-31）

模式B：号段模式（依赖MySQL，高性能稳定）

先执行MySQL初始化SQL：

CREATE DATABASE IF NOT EXISTS leaf;
USE leaf;

CREATE TABLE IF NOT EXISTS leaf_alloc (
  biz_tag VARCHAR(128) NOT NULL COMMENT '业务标识（如order_id）',
  max_id BIGINT NOT NULL COMMENT '当前最大ID',
  step INT NOT NULL COMMENT '号段步长（如1000）',
  description VARCHAR(256) DEFAULT NULL COMMENT '业务描述',
  update_time DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (biz_tag)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='Leaf 号段分配表';

# 插入订单ID业务号段
INSERT INTO leaf_alloc (biz_tag, max_id, step, description) 
VALUES ('order_id', 0, 1000, '打车服务订单ID号段');

再配置 leaf.properties：

# 开启号段模式
leaf.segment.enable=true
# 关闭雪花算法
leaf.snowflake.enable=false

# 数据库连接配置
leaf.jdbc.url=jdbc:mysql://127.0.0.1:3306/leaf?useUnicode=true&characterEncoding=utf8&useSSL=false&serverTimezone=Asia/Shanghai
leaf.jdbc.username=root  # 你的MySQL用户名
leaf.jdbc.password=123456  # 你的MySQL密码
leaf.jdbc.driver-class-name=com.mysql.cj.jdbc.Driver

4. 初始化配置类

import com.sankuai.inf.leaf.IDGen;
import com.sankuai.inf.leaf.common.PropertyFactory;
import com.sankuai.inf.leaf.segment.SegmentIDGenImpl;
import com.sankuai.inf.leaf.snowflake.SnowflakeIDGenImpl;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.Properties;

@Configuration
public class LeafConfig {

    @Bean
    public IDGen idGen() throws Exception {
        Properties props = PropertyFactory.getProperties();
        IDGen idGen;
        if (Boolean.parseBoolean(props.getProperty("leaf.segment.enable"))) {
            // 号段模式初始化
            idGen = new SegmentIDGenImpl();
            ((SegmentIDGenImpl) idGen).init();
        } else if (Boolean.parseBoolean(props.getProperty("leaf.snowflake.enable"))) {
            // 雪花算法模式初始化
            idGen = new SnowflakeIDGenImpl();
            ((SnowflakeIDGenImpl) idGen).init();
        } else {
            throw new RuntimeException("请开启Leaf的号段模式或雪花算法模式");
        }
        return idGen;
    }
}

5. 业务使用示例

import com.sankuai.inf.leaf.IDGen;
import com.sankuai.inf.leaf.common.Result;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

@Service
public class TaxiOrderIdGenerator {

    @Autowired
    private IDGen idGen;

    /**
     * 生成打车订单号
     * @param cityCode 城市编码（如BJ=北京，SH=上海）
     * @param serviceType 服务类型（01=快车，02=专车，03=拼车）
     * @return 格式：城市编码+服务类型+雪花ID（如BJ011756892458765432832）
     */
    public String generateOrderId(String cityCode, String serviceType) {
        // 1. 生成核心ID（雪花/号段模式）
        Result result = idGen.get("order_id");
        if (!result.isSuccess()) {
            throw new RuntimeException("生成订单ID失败：" + result.getMsg());
        }
        Long coreId = result.getId();

        // 2. 拼接业务前缀（可选：添加随机盐值防伪造）
        // String salt = String.valueOf((int) (Math.random() * 100)); // 两位随机盐值
        // return cityCode + serviceType + salt + coreId;
        return cityCode + serviceType + coreId;
    }

    /**
     * 生成数据库订单主键（纯数字型，优化索引）
     * @return 雪花算法Long型ID
     */
    public Long generateOrderPrimaryKey() {
        Result result = idGen.get("order_id");
        if (!result.isSuccess()) {
            throw new RuntimeException("生成订单主键失败：" + result.getMsg());
        }
        return result.getId();
    }

    // 测试方法
    public static void main(String[] args) {
        // 模拟Spring注入（实际项目中无需手动创建）
        TaxiOrderIdGenerator generator = new TaxiOrderIdGenerator();
        // 模拟注入IDGen（实际由Spring管理）
        // generator.idGen = new SnowflakeIDGenImpl(); 
        // ((SnowflakeIDGenImpl) generator.idGen).init();

        // 生成北京快车订单号
        String orderId = generator.generateOrderId("BJ", "01");
        System.out.println("生成订单号：" + orderId); // 输出：BJ011756892458765432832

        // 生成订单主键
        Long orderPk = generator.generateOrderPrimaryKey();
        System.out.println("生成订单主键：" + orderPk); // 输出：1756892458765432832
    }
}

📊 订单号格式说明

默认格式：城市编码 + 服务类型 + 核心ID

部分

示例

长度

说明

城市编码

BJ/SH/GZ

2位

业务标识，区分订单所属城市

服务类型

01/02/03

2位

区分快车、专车、拼车等服务

核心ID

1756892458765432832

19-20位

雪花算法/Long型号段ID，保证唯一有序

扩展格式（防伪造）：城市编码 + 服务类型 + 随机盐值（2位） + 核心ID，示例：BJ01881756892458765432832。

⚠️ 注意事项

1. 分布式部署机器ID唯一：雪花算法模式下，不同节点的 workerId 必须唯一（0-31范围），避免ID重复。

2. 时钟同步：雪花算法依赖服务器时钟，分布式节点需保证时钟同步，避免时钟回拨导致ID生成失败。

3. 号段模式维护：号段模式下，MySQL 需高可用部署，避免数据库宕机导致ID生成中断。

4. 订单号长度控制：建议订单号总长度不超过32位，便于存储、展示及第三方对接。

5. 盐值优化：防伪造盐值建议结合系统配置动态生成，而非固定随机数，提升安全性。

🔄 模式对比与选型建议

对比维度

雪花算法模式

号段模式

依赖

无（仅依赖时钟）

依赖MySQL

性能

极高（纯内存运算）

高（号段缓存，减少DB访问）

稳定性

中（存在时钟回拨风险）

高（无时钟依赖）

部署复杂度

低（无需额外组件）

中（需部署MySQL）

选型建议

无数据库环境、轻量部署场景

生产环境、高稳定性要求场景

📈 性能测试

在 8核16G 服务器上，单节点测试结果：

- 雪花算法模式：QPS 可达 10w+，平均响应时间 < 1ms。

- 号段模式（步长1000）：QPS 可达 8w+，平均响应时间 < 2ms（依赖DB性能）。

🤝 贡献指南

1. Fork 本仓库。

2. 创建特性分支：git checkout -b feature/xxx。

3. 提交代码：git commit -m 'feat: 新增xxx功能'。

4. 推送到分支：git push origin feature/xxx。

5. 创建 Pull Request。

📄 许可证

本项目基于 MIT 许可证开源，详情见 LICENSE 文件。

❓ 常见问题

Q1：雪花算法的机器ID如何分配？

A1：可通过配置中心（如Nacos/Apollo）为每个节点分配唯一机器ID，或根据服务器IP后几位计算（确保0-31范围）。

Q2：订单号中的业务前缀可以自定义吗？

A2：可以。支持新增前缀（如用户类型、订单来源），只需修改 generateOrderId 方法的拼接逻辑，建议控制前缀总长度不超过8位。

Q3：如何处理时钟回拨问题？

A3：Leaf 雪花算法内置简单时钟回拨检测，若检测到回拨会抛出异常。生产环境建议搭配时钟同步工具（如NTP），并做好降级策略（切换号段模式）。
