# 本体驱动的软件建模方案
**Ontology-Driven Software Modeling Framework**

> 完整建模规范 · 八大模型元文件 · 实施指南
> 版本 1.0 | 2026年3月
> 作者 @人月聊IT 

---

## 目录

1. [方案概述与设计哲学](#第一章--方案概述与设计哲学)
2. [M1 对象模型](#第二章--m1-对象模型)
3. [M2 行为模型](#第三章--m2-行为模型)
4. [M3 规则模型](#第四章--m3-规则模型)
5. [ME 事件模型](#第五章--me-事件模型)
6. [M4 场景模型](#第六章--m4-场景模型)
7. [M5 主体模型](#第七章--m5-主体模型)
8. [M6 异常补偿模型](#第八章--m6-异常补偿模型)
9. [M7 质量约束模型](#第九章--m7-质量约束模型)
10. [传统需求覆盖度分析](#第十章--传统需求覆盖度分析)
11. [实施指南与最佳实践](#第十一章--实施指南与最佳实践)
12. [附录：术语对照表](#附录--术语对照表)

---

# 第一章  方案概述与设计哲学

## 1.1  框架定位

本方案面向中小规模软件系统，以"本体"（Ontology）作为系统设计的核心隐喻，将业务世界中的"存在（What）"、"行为（How）"、"规则（Why）"、"场景（When/Flow）"分离建模，并通过事件驱动架构（EDA）实现运行时的松耦合。

与传统需求分析方法（如UML用例图、功能规格说明书）相比，本框架具有以下核心差异：

- **语义驱动**：模型元素具有明确的业务语义，而非纯粹的技术描述
- **正交分解**：对象、行为、规则、场景四个维度相互独立，可单独演进
- **事件解耦**：通过事件链替代长事务，避免强耦合的同步调用链
- **可追溯性**：每个实现单元都可以溯源到具体的本体模型定义

---

## 1.2  八大本体模型元文件总览

在初始四模型基础上，补充了独立事件模型、权限主体模型、异常补偿模型和非功能性约束模型，形成完整的八模型体系：

| 编号 | 模型名称 | 核心职责 |
|------|----------|----------|
| **M1** | 对象模型 Object Model | 定义数据实体、实体属性、实体间的双向关联与参照完整性约束 |
| **M2** | 行为模型 Behavior Model | 定义对象的原子行为方法、触发条件、前置后置约束 |
| **M3** | 规则模型 Rule Model | 定义可复用的解耦业务规则，包括计算规则、验证规则、推导规则 |
| **ME** | 事件模型 Event Model | 定义业务事件、事件生产者、事件订阅者及事件链路关系 |
| **M4** | 场景模型 Scenario Model | 定义业务流程与用例场景，通过对象行为和事件链组装完整业务流 |
| **M5** | 主体模型 Actor Model | 定义系统参与者、角色、权限边界及行为执行授权关系 |
| **M6** | 异常补偿模型 Compensation Model | 定义行为失败的异常分类、补偿策略及Saga协调机制 |
| **M7** | 质量约束模型 Quality Model | 定义非功能性约束，包括性能SLA、可靠性要求、并发策略 |

---

## 1.3  模型间关系全景

八个模型之间的依赖与引用关系如下：

```
M4 场景模型      依赖   M2 行为模型  +  ME 事件模型  +  M6 异常补偿模型
M2 行为模型      依赖   M1 对象模型  +  M3 规则模型  +  M5 主体模型
ME 事件模型      依赖   M2 行为模型（生产者与订阅者）
M3 规则模型      引用   M1 对象模型（只读）
M5 主体模型      引用   M1 对象模型  +  M2 行为模型（权限绑定）
M6 异常补偿模型  引用   M2 行为模型（补偿行为）  +  M4 场景模型（Saga边界）
M7 质量约束模型  标注   M2 行为模型  +  M4 场景模型（QoS Annotation）
```

**关键设计决策**：事件模型从行为模型中独立出来，作为一等公民存在。事件不再内嵌于行为定义中，而是通过 `producerBehaviorRef` 和 `subscriberBehaviorRefs` 建立与行为的引用关系，实现松耦合的事件驱动架构。

---

# 第二章  M1 对象模型

## 2.1  设计目标与领域建模原则

对象模型是整个本体体系的基础，负责描述业务域中的核心领域对象及其关系，对应领域驱动设计（DDD）中的领域模型层。其核心设计原则是：

- **业务完整性优先**：从业务视角建模，而非数据库表结构视角
- **聚合边界清晰**：通过聚合根（Aggregate Root）保证业务不变性
- **实现延迟绑定**：对象模型关注业务语义，数据库拆分在实现阶段决策

> **关键设计理念**：对象模型描述的是"业务对象"而非"数据库表"。例如，订单（包含订单头和订单明细）在业务层面是一个完整的聚合，不应拆分为两个独立实体建模。数据库层面的表拆分是实现细节，属于持久化映射阶段的技术决策。

---

## 2.2  领域对象分类体系

本框架采用DDD的对象分类体系，将领域对象分为三类：

| 对象类型 | 说明 | 标识特征 | 生命周期 |
|----------|------|----------|----------|
| **聚合根（Aggregate Root）** | 业务完整性的边界，对外暴露的唯一入口，包含子实体和值对象 | 有全局唯一标识 | 有独立生命周期，可独立存在 |
| **实体（Entity）** | 聚合内部的子对象，有标识但不能脱离聚合根独立存在 | 有聚合内唯一标识 | 依赖聚合根生命周期，级联删除 |
| **值对象（Value Object）** | 无标识的不可变对象，通过属性值判断相等性，描述聚合的某个特征 | 无标识，通过值相等 | 依附于聚合根或实体，不可变 |

### 2.2.1  聚合根设计原则

聚合根是业务完整性的保护边界，遵循以下原则：

1. **唯一入口原则**：外部只能通过聚合根访问聚合内部对象，不能直接操作子实体
2. **事务边界原则**：一个事务只修改一个聚合根，跨聚合通过事件驱动
3. **不变性保护**：聚合根负责维护聚合内的业务不变性约束
4. **引用原则**：聚合之间通过ID引用，而非对象引用，避免聚合边界模糊

### 2.2.2  聚合识别指南

如何判断是否应该建模为一个聚合？

| 判断维度 | 聚合内（同一聚合） | 聚合间（独立聚合） |
|----------|-------------------|-------------------|
| 业务完整性 | 必须同时存在才有业务意义 | 可以独立存在 |
| 生命周期 | 同生共死，级联删除 | 独立生命周期 |
| 事务边界 | 必须在同一事务中修改 | 可以在不同事务中修改 |
| 访问路径 | 只能通过聚合根访问 | 可以直接访问 |
| 引用方式 | 对象引用（组合/聚合） | ID引用（关联） |

**示例**：
- ✅ 订单 + 订单明细 → 同一聚合（订单是聚合根，明细是子实体）
- ✅ 合同 + 付款条款 → 同一聚合（合同是聚合根，条款是子实体）
- ❌ 订单 + 客户 → 独立聚合（通过客户ID关联，而非组合）
- ❌ 订单 + 产品 → 独立聚合（产品有独立生命周期）

---

## 2.3  模型元素规范

### 2.3.1  聚合根（Aggregate Root）

| 属性名 | 类型 | 说明 |
|--------|------|------|
| id | String / UUID | 聚合根全局唯一标识，建议使用UUID v4 |
| name | String | 聚合根业务名称（中文，需唯一） |
| alias | String | 英文标识符，用于代码映射 |
| description | String | 业务含义说明 |
| aggregateType | Enum | AGGREGATE_ROOT（标识为聚合根） |
| lifecycle | Enum[] | 聚合根生命周期状态列表，如 [草稿, 生效, 注销] |
| attributes | Attribute[] | 聚合根自身属性集合 |
| entities | Entity[] | 聚合内子实体集合（组合关系） |
| valueObjects | ValueObject[] | 聚合内值对象集合 |
| invariants | Invariant[] | 聚合不变性约束（业务规则） |
| tags | String[] | 分类标签，如 [核心域, 支撑域] |

### 2.3.2  子实体（Entity）

| 属性名 | 类型 | 说明 |
|--------|------|------|
| name | String | 子实体名称 |
| alias | String | 英文标识符 |
| description | String | 业务含义说明 |
| localId | String | 聚合内唯一标识字段名（如 itemId） |
| attributes | Attribute[] | 子实体属性集合 |
| cardinality | Enum | 与聚合根的基数关系：ONE / ZERO_OR_ONE / ONE_OR_MORE / ZERO_OR_MORE |
| cascadeDelete | Boolean | 是否级联删除（默认true） |

### 2.3.3  值对象（Value Object）

| 属性名 | 类型 | 说明 |
|--------|------|------|
| name | String | 值对象名称 |
| alias | String | 英文标识符 |
| description | String | 业务含义说明 |
| attributes | Attribute[] | 值对象属性集合 |
| immutable | Boolean | 是否不可变（默认true） |
| equalityFields | String[] | 用于判断相等性的字段列表 |

### 2.3.4  属性（Attribute）

| 属性名 | 类型 | 说明 |
|--------|------|------|
| name | String | 属性名（英文camelCase） |
| label | String | 业务展示名（中文） |
| type | DataType | 基础类型：String / Integer / Decimal / Date / DateTime / Boolean / Enum / Money / ValueObject |
| required | Boolean | 是否必填 |
| unique | Boolean | 是否唯一（仅对聚合根属性有效） |
| defaultValue | Any | 默认值 |
| enumValues | String[] | 当type=Enum时的枚举值列表 |
| valueObjectRef | String | 当type=ValueObject时引用的值对象名称 |

### 2.3.5  聚合不变性约束（Invariant）

聚合不变性是必须始终满足的业务规则，由聚合根负责维护：

| 属性名 | 类型 | 说明 |
|--------|------|------|
| name | String | 约束名称 |
| expression | String | 约束表达式，可跨聚合内的子实体和值对象 |
| violationMessage | String | 违反约束时的业务错误消息 |
| enforcedAt | Enum | 执行时机：ON_CREATE / ON_UPDATE / ON_DELETE / ALWAYS |

### 2.3.6  聚合间关联（Aggregate Association）

聚合之间通过ID引用建立关联，而非对象引用：

| 属性名 | 类型 | 说明 |
|--------|------|------|
| id | String | 关联唯一标识 |
| sourceAggregate | AggregateRef | 来源聚合根 |
| targetAggregate | AggregateRef | 目标聚合根 |
| associationType | Enum | REFERENCE（引用）/ DEPENDENCY（依赖） |
| sourceRole | String | 来源端角色名 |
| targetRole | String | 目标端角色名 |
| cardinality | Enum | 基数关系：ONE_TO_ONE / ONE_TO_MANY / MANY_TO_MANY |
| referenceField | String | 存储目标聚合根ID的字段名 |

---

## 2.4  YAML 元文件模板

### 2.4.1  聚合根建模示例（订单聚合）

```yaml
# M1 对象模型元文件 - object-model.yaml
model_type: OBJECT
version: "2.0"
domain: "订单管理"

aggregates:
  # ══════════════════════════════════════════════════════════
  # 订单聚合（Aggregate Root）
  # ══════════════════════════════════════════════════════════
  - id: AGG-ORDER-001
    name: 订单
    alias: Order
    aggregateType: AGGREGATE_ROOT
    description: 客户下单的完整业务对象，包含订单头信息和订单明细列表
    lifecycle: [草稿, 待支付, 已支付, 已发货, 已完成, 已取消]
    tags: [核心域]
    
    # 聚合根自身属性
    attributes:
      - name: orderId
        label: 订单号
        type: String
        required: true
        unique: true
      - name: orderDate
        label: 下单日期
        type: DateTime
        required: true
      - name: status
        label: 订单状态
        type: Enum
        enumValues: [草稿, 待支付, 已支付, 已发货, 已完成, 已取消]
        required: true
      - name: customerId
        label: 客户ID
        type: String
        required: true
        description: 引用客户聚合根的ID
      - name: totalAmount
        label: 订单总额
        type: Money
        required: true
      - name: shippingAddress
        label: 收货地址
        type: ValueObject
        valueObjectRef: Address
        required: true
    
    # 聚合内子实体（订单明细）
    entities:
      - name: 订单明细
        alias: OrderItem
        description: 订单中的商品明细行
        localId: itemId
        cardinality: ONE_OR_MORE
        cascadeDelete: true
        attributes:
          - name: itemId
            label: 明细ID
            type: String
            required: true
            description: 聚合内唯一标识
          - name: productId
            label: 商品ID
            type: String
            required: true
            description: 引用商品聚合根的ID
          - name: productName
            label: 商品名称
            type: String
            required: true
            description: 冗余字段，避免查询商品聚合
          - name: quantity
            label: 数量
            type: Integer
            required: true
          - name: unitPrice
            label: 单价
            type: Money
            required: true
          - name: subtotal
            label: 小计
            type: Money
            required: true
    
    # 聚合内值对象
    valueObjects:
      - name: 收货地址
        alias: Address
        description: 订单的收货地址信息
        immutable: true
        equalityFields: [province, city, district, street, detail]
        attributes:
          - name: province
            label: 省份
            type: String
            required: true
          - name: city
            label: 城市
            type: String
            required: true
          - name: district
            label: 区县
            type: String
            required: true
          - name: street
            label: 街道
            type: String
          - name: detail
            label: 详细地址
            type: String
            required: true
          - name: postalCode
            label: 邮编
            type: String
          - name: contactName
            label: 联系人
            type: String
            required: true
          - name: contactPhone
            label: 联系电话
            type: String
            required: true
    
    # 聚合不变性约束
    invariants:
      - name: 订单总额一致性
        expression: "totalAmount == SUM(items.subtotal)"
        violationMessage: 订单总额必须等于所有明细小计之和
        enforcedAt: ALWAYS
      - name: 明细小计正确性
        expression: "FORALL item IN items: item.subtotal == item.quantity * item.unitPrice"
        violationMessage: 每个明细的小计必须等于数量乘以单价
        enforcedAt: ALWAYS
      - name: 订单明细非空
        expression: "items.size() >= 1"
        violationMessage: 订单必须至少包含一个明细
        enforcedAt: ON_CREATE
      - name: 已支付订单不可修改
        expression: "IF status IN ['已支付', '已发货', '已完成'] THEN items.immutable == true"
        violationMessage: 已支付的订单不能修改明细
        enforcedAt: ON_UPDATE

  # ══════════════════════════════════════════════════════════
  # 客户聚合（独立聚合根）
  # ══════════════════════════════════════════════════════════
  - id: AGG-CUSTOMER-001
    name: 客户
    alias: Customer
    aggregateType: AGGREGATE_ROOT
    description: 客户主数据
    lifecycle: [潜在客户, 正式客户, 休眠客户, 注销]
    tags: [支撑域]
    
    attributes:
      - name: customerId
        label: 客户ID
        type: String
        required: true
        unique: true
      - name: customerName
        label: 客户名称
        type: String
        required: true
      - name: customerType
        label: 客户类型
        type: Enum
        enumValues: [个人, 企业]
        required: true
      - name: contactInfo
        label: 联系信息
        type: ValueObject
        valueObjectRef: ContactInfo
    
    valueObjects:
      - name: 联系信息
        alias: ContactInfo
        immutable: false
        attributes:
          - name: email
            label: 邮箱
            type: String
          - name: phone
            label: 电话
            type: String
            required: true
          - name: wechat
            label: 微信号
            type: String

  # ══════════════════════════════════════════════════════════
  # 商品聚合（独立聚合根）
  # ══════════════════════════════════════════════════════════
  - id: AGG-PRODUCT-001
    name: 商品
    alias: Product
    aggregateType: AGGREGATE_ROOT
    description: 商品主数据
    lifecycle: [草稿, 上架, 下架, 停售]
    tags: [支撑域]
    
    attributes:
      - name: productId
        label: 商品ID
        type: String
        required: true
        unique: true
      - name: productName
        label: 商品名称
        type: String
        required: true
      - name: price
        label: 标准价格
        type: Money
        required: true
      - name: category
        label: 商品类别
        type: String

# ══════════════════════════════════════════════════════════
# 聚合间关联（通过ID引用）
# ══════════════════════════════════════════════════════════
aggregate_associations:
  - id: ASSOC-001
    sourceAggregate: AGG-ORDER-001
    targetAggregate: AGG-CUSTOMER-001
    associationType: REFERENCE
    sourceRole: 所属客户
    targetRole: 客户订单
    cardinality: MANY_TO_ONE
    referenceField: customerId
    description: 订单通过customerId引用客户聚合根

  - id: ASSOC-002
    sourceAggregate: AGG-ORDER-001
    targetAggregate: AGG-PRODUCT-001
    associationType: REFERENCE
    sourceRole: 订购商品
    targetRole: 商品订单
    cardinality: MANY_TO_MANY
    referenceField: items[].productId
    description: 订单明细通过productId引用商品聚合根
```

---

## 2.5  聚合建模最佳实践

### 2.5.1  聚合大小控制

- **小聚合优先**：聚合越小，并发冲突越少，性能越好
- **业务完整性优先**：不能为了性能牺牲业务完整性
- **经验法则**：一个聚合包含的子实体不超过3-5个，总字段数不超过30个

### 2.5.2  聚合拆分时机

当聚合过大时，考虑拆分：

| 拆分信号 | 处理方式 |
|----------|----------|
| 子实体有独立生命周期 | 提升为独立聚合根，通过ID引用 |
| 子实体被多个聚合引用 | 提升为独立聚合根 |
| 聚合内部分字段很少一起修改 | 拆分为多个聚合，通过事件保持一致性 |
| 并发冲突频繁 | 缩小聚合边界，减少锁竞争 |

### 2.5.3  数据冗余策略

为了避免跨聚合查询，可以在聚合内冗余其他聚合的关键信息：

```yaml
# 订单明细中冗余商品名称
- name: productName
  label: 商品名称
  type: String
  description: 冗余字段，避免查询商品聚合。通过事件同步更新。
```

**冗余原则**：
- 只冗余展示用的稳定字段（如名称、编码）
- 不冗余频繁变化的字段（如价格、库存）
- 通过领域事件保持冗余数据的最终一致性

### 2.5.4  聚合根ID设计

| ID类型 | 适用场景 | 示例 |
|--------|----------|------|
| UUID | 分布式系统，需要客户端生成ID | `550e8400-e29b-41d4-a716-446655440000` |
| 业务编号 | 有业务规则的编号体系 | `ORD-2026-03-0001` |
| 雪花ID | 高并发，需要趋势递增 | `1234567890123456789` |

---

## 2.6  与数据库实现的映射

对象模型是业务层的概念模型，与数据库表的映射在实现阶段决策：

| 对象模型 | 数据库映射策略 | 说明 |
|----------|---------------|------|
| 聚合根 | 主表 | 一个聚合根对应一个主表 |
| 子实体（集合） | 从表 | 通过外键关联主表，一对多关系 |
| 子实体（单个） | 主表字段 | 如果字段不多，可以平铺到主表 |
| 值对象 | 主表字段 | 通过字段前缀区分，如 `shipping_address_province` |
| 值对象（复杂） | JSON字段 | 使用数据库的JSON类型存储 |
| 聚合间关联 | 外键字段 | 存储目标聚合根的ID |

**示例映射**：

```
订单聚合 → 数据库表设计

orders 表（聚合根）:
  - order_id (PK)
  - order_date
  - status
  - customer_id (FK → customers.customer_id)
  - total_amount
  - shipping_address_province
  - shipping_address_city
  - shipping_address_detail
  - shipping_address_contact_name
  - shipping_address_contact_phone

order_items 表（子实体）:
  - item_id (PK)
  - order_id (FK → orders.order_id)
  - product_id (FK → products.product_id)
  - product_name (冗余)
  - quantity
  - unit_price
  - subtotal
```

> **重要提示**：数据库表拆分是实现细节，不应影响对象模型的设计。对象模型始终从业务完整性出发，保持聚合的业务语义清晰。

---

# 第三章  M2 行为模型

## 3.1  设计目标

行为模型定义对象能够执行的原子行为方法。每个行为是单一对象发出的、不可再分的核心操作单元。复合行为（Composite Behavior）由多个原子行为通过场景模型编排，而非在行为模型内部内嵌。

> **关键设计原则**：行为模型的核心约束是"原子性"——每个行为方法只做一件事，操作一个对象，产生确定性的状态变更。复杂的业务逻辑通过规则模型注入，通过场景模型编排，而不是在行为方法内扩张。

---

## 3.2  模型元素规范

### 3.2.1  行为（Behavior）

| 属性名 | 类型 | 说明 |
|--------|------|------|
| id | String | 行为唯一标识，建议格式：{EntityAlias}_{ActionName} |
| name | String | 行为业务名称（中文动宾短语） |
| ownerEntity | EntityRef | 行为所属对象 |
| behaviorType | Enum | COMMAND（指令，改变状态）/ QUERY（查询，只读）/ EVENT_HANDLER（事件响应处理器） |
| triggerType | Enum | USER_ACTION（用户触发）/ SYSTEM（系统定时）/ EVENT（事件订阅）/ EXTERNAL（外部接口调用） |
| preconditions | Condition[] | 前置条件集合，全部满足才可执行 |
| postconditions | StateChange[] | 执行后的状态变更描述 |
| appliedRules | RuleRef[] | 调用的规则模型引用 |
| requiredPermissions | PermissionRef[] | 执行所需权限（引用M5主体模型） |
| producedEvents | Event[] | 执行成功后产生的事件 |
| qualityAnnotation | QoSRef | 质量约束标注（引用M7） |
| compensationRef | CompensationRef | 关联的补偿行为（引用M6） |

### 3.2.2  事件引用（Event Reference）— 简化版

在行为模型中，`producedEvents` 字段仅包含事件ID的引用列表，不再内嵌完整的事件定义：

```yaml
behaviors:
  - id: Order_ConfirmPayment
    name: 确认订单支付
    # ... 其他字段 ...
    producedEvents:
      - Order.PaymentConfirmed  # 仅引用事件ID
```

事件的完整定义（包括生产者、订阅者、载荷、顺序语义等）统一在 **ME 事件模型** 中维护。这种设计避免了信息重复和不一致，使事件成为可独立演进的一等公民。

> **重要说明**：如果您的项目尚未采用独立事件模型，也可以在行为模型中内嵌完整的事件定义（参见3.2.3节的传统方式）。但推荐采用独立事件模型，以获得更好的松耦合和可维护性。

### 3.2.3  事件（Event）— 传统内嵌方式（不推荐）

> **注意**：此方式已被独立的ME事件模型取代，仅作为向后兼容保留。新项目请使用ME事件模型。

| 属性名 | 类型 | 说明 |
|--------|------|------|
| eventId | String | 事件唯一标识，建议格式：{Entity}.{State}_{Past}，如 Order.PaymentConfirmed |
| eventName | String | 事件业务名称（中文） |
| triggerCondition | String | 触发事件的条件表达式 |
| payload | FieldRef[] | 事件携带的对象字段引用 |
| subscribers | BehaviorRef[] | 已知订阅者（可选，用于文档化） |
| ordering | Enum | 事件顺序语义：AT_LEAST_ONCE / EXACTLY_ONCE / BEST_EFFORT |

### 3.2.4  前置/后置条件（Condition / StateChange）

使用简洁的谓词表达式语法，避免引入完整编程语言的复杂度：

- 前置条件示例：`order.status == '待支付'  AND  order.totalAmount > 0`
- 状态变更示例：`order.status = '已支付'  |  order.paidAt = NOW()`
- 支持跨实体引用：`stock.availableQty >= orderItem.quantity`

---

## 3.3  YAML 元文件模板

```yaml
# M2 行为模型元文件 - behavior-model.yaml
model_type: BEHAVIOR
version: "1.0"
domain: "订单管理"

behaviors:
  - id: Order_ConfirmPayment
    name: 确认订单支付
    ownerEntity: ORD-001
    behaviorType: COMMAND
    triggerType: USER_ACTION
    preconditions:
      - "order.status == '待支付'"
      - "order.totalAmount > 0"
    postconditions:
      - field: order.status
        setValue: "已支付"
      - field: order.paidAt
        setValue: NOW()
    appliedRules:
      - RULE-PAY-001   # 支付金额校验规则
      - RULE-FRAUD-001 # 风控检查规则
    requiredPermissions:
      - PERM-ORDER-PAY
    producedEvents:
      - Order.PaymentConfirmed  # 仅引用事件ID，完整定义在ME事件模型中
    qualityAnnotation: QA-ORDER-001
    compensationRef: COMP-Order_ConfirmPayment

  - id: Inventory_DeductStock
    name: 扣减库存
    ownerEntity: INV-001
    behaviorType: COMMAND
    triggerType: EVENT  # 声明为事件触发，订阅关系在ME事件模型中定义
    preconditions:
      - "stock.availableQty >= orderItem.quantity"
    postconditions:
      - field: stock.availableQty
        setValue: "stock.availableQty - orderItem.quantity"
      - field: stock.lockedQty
        setValue: "stock.lockedQty + orderItem.quantity"
    appliedRules:
      - RULE-INV-001   # 库存扣减幂等规则
    producedEvents:
      - Inventory.StockDeducted  # 仅引用事件ID
    compensationRef: COMP-Inventory_DeductStock
```

> **说明**：`producedEvents` 字段仅包含事件ID引用。事件的完整定义（生产者、订阅者、载荷等）在 `me-event-model.yaml` 中维护。

---

# 第四章  M3 规则模型

## 4.1  设计目标与边界

规则模型专注于可复用的解耦业务规则，是从行为逻辑中分离出来的独立关切。其核心价值在于：同一规则可以被多个行为引用，规则变更不影响行为定义。

> **重要边界说明**：参照完整性约束（外键约束、唯一约束等）不在规则模型中定义，而是归属M1对象模型。规则模型处理的是业务逻辑层面的约束，而非数据结构层面的约束。

> **重大扩展**：规则不再只是被动的验证器或计算器，而是可以作为事件驱动架构中的主动参与者。规则可以订阅事件、执行业务逻辑判断、并在满足条件时触发新的事件，成为连接行为与行为之间的智能决策节点。

---

## 4.2  规则分类体系

| 规则类型 | 说明 | 触发方式 |
|----------|------|----------|
| 验证规则（Validation Rule） | 判断输入数据是否符合业务要求，返回 true/false，不改变状态 | 被行为调用 |
| 计算规则（Calculation Rule） | 根据输入参数计算并返回结果值，如计算运费、折扣金额 | 被行为调用 |
| 推导规则（Derivation Rule） | 基于已知属性推导出其他属性值，如根据级别推导权限范围 | 被行为调用 |
| 转换规则（Transformation Rule） | 将一种数据格式转换为另一种，适用于外部系统集成场景 | 被行为调用 |
| 风控规则（Risk Rule） | 评估业务风险，返回风险等级或通过/拒绝决策（可接入外部规则引擎） | 被行为调用 |
| **事件驱动规则（Event-Driven Rule）** | **订阅事件，执行条件判断，满足条件时触发新事件** | **事件触发** |

### 4.2.1  事件驱动规则的特殊性

事件驱动规则是规则模型的重要扩展，具有以下特征：

- **主动性**：不是被动等待行为调用，而是主动订阅事件
- **决策性**：作为事件链中的决策节点，判断是否继续传播
- **事件生产**：满足条件时可以触发新的事件
- **解耦性**：将复杂的条件判断从行为中分离，提高可维护性

**典型场景**：
- 合同激活后，配送规则判断是否满足配送条件（合同生效 AND 配送员空闲）
- 订单支付后，风控规则判断是否需要人工审核（金额超限 OR 异常地址）
- 库存扣减后，补货规则判断是否触发补货流程（库存低于安全阈值）

---

## 4.3  模型元素规范

### 4.3.1  通用规则属性

| 属性名 | 类型 | 说明 |
|--------|------|------|
| id | String | 规则唯一标识，建议格式：RULE-{Domain}-{Seq} |
| name | String | 规则业务名称 |
| ruleType | Enum | VALIDATION / CALCULATION / DERIVATION / TRANSFORMATION / RISK / EVENT_DRIVEN |
| description | String | 规则的业务逻辑说明 |
| inputParams | Param[] | 输入参数定义（名称、类型、来源字段） |
| outputType | DataType | 返回值类型（对于事件驱动规则，可以是 Boolean 或 Event） |
| expression | String | 规则表达式（支持伪代码或DSL，不限定具体语言） |
| reusedBy | BehaviorRef[] | 引用本规则的行为列表（反向追踪） |
| externalEngine | String | 若委托外部规则引擎，填写引擎名称（如 Drools, OPA） |
| version | String | 规则版本，支持规则的独立版本管理 |

### 4.3.2  事件驱动规则专属属性

| 属性名 | 类型 | 说明 |
|--------|------|------|
| triggerType | Enum | EVENT（标识为事件触发型规则） |
| subscribedEvents | EventRef[] | 订阅的事件ID列表，规则监听这些事件 |
| conditionExpression | String | 触发条件表达式，满足条件时才执行规则逻辑 |
| producedEvents | EventRef[] | 规则执行成功后可能产生的事件列表 |
| eventTriggerCondition | String | 触发事件的条件，如 "result == true" 或 "riskLevel == 'HIGH'" |
| executionMode | Enum | SYNC（同步执行）/ ASYNC（异步执行） |
| timeout | Duration | 规则执行超时时间（异步模式） |

### 4.3.3  输入参数（Param）

| 属性名 | 类型 | 说明 |
|--------|------|------|
| name | String | 参数名称 |
| type | DataType | 参数类型 |
| sourceField | String | 来源字段路径，如 "order.totalAmount" 或 "event.payload.contractId" |
| required | Boolean | 是否必填 |
| description | String | 参数说明 |

---

## 4.4  YAML 元文件模板

### 4.4.1  传统规则示例（被行为调用）

```yaml
# M3 规则模型元文件 - rule-model.yaml
model_type: RULE
version: "2.0"
domain: "订单管理"

rules:
  - id: RULE-PAY-001
    name: 支付金额一致性验证
    ruleType: VALIDATION
    description: 验证支付金额与订单总额一致，允许误差0.01元
    inputParams:
      - name: paymentAmount
        type: Decimal
        sourceField: Payment.amount
        required: true
      - name: orderAmount
        type: Decimal
        sourceField: Order.totalAmount
        required: true
    outputType: Boolean
    expression: |
      ABS(paymentAmount - orderAmount) <= 0.01
    version: "1.2"
    reusedBy:
      - Order_ConfirmPayment
      - Payment_Reconcile

  - id: RULE-SHIP-001
    name: 运费计算规则
    ruleType: CALCULATION
    description: 根据包裹重量和目的地区域计算运费
    inputParams:
      - name: weight
        type: Decimal
        required: true
      - name: destRegion
        type: String
        required: true
    outputType: Decimal
    expression: |
      IF destRegion IN ['华东', '华南'] THEN
        IF weight <= 1 THEN 8 ELSE 8 + (weight - 1) * 2
      ELSE
        IF weight <= 1 THEN 12 ELSE 12 + (weight - 1) * 3
    version: "2.0"

  - id: RULE-FRAUD-001
    name: 支付风控检查
    ruleType: RISK
    description: 对支付行为进行风险评估，超过阈值需人工审核
    inputParams:
      - name: userId
        type: String
        required: true
      - name: paymentAmount
        type: Decimal
        required: true
      - name: deviceFingerprint
        type: String
        required: true
    outputType: Enum  # PASS / REVIEW / REJECT
    externalEngine: "RiskEngineService"
    version: "3.1"
```

### 4.4.2  事件驱动规则示例（订阅事件）

```yaml
# M3 规则模型元文件 - rule-model.yaml（续）
rules:
  # ══════════════════════════════════════════════════════════
  # 事件驱动规则
  # ══════════════════════════════════════════════════════════
  
  - id: RULE-DELIVERY-001
    name: 配送条件检查规则
    ruleType: EVENT_DRIVEN
    description: 监听合同激活事件，判断是否满足配送条件，满足则触发配送就绪事件
    triggerType: EVENT
    
    # 订阅的事件
    subscribedEvents:
      - Contract.Activated
    
    # 输入参数（从事件载荷中提取）
    inputParams:
      - name: contractId
        type: String
        sourceField: event.payload.contractId
        required: true
      - name: productType
        type: String
        sourceField: event.payload.productType
        required: true
      - name: deliveryAddress
        type: String
        sourceField: event.payload.deliveryAddress
        required: true
    
    # 条件表达式（满足条件才执行规则）
    conditionExpression: |
      productType == '实物商品' AND deliveryAddress IS NOT NULL
    
    # 规则逻辑表达式
    expression: |
      // 查询配送员状态
      availableDrivers = DeliveryService.getAvailableDrivers(deliveryAddress.region)
      
      // 判断是否有空闲配送员
      IF availableDrivers.size() > 0 THEN
        RETURN {
          canDeliver: true,
          assignedDriver: availableDrivers[0].driverId,
          estimatedTime: availableDrivers[0].estimatedArrival
        }
      ELSE
        RETURN {
          canDeliver: false,
          reason: '当前区域无可用配送员'
        }
    
    outputType: Object
    
    # 规则执行成功后产生的事件
    producedEvents:
      - Delivery.ReadyToDispatch
    
    # 触发事件的条件
    eventTriggerCondition: "result.canDeliver == true"
    
    executionMode: ASYNC
    timeout: PT5S
    version: "1.0"

  - id: RULE-RESTOCK-001
    name: 自动补货规则
    ruleType: EVENT_DRIVEN
    description: 监听库存扣减事件，判断是否需要触发补货流程
    triggerType: EVENT
    
    subscribedEvents:
      - Inventory.StockDeducted
    
    inputParams:
      - name: productId
        type: String
        sourceField: event.payload.productId
        required: true
      - name: remainingQty
        type: Integer
        sourceField: event.payload.remainingQty
        required: true
    
    expression: |
      // 查询商品的安全库存阈值
      safetyStock = ProductService.getSafetyStock(productId)
      
      // 判断是否低于安全库存
      IF remainingQty < safetyStock THEN
        restockQty = safetyStock * 2 - remainingQty
        RETURN {
          needRestock: true,
          productId: productId,
          restockQty: restockQty,
          priority: IF remainingQty == 0 THEN 'URGENT' ELSE 'NORMAL'
        }
      ELSE
        RETURN { needRestock: false }
    
    outputType: Object
    
    producedEvents:
      - Inventory.RestockRequired
    
    eventTriggerCondition: "result.needRestock == true"
    
    executionMode: ASYNC
    timeout: PT3S
    version: "1.0"

  - id: RULE-AUDIT-001
    name: 大额订单审核规则
    ruleType: EVENT_DRIVEN
    description: 监听订单支付确认事件，判断是否需要人工审核
    triggerType: EVENT
    
    subscribedEvents:
      - Order.PaymentConfirmed
    
    inputParams:
      - name: orderId
        type: String
        sourceField: event.payload.orderId
        required: true
      - name: totalAmount
        type: Decimal
        sourceField: event.payload.totalAmount
        required: true
      - name: customerId
        type: String
        sourceField: event.payload.customerId
        required: true
    
    expression: |
      // 获取客户信用等级
      customerLevel = CustomerService.getCreditLevel(customerId)
      
      // 判断是否需要审核
      IF totalAmount > 10000 AND customerLevel IN ['新客户', '低信用'] THEN
        RETURN {
          needAudit: true,
          reason: '大额订单且客户信用等级较低',
          auditLevel: 'MANAGER'
        }
      ELSE IF totalAmount > 50000 THEN
        RETURN {
          needAudit: true,
          reason: '超大额订单',
          auditLevel: 'DIRECTOR'
        }
      ELSE
        RETURN { needAudit: false }
    
    outputType: Object
    
    producedEvents:
      - Order.AuditRequired
    
    eventTriggerCondition: "result.needAudit == true"
    
    executionMode: SYNC
    version: "1.0"
```

---

## 4.5  事件驱动规则的执行流程

```
┌─────────────────────────────────────────────────────────────┐
│                     事件驱动规则执行流程                      │
└─────────────────────────────────────────────────────────────┘

1. 事件发布
   行为执行 → 产生事件 → 发布到事件总线

2. 规则订阅匹配
   事件总线 → 查找订阅该事件的规则 → 触发规则执行

3. 条件判断
   规则引擎 → 评估 conditionExpression → 决定是否执行规则逻辑

4. 规则执行
   执行 expression → 计算结果 → 返回 outputType

5. 事件触发判断
   评估 eventTriggerCondition → 决定是否触发新事件

6. 新事件发布
   满足条件 → 产生新事件 → 发布到事件总线 → 继续传播

7. 事件链传播
   新事件 → 触发订阅该事件的行为或规则 → 形成事件链
```

**示例流程**：
```
合同管理员.激活合同行为
  ↓ 产生
Contract.Activated 事件
  ↓ 订阅
RULE-DELIVERY-001 配送条件检查规则
  ↓ 判断：合同生效 AND 配送员空闲
  ↓ 满足条件，产生
Delivery.ReadyToDispatch 事件
  ↓ 订阅
配送对象.创建配送单行为
  ↓ 执行
创建配送单
```

---

## 4.6  事件驱动规则的最佳实践

### 4.6.1  规则粒度控制

- **单一职责**：一个规则只做一件事，只判断一个业务条件
- **避免过度嵌套**：规则逻辑不应过于复杂，复杂逻辑拆分为多个规则
- **明确输入输出**：清晰定义输入参数和输出结果

### 4.6.2  性能考虑

- **异步优先**：非关键路径的规则使用异步执行模式
- **超时设置**：为规则设置合理的超时时间，避免阻塞
- **缓存策略**：频繁查询的数据（如安全库存阈值）应该缓存

### 4.6.3  错误处理

- **规则失败不应阻断事件链**：规则执行失败应记录日志，但不影响其他订阅者
- **重试机制**：对于临时性失败（如网络超时），应该有重试机制
- **降级策略**：规则不可用时，应该有默认行为

### 4.6.4  可观测性

- **规则执行日志**：记录规则的输入、输出、执行时间
- **事件链追踪**：记录事件的完整传播路径（行为→事件→规则→事件→行为）
- **规则性能监控**：监控规则的执行时间、成功率、失败率

### 4.6.5  规则版本管理

- **向后兼容**：规则升级时保持输入输出接口的兼容性
- **灰度发布**：新规则先在小范围测试，再全量发布
- **回滚机制**：规则出现问题时，能够快速回滚到上一版本

---

# 第五章  ME 事件模型

## 5.1  设计目标与边界

事件模型是从行为模型中独立出来的一等公民，专注于定义业务事件及其在系统中的传播路径。其核心价值在于：

- **松耦合架构**：事件生产者与订阅者之间无直接依赖，通过事件中介实现解耦
- **事件链可视化**：通过生产者和订阅者引用，清晰表达事件驱动的业务流程
- **异步处理支持**：事件天然支持异步处理模式，适合长流程和跨服务协作
- **可追溯性**：每个事件都有明确的生产者和订阅者，便于业务流程追踪和问题排查
- **智能决策节点**：规则可以作为事件的订阅者和生产者，实现复杂的业务编排

> **关键设计决策**：事件不再内嵌于行为模型的 `producedEvents` 字段中，而是作为独立模型存在。事件的完整定义（包括生产者、订阅者、载荷）统一在事件模型中维护。

> **重大扩展**：事件的生产者和订阅者不再局限于行为，规则也可以作为事件的生产者和订阅者，形成"行为→事件→规则→事件→行为"的完整闭环。

---

## 5.2  事件模型的核心关切

| 关切维度 | 说明 |
|----------|------|
| 事件定义 | 业务事件的唯一标识、名称、业务含义 |
| 生产者 | 哪个行为或规则触发了该事件（单一生产者原则） |
| 订阅者 | 哪些行为或规则订阅了该事件（支持多个订阅者） |
| 事件载荷 | 事件携带的业务数据字段，供订阅者使用 |
| 顺序语义 | 事件的投递保证级别（至少一次、恰好一次、尽力而为） |
| 事件链构建 | 通过生产者和订阅者引用，构建完整的事件驱动流程链路 |

---

## 5.3  事件链的构建机制

事件模型通过以下机制构建完整的事件链：

### 5.3.1  单一生产者原则

每个事件有且仅有一个生产者（行为或规则），这确保了事件来源的明确性和可追溯性。

```
生产者类型：
- 行为（Behavior）：行为执行成功后触发事件
- 规则（Rule）：规则判断满足条件后触发事件
```

### 5.3.2  多订阅者支持

一个事件可以被多个行为或规则订阅，实现一对多的事件广播模式。订阅者之间相互独立，互不影响。

```
订阅者类型：
- 行为（Behavior）：接收事件后执行业务操作
- 规则（Rule）：接收事件后执行条件判断，可能触发新事件
```

### 5.3.3  事件链的传递模式

#### 模式1：行为 → 事件 → 行为
```
订单支付行为 --触发--> Order.PaymentConfirmed事件 --订阅--> 库存扣减行为
```

#### 模式2：行为 → 事件 → 规则 → 事件 → 行为
```
合同激活行为 --触发--> Contract.Activated事件 
                    --订阅--> 配送条件检查规则
                              --触发--> Delivery.ReadyToDispatch事件
                                        --订阅--> 创建配送单行为
```

#### 模式3：规则 → 事件 → 规则（规则链）
```
库存扣减事件 --订阅--> 补货规则 --触发--> Inventory.RestockRequired事件
                                      --订阅--> 采购审批规则
```

### 5.3.4  跨对象协作

事件模型天然支持跨聚合的业务协作。例如：

- 订单聚合的"支付确认"行为触发 `Order.PaymentConfirmed` 事件
- 库存聚合的"扣减库存"行为订阅该事件
- 通知聚合的"发送支付成功通知"行为也订阅该事件
- 风控规则订阅该事件，判断是否需要人工审核

这种模式避免了聚合之间的直接调用，实现了真正的松耦合。

---

## 5.4  模型元素规范

### 5.4.1  事件（Event）

| 属性名 | 类型 | 说明 |
|--------|------|------|
| eventId | String | 事件唯一标识，建议格式：{Entity}.{State}_{Past}，如 Order.PaymentConfirmed |
| eventName | String | 事件业务名称（中文） |
| description | String | 事件的业务含义说明 |
| **producerType** | **Enum** | **生产者类型：BEHAVIOR（行为）/ RULE（规则）** |
| **producerBehaviorRef** | **BehaviorRef** | **当producerType=BEHAVIOR时，生产该事件的行为引用** |
| **producerRuleRef** | **RuleRef** | **当producerType=RULE时，生产该事件的规则引用** |
| producerEntityRef | EntityRef | 生产者所属的聚合根引用（用于可视化） |
| triggerCondition | String | 触发事件的条件表达式 |
| payload | PayloadField[] | 事件携带的数据字段列表 |
| **subscribers** | **Subscriber[]** | **订阅者列表（行为或规则）** |
| ordering | Enum | 事件顺序语义：AT_LEAST_ONCE / EXACTLY_ONCE / BEST_EFFORT |
| version | String | 事件版本，支持事件schema的演进 |

### 5.4.2  订阅者（Subscriber）

| 属性名 | 类型 | 说明 |
|--------|------|------|
| **subscriberType** | **Enum** | **订阅者类型：BEHAVIOR（行为）/ RULE（规则）** |
| **subscriberBehaviorRef** | **BehaviorRef** | **当subscriberType=BEHAVIOR时，订阅该事件的行为引用** |
| **subscriberRuleRef** | **RuleRef** | **当subscriberType=RULE时，订阅该事件的规则引用** |
| subscriberEntityRef | EntityRef | 订阅者所属的聚合根引用（仅行为订阅者有效） |
| priority | Integer | 订阅优先级，数字越小优先级越高（可选） |
| filterCondition | String | 订阅过滤条件，满足条件才触发（可选） |

### 5.4.3  事件载荷字段（PayloadField）

| 属性名 | 类型 | 说明 |
|--------|------|------|
| name | String | 字段名称 |
| type | DataType | 字段类型：String / Integer / Decimal / Date / DateTime / Boolean / Object |
| required | Boolean | 是否必填 |
| description | String | 字段业务含义 |
| sourceField | String | 来源字段路径，如 "order.orderId" |

---

## 5.5  事件顺序语义说明

| 顺序语义 | 说明 | 适用场景 |
|----------|------|----------|
| AT_LEAST_ONCE | 保证事件至少被投递一次，可能重复投递。订阅者需要实现幂等性。 | 大多数业务事件的默认选择，平衡可靠性和性能 |
| EXACTLY_ONCE | 保证事件恰好被投递一次，无重复。需要分布式事务或幂等键支持。 | 金融交易、库存扣减等对精确性要求极高的场景 |
| BEST_EFFORT | 尽力投递，不保证一定送达。性能最优，但可能丢失。 | 日志记录、统计分析等允许少量丢失的场景 |

---

## 5.6  YAML 元文件模板

```yaml
# ME 事件模型元文件 - event-model.yaml
model_type: EVENT
version: "1.0"
domain: "订单管理"

events:
  # ══════════════════════════════════════════════════════════
  # 订单相关事件
  # ══════════════════════════════════════════════════════════

  - eventId: Order.Created
    eventName: 订单已创建
    description: 订单信息录入成功后触发，用于启动后续流程
    producerBehaviorRef: Order_Create
    producerEntityRef: ORD-001
    triggerCondition: "postcondition.success == true"
    payload:
      - name: orderId
        type: String
        required: true
        description: 订单唯一标识
        sourceField: order.orderId
      - name: customerId
        type: String
        required: true
        description: 客户ID
        sourceField: order.customerId
      - name: totalAmount
        type: Decimal
        required: true
        description: 订单总额
        sourceField: order.totalAmount
      - name: createdAt
        type: DateTime
        required: true
        description: 创建时间
        sourceField: order.createdAt
    subscriberBehaviorRefs:
      - Inventory_ReserveStock
      - Notification_SendOrderConfirmation
    ordering: AT_LEAST_ONCE
    version: "1.0"

  - eventId: Order.PaymentConfirmed
    eventName: 订单支付已确认
    description: 订单支付成功后触发，启动库存扣减和发货流程
    producerBehaviorRef: Order_ConfirmPayment
    producerEntityRef: ORD-001
    triggerCondition: "postcondition.success == true"
    payload:
      - name: orderId
        type: String
        required: true
        sourceField: order.orderId
      - name: totalAmount
        type: Decimal
        required: true
        sourceField: order.totalAmount
      - name: paidAt
        type: DateTime
        required: true
        sourceField: order.paidAt
      - name: userId
        type: String
        required: true
        sourceField: order.customerId
    subscriberBehaviorRefs:
      - Inventory_DeductStock
      - Notification_SendPaymentSuccess
      - Order_TriggerFulfillment
    ordering: EXACTLY_ONCE
    version: "1.0"

  # ══════════════════════════════════════════════════════════
  # 库存相关事件
  # ══════════════════════════════════════════════════════════

  - eventId: Inventory.StockDeducted
    eventName: 库存已扣减
    description: 库存扣减成功后触发，通知下游系统
    producerBehaviorRef: Inventory_DeductStock
    producerEntityRef: INV-001
    triggerCondition: "postcondition.success == true"
    payload:
      - name: productId
        type: String
        required: true
        sourceField: stock.productId
      - name: deductedQty
        type: Integer
        required: true
        sourceField: orderItem.quantity
      - name: remainingQty
        type: Integer
        required: true
        sourceField: stock.availableQty
      - name: orderId
        type: String
        required: true
        description: 关联的订单ID
    subscriberBehaviorRefs:
      - Logistics_CreateShipment
    ordering: EXACTLY_ONCE
    version: "1.0"

  - eventId: Inventory.StockRestored
    eventName: 库存已恢复
    description: 订单取消或退款时，恢复已扣减的库存
    producerBehaviorRef: Inventory_RestoreStock
    producerEntityRef: INV-001
    triggerCondition: "postcondition.success == true"
    payload:
      - name: productId
        type: String
        required: true
        sourceField: stock.productId
      - name: restoredQty
        type: Integer
        required: true
      - name: orderId
        type: String
        required: true
    subscriberBehaviorRefs: []
    ordering: AT_LEAST_ONCE
    version: "1.0"
```

---

## 5.7  事件模型与行为模型的协作

### 5.7.1  行为模型中的事件引用

在行为模型中，行为定义仍然保留 `producedEvents` 字段，但仅包含事件ID引用：

```yaml
# M2 行为模型中的事件引用示例
behaviors:
  - id: Order_ConfirmPayment
    name: 确认订单支付
    ownerEntity: ORD-001
    behaviorType: COMMAND
    # ... 其他字段 ...
    producedEvents:
      - Order.PaymentConfirmed  # 仅引用事件ID
```

事件的完整定义（载荷、订阅者等）在事件模型中维护，避免重复和不一致。

### 5.7.2  事件驱动的行为触发

订阅者行为在行为模型中声明 `triggerType: EVENT`，并在事件模型中通过 `subscriberBehaviorRefs` 建立订阅关系：

```yaml
# M2 行为模型中的事件订阅者
behaviors:
  - id: Inventory_DeductStock
    name: 扣减库存
    ownerEntity: INV-001
    behaviorType: COMMAND
    triggerType: EVENT  # 声明为事件触发
    # ... 其他字段 ...
```

```yaml
# ME 事件模型中的订阅关系
events:
  - eventId: Order.PaymentConfirmed
    # ... 其他字段 ...
    subscriberBehaviorRefs:
      - Inventory_DeductStock  # 建立订阅关系
```

---

## 5.8  事件链的可视化表达

通过事件模型，可以清晰地可视化业务流程的事件链路：

```
订单录入流程事件链：
┌─────────────────┐
│ Order_Create    │ (行为)
└────────┬────────┘
         │ 触发
         ▼
┌─────────────────┐
│ Order.Created   │ (事件)
└────────┬────────┘
         │ 订阅
         ├──────────────────┐
         ▼                  ▼
┌──────────────────┐  ┌──────────────────┐
│ Inventory_       │  │ Notification_    │ (行为)
│ ReserveStock     │  │ SendConfirmation │
└──────────────────┘  └──────────────────┘

支付确认流程事件链：
┌─────────────────────┐
│ Order_ConfirmPayment│ (行为)
└──────────┬──────────┘
           │ 触发
           ▼
┌─────────────────────┐
│ Order.Payment       │ (事件)
│ Confirmed           │
└──────────┬──────────┘
           │ 订阅
           ├────────────────┬────────────────┐
           ▼                ▼                ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ Inventory_   │  │ Notification_│  │ Order_Trigger│ (行为)
│ DeductStock  │  │ SendPayment  │  │ Fulfillment  │
└──────┬───────┘  │ Success      │  └──────────────┘
       │          └──────────────┘
       │ 触发
       ▼
┌──────────────┐
│ Inventory.   │ (事件)
│ StockDeducted│
└──────┬───────┘
       │ 订阅
       ▼
┌──────────────┐
│ Logistics_   │ (行为)
│ CreateShipment│
└──────────────┘
```

---

## 5.9  事件模型的最佳实践

### 5.9.1  事件命名规范

- 使用过去时态：`Order.Created`（已创建）而非 `Order.Create`（创建）
- 格式：`{Entity}.{State}_{Past}`，如 `Payment.Confirmed`、`Inventory.StockDeducted`
- 避免使用动词原形或进行时

### 5.9.2  事件粒度控制

- 事件应该表达业务上有意义的状态变更，而非技术层面的细节
- 避免过细粒度：不要为每个字段变更都发布事件
- 避免过粗粒度：一个事件不应该包含多个不相关的业务含义

### 5.9.3  事件载荷设计

- 包含订阅者所需的核心业务数据，避免订阅者回查数据库
- 不要包含敏感信息（如密码、完整银行卡号）
- 保持载荷稳定，字段变更需要版本管理

### 5.9.4  订阅者独立性

- 订阅者之间应该相互独立，一个订阅者的失败不应影响其他订阅者
- 避免订阅者之间的隐式依赖和执行顺序假设
- 如果需要顺序执行，应该通过事件链串联，而非在同一事件的多个订阅者中实现

### 5.9.5  幂等性保证

- 对于 `AT_LEAST_ONCE` 和 `EXACTLY_ONCE` 语义的事件，订阅者必须实现幂等性
- 使用幂等键（如 `orderId + eventId + timestamp`）防止重复处理
- 在行为模型的补偿模型中定义幂等键构成规则

---

## 5.10  事件模型与其他模型的关系

| 关系 | 说明 |
|------|------|
| ME → M2 | 事件引用行为模型中的生产者行为和订阅者行为 |
| M2 → ME | 行为模型中的 `producedEvents` 字段引用事件模型中的事件ID |
| M4 → ME | 场景模型通过事件链编排业务流程，引用事件模型中的事件 |
| M6 → ME | 异常补偿模型中，事件失败可能触发补偿流程 |

---

# 第六章  M4 场景模型

## 5.1  双层场景体系

场景模型分为两个层次，形成层次化的业务描述：

| 层次 | 说明 |
|------|------|
| 业务流程（Business Process） | 跨多个业务用例的大粒度流程，描述完整的业务价值链，如"订单履约流程"。由多个业务用例串联或并联组成。 |
| 业务用例（Business Use Case） | 单一业务场景，对应一次完整的用户意图达成，如"客户下单"。由对象行为调用序列和事件链组成。 |

---

## 5.2  事件链编排语义

场景模型的核心是事件链编排，需要明确表达以下语义：

| 编排语义 | 说明 |
|----------|------|
| 顺序（Sequence） | A行为完成后执行B行为 |
| 并行（Parallel / AND-Split） | 同时触发多个行为，全部完成后继续 |
| 聚合等待（AND-Join） | 等待多个事件全部到达后触发下一步 |
| 选择分支（XOR-Split） | 根据条件选择一个分支执行 |
| 事件等待（Event-Based Gateway） | 等待特定事件到来再继续 |
| 超时处理（Timer Event） | 等待超时后触发补偿或替代路径 |

---

## 5.3  模型元素规范

### 5.3.1  业务用例（UseCase）

| 属性名 | 类型 | 说明 |
|--------|------|------|
| id | String | 用例唯一标识 |
| name | String | 用例名称（动宾短语，如"客户确认支付"） |
| actors | ActorRef[] | 参与的主体（引用M5） |
| primaryFlow | FlowStep[] | 主成功路径步骤列表 |
| alternativeFlows | AltFlow[] | 替代路径（条件分支） |
| exceptionFlows | ExcFlow[] | 异常路径（引用M6） |
| preconditions | String[] | 用例启动前提条件 |
| postconditions | String[] | 用例成功完成后的系统状态 |
| qualityAnnotation | QoSRef | 端到端质量要求（引用M7） |

### 5.3.2  流程步骤（FlowStep）

| 属性名 | 类型 | 说明 |
|--------|------|------|
| stepId | String | 步骤标识 |
| stepType | Enum | BEHAVIOR_CALL（行为调用）/ EVENT_EMIT（事件发布）/ EVENT_WAIT（事件等待）/ GATEWAY（网关）/ EXTERNAL_CALL（外部调用） |
| behaviorRef | BehaviorRef | 当stepType=BEHAVIOR_CALL时的行为引用 |
| gatewayType | Enum | 当stepType=GATEWAY时：AND_SPLIT / AND_JOIN / XOR_SPLIT / EVENT_GATEWAY |
| condition | String | XOR分支条件表达式 |
| timeout | Duration | 步骤超时时间，超时后触发compensationRef |
| nextSteps | StepRef[] | 后继步骤（支持多个，用于并行和分支） |

### 5.3.3  业务流程（Business Process）

| 属性名 | 类型 | 说明 |
|--------|------|------|
| id | String | 流程唯一标识 |
| name | String | 流程名称 |
| useCases | UseCaseRef[] | 组成该流程的用例列表 |
| orchestration | FlowStep[] | 用例间的编排步骤（同FlowStep规范） |
| sagaBoundary | SagaRef | 该流程对应的Saga边界（引用M6） |

---

## 5.4  YAML 元文件模板

```yaml
# M4 场景模型元文件 - scenario-model.yaml
model_type: SCENARIO
version: "1.0"
domain: "订单管理"

use_cases:
  - id: UC-ORDER-003
    name: 客户确认支付
    actors:
      - ACTOR-CUSTOMER
      - ACTOR-PAYMENT-GATEWAY
    preconditions:
      - "order.status == '待支付'"
      - "当前用户是订单所有人"
    postconditions:
      - "order.status == '已支付'"
      - "库存已完成扣减"
    primaryFlow:
      - stepId: S01
        stepType: BEHAVIOR_CALL
        behaviorRef: Order_ConfirmPayment
        timeout: PT30S
        nextSteps: [S02]
      - stepId: S02
        stepType: EVENT_WAIT
        waitForEvent: Order.PaymentConfirmed
        timeout: PT60S
        nextSteps: [S03]
        onTimeout: S_TIMEOUT
      - stepId: S03
        stepType: GATEWAY
        gatewayType: AND_SPLIT
        nextSteps: [S04, S05]
      - stepId: S04
        stepType: BEHAVIOR_CALL
        behaviorRef: Inventory_DeductStock
        nextSteps: [S06]
      - stepId: S05
        stepType: BEHAVIOR_CALL
        behaviorRef: Notification_SendPaymentSuccess
        nextSteps: [S06]
      - stepId: S06
        stepType: GATEWAY
        gatewayType: AND_JOIN
        waitFor: [S04, S05]
        nextSteps: [S07]
      - stepId: S07
        stepType: BEHAVIOR_CALL
        behaviorRef: Order_TriggerFulfillment
    alternativeFlows:
      - id: ALT-01
        triggerAt: S01
        condition: "riskRule.result == 'REVIEW'"
        steps:
          - stepId: A01
            stepType: BEHAVIOR_CALL
            behaviorRef: Order_PendingRiskReview
    exceptionFlows:
      - id: EXC-01
        triggerAt: S_TIMEOUT
        compensationRef: COMP-UC-ORDER-003
    qualityAnnotation: QA-ORDER-003

business_processes:
  - id: BP-FULFILL-001
    name: 订单履约流程
    useCases:
      - UC-ORDER-003  # 确认支付
      - UC-ORDER-004  # 发货处理
      - UC-ORDER-005  # 物流跟踪
    sagaBoundary: SAGA-FULFILL-001
```

---

# 第七章  M5 主体模型

## 6.1  设计目标

主体模型解决"谁能做什么"的问题，是行为模型与对象模型之外的独立关切。采用RBAC（基于角色的访问控制）为基础，支持ABAC（基于属性的访问控制）扩展。

> **设计决策**：权限定义不内嵌于行为模型（避免耦合），而是在行为模型中声明 `requiredPermissions`，在主体模型中定义权限的授予关系。这样角色权限的变化不影响行为模型定义。

---

## 6.2  模型层次

| 层次 | 说明 |
|------|------|
| 参与者（Actor） | 与系统交互的主体，分为：人类用户（Human）、系统账户（System）、外部系统（External） |
| 角色（Role） | 权限的集合单元，一个参与者可拥有多个角色，角色支持继承 |
| 权限（Permission） | 对特定对象或行为的操作授权，粒度到行为级别 |
| 权限组（PermissionGroup） | 权限的分组管理，便于批量授予 |
| 外部实体（ExternalEntity） | 外部系统的边界定义，包括接口契约、协议、数据契约 |

---

## 6.3  模型元素规范

### 6.3.1  参与者（Actor）

| 属性名 | 类型 | 说明 |
|--------|------|------|
| actorId | String | 主体唯一标识 |
| actorType | Enum | HUMAN / SYSTEM / EXTERNAL |
| roles | RoleRef[] | 拥有的角色列表 |
| attributes | Map | 主体属性（用于ABAC条件评估，如 department, level） |
| externalContract | Contract | 当actorType=EXTERNAL时的接口契约定义 |
| └─ protocol | Enum | 接口协议：REST / SOAP / MQ / gRPC / SFTP |
| └─ dataFormat | Enum | 数据格式：JSON / XML / CSV / Binary |
| └─ authMethod | Enum | 认证方式：API_KEY / OAuth2 / mTLS / NONE |
| └─ timeoutMs | Integer | 调用超时限制（毫秒） |
| └─ retryPolicy | RetryRef | 重试策略引用（引用M6） |

### 6.3.2  角色（Role）

| 属性名 | 类型 | 说明 |
|--------|------|------|
| roleId | String | 角色唯一标识 |
| name | String | 角色名称 |
| inheritsFrom | RoleRef[] | 继承的父角色（支持多继承） |
| permissions | PermissionRef[] | 直接授予的权限列表 |
| permissionGroups | GroupRef[] | 授予的权限组 |

### 6.3.3  权限（Permission）

| 属性名 | 类型 | 说明 |
|--------|------|------|
| permissionId | String | 权限标识，建议格式：PERM-{Domain}-{Action} |
| targetType | Enum | 授权目标类型：BEHAVIOR（行为）/ ENTITY（实体数据） |
| targetRef | Ref | 授权目标引用 |
| dataScope | Enum | 数据范围：ALL / OWN / DEPT / CUSTOM |
| abacCondition | String | ABAC条件表达式，如 `actor.dept == resource.dept` |

---

## 6.4  YAML 元文件模板

```yaml
# M5 主体模型元文件 - actor-model.yaml
model_type: ACTOR
version: "1.0"
domain: "订单管理"

actors:
  - actorId: ACTOR-CUSTOMER
    name: 客户
    actorType: HUMAN
    roles:
      - ROLE-CUSTOMER

  - actorId: ACTOR-PAYMENT-GATEWAY
    name: 支付网关
    actorType: EXTERNAL
    externalContract:
      protocol: REST
      dataFormat: JSON
      authMethod: OAuth2
      timeoutMs: 30000
      retryPolicy: RETRY-EXTERNAL-001

roles:
  - roleId: ROLE-CUSTOMER
    name: 普通客户
    permissions:
      - PERM-ORDER-CREATE
      - PERM-ORDER-PAY
      - PERM-ORDER-CANCEL-OWN

  - roleId: ROLE-ORDER-ADMIN
    name: 订单管理员
    inheritsFrom: [ROLE-CUSTOMER]
    permissions:
      - PERM-ORDER-CANCEL-ALL
      - PERM-ORDER-REFUND

permissions:
  - permissionId: PERM-ORDER-PAY
    targetType: BEHAVIOR
    targetRef: Order_ConfirmPayment
    dataScope: OWN
    abacCondition: "actor.userId == order.customerId"

  - permissionId: PERM-ORDER-CANCEL-ALL
    targetType: BEHAVIOR
    targetRef: Order_Cancel
    dataScope: ALL
```

---

# 第八章  M6 异常补偿模型

## 7.1  为什么需要独立的异常补偿模型

在EDA事件驱动架构下，长流程由多个异步步骤串联完成。当中间某个步骤失败时，传统数据库事务回滚不再适用，需要通过"补偿行为"来撤销已完成的步骤。这一复杂度需要在建模阶段明确表达，否则会成为实现阶段的隐性债务。

---

## 7.2  异常分类体系

| 异常类型 | 处理策略 |
|----------|----------|
| 业务异常（BusinessException） | 违反业务规则导致的可预期失败，如库存不足、余额不足。处理策略：直接终止并返回业务错误，通常不需要补偿。 |
| 技术异常（TechnicalException） | 系统层面的临时故障，如网络超时、数据库不可用。处理策略：重试 + 幂等保护。 |
| 业务冲突（ConflictException） | 并发操作导致的业务状态冲突，如超卖。处理策略：乐观锁冲突检测 + 重试或拒绝。 |
| 补偿失败（CompensationFailure） | 补偿操作本身失败，需人工介入。处理策略：触发人工处理流程 + 告警。 |

---

## 7.3  Saga 协调模型

对于跨多个对象/服务的长流程，采用Saga模式管理补偿：

| 概念 | 说明 |
|------|------|
| Saga定义 | 一组需要保持最终一致性的行为序列，每个步骤都有对应的补偿步骤 |
| 编排模式（Orchestration） | 由集中的Saga协调器（场景模型中定义）控制执行顺序和补偿触发 |
| 补偿顺序 | 逆序执行补偿，即最后执行的步骤最先补偿（LIFO） |
| 幂等性要求 | 所有补偿行为必须声明幂等性，支持多次安全执行 |

---

## 7.4  模型元素规范

| 属性名 | 类型 | 说明 |
|--------|------|------|
| compensationId | String | 补偿定义唯一标识 |
| targetBehavior | BehaviorRef | 被补偿的目标行为 |
| compensationBehavior | BehaviorRef | 执行补偿的行为引用（也在M2中定义） |
| exceptionTypes | Enum[] | 触发此补偿的异常类型列表 |
| retryPolicy.maxRetries | Integer | 最大重试次数（用于TechnicalException） |
| retryPolicy.backoffType | Enum | 退避策略：FIXED / EXPONENTIAL / JITTER |
| retryPolicy.initialDelayMs | Integer | 初始重试延迟（毫秒） |
| fallback | Enum | 重试耗尽后的兜底策略：MANUAL_PROCESS / DEAD_LETTER / REJECT |
| idempotencyKey | String | 幂等键构成字段表达式，确保补偿操作安全重复执行 |
| sagaRef | SagaRef | 所属Saga范围引用 |

---

## 7.5  YAML 元文件模板

```yaml
# M6 异常补偿模型元文件 - compensation-model.yaml
model_type: COMPENSATION
version: "1.0"
domain: "订单管理"

compensations:
  - compensationId: COMP-Order_ConfirmPayment
    targetBehavior: Order_ConfirmPayment
    compensationBehavior: Order_ReversePayment
    exceptionTypes: [TechnicalException, CompensationFailure]
    retryPolicy:
      maxRetries: 3
      backoffType: EXPONENTIAL
      initialDelayMs: 500
      maxDelayMs: 10000
    fallback: DEAD_LETTER
    idempotencyKey: "orderId + '_' + paymentId"

  - compensationId: COMP-Inventory_DeductStock
    targetBehavior: Inventory_DeductStock
    compensationBehavior: Inventory_RestoreStock
    exceptionTypes: [TechnicalException]
    retryPolicy:
      maxRetries: 5
      backoffType: EXPONENTIAL
      initialDelayMs: 200
    fallback: MANUAL_PROCESS
    idempotencyKey: "orderId + '_' + productId + '_' + deductionId"

sagas:
  - sagaId: SAGA-FULFILL-001
    name: 订单履约Saga
    steps:
      - stepOrder: 1
        behavior: Order_ConfirmPayment
        compensation: COMP-Order_ConfirmPayment
      - stepOrder: 2
        behavior: Inventory_DeductStock
        compensation: COMP-Inventory_DeductStock
      - stepOrder: 3
        behavior: Logistics_CreateShipment
        compensation: COMP-Logistics_CreateShipment
    compensationOrder: LIFO
    onCompensationFailure: MANUAL_PROCESS
```

---

# 第九章  M7 质量约束模型

## 8.1  设计目标

质量约束模型不独立承载功能，而是以"标注"（Annotation）的形式挂载到行为模型和场景模型上，将非功能性需求（NFR）与功能定义绑定，确保实现阶段不遗漏质量要求。

---

## 8.2  质量维度框架

| 质量维度 | 关键指标 |
|----------|----------|
| 性能（Performance） | 响应时间SLA、吞吐量要求、最大并发数 |
| 可靠性（Reliability） | 可用性目标（SLA %）、故障恢复时间（RTO/RPO） |
| 一致性（Consistency） | 数据一致性级别：强一致 / 最终一致 / 读自写一致 |
| 并发安全（Concurrency） | 并发冲突处理策略：乐观锁 / 悲观锁 / 无锁队列 |
| 可审计（Auditability） | 是否需要操作日志、变更历史追踪 |
| 幂等性（Idempotency） | 接口是否需要幂等保护及幂等键设计 |

---

## 8.3  模型元素规范（QoS Annotation）

| 属性名 | 类型 | 说明 |
|--------|------|------|
| qaId | String | 质量约束标注唯一标识 |
| appliesTo | Ref[] | 标注目标：行为引用或场景用例引用 |
| performance.maxResponseMs | Integer | 最大响应时间（毫秒） |
| performance.targetTps | Integer | 目标吞吐量（每秒事务数） |
| performance.maxConcurrency | Integer | 最大并发执行数 |
| reliability.availabilityTarget | String | 可用性目标，如 "99.9%" |
| reliability.rtoMinutes | Integer | 恢复时间目标（分钟） |
| reliability.rpoMinutes | Integer | 恢复点目标（分钟） |
| consistency.level | Enum | STRONG / EVENTUAL / READ_YOUR_WRITES |
| concurrency.strategy | Enum | OPTIMISTIC_LOCK / PESSIMISTIC_LOCK / QUEUE_SERIALIZE |
| auditRequired | Boolean | 是否需要操作审计日志 |
| idempotencyRequired | Boolean | 是否需要幂等性保护 |

---

## 8.4  YAML 元文件模板

```yaml
# M7 质量约束模型元文件 - quality-model.yaml
model_type: QUALITY
version: "1.0"
domain: "订单管理"

quality_annotations:
  - qaId: QA-ORDER-001
    name: 支付确认接口质量要求
    appliesTo:
      - Order_ConfirmPayment
    performance:
      maxResponseMs: 3000
      targetTps: 500
      maxConcurrency: 200
    reliability:
      availabilityTarget: "99.95%"
      rtoMinutes: 5
      rpoMinutes: 1
    consistency:
      level: STRONG
    concurrency:
      strategy: OPTIMISTIC_LOCK
    auditRequired: true
    idempotencyRequired: true

  - qaId: QA-ORDER-003
    name: 下单用例端到端质量要求
    appliesTo:
      - UC-ORDER-003
    performance:
      maxResponseMs: 5000   # 端到端完成时间
    reliability:
      availabilityTarget: "99.9%"
    consistency:
      level: EVENTUAL        # 库存扣减允许最终一致
    auditRequired: true
```

---

# 第十章  传统需求覆盖度分析

## 10.1  与传统需求工程的映射

| 传统需求维度 | 本框架覆盖 | 承载模型 |
|--------------|-----------|----------|
| 数据需求 / 概念模型 | ✅ 完整覆盖 | M1 对象模型 |
| 功能需求（细粒度） | ✅ 完整覆盖 | M2 行为模型 |
| 业务规则 | ✅ 完整覆盖 | M3 规则模型 |
| 事件 / 消息需求 | ✅ 完整覆盖 | ME 事件模型 |
| 业务流程 / 用例 | ✅ 完整覆盖 | M4 场景模型 |
| 访问控制 / 权限 | ✅ 完整覆盖 | M5 主体模型 |
| 异常处理 / 补偿 | ✅ 完整覆盖 | M6 异常补偿模型 |
| 非功能性需求（NFR） | ✅ 完整覆盖 | M7 质量约束模型 |
| 外部系统集成 | ✅ 完整覆盖 | M5 ExternalEntity + M2 EXTERNAL_CALL |
| 事件驱动流程 | ✅ 完整覆盖 | ME 事件模型 + M4 事件链编排 |
| 参照完整性约束 | ✅ 完整覆盖 | M1 ReferentialConstraint |
| 并发冲突处理 | ✅ 完整覆盖 | M6 ConflictException + M7 并发策略 |
| 异步处理需求 | ✅ 完整覆盖 | ME 事件模型（事件订阅机制） |
| 消息顺序保证 | ✅ 完整覆盖 | ME 事件模型（ordering字段） |
| UI交互（明确排除） | ⭕ 框架外 | 不在本方案范围内 |

---

## 10.2  框架整体覆盖评估

### 核心优势

- 语义驱动，业务与技术高度一致
- 正交分层，各模型可独立演进
- 事件模型独立，支持松耦合的事件驱动架构
- 事件链使复杂流程可视化，生产者和订阅者关系清晰
- NFR通过标注机制内嵌需求
- 异常路径与正常路径对等表达
- 外部系统边界通过ExternalEntity显式建模
- 规则和事件模型支持独立版本管理，支持热更新
- 事件载荷明确定义，避免订阅者回查数据库
- 支持多种事件顺序语义，满足不同业务场景需求

### 已知局限（框架外）

- 不覆盖UI/UX交互需求（明确排除）
- 报表/BI查询需求需独立补充
- 基础设施部署需求需独立说明
- 多语言/国际化需求框架外
- 事件存储和事件溯源的具体实现细节框架外（由实现层决定）

---

# 第十一章  实施指南与最佳实践

## 11.1  建模顺序推荐

八个模型之间存在依赖，建议按以下顺序建模，避免循环依赖：

| 阶段 | 建模对象 | 说明 |
|------|----------|------|
| ① | M1 对象模型 | 从业务词汇表开始，识别核心实体、属性和关联，先不考虑行为 |
| ② | M5 主体模型（Actor部分） | 识别系统参与者和角色，此时只定义角色，权限后续补充 |
| ③ | M3 规则模型 | 梳理独立于具体行为的业务规则，可以和业务专家协作完成 |
| ④ | M2 行为模型 | 为每个对象定义行为，引用规则，定义产生的事件ID引用 |
| ⑤ | ME 事件模型 | 定义事件的完整信息，包括生产者、订阅者、载荷，构建事件链 |
| ⑥ | M5 主体模型（权限部分） | 为行为绑定权限，定义外部实体接口契约 |
| ⑦ | M6 异常补偿模型 | 为关键行为定义补偿策略，识别Saga边界 |
| ⑧ | M4 场景模型 | 组装业务用例和流程，包含正常路径和异常分支，引用事件链 |
| ⑨ | M7 质量约束模型 | 为关键行为和场景标注NFR，与架构团队对齐 |

---

## 11.2  模型评审检查清单

### M1 对象模型评审
- [ ] 聚合边界是否清晰？每个聚合根是否代表一个完整的业务概念？
- [ ] 聚合内的子实体是否真的需要与聚合根同生共死？
- [ ] 聚合之间是否通过ID引用而非对象引用？
- [ ] 聚合不变性约束是否完整覆盖了业务规则？
- [ ] 聚合大小是否合理（子实体不超过3-5个）？
- [ ] 值对象是否真正不可变？是否通过值相等判断？
- [ ] 是否存在应该独立为聚合根的子实体（有独立生命周期）？
- [ ] 数据冗余是否合理？冗余字段是否有同步机制？
- [ ] 聚合根ID设计是否符合业务需求（UUID vs 业务编号）？

### M2 行为模型评审
- [ ] 每个行为是否真正原子化，只操作一个对象？
- [ ] 前置条件是否完整覆盖了行为可执行的业务前提？
- [ ] 后置状态变更是否完整描述了行为的全部副作用？
- [ ] 每个COMMAND行为是否定义了对应的补偿行为引用？
- [ ] producedEvents字段是否只包含事件ID引用（完整定义在事件模型中）？

### M3 规则模型评审
- [ ] 规则是否真正可复用（被2个以上行为引用才值得独立）？
- [ ] 规则表达式是否无副作用（不改变系统状态）？
- [ ] 规则版本是否独立管理，与行为版本解耦？

### ME 事件模型评审
- [ ] 每个事件是否有明确的单一生产者行为？
- [ ] 事件载荷是否包含订阅者所需的核心业务数据？
- [ ] 事件命名是否使用过去时态（如Order.Created而非Order.Create）？
- [ ] 订阅者行为之间是否相互独立，无隐式依赖？
- [ ] 对于AT_LEAST_ONCE和EXACTLY_ONCE语义，订阅者是否实现了幂等性？
- [ ] 事件链路是否清晰可追溯（生产者→事件→订阅者）？

### M4 场景模型评审
- [ ] 是否覆盖了主成功路径和所有可预期的替代路径？
- [ ] 并行执行的步骤是否定义了AND-JOIN聚合条件？
- [ ] 每个步骤是否定义了超时处理策略？
- [ ] Saga边界是否清晰，补偿顺序是否正确定义？
- [ ] 用例前置/后置条件是否与M2行为的前后置条件一致？
- [ ] 事件链编排是否与事件模型中的订阅关系一致？

### M5 主体模型评审
- [ ] 外部系统是否定义了接口契约（协议、超时、重试）？
- [ ] ABAC条件是否覆盖了数据隔离需求（如多租户）？
- [ ] 角色继承关系是否符合最小权限原则？

### M6 异常补偿模型评审
- [ ] 所有写操作行为是否都有补偿行为对应？
- [ ] 补偿行为是否声明了幂等性保证？
- [ ] 死信队列的后续处理流程是否有对应的运维规程？

### M7 质量约束模型评审
- [ ] 核心业务路径是否都有性能SLA标注？
- [ ] 一致性级别选择是否经过权衡（强一致 vs 可用性）？
- [ ] 审计要求是否覆盖了合规和安全的需要？

---

## 11.3  文件命名与版本管理建议

建议将八个模型元文件纳入版本控制（Git），采用如下目录约定：

```
models/
├── {domain}/
│   ├── m1-object-model.yaml
│   ├── m2-behavior-model.yaml
│   ├── m3-rule-model.yaml
│   ├── me-event-model.yaml
│   ├── m4-scenario-model.yaml
│   ├── m5-actor-model.yaml
│   ├── m6-compensation-model.yaml
│   └── m7-quality-model.yaml
├── shared/
│   ├── m3-shared-rules.yaml    # 跨域复用规则
│   ├── me-shared-events.yaml   # 跨域共享事件
│   └── m5-global-roles.yaml    # 全局角色定义
└── CHANGELOG.md
```

**版本管理约定**：

- 每个元文件内部维护自己的 `version` 字段
- 跨模型的破坏性变更（如实体删除、行为签名变更、事件载荷变更）需要在 `CHANGELOG.md` 中记录
- 规则模型和事件模型支持独立版本，允许热更新而不触发其他模型变更
- 事件载荷的字段变更需要特别注意向后兼容性，建议采用增量添加而非修改现有字段

---

## 11.4  工具链建议

| 工具场景 | 建议方案 |
|----------|----------|
| 模型编辑 | VS Code + YAML插件 + 自定义JSON Schema校验，保证模型文件的结构合规性 |
| 可视化 | 将YAML解析后生成PlantUML或Mermaid图表，可集成到CI/CD流水线自动生成文档。特别是事件链的可视化，可以清晰展示事件驱动流程 |
| 一致性检查 | 编写校验脚本：检查行为模型中的entityRef是否都存在于对象模型中；检查事件模型中的producerBehaviorRef和subscriberBehaviorRefs是否都存在于行为模型中 |
| 代码生成 | 基于模型元文件生成：实体类骨架、接口签名、事件常量、事件载荷DTO类、权限枚举等 |
| 变更影响分析 | 当M1实体变更时，自动分析影响的M2行为、M3规则、ME事件、M4场景；当事件载荷变更时，分析影响的所有订阅者 |
| 模型验证CI | 每次PR合并前运行模型一致性检查，防止引用悬空（dangling reference）；特别检查事件链的完整性 |
| 事件链追踪 | 开发工具支持从任意行为追踪其产生的事件链路，以及从任意事件追踪其生产者和所有订阅者 |

---

## 11.5  与实现层的映射建议

本框架是需求/设计层的语义模型，与实现技术的映射建议如下：

| 本体模型元素 | 实现层对应 |
|--------------|-----------|
| M1 聚合根 | DDD Aggregate Root / JPA Entity（主表） / Domain Model |
| M1 子实体 | JPA Entity（从表，外键关联） / Embedded Entity |
| M1 值对象 | Value Object / Embeddable / JSON字段 |
| M1 聚合不变性 | Domain Service / Aggregate Method / Validation Logic |
| M1 聚合间关联 | 外键字段（存储目标聚合根ID） / Repository查询 |
| M2 行为（COMMAND） | Application Service Method / Use Case Handler / Aggregate Method |
| M2 行为（QUERY） | Query Service / Repository Query Method / CQRS Query Handler |
| M2 行为（EVENT_HANDLER） | Event Listener / Message Consumer |
| ME 事件 | Domain Event Class / Message Topic / Event Schema |
| ME 事件载荷 | Event DTO / Message Payload Class |
| ME 生产者行为 | Event Publisher / Message Producer |
| ME 订阅者行为 | Event Subscriber / Message Consumer / Event Handler |
| ME 事件链 | Event-Driven Architecture / Message Queue / Event Bus |
| M3 规则 | Domain Service / Policy Object / Rule Engine Rule |
| M4 场景（流程） | Saga Orchestrator / Process Manager / Workflow Engine |
| M5 主体模型 | Spring Security / Casbin / OPA Policy |
| M6 异常补偿 | Saga Compensating Transaction / Dead Letter Queue |
| M7 质量标注 | SLA监控指标 / 限流配置 / 锁策略配置 |

**聚合根的实现技术选型**：

- **ORM映射**：JPA/Hibernate（Java）、Entity Framework（.NET）、TypeORM（Node.js）
- **聚合持久化**：Repository模式、Unit of Work模式
- **聚合加载策略**：Eager Loading（小聚合）、Lazy Loading（大聚合）
- **并发控制**：乐观锁（Version字段）、悲观锁（SELECT FOR UPDATE）
- **值对象存储**：Embeddable、JSON字段、字段前缀平铺

**事件模型的实现技术选型**：

- **消息中间件**：Kafka（高吞吐）、RabbitMQ（灵活路由）、Pulsar（多租户）
- **事件总线**：Spring Cloud Stream、Axon Framework、EventBus
- **事件存储**：EventStore、Kafka（作为事件日志）、数据库事件表
- **顺序保证**：Kafka分区键、RabbitMQ消息优先级、数据库事务
- **幂等性实现**：Redis去重、数据库唯一索引、幂等键表

---

# 附录  术语对照表

| 术语 | 定义 |
|------|------|
| 本体（Ontology） | 对某个领域中概念及其关系的形式化表达，借鉴自知识工程和OWL规范 |
| 聚合（Aggregate） | DDD中的核心概念，是业务完整性的边界，由聚合根、子实体和值对象组成 |
| 聚合根（Aggregate Root） | 聚合的唯一入口，对外暴露的访问点，负责维护聚合内的业务不变性 |
| 子实体（Entity） | 聚合内部有标识的对象，依赖聚合根生命周期，不能独立存在 |
| 值对象（Value Object） | 无标识的不可变对象，通过属性值判断相等性，描述聚合的某个特征 |
| 聚合不变性（Invariant） | 聚合内必须始终满足的业务规则，由聚合根负责维护 |
| 原子行为（Atomic Behavior） | 不可再分的最小行为单元，只操作单一对象，产生确定性副作用 |
| 事件（Event） | 业务状态变更的通知消息，由生产者行为触发，被订阅者行为消费 |
| 事件链（Event Chain） | 由行为产生事件、事件触发新行为形成的异步处理链路 |
| 生产者（Producer） | 触发事件的行为，每个事件有且仅有一个生产者 |
| 订阅者（Subscriber） | 消费事件的行为，一个事件可以有多个订阅者 |
| 事件载荷（Event Payload） | 事件携带的业务数据字段，供订阅者使用 |
| 事件顺序语义（Event Ordering） | 事件投递的保证级别：至少一次、恰好一次、尽力而为 |
| EDA（Event-Driven Architecture） | 事件驱动架构，通过消息事件实现组件间的松耦合 |
| Saga模式 | 分布式事务的补偿模式，将长事务分解为一系列本地事务，每步有对应补偿 |
| RBAC | 基于角色的访问控制（Role-Based Access Control） |
| ABAC | 基于属性的访问控制（Attribute-Based Access Control），比RBAC更细粒度 |
| NFR（Non-Functional Requirement） | 非功能性需求，包括性能、可靠性、安全性、可维护性等 |
| QoS Annotation | 质量服务标注，将NFR以标注形式绑定到功能元素上 |
| 幂等性（Idempotency） | 操作执行一次与执行多次产生相同结果的特性，分布式系统的关键要求 |
| 幂等键（Idempotency Key） | 用于保证操作幂等性的唯一标识，通常由业务字段组合而成 |
| RTO（Recovery Time Objective） | 恢复时间目标，系统故障后允许的最长恢复时间 |
| RPO（Recovery Point Objective） | 恢复点目标，系统故障后允许的最大数据丢失时间窗口 |
| AND-Join | 流程编排中等待多个并行分支全部完成的聚合节点 |
| XOR-Split | 流程编排中根据条件选择唯一一个分支执行的网关节点 |
| Dead Letter | 无法处理的消息被放入死信队列，等待人工干预或后续处理 |
| LIFO | 后进先出（Last In First Out），Saga补偿的默认执行顺序 |
| Dangling Reference | 悬空引用，模型中引用了不存在的目标，需通过CI校验防止 |
| OWL | 网络本体语言（Web Ontology Language），W3C标准的语义本体描述语言 |
| 消息中间件（Message Broker） | 实现事件传递的基础设施，如Kafka、RabbitMQ、Pulsar |
| 事件总线（Event Bus） | 应用内的事件分发机制，支持发布-订阅模式 |
| 事件溯源（Event Sourcing） | 将所有状态变更存储为事件序列，而非仅存储当前状态 |
| 数据冗余（Data Redundancy） | 在聚合内复制其他聚合的关键信息，避免跨聚合查询 |
| 最终一致性（Eventual Consistency） | 分布式系统中，数据在一段时间后达到一致状态 |
| 事务边界（Transaction Boundary） | 一个事务的范围，在DDD中通常是一个聚合根 |

---

*© 2026  Ontology-Driven Software Modeling Framework  v1.0*
