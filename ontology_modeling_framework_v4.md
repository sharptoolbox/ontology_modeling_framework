# 本体驱动的软件建模方案
**Ontology-Driven Software Modeling Framework**

> 完整建模规范 · 八大模型元文件 · 实施指南
> 版本 4.0 | 2026年7月
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
8. [M6 流程模型](#第八章--m6-流程模型)
9. [M7 查询统计与报表模型](#第九章--m7-查询统计与报表模型)
10. [传统需求覆盖度分析](#第十章--传统需求覆盖度分析)
11. [实施指南与最佳实践](#第十一章--实施指南与最佳实践)
12. [附录：术语对照表](#附录--术语对照表)

---

# 第一章  方案概述与设计哲学

## 1.1  框架定位

本方案面向具有明确领域边界的中大型业务软件，以"本体"（Ontology）作为系统设计的核心隐喻，将业务世界中的"存在（What）"、"行为（How）"、"规则（Why）"、"事件协同（When/Event Collaboration）"、"流程流转（Workflow）"和"查询表达（Read/Report）"分离建模，并通过事件驱动架构（EDA）与流程编排实现松耦合协作。

与传统需求分析方法（如UML用例图、功能规格说明书）相比，本框架具有以下核心差异：

- **语义驱动**：模型元素具有明确的业务语义，而非纯粹的技术描述
- **正交分解**：对象、行为、规则、事件、事件协同场景、主体、流程和查询报表各自承担单一职责，可独立演进
- **事件解耦**：通过事件链替代长事务，避免强耦合的同步调用链
- **流程分离**：端到端协同流和审批流独立建模，不与事件协同场景混合
- **读写分离**：跨对象查询、统计和固定报表独立建模，不把复杂查询结构塞入原子行为或业务规则
- **可追溯性**：每个实现单元都可以溯源到具体的本体模型定义

---

## 1.2  八大本体模型元文件总览

在对象、行为和规则等基础模型之上，独立定义事件、事件协同场景、权限主体、业务流程和查询报表，形成八模型体系：

| 编号 | 模型名称 | 核心职责 |
|------|----------|----------|
| **M1** | 对象模型 Object Model | 定义数据实体、实体属性、实体间的双向关联与参照完整性约束 |
| **M2** | 行为模型 Behavior Model | 定义对象的原子行为方法、触发条件、前置后置约束 |
| **M3** | 规则模型 Rule Model | 定义跨对象、跨行为、事件驱动或需要独立复用的业务决策规则 |
| **ME** | 事件模型 Event Model | 定义业务事件、事件生产者、事件订阅者及事件链路关系 |
| **M4** | 场景模型 Scenario Model | 定义跨对象、基于事件解耦的业务协同场景，表达行为、事实事件、规则和后续行为之间的语义链 |
| **M5** | 主体模型 Actor Model | 定义系统参与者、角色、权限边界及行为执行授权关系 |
| **M6** | 流程模型 Flow Model | 定义端到端业务协同流和审批流，描述角色任务、系统活动、网关、条件和子流程调用 |
| **M7** | 查询统计与报表模型 Query & Report Model | 定义跨对象查询、统计分析和固定业务报表的来源、关联、条件、结果列、聚合及参考SQL |

---

## 1.3  模型间关系全景

八个模型之间的依赖与引用关系如下：

```
M6 流程模型      依赖   M1 对象模型 + M2 行为模型 + M3 规则模型 + ME 事件模型 + M4 场景模型 + M5 角色模型
M7 查询报表模型  依赖   M1 对象模型 + M2 查询行为（一对一绑定）
M4 场景模型      依赖   M1 对象模型 + M2 行为模型 + M3 规则模型 + ME 事件模型
M2 行为模型      依赖   M1 对象模型  +  M3 规则模型  +  M5 主体模型
ME 事件模型      依赖   M2 行为模型 + M3 规则模型（生产者与订阅者）
M3 规则模型      引用   M1 对象模型（只读）
M5 主体模型      引用   M1 对象模型  +  M2 行为模型（权限绑定）
```

**关键设计决策**：

1. 事件模型从行为模型中独立出来，事件通过产生者字段和 `subscribers` 建立与行为、规则的引用关系。
2. M4只描述真实存在的跨对象事件协同，不描述人工任务、审批人、通用顺序流程或端到端业务阶段。
3. M6统一承载端到端业务协同流和审批流。人工活动只能通过 `roleRef` 引用M5中的 `roleId`，不得直接引用 `actorId` 或填写自由文本参与人。
4. M6可以调用M2行为、M3规则、M4事件协同场景及其他M6子流程，但不得复制这些模型中的定义。
5. M7只与M1、M2发生直接关系。M7定义查询内容，M2定义执行该查询报表对象的原子行为，两者严格一对一；M7不直接引用M3、ME、M4、M5或M6。

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
| type | DataType | 类型：String / Integer / Decimal / Date / DateTime / Boolean / Enum / Money / ValueObject / AggregateRootRef / DictionaryRef |
| required | Boolean | 是否必填 |
| unique | Boolean | 是否唯一（仅对聚合根属性有效） |
| defaultValue | Any | 默认值 |
| enumValues | String[] | 当type=Enum时的枚举值列表 |
| valueObjectRef | String | 当type=ValueObject时引用的值对象名称 |
| targetAggregate | AggregateRef | 当type=AggregateRootRef时必填，指向被引用聚合根的ID |
| dictionaryRef | DictionaryTypeRef | 当type=DictionaryRef时必填，包含dictionaryId和typeCode |
| refRules | AttributeRule[] | 可选；只依赖当前属性值、可在该属性内部完整定义的扩展规则 |

`AggregateRootRef`表示“存储另一个聚合根ID的标量属性”，而不是内嵌对象或数据库外键对象。该类型在代码生成时仍可映射为目标聚合根ID的实际标量类型，但在本体层必须通过`targetAggregate`显式声明目标，禁止只在`description`中写“引用某聚合根ID”。建模工具应对该类型使用独立图标、聚合选择器并执行悬空引用校验。

`DictionaryRef`表示属性值来自M1对象模型中的数据字典类型。业务数据保存字典项稳定的`code`，界面显示`label`。`DictionaryRef`不得同时声明`enumValues`，也不生成聚合关联。

#### 属性扩展规则（AttributeRule / refRules）

`refRules`用于定义只依赖当前属性值即可判断、无需读取其他属性、其他对象或行为上下文的局部规则。此类规则属于M1对象模型，不得重复放入M3规则模型。

| 属性名 | 类型 | 说明 |
|--------|------|------|
| name | String | 属性规则名称，在当前属性内唯一 |
| description | String | 规则业务说明 |
| expression | String | 属性规则表达式，使用`value`表示当前属性值 |
| violationMessage | String | 规则不满足时返回的业务提示 |
| enforcedAt | Enum | ON_CREATE / ON_UPDATE / ON_DELETE / ALWAYS |

属性规则分层原则：

1. 可以使用内置字段表达的约束，优先使用内置字段，不重复写入`refRules`。例如“必须填写”使用`required: true`，“唯一”使用`unique: true`。
2. 无法由内置字段表达、但只依赖当前属性值的规则使用`refRules`。例如金额必须大于0、名称长度不超过100、日期不能晚于当前日期。
3. 依赖同一对象多个属性或聚合内部子实体的规则使用聚合`invariants`。例如开始日期不能晚于结束日期、配件明细金额合计必须等于设备总价。
4. 依赖其他独立对象、多个行为结果、事件、外部数据或需要被多个行为复用的规则，才进入M3规则模型。

示例：

```yaml
attributes:
  - name: contractAmount
    label: 合同金额
    type: Decimal
    required: true
    refRules:
      - name: 合同金额必须为正数
        description: 合同金额必须大于0
        expression: "value > 0"
        violationMessage: 合同金额必须大于0
        enforcedAt: ALWAYS

  - name: contractName
    label: 合同名称
    type: String
    required: true
    refRules:
      - name: 合同名称长度限制
        description: 合同名称最多100个字符
        expression: "LENGTH(value) <= 100"
        violationMessage: 合同名称不能超过100个字符
        enforcedAt: ALWAYS
```

### 2.3.5 数据字典（Data Dictionary）

数据字典是对象模型中的引用数据定义，不是值对象、聚合根或运行时业务实体。同一个数据字典对象可以维护多个平级字典类型；当前版本不支持字典项父子层级。

#### 数据字典对象

| 属性名 | 类型 | 说明 |
|--------|------|------|
| id | String | 数据字典对象唯一ID |
| name | String | 数据字典对象名称 |
| description | String | 可选说明 |
| types | DictionaryType[] | 字典类型集合 |

#### 字典类型

| 属性名 | 类型 | 说明 |
|--------|------|------|
| typeCode | String | 数据字典对象内唯一的稳定编码 |
| typeName | String | 中文显示名称 |
| description | String | 可选说明 |
| items | DictionaryItem[] | 平级字典项，不允许parentCode |

#### 字典项

| 属性名 | 类型 | 说明 |
|--------|------|------|
| code | String | 类型内唯一的稳定业务存储值 |
| label | String | 界面显示文本，可修改但不影响已有业务数据 |
| enabled | Boolean | 是否可被新增数据选择；已使用项应停用而非删除 |
| sortOrder | Integer | 显示顺序 |
| description | String | 可选说明 |

#### 字典引用约束

- `dictionaryRef.dictionaryId`必须指向存在的数据字典对象。
- `dictionaryRef.typeCode`必须指向该对象内存在的字典类型。
- `defaultValue`如存在，必须是该类型中已启用字典项的`code`。
- 数据字典ID、类型编码和字典项编码被引用后，重命名或删除必须进行影响分析。
- 数据字典项当前为平级结构，规范中不定义`parentCode`或其他层级字段。
- 客户状态等参与生命周期、行为条件或规则表达式的字典项，修改时必须同步校验所有引用。

### 2.3.6  聚合不变性约束（Invariant）

聚合不变性是必须始终满足的业务规则，由聚合根负责维护：

| 属性名 | 类型 | 说明 |
|--------|------|------|
| name | String | 约束名称 |
| expression | String | 约束表达式，可跨聚合内的子实体和值对象 |
| violationMessage | String | 违反约束时的业务错误消息 |
| enforcedAt | Enum | 执行时机：ON_CREATE / ON_UPDATE / ON_DELETE / ALWAYS |

### 2.3.7  聚合间关联（Aggregate Association）

聚合之间通过ID引用建立关联，而非对象引用：

| 属性名 | 类型 | 说明 |
|--------|------|------|
| id | String | 关联唯一标识 |
| sourceAggregate | AggregateRef | 来源聚合根 |
| targetAggregate | AggregateRef | 目标聚合根 |
| associationType | Enum | REFERENCE（引用）/ DEPENDENCY（依赖） |
| sourceRole | String | 来源端角色名 |
| targetRole | String | 目标端角色名 |
| cardinality | Enum | 基数关系：ONE_TO_ONE / ONE_TO_MANY / MANY_TO_ONE / MANY_TO_MANY |
| referenceField | String | 存储目标聚合根ID的字段名 |

---

## 2.4  YAML 元文件模板

### 2.4.1  聚合根建模示例（订单聚合）

```yaml
# M1 对象模型元文件 - m1-object-model.yaml
model_type: OBJECT
version: "2.1"
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
        type: AggregateRootRef
        targetAggregate: AGG-CUSTOMER-001
        required: true
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
            type: AggregateRootRef
            targetAggregate: AGG-PRODUCT-001
            required: true
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
data_dictionaries:
  - id: DICT-ORDER-BASE
    name: 订单基础数据字典
    types:
      - typeCode: ORDER_CHANNEL
        typeName: 下单渠道
        items:
          - code: ONLINE
            label: 线上
            enabled: true
            sortOrder: 10
          - code: OFFLINE
            label: 线下
            enabled: true
            sortOrder: 20

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

行为模型定义对象能够执行的原子行为方法。每个行为是单一对象发出的、不可再分的核心操作单元。跨对象事件协同由M4场景模型表达，端到端协同和人工审批由M6流程模型编排，而非在行为模型内部内嵌。

> **关键设计原则**：行为模型的核心约束是"原子性"——每个行为方法只做一件事，操作一个对象，产生确定性的状态变更。复杂判断通过规则模型注入；事件驱动的跨对象协同通过M4表达；端到端和审批流转通过M6表达。

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
| queryReportRef | QueryReportRef | 当behaviorType=QUERY且行为执行M7查询统计或报表对象时填写；与M7.behaviorRef严格一对一 |

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
# M2 行为模型元文件 - m2-behavior-model.yaml
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

  - id: Contract_QueryExecutionAnalysis
    name: 查询合同执行情况分析
    ownerEntity: OBJ-CONTRACT-001
    behaviorType: QUERY
    triggerType: USER_ACTION
    preconditions: []
    postconditions: []
    appliedRules: []
    requiredPermissions:
      - PERM-CONTRACT-ANALYSIS
    producedEvents: []
    queryReportRef: QR-CONTRACT-EXECUTION-001
```

> **说明**：`producedEvents` 字段仅包含事件ID引用。事件的完整定义（生产者、订阅者、载荷等）在 `me-event-model.yaml` 中维护。

> **M7绑定说明**：普通单聚合详情或列表查询可以不填写 `queryReportRef`。只有查询行为对应一个正式定义的跨对象查询、统计分析或固定报表对象时，才与M7建立一对一引用。M2继续通过 `requiredPermissions` 关联M5，M7不直接维护权限。

---

# 第四章  M3 规则模型

## 4.1  设计目标与边界

规则模型专注于跨对象、跨行为、事件驱动或需要独立复用的解耦业务规则，是从对象局部约束和行为逻辑中分离出来的独立关切。其核心价值在于：同一规则可以被多个行为或事件链引用，规则变更不影响对象结构和行为定义。

> **重要边界说明**：能够在单个属性内部定义清楚的规则不进入M3。必填、唯一、类型、枚举和数据字典约束直接使用M1属性字段；只依赖当前属性值的扩展表达式使用属性`refRules`；依赖同一对象多个属性或聚合内部子实体的规则使用聚合`invariants`。M3只处理超出单个对象内部边界的业务判断、计算、推导和事件驱动决策。

### 4.1.1  M1局部规则与M3规则的判定

| 规则特征 | 归属位置 | 示例 |
|----------|----------|------|
| 属性内置约束 | M1 Attribute字段 | 必填、唯一、枚举、字典引用 |
| 只读取当前属性值 | M1 Attribute.refRules | 金额大于0、名称长度不超过100 |
| 读取同一对象多个属性或聚合内部子实体 | M1 Aggregate.invariants | 开始日期不晚于结束日期、明细合计等于总额 |
| 读取两个或多个独立对象 | M3 Rule | 累计收款金额达到合同总金额 |
| 依赖行为执行结果或被多个行为复用 | M3 Rule | 合同关闭资格校验、统一风险评估 |
| 订阅事件并决定后续行为 | M3 EVENT_DRIVEN Rule | 收款事件触发合同关闭校验 |
| 调用外部规则引擎或外部数据决策 | M3 Rule | 信用风险评分、合规名单校验 |

判断顺序：

```text
内置字段能表达？
-> 是：使用属性内置字段
-> 否：是否只依赖当前属性值？
   -> 是：使用Attribute.refRules
   -> 否：是否只依赖同一聚合内部数据？
      -> 是：使用Aggregate.invariants
      -> 否：进入M3规则模型
```

> **重大扩展**：规则不再只是被动的验证器或计算器，而是可以作为事件驱动架构中的主动参与者。规则可以订阅事实事件、执行业务逻辑判断，并在满足条件时触发后续行为；只有规则执行本身又形成了一个新的、独立的业务事实时，才继续产生新事件。

> **强制语义边界**：事件、规则、行为不可合并表达。事件只回答“刚刚发生了什么”；规则只回答“条件是否满足”；行为只回答“接下来改变什么状态”。例如：`收款录入行为 -> 合同新增一笔收款事件 -> 合同关闭校验规则 -> 合同关闭行为`。禁止把“所有开票均已收款且金额相等时关闭合同”写成事件说明，这段内容必须拆为规则条件和后续行为。

---

## 4.2  规则分类体系

| 规则类型 | 说明 | 触发方式 |
|----------|------|----------|
| 验证规则（Validation Rule） | 对跨属性、跨对象或行为上下文执行验证，返回 true/false，不改变状态；单属性局部验证不进入M3 | 被行为调用 |
| 计算规则（Calculation Rule） | 根据输入参数计算并返回结果值，如计算运费、折扣金额 | 被行为调用 |
| 推导规则（Derivation Rule） | 基于已知属性推导出其他属性值，如根据级别推导权限范围 | 被行为调用 |
| 转换规则（Transformation Rule） | 将一种数据格式转换为另一种，适用于外部系统集成场景 | 被行为调用 |
| 风控规则（Risk Rule） | 评估业务风险，返回风险等级或通过/拒绝决策（可接入外部规则引擎） | 被行为调用 |
| **事件驱动规则（Event-Driven Rule）** | **订阅事实事件，执行条件判断，满足条件时触发行为；必要时也可产生另一个独立事实事件** | **事件触发** |

### 4.2.1  事件驱动规则的特殊性

事件驱动规则是规则模型的重要扩展，具有以下特征：

- **主动性**：不是被动等待行为调用，而是主动订阅事件
- **决策性**：作为事件链中的决策节点，判断是否继续传播
- **行为触发**：满足条件时通过 `triggeredBehaviors` 触发一个或多个后续行为
- **可选事件生产**：只有规则执行后确实形成新的业务事实时才使用 `producedEvents`
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
| outputType | DataType | 返回值类型；事件驱动规则通常为 Boolean 或结构化判断结果 |
| expression | String | 规则表达式（支持伪代码或DSL，不限定具体语言） |
| reusedBy | BehaviorRef[] | 引用本规则的行为列表（反向追踪） |
| externalEngine | String | 若委托外部规则引擎，填写引擎名称（如 Drools, OPA） |
| version | String | 规则版本，支持规则的独立版本管理 |

### 4.3.2  事件驱动规则专属属性

| 属性名 | 类型 | 说明 |
|--------|------|------|
| triggerType | Enum | EVENT（标识为事件触发型规则） |
| subscribedEvents | EventRef[] | 订阅的事件ID列表，规则监听这些事件 |
| triggeredBehaviors | BehaviorRef[] | 规则判断满足后直接触发的行为列表 |
| producedEvents | EventRef[] | 可选；规则执行后新形成的独立事实事件列表，不得用它代替后续行为 |
| eventTriggerCondition | String | 仅当存在producedEvents时使用，描述新事实事件的产生条件 |
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
# M3 规则模型元文件 - m3-rule-model.yaml
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
# M3 规则模型元文件 - m3-rule-model.yaml（续）
rules:
  # ══════════════════════════════════════════════════════════
  # 事件驱动规则
  # ══════════════════════════════════════════════════════════
  
  - id: RULE-DELIVERY-001
    name: 配送条件检查规则
    ruleType: EVENT_DRIVEN
    description: 监听合同激活事实，判断是否满足创建配送单的条件
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
    triggeredBehaviors:
      - Delivery_CreateOrder
    
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
    
    triggeredBehaviors:
      - Inventory_CreateRestockRequest
    
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
    
    triggeredBehaviors:
      - Order_CreateManualAudit
    
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

3. 规则判断
   规则引擎 → 执行 expression → 返回是否满足以及必要的计算结果

4. 后续行为触发
   规则满足 → 调用 triggeredBehaviors 中的行为 → 行为负责改变对象状态

5. 可选的新事实发布
   只有后续处理又形成独立业务事实时，才由对应行为或规则产生新事件
```

**示例流程**：
```
合同管理员.激活合同行为
  ↓ 产生
Contract.Activated 事件
  ↓ 订阅
RULE-DELIVERY-001 配送条件检查规则
  ↓ 判断：合同生效 AND 配送员空闲
  ↓ 满足条件，触发
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
- **事件链追踪**：记录事件的完整传播路径（典型为行为→事件→规则→行为；只有出现新事实时才继续接事件）
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

> **重大扩展**：事件的生产者和消费者不再局限于行为，规则也可以作为事件的生产者和消费者。最常见闭环是“行为→事实事件→规则→行为”；只有规则或后续行为又形成新事实时才继续接入另一个事件。

### 5.1.1  事件事实性约束（强制）

1. 事件名称使用已完成事实，如“合同新增一笔收款”“合同已关闭”，不使用“合同满足关闭条件”“合同需要关闭”。
2. `description` 只解释该事实本身，不得包含资格判断、规则公式或后续处理结论。
3. `triggerCondition` 只允许描述生产行为成功或源对象状态已经变化，例如 `postcondition.success == true`。
4. 每个事件必须有一个产生者。产生者由 `producerEntityRef` 加 `producerBehaviorRef` 或 `producerRuleRef` 组成。
5. 每个具有下游影响的事件必须有一个或多个消费者。消费者可以是规则或行为；规则消费者必须与M3的 `subscribedEvents` 双向一致。
6. 在浏览树中，任何事件固定展开为两个逻辑子节点：`产生者` 和 `消费者`；消费者节点下允许多条记录。载荷仍属于事件详情，不作为第三个逻辑子节点。

### 5.1.2  跨对象CUD事件识别原则（强制）

在业务需求或业务功能描述中，当一个业务功能对当前源对象完成新增、修改、删除、归档、状态变更或提交后，如果该操作结果还需要驱动另一个独立业务对象执行新增、修改、删除、归档、状态流转或流程启动，则必须识别为事件驱动关系。

这里的“另一个独立业务对象”包括另一个聚合根及其属性、子实体和值对象。只要同一个业务功能原本准备在完成源对象操作后，继续直接新增或变更另一个聚合根范围内的数据，就必须拆成“源对象行为 -> 事实事件 -> 下游订阅者”的解耦结构，禁止由源对象行为直接跨聚合写入目标对象。

标准拆分结构为：

```text
源对象行为
-> 源对象产生事实事件
-> 消费规则或消费行为
-> 目标对象执行CUD或状态变更行为
```

识别时必须满足以下条件：

1. 源业务功能已经对源对象完成CUD或状态变更。
2. 后续还需要操作另一个独立业务对象。
3. 后续操作包括新增、修改、删除、归档、状态流转或流程启动。
4. 源对象操作与目标对象操作属于不同对象职责或事务边界。
5. 目标操作由源对象已经发生的业务事实触发，而不是用户发起的另一次独立操作。

消费者的选择规则：

- 如果下游操作需要先判断业务资格、金额、数量、状态或其他条件，由规则消费事件，再由规则通过 `triggeredBehaviors` 触发目标行为。
- 如果下游操作在事实发生后无条件执行，可以由目标行为直接消费事件。
- 同一个事件可以有多个相互独立的消费者。
- 事件订阅者只能是行为或规则：行为订阅表示无条件进入目标对象操作；规则订阅表示先判定条件，满足后再触发一个或多个目标行为。

以下情况通常不识别为事件：

- 只查询另一个对象，没有导致该对象发生CUD或状态变化。
- 只修改当前对象自身的一个或多个属性。
- 修改同一聚合根内部、与主对象同生共死且可在同一事务内完成的从属子实体。
- 只执行数据格式转换、页面展示或业务校验，没有产生后续对象变化。

> **判断口诀**：一个对象完成变化后，是否还要因为这个事实去改变另一个独立对象？如果是，则识别事件；如果只是当前对象内部完成操作，则不识别事件。

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
- 规则（Rule）：规则执行后形成新的简单业务事实时触发事件；普通判断满足后应直接触发行为
```

### 5.3.2  多订阅者支持

一个事件可以被多个行为或规则订阅，实现一对多的事件广播模式。订阅者之间相互独立，互不影响。

```
订阅者类型：
- 行为（Behavior）：接收事件后执行业务操作
- 规则（Rule）：接收事件后执行条件判断，满足时触发后续行为，必要时产生新的独立事实事件
```

### 5.3.3  事件链的传递模式

#### 模式1：行为 → 事件 → 行为
```
订单支付行为 --触发--> Order.PaymentConfirmed事件 --订阅--> 库存扣减行为
```

#### 模式2：行为 → 事件 → 规则 → 行为
```
收款录入行为 --触发--> Contract.PaymentRecorded事件
                    --订阅--> 合同关闭校验规则
                              --满足--> 合同关闭行为
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

### 5.3.5  案例一：合同废弃后归档关联业务信息

业务需求：

> 合同废弃后，需要归档该合同对应的开票信息和收款信息。

该需求中，合同、合同开票和合同收款属于不同的独立业务对象。合同状态变更后继续驱动开票对象和收款对象执行归档，因此必须使用事件衔接：

```text
合同废弃行为
-> “合同已废弃”事件
   -> 开票信息归档行为
   -> 收款信息归档行为
```

模型职责：

- 产生对象：合同。
- 产生行为：合同废弃。
- 事实事件：合同已废弃。
- 消费者一：开票信息归档行为。
- 消费者二：收款信息归档行为。

如果所有开票和收款信息都必须无条件归档，两个归档行为可以直接消费事件。如果只有满足特定状态的记录才能归档，应分别增加归档资格规则，形成“事件 -> 规则 -> 归档行为”。

### 5.3.6  案例二：合同提交后启动M6审批流程

业务需求：

> 合同提交保存成功后，自动启动合同审批流程。

合同提交修改合同对象，M6审批流程属于独立流程实例，二者属于不同事务边界，因此使用事件衔接：

```text
合同提交行为
-> “合同已提交”事件
-> M6合同审批流程启动行为
```

模型职责：

- 产生对象：合同。
- 产生行为：合同提交。
- 事实事件：合同已提交。
- 消费对象：流程实例。
- 消费行为：启动 `FLOW-CONTRACT-APPROVAL-001` 审批流程。

如果流程启动前需要根据合同金额、合同类型或组织策略选择审批流程，则增加审批流程匹配规则：

```text
合同提交行为
-> “合同已提交”事件
-> 审批流程匹配规则
-> M6合同审批流程启动行为
```

“合同金额达到多少时使用哪类审批流程”属于规则或M6流程网关条件，不得写入“合同已提交”事件。具体财务经理、总经理审批活动及其角色分配只在M6中定义。

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
| triggerCondition | String | 只描述生产行为成功或源对象状态已变化，不得包含业务资格判断 |
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
# ME 事件模型元文件 - me-event-model.yaml
model_type: EVENT
version: "1.0"
domain: "订单管理"

events:
  # ══════════════════════════════════════════════════════════
  # 订单相关事件
  # ══════════════════════════════════════════════════════════

  - eventId: Order.Created
    eventName: 订单已创建
    description: 订单信息录入成功后形成的业务事实
    producerType: BEHAVIOR
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
    subscribers:
      - subscriberType: BEHAVIOR
        subscriberBehaviorRef: Inventory_ReserveStock
        subscriberEntityRef: INV-001
        priority: 1
      - subscriberType: BEHAVIOR
        subscriberBehaviorRef: Notification_SendOrderConfirmation
        subscriberEntityRef: OBJ-NOTIFICATION
        priority: 2
    ordering: AT_LEAST_ONCE
    version: "1.0"

  - eventId: Order.PaymentConfirmed
    eventName: 订单支付已确认
    description: 订单支付确认行为成功后形成的业务事实
    producerType: BEHAVIOR
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
    subscribers:
      - subscriberType: RULE
        subscriberRuleRef: RULE-ORDER-FULFILLMENT-CHECK
        priority: 1
    ordering: EXACTLY_ONCE
    version: "1.0"

  # ══════════════════════════════════════════════════════════
  # 库存相关事件
  # ══════════════════════════════════════════════════════════

  - eventId: Inventory.StockDeducted
    eventName: 库存已扣减
    description: 库存扣减行为成功后形成的业务事实
    producerType: BEHAVIOR
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
    subscribers:
      - subscriberType: BEHAVIOR
        subscriberBehaviorRef: Logistics_CreateShipment
        subscriberEntityRef: OBJ-LOGISTICS
        priority: 1
    ordering: EXACTLY_ONCE
    version: "1.0"

  - eventId: Inventory.StockRestored
    eventName: 库存已恢复
    description: 订单取消或退款时，恢复已扣减的库存
    producerType: BEHAVIOR
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
    subscribers: []
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

消费者行为在行为模型中声明 `triggerType: EVENT`，并在事件模型中通过 `subscribers` 建立消费关系：

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
    subscribers:
      - subscriberType: BEHAVIOR
        subscriberBehaviorRef: Inventory_DeductStock
        subscriberEntityRef: INV-001
        priority: 1
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
- 避免过细粒度：同一聚合内部的普通字段变更不要逐字段发布事件；但该变化需要驱动另一个独立聚合执行CUD或状态变化时，必须发布一个能够表达源对象业务事实的事件
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
- 幂等键及处理策略属于实现层约束，不在当前八模型中单独建模

---

## 5.10  事件模型与其他模型的关系

| 关系 | 说明 |
|------|------|
| ME → M2 | 事件引用行为模型中的生产者行为和订阅者行为 |
| ME → M3 | 事件引用规则模型中的生产者规则或订阅者规则；规则可在判定满足后触发后续行为 |
| M2 → ME | 行为模型中的 `producedEvents` 字段引用事件模型中的事件ID |
| M4 → ME | 场景模型引用事实事件，表达跨对象事件协同链 |
| M6 → ME | 流程可通过启动事件或等待事件衔接异步业务，但不重复定义事件 |

---

# 第六章  M4 场景模型

## 6.1  设计目标与边界

M4场景模型只描述**跨对象、基于事件解耦的业务协同场景**。它回答的是：一个对象完成状态变化后，如何通过事实事件通知规则或行为，并最终驱动另一个独立对象发生变化。

M4不再承载端到端业务流程、人工任务、审批人、角色泳道、通用网关、退回或驳回路径。这些内容统一进入M6流程模型。

只有存在真实跨对象影响和事件解耦时才定义M4场景。项目可以没有事件协同场景，不得为了满足数量要求虚构场景。

标准语义链为：

```text
源对象行为
-> 已发生的事实事件
-> 消费规则或消费行为
-> 目标对象行为
```

存在资格、金额、数量、状态或阈值判断时，必须使用：

```text
BEHAVIOR_CALL -> EVENT_EMIT -> RULE_EVALUATE -> BEHAVIOR_CALL
```

---

## 6.2  场景与流程的职责分界

| 判断问题 | M4 场景模型 | M6 流程模型 |
|----------|-------------|-------------|
| 核心目的 | 描述跨对象事件协同的业务语义 | 描述业务从开始到结束如何流转 |
| 主要元素 | 行为、事件、规则、目标行为 | 角色任务、系统活动、网关、子流程、场景调用 |
| 人工参与者 | 不定义人工任务参与人 | 人工活动必须引用M5角色 |
| 条件判断 | 通过M3规则节点表达 | 可引用M3规则或使用流程条件表达式 |
| 端到端流程 | 不承载 | 由 `COLLABORATION` 流程承载 |
| 审批流程 | 不承载 | 由 `APPROVAL` 流程承载 |

当M6流程执行到一个跨对象事件协同片段时，应通过 `SCENARIO_CALL` 引用M4场景，而不是把M4中的行为、事件和规则链复制到流程文件中。

---

## 6.3  模型元素规范

### 6.3.1  事件协同场景（Event Collaboration Scenario）

| 属性名 | 类型 | 说明 |
|--------|------|------|
| id | String | 场景唯一标识，建议格式 `SCN-{DOMAIN}-{NNN}` |
| name | String | 场景名称，如"收款完成驱动合同关闭" |
| description | String | 场景业务目的及跨对象影响说明 |
| sourceObjectRef | AggregateRef | 事件源对象，引用M1聚合根 |
| targetObjectRefs | AggregateRef[] | 被事件影响的目标对象，引用M1聚合根 |
| triggerEventRef | EventRef | 启动该协同场景的事实事件 |
| preconditions | String[] | 场景启动前提条件 |
| postconditions | String[] | 场景成功完成后的整体业务状态 |
| steps | ScenarioStep[] | 行为、事件和规则组成的事件协同链 |

### 6.3.2  场景步骤（ScenarioStep）

| 属性名 | 类型 | 说明 |
|--------|------|------|
| stepId | String | 步骤标识 |
| stepType | Enum | BEHAVIOR_CALL（行为调用）/ EVENT_EMIT（事件发布）/ RULE_EVALUATE（规则评估） |
| behaviorRef | BehaviorRef | 当stepType=BEHAVIOR_CALL时引用已存在的M2行为 |
| eventRef | EventRef | 当stepType=EVENT_EMIT时引用已存在的ME事实事件 |
| ruleRef | RuleRef | 当stepType=RULE_EVALUATE时引用已存在的M3规则 |
| nextSteps | StepRef[] | 后继步骤；允许一个事件连接多个相互独立的消费者 |

M4不允许 `GATEWAY`、`USER_TASK`、`APPROVAL_TASK`、`SUB_FLOW_CALL` 或角色分配。业务条件必须进入M3规则；流程分支必须进入M6。

## 6.4  YAML 元文件模板

```yaml
# M4 场景模型元文件 - m4-scenario-model.yaml
model_type: SCENARIO
version: "4.0"
domain: "合同管理"

event_scenarios:
  - id: SCN-CONTRACT-001
    name: 收款完成驱动合同关闭
    description: 收款录入后通过事实事件触发合同关闭资格判断，满足条件时关闭合同
    sourceObjectRef: OBJ-PAYMENT-001
    targetObjectRefs:
      - OBJ-CONTRACT-001
    triggerEventRef: Payment.Recorded
    preconditions:
      - "收款记录已成功保存"
    postconditions:
      - "满足关闭条件时合同状态变为CLOSED"
    steps:
      - stepId: S01
        stepType: BEHAVIOR_CALL
        behaviorRef: Payment_Record
        nextSteps: [S02]
      - stepId: S02
        stepType: EVENT_EMIT
        eventRef: Payment.Recorded
        nextSteps: [S03]
      - stepId: S03
        stepType: RULE_EVALUATE
        ruleRef: RULE-CONTRACT-CLOSE-CHECK
        nextSteps: [S04]
      - stepId: S04
        stepType: BEHAVIOR_CALL
        behaviorRef: Contract_Close
        nextSteps: []
```

> **场景使用原则**：不是每个行为都需要M4场景。只有一个对象的状态变化通过事实事件驱动另一个独立对象发生变化时，才建立事件协同场景。普通查询、单对象修改、人工审批和常规顺序流程均不属于M4。

## 6.5  场景一致性约束

1. `sourceObjectRef`、`targetObjectRefs` 必须引用M1已存在的不同聚合根；同一聚合内部变化不得建立跨对象场景。
2. `triggerEventRef` 必须引用ME已存在的事实事件，并与场景中对应 `EVENT_EMIT.eventRef` 一致。
3. `EVENT_EMIT` 前的 `BEHAVIOR_CALL` 必须等于该事件的 `producerBehaviorRef`。
4. `EVENT_EMIT` 后连接规则时，规则必须在 `subscribedEvents` 中订阅该事件，事件 `subscribers` 也必须反向包含该规则。
5. `RULE_EVALUATE` 后的行为必须包含在规则 `triggeredBehaviors` 中。
6. 场景只允许 `BEHAVIOR_CALL`、`EVENT_EMIT`、`RULE_EVALUATE` 三类步骤；流程网关、人工活动、审批结果和子流程调用一律进入M6。
7. 无消费者事件不得建立场景；一个事件存在多个独立消费者时，允许 `nextSteps` 指向多个规则或行为节点。

---

# 第七章  M5 主体模型

## 7.1  设计目标

主体模型解决"谁能做什么"的问题，是行为模型与对象模型之外的独立关切。采用RBAC（基于角色的访问控制）为基础，支持ABAC（基于属性的访问控制）扩展。

> **设计决策**：权限定义不内嵌于行为模型（避免耦合），而是在行为模型中声明 `requiredPermissions`，在主体模型中定义权限的授予关系。这样角色权限的变化不影响行为模型定义。

---

## 7.2  模型层次

| 层次 | 说明 |
|------|------|
| 参与者（Actor） | 与系统交互的主体，分为：人类用户（Human）、系统账户（System）、外部系统（External） |
| 角色（Role） | 权限的集合单元，一个参与者可拥有多个角色，角色支持继承 |
| 权限（Permission） | 对特定对象或行为的操作授权，粒度到行为级别 |
| 权限组（PermissionGroup） | 权限的分组管理，便于批量授予 |
| 外部实体（ExternalEntity） | 外部系统的边界定义，包括接口契约、协议、数据契约 |

---

## 7.3  模型元素规范

### 7.3.1  参与者（Actor）

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

### 7.3.2  角色（Role）

角色除了承载权限集合，也是M6人工任务的唯一参与人类型。删除或重命名角色前，必须分析所有M6流程中的 `roleRefs` 和活动 `roleRef`。

| 属性名 | 类型 | 说明 |
|--------|------|------|
| roleId | String | 角色唯一标识 |
| name | String | 角色名称 |
| inheritsFrom | RoleRef[] | 继承的父角色（支持多继承） |
| permissions | PermissionRef[] | 直接授予的权限列表 |
| permissionGroups | GroupRef[] | 授予的权限组 |

### 7.3.3  权限（Permission）

| 属性名 | 类型 | 说明 |
|--------|------|------|
| permissionId | String | 权限标识，建议格式：PERM-{Domain}-{Action} |
| targetType | Enum | 授权目标类型：BEHAVIOR（行为）/ ENTITY（实体数据） |
| targetRef | Ref | 授权目标引用 |
| dataScope | Enum | 数据范围：ALL / OWN / DEPT / CUSTOM |
| abacCondition | String | ABAC条件表达式，如 `actor.dept == resource.dept` |

---

## 7.4  YAML 元文件模板

```yaml
# M5 主体模型元文件 - m5-actor-model.yaml
model_type: ACTOR
version: "1.0"
domain: "订单与合同管理示例"

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

  - actorId: ACTOR-FINANCE-MANAGER
    name: 财务经理
    actorType: HUMAN
    roles:
      - ROLE-FINANCE-MANAGER

  - actorId: ACTOR-CONTRACT-SPECIALIST
    name: 合同专员
    actorType: HUMAN
    roles:
      - ROLE-CONTRACT-SPECIALIST

  - actorId: ACTOR-GENERAL-MANAGER
    name: 公司总经理
    actorType: HUMAN
    roles:
      - ROLE-GENERAL-MANAGER

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

  - roleId: ROLE-FINANCE-MANAGER
    name: 财务经理
    permissions:
      - PERM-CONTRACT-APPROVE-FINANCE

  - roleId: ROLE-CONTRACT-SPECIALIST
    name: 合同专员
    permissions:
      - PERM-CONTRACT-CREATE

  - roleId: ROLE-GENERAL-MANAGER
    name: 公司总经理
    permissions:
      - PERM-CONTRACT-APPROVE-LARGE

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

  - permissionId: PERM-CONTRACT-APPROVE-FINANCE
    targetType: BEHAVIOR
    targetRef: Contract_ApproveFinance
    dataScope: ALL

  - permissionId: PERM-CONTRACT-CREATE
    targetType: BEHAVIOR
    targetRef: Contract_Create
    dataScope: DEPT

  - permissionId: PERM-CONTRACT-APPROVE-LARGE
    targetType: BEHAVIOR
    targetRef: Contract_ApproveGeneralManager
    dataScope: ALL
```

---

# 第八章  M6 流程模型

## 8.1  设计目标与边界

M6流程模型定义业务工作如何从开始流转到结束，承载两类流程：

1. `COLLABORATION`：端到端业务协同流，例如合同创建、审批、生效、开票、收款和关闭。
2. `APPROVAL`：围绕提交、审批、会签、驳回、退回和通过形成的审批流，可被协同流作为子流程调用。

M6负责角色任务、系统活动、顺序、并行、条件网关、事件等待、场景调用和子流程调用。M6不重新定义对象、行为、规则、事件、场景或角色，而是通过稳定ID引用其他模型。

> **强制角色约束**：所有 `USER_TASK` 和 `APPROVAL_TASK` 的参与人只能通过 `roleRef` 引用M5 `roles.roleId`。不得引用 `actorId`，不得填写“财务经理”等自由文本，也不得直接绑定具体用户。

## 8.2  流程类型及组合关系

### 8.2.1  端到端业务协同流（COLLABORATION）

端到端协同流描述一个业务目标跨阶段、跨对象的完整生命周期。它可以调用M2行为、M4事件协同场景和M6审批子流程。

```text
合同创建
-> 调用合同审批子流程
-> 合同生效
-> 合同开票
-> 合同收款
-> 调用收款完成关闭合同事件场景
-> 合同结束
```

### 8.2.2  审批流（APPROVAL）

审批流描述由角色承担的人工决策过程。审批条件可以直接使用流程表达式，也可以通过 `ruleRef` 引用M3规则。金额、数量、状态、组织层级等可复用业务判断应优先进入M3。

```text
合同提交
-> 财务经理审批
-> 合同金额判断
   -> 不超过100万元：审批通过
   -> 超过100万元：公司总经理审批
-> 审批完成
```

### 8.2.3  M4与M6的组合

- M4只定义“行为 -> 事件 -> 规则/行为 -> 目标行为”的事件协同语义。
- M6通过 `SCENARIO_CALL.scenarioRef` 调用M4，不复制事件链。
- M6通过 `SUB_FLOW_CALL.subFlowRef` 调用另一条M6流程；协同流可以调用审批流。
- M4不得反向引用M6，避免事件语义依赖具体流程实现。

## 8.3  模型元素规范

### 8.3.1  流程（Flow）

| 属性名 | 类型 | 说明 |
|--------|------|------|
| id | String | 流程唯一标识，建议格式 `FLOW-{DOMAIN}-{NNN}` |
| name | String | 流程业务名称 |
| flowType | Enum | `COLLABORATION` / `APPROVAL` |
| description | String | 流程目标和边界说明 |
| businessObjectRefs | AggregateRef[] | 流程涉及的M1聚合根 |
| roleRefs | RoleRef[] | 流程中允许承担人工活动的M5角色集合 |
| trigger | FlowTrigger | 流程启动方式 |
| preconditions | String[] | 流程启动前提 |
| postconditions | String[] | 流程完成后的业务状态 |
| startActivity | ActivityRef | 唯一开始活动 |
| endActivities | ActivityRef[] | 一个或多个合法结束活动 |
| activities | FlowActivity[] | 流程活动和网关集合 |
| version | String | 流程定义版本 |

### 8.3.2  流程触发器（FlowTrigger）

| 属性名 | 类型 | 说明 |
|--------|------|------|
| triggerType | Enum | `MANUAL` / `BEHAVIOR` / `EVENT` / `SCHEDULE` / `SUB_FLOW` |
| behaviorRef | BehaviorRef | `BEHAVIOR`触发时引用M2行为 |
| eventRef | EventRef | `EVENT`触发时引用ME事件 |
| scheduleExpression | String | `SCHEDULE`触发时的Cron或伪代码表达式 |

### 8.3.3  流程活动（FlowActivity）

| 属性名 | 类型 | 说明 |
|--------|------|------|
| activityId | String | 流程内唯一活动标识 |
| name | String | 活动业务名称 |
| activityType | Enum | `START` / `END` / `USER_TASK` / `APPROVAL_TASK` / `SYSTEM_TASK` / `BEHAVIOR_CALL` / `SCENARIO_CALL` / `SUB_FLOW_CALL` / `GATEWAY` / `EVENT_WAIT` |
| roleRef | RoleRef | `USER_TASK`和`APPROVAL_TASK`必填，只能引用M5角色 |
| behaviorRef | BehaviorRef | `BEHAVIOR_CALL`或需要落到领域行为的任务引用M2行为 |
| scenarioRef | ScenarioRef | `SCENARIO_CALL`引用M4事件协同场景；从场景的triggerEventRef开始衔接，不重复执行事件上游的来源行为 |
| subFlowRef | FlowRef | `SUB_FLOW_CALL`引用M6中的另一流程，禁止直接或间接循环调用 |
| ruleRef | RuleRef | 可选，引用M3规则作为进入、完成或网关判断条件 |
| conditionExpression | String | 不需要独立复用时可使用的流程局部条件表达式或伪代码 |
| approvalOutcomes | Enum[] | 审批任务允许的结果，如 `APPROVE` / `REJECT` / `RETURN` |
| eventRef | EventRef | `EVENT_WAIT`等待的ME事件 |
| timeout | Duration | 活动超时约束 |
| nextActivities | ActivityRef[] | 普通活动的后继活动 |
| branches | FlowBranch[] | `GATEWAY`的条件分支 |

### 8.3.4  流程分支（FlowBranch）

| 属性名 | 类型 | 说明 |
|--------|------|------|
| branchName | String | 分支名称 |
| ruleRef | RuleRef | 可选，引用M3规则 |
| conditionExpression | String | 可选，流程局部判断表达式；与ruleRef至少填写一个，默认分支除外 |
| approvalOutcome | Enum | 可选，按 `APPROVE` / `REJECT` / `RETURN` 等审批结果分支 |
| targetActivity | ActivityRef | 目标活动 |
| isDefault | Boolean | 是否默认分支；同一网关最多一个默认分支 |

## 8.4  YAML 元文件模板

以下同一文件同时给出端到端协同流和审批流示例。示例中的 `ROLE-CONTRACT-SPECIALIST`、`ROLE-FINANCE-MANAGER`、`ROLE-GENERAL-MANAGER` 必须已在 `m5-actor-model.yaml` 中定义。

```yaml
# M6 流程模型元文件 - m6-flow-model.yaml
model_type: FLOW
version: "1.0"
domain: "合同管理"

flows:
  - id: FLOW-CONTRACT-001
    name: 合同全生命周期协同流
    flowType: COLLABORATION
    description: 从合同创建、审批、生效、开票和收款到合同关闭的端到端协同流程
    businessObjectRefs:
      - OBJ-CONTRACT-001
      - OBJ-INVOICE-001
      - OBJ-PAYMENT-001
    roleRefs:
      - ROLE-CONTRACT-SPECIALIST
      - ROLE-FINANCE-MANAGER
      - ROLE-GENERAL-MANAGER
    trigger:
      triggerType: MANUAL
    preconditions:
      - "用户具备合同创建权限"
    postconditions:
      - "合同已关闭或流程已明确终止"
    startActivity: A01
    endActivities: [A08]
    activities:
      - activityId: A01
        name: 开始
        activityType: START
        nextActivities: [A02]
      - activityId: A02
        name: 创建合同
        activityType: USER_TASK
        roleRef: ROLE-CONTRACT-SPECIALIST
        behaviorRef: Contract_Create
        nextActivities: [A03]
      - activityId: A03
        name: 合同审批
        activityType: SUB_FLOW_CALL
        subFlowRef: FLOW-CONTRACT-APPROVAL-001
        nextActivities: [A04]
      - activityId: A04
        name: 合同生效
        activityType: BEHAVIOR_CALL
        behaviorRef: Contract_Activate
        nextActivities: [A05]
      - activityId: A05
        name: 合同开票
        activityType: BEHAVIOR_CALL
        behaviorRef: Invoice_Issue
        nextActivities: [A06]
      - activityId: A06
        name: 合同收款
        activityType: BEHAVIOR_CALL
        behaviorRef: Payment_Record
        nextActivities: [A07]
      - activityId: A07
        name: 收款完成关闭合同事件协同
        activityType: SCENARIO_CALL
        scenarioRef: SCN-CONTRACT-001
        nextActivities: [A08]
      - activityId: A08
        name: 结束
        activityType: END
        nextActivities: []

  - id: FLOW-CONTRACT-APPROVAL-001
    name: 合同创建审批流
    flowType: APPROVAL
    description: 合同先由财务经理审批，金额超过100万元时增加公司总经理审批
    businessObjectRefs:
      - OBJ-CONTRACT-001
    roleRefs:
      - ROLE-FINANCE-MANAGER
      - ROLE-GENERAL-MANAGER
    trigger:
      triggerType: BEHAVIOR
      behaviorRef: Contract_Submit
    preconditions:
      - "contract.status == 'SUBMITTED'"
    postconditions:
      - "合同审批通过、驳回或退回修改"
    startActivity: P01
    endActivities: [P07, P08]
    activities:
      - activityId: P01
        name: 开始
        activityType: START
        nextActivities: [P02]
      - activityId: P02
        name: 财务经理审批
        activityType: APPROVAL_TASK
        roleRef: ROLE-FINANCE-MANAGER
        behaviorRef: Contract_ApproveFinance
        approvalOutcomes: [APPROVE, REJECT, RETURN]
        nextActivities: [P03]
      - activityId: P03
        name: 财务审批结果判断
        activityType: GATEWAY
        branches:
          - branchName: 驳回或退回
            conditionExpression: "approval.outcome IN ['REJECT', 'RETURN']"
            targetActivity: P08
            isDefault: false
          - branchName: 财务审批通过
            conditionExpression: "approval.outcome == 'APPROVE'"
            targetActivity: P04
            isDefault: true
      - activityId: P04
        name: 大额合同判断
        activityType: GATEWAY
        branches:
          - branchName: 超过100万元
            conditionExpression: "contract.totalAmount > 1000000"
            targetActivity: P05
            isDefault: false
          - branchName: 普通金额合同
            conditionExpression: null
            targetActivity: P06
            isDefault: true
      - activityId: P05
        name: 公司总经理审批
        activityType: APPROVAL_TASK
        roleRef: ROLE-GENERAL-MANAGER
        behaviorRef: Contract_ApproveGeneralManager
        approvalOutcomes: [APPROVE, REJECT, RETURN]
        nextActivities: [P09]
      - activityId: P09
        name: 总经理审批结果判断
        activityType: GATEWAY
        branches:
          - branchName: 总经理审批通过
            approvalOutcome: APPROVE
            targetActivity: P06
            isDefault: false
          - branchName: 总经理驳回
            approvalOutcome: REJECT
            targetActivity: P08
            isDefault: false
          - branchName: 总经理退回
            approvalOutcome: RETURN
            targetActivity: P08
            isDefault: false
      - activityId: P06
        name: 标记审批通过
        activityType: BEHAVIOR_CALL
        behaviorRef: Contract_MarkApproved
        nextActivities: [P07]
      - activityId: P07
        name: 审批通过结束
        activityType: END
        nextActivities: []
      - activityId: P08
        name: 驳回或退回结束
        activityType: END
        nextActivities: []
```

## 8.5  流程约束

1. 每条流程必须有且仅有一个 `startActivity`，并至少有一个 `endActivities`。
2. 除 `END` 外的可达活动必须存在合法后继路径；所有 `endActivities` 必须指向 `END` 活动。
3. `USER_TASK`、`APPROVAL_TASK` 必须填写有效 `roleRef`，且该角色必须包含在流程 `roleRefs` 中。
4. `GATEWAY` 至少有两个分支，最多一个默认分支；非默认分支必须提供 `ruleRef`、`conditionExpression` 或 `approvalOutcome`。
5. `SUB_FLOW_CALL` 不得形成直接或间接递归调用环。
6. `SCENARIO_CALL` 只能引用M4 `event_scenarios.id`。
7. 审批活动的处理结果必须显式覆盖通过、驳回、退回等实际业务结果，不得只有成功路径。
8. 流程局部条件可使用表达式；跨流程复用、跨对象或复杂决策应定义为M3规则并通过 `ruleRef` 引用。
9. `SCENARIO_CALL` 从被引用M4场景的 `triggerEventRef` 进入事件协同链；M4中记录的来源行为用于语义追溯，不得被流程重复执行。

## 8.6  跨模型引用矩阵

| M6字段 | 引用目标 | 一致性要求 |
|--------|----------|------------|
| `businessObjectRefs` | M1 `aggregates.id` | 引用对象必须存在，删除对象前必须分析流程影响 |
| `roleRefs`、`activity.roleRef` | M5 `roles.roleId` | 只允许角色，不允许Actor或自由文本；活动角色必须属于流程roleRefs |
| `trigger.behaviorRef`、`activity.behaviorRef` | M2 `behaviors.id` | 引用行为必须存在，任务所需权限应与角色权限一致 |
| `trigger.eventRef`、`activity.eventRef` | ME `events.eventId` | 事件必须存在；EVENT_WAIT需明确超时或替代路径 |
| `activity.ruleRef`、`branch.ruleRef` | M3 `rules.id` | 规则必须无副作用，流程只消费判断结果 |
| `activity.scenarioRef` | M4 `event_scenarios.id` | 场景必须存在，且从其triggerEventRef衔接 |
| `activity.subFlowRef` | M6 `flows.id` | 子流程必须存在，且整个调用图无环 |

M6中的稳定引用不得通过名称猜测。角色、行为、规则、事件、场景和子流程的ID发生变更时，必须展示影响范围并级联更新或阻止保存。

---

# 第九章  M7 查询统计与报表模型

## 9.1  设计目标与边界

M7定义业务应用中的查询统计对象和固定报表对象，专门承载多个M1业务对象之间的关联查询、条件过滤、结果列、分组聚合、排序、分页和参考SQL。

M7不是BI指标模型，不定义事实表、维度表、宽表、数据集市、OLAP Cube、ETL任务或自助分析语义。M7也不定义独立指标资产；`COUNT`、`SUM`、`AVG`、`MIN`、`MAX`等只作为具体查询对象内部的聚合表达式存在。

M7的直接依赖严格限制为：

```text
M7 查询统计或报表对象 -> M1 对象及字段
M7 查询统计或报表对象 <-> M2 QUERY行为（一对一）
```

M7不直接引用M3规则、ME事件、M4场景、M5主体或M6流程。查询条件、关联条件、聚合公式和SQL表达式属于查询对象自身定义，不视为M3业务规则。数据访问权限暂不在M7建模，仍由M2行为通过 `requiredPermissions` 与M5建立关系。

## 9.2  M2查询行为与M7对象的分界

| 判断问题 | M2 QUERY行为 | M7 查询统计与报表对象 |
|----------|--------------|-----------------------|
| 核心职责 | 定义“执行一次查询或生成报表”的原子行为 | 定义“查什么、如何关联、如何统计、返回什么” |
| 单聚合简单查询 | 可独立定义，不要求M7 | 通常不建立 |
| 跨对象关联查询 | 通过queryReportRef调用M7 | 定义来源对象、Join、条件和结果列 |
| 分组统计 | 负责执行入口 | 定义聚合、GROUP BY、HAVING |
| 固定报表 | 负责生成或导出行为 | 定义报表列、分组、小计、合计和导出格式 |
| 权限 | 通过requiredPermissions关联M5 | 不定义权限或角色 |

一条M2行为最多引用一个M7对象，一个M7对象也必须且只能绑定一条M2行为。

## 9.3  对象类型

| objectType | 说明 | 典型示例 |
|------------|------|----------|
| `DETAIL_QUERY` | 跨对象详情查询，返回一条主要业务记录及关联信息 | 合同执行详情 |
| `LIST_QUERY` | 跨对象条件列表查询，通常支持分页和排序 | 已开票未收款合同列表 |
| `STATISTICAL_QUERY` | 包含聚合或分组统计的查询分析 | 按部门统计合同金额 |
| `REPORT` | 具有固定列、分组、小计、合计和导出要求的业务报表 | 部门合同执行汇总报表 |

## 9.4  模型元素规范

### 9.4.1  查询统计或报表对象（QueryReportObject）

| 属性名 | 类型 | 说明 |
|--------|------|------|
| id | String | 对象唯一标识；查询建议 `QR-{DOMAIN}-{NNN}`，报表建议 `RPT-{DOMAIN}-{NNN}` |
| name | String | 查询统计或报表名称 |
| alias | String | 稳定英文别名，使用lowerCamelCase |
| objectType | Enum | `DETAIL_QUERY` / `LIST_QUERY` / `STATISTICAL_QUERY` / `REPORT` |
| description | String | 业务目的和结果口径说明 |
| behaviorRef | BehaviorRef | 唯一绑定的M2 QUERY行为 |
| sourceObjects | QuerySource[] | 查询涉及的M1对象及别名 |
| joins | QueryJoin[] | 多对象关联定义 |
| parameters | QueryParameter[] | 查询输入参数和允许操作符 |
| conditions | QueryCondition[] | WHERE条件语义定义 |
| resultColumns | ResultColumn[] | 查询结果列及聚合表达式 |
| groupBy | FieldExpression[] | 分组字段表达式 |
| having | QueryCondition[] | 聚合后的过滤条件 |
| orderBy | OrderBy[] | 默认排序 |
| pagination | Pagination | 分页约束 |
| reportOptions | ReportOptions | objectType=REPORT时的固定报表设置 |
| referenceSql | ReferenceSql | 参考SQL、方言、参数绑定及结果映射 |
| version | String | 对象定义版本 |

### 9.4.2  查询来源（QuerySource）

| 属性名 | 类型 | 说明 |
|--------|------|------|
| objectRef | AggregateRef | M1聚合根ID |
| alias | String | 查询内部唯一别名，如contract、invoice、payment |
| entityPath | FieldPath | 可选，查询聚合内某个子实体时填写 |
| primary | Boolean | 是否为主查询对象；每个M7对象必须且只能有一个主对象 |
| preAggregation | SourcePreAggregation | 可选；在参与Join前按关联键预聚合一对多明细，避免多个明细来源相乘 |

`SourcePreAggregation`结构：

| 属性名 | 类型 | 说明 |
|--------|------|------|
| groupBy | FieldPath[] | 预聚合分组字段，通常为指向主对象的外键 |
| columns | PreAggregateColumn[] | 预聚合结果列；包含`name`、`sourceExpression`和`aggregateFunction` |

当两个或更多一对多来源同时关联主对象并参与聚合时，必须先按Join键分别预聚合。不得使用`SUM(DISTINCT amount)`规避重复行，因为不同业务记录可能具有相同金额。

### 9.4.3  对象关联（QueryJoin）

| 属性名 | 类型 | 说明 |
|--------|------|------|
| joinId | String | 查询对象内唯一关联标识 |
| joinType | Enum | `INNER` / `LEFT` / `RIGHT` / `FULL` |
| leftSource | SourceAlias | 左侧来源别名 |
| rightSource | SourceAlias | 右侧来源别名 |
| relationRef | AssociationRef | 可选，引用M1 `aggregate_associations.id` |
| conditionExpression | String | 关联字段表达式，如 `invoice.contractId == contract.contractId` |

优先通过 `relationRef` 复用M1已定义的聚合关联。没有显式关联对象时，可以使用 `conditionExpression`，但字段路径必须存在于M1。

### 9.4.4  查询参数（QueryParameter）

| 属性名 | 类型 | 说明 |
|--------|------|------|
| name | String | 参数名 |
| label | String | 参数显示名称 |
| dataType | String | String / Integer / Decimal / Boolean / Date / DateTime / Enum / DictionaryRef |
| required | Boolean | 是否必填 |
| defaultValue | Any | 默认值或表达式 |
| allowedOperators | Enum[] | `EQ` / `NE` / `GT` / `GE` / `LT` / `LE` / `IN` / `NOT_IN` / `LIKE` / `BETWEEN` / `IS_NULL` / `IS_NOT_NULL` |
| sourceField | FieldPath | 参数通常约束的M1字段路径，可选 |

### 9.4.5  查询条件（QueryCondition）

| 属性名 | 类型 | 说明 |
|--------|------|------|
| conditionId | String | 条件唯一标识 |
| leftExpression | String | M1字段路径或查询表达式 |
| operator | Enum | 使用QueryParameter允许的操作符 |
| parameterRef | ParameterRef | 右值来源于查询参数时填写 |
| fixedValue | Any | 使用固定值时填写；与parameterRef二选一 |
| logicalConnector | Enum | `AND` / `OR` |
| group | String | 条件组标识，用于表达括号组合 |
| skipWhenParameterEmpty | Boolean | 可选参数为空时是否跳过该条件 |

### 9.4.6  结果列（ResultColumn）

| 属性名 | 类型 | 说明 |
|--------|------|------|
| name | String | 稳定结果字段名 |
| label | String | 业务显示名称 |
| dataType | String | 结果数据类型 |
| sourceExpression | String | M1字段路径、计算表达式或聚合表达式 |
| aggregateFunction | Enum | `NONE` / `COUNT` / `COUNT_DISTINCT` / `SUM` / `AVG` / `MIN` / `MAX` |
| format | String | 日期、金额、百分比等格式 |
| nullable | Boolean | 是否允许空值 |
| visible | Boolean | 默认是否输出 |
| sortable | Boolean | 是否允许排序 |

### 9.4.7  分组、排序与分页

`groupBy`直接保存字段或表达式列表。`having`复用QueryCondition结构，但左侧通常是聚合表达式。`orderBy`结构如下：

| 属性名 | 类型 | 说明 |
|--------|------|------|
| expression | String | 结果列名、字段路径或聚合表达式 |
| direction | Enum | `ASC` / `DESC` |

分页结构：

| 属性名 | 类型 | 说明 |
|--------|------|------|
| enabled | Boolean | 是否分页 |
| defaultPageSize | Integer | 默认每页条数 |
| maxPageSize | Integer | 最大每页条数 |

### 9.4.8  固定报表设置（ReportOptions）

仅当 `objectType=REPORT` 时使用：

| 属性名 | 类型 | 说明 |
|--------|------|------|
| title | String | 报表标题 |
| layout | Enum | 当前仅支持 `TABLE`，不定义BI图表和仪表盘 |
| groupFields | ResultColumnRef[] | 报表分组列 |
| subtotalFields | ResultColumnRef[] | 分组小计列 |
| totalFields | ResultColumnRef[] | 全表合计列 |
| exportFormats | Enum[] | `XLSX` / `CSV` / `PDF` |
| emptyValueDisplay | String | 空值显示文本 |

### 9.4.9  参考SQL（ReferenceSql）

| 属性名 | 类型 | 说明 |
|--------|------|------|
| dialect | String | PostgreSQL / MySQL / SQLServer / Oracle / SQLite等 |
| sql | String | 带命名参数的完整参考SQL |
| parameterBindings | SqlParameterBinding[] | M7参数到SQL占位符的映射 |
| resultMappings | SqlResultMapping[] | SQL结果列到M7 resultColumns的映射 |

参考SQL用于实现指导、人工复核和生成代码，不是唯一权威来源。M1对象引用、Join、查询条件、结果列和聚合定义才是稳定业务语义。SQL可以使用实际物理表名，但必须通过参数和结果映射与M7语义定义保持一致。

## 9.5  YAML 元文件模板

以下示例覆盖合同执行分析、已开票未收款分析和按部门合同汇总报表，不引入指标、维度表或宽表。

```yaml
# M7 查询统计与报表模型元文件 - m7-report-model.yaml
model_type: REPORT
version: "1.0"
domain: "合同管理"

query_reports:
  - id: QR-CONTRACT-EXECUTION-001
    name: 合同执行情况分析
    alias: contractExecutionAnalysis
    objectType: STATISTICAL_QUERY
    description: 按合同汇总合同金额、已开票金额、已收款金额、未收款金额和执行比例
    behaviorRef: Contract_QueryExecutionAnalysis
    sourceObjects:
      - objectRef: OBJ-CONTRACT-001
        alias: contract
        primary: true
      - objectRef: OBJ-INVOICE-001
        alias: invoiceAgg
        primary: false
        preAggregation:
          groupBy: [contractId]
          columns:
            - name: invoicedAmount
              sourceExpression: invoiceAmount
              aggregateFunction: SUM
      - objectRef: OBJ-PAYMENT-001
        alias: paymentAgg
        primary: false
        preAggregation:
          groupBy: [contractId]
          columns:
            - name: receivedAmount
              sourceExpression: receivedAmount
              aggregateFunction: SUM
    joins:
      - joinId: J01
        joinType: LEFT
        leftSource: contract
        rightSource: invoiceAgg
        relationRef: ASSOC-CONTRACT-INVOICE
        conditionExpression: "invoiceAgg.contractId == contract.contractId"
      - joinId: J02
        joinType: LEFT
        leftSource: contract
        rightSource: paymentAgg
        relationRef: ASSOC-CONTRACT-PAYMENT
        conditionExpression: "paymentAgg.contractId == contract.contractId"
    parameters:
      - name: departmentId
        label: 所属部门
        dataType: String
        required: false
        defaultValue: null
        allowedOperators: [EQ]
        sourceField: contract.departmentId
      - name: dateStart
        label: 合同开始日期
        dataType: Date
        required: false
        defaultValue: null
        allowedOperators: [GE]
        sourceField: contract.signDate
      - name: dateEnd
        label: 合同结束日期
        dataType: Date
        required: false
        defaultValue: null
        allowedOperators: [LE]
        sourceField: contract.signDate
    conditions:
      - conditionId: C01
        leftExpression: contract.departmentId
        operator: EQ
        parameterRef: departmentId
        fixedValue: null
        logicalConnector: AND
        group: BASE
        skipWhenParameterEmpty: true
      - conditionId: C02
        leftExpression: contract.signDate
        operator: GE
        parameterRef: dateStart
        fixedValue: null
        logicalConnector: AND
        group: BASE
        skipWhenParameterEmpty: true
      - conditionId: C03
        leftExpression: contract.signDate
        operator: LE
        parameterRef: dateEnd
        fixedValue: null
        logicalConnector: AND
        group: BASE
        skipWhenParameterEmpty: true
    resultColumns:
      - name: contractId
        label: 合同编号
        dataType: String
        sourceExpression: contract.contractId
        aggregateFunction: NONE
        format: null
        nullable: false
        visible: true
        sortable: true
      - name: contractName
        label: 合同名称
        dataType: String
        sourceExpression: contract.contractName
        aggregateFunction: NONE
        format: null
        nullable: false
        visible: true
        sortable: true
      - name: contractAmount
        label: 合同金额
        dataType: Decimal
        sourceExpression: contract.totalAmount
        aggregateFunction: MAX
        format: "#,##0.00"
        nullable: false
        visible: true
        sortable: true
      - name: invoicedAmount
        label: 已开票金额
        dataType: Decimal
        sourceExpression: invoiceAgg.invoicedAmount
        aggregateFunction: MAX
        format: "#,##0.00"
        nullable: false
        visible: true
        sortable: true
      - name: receivedAmount
        label: 已收款金额
        dataType: Decimal
        sourceExpression: paymentAgg.receivedAmount
        aggregateFunction: MAX
        format: "#,##0.00"
        nullable: false
        visible: true
        sortable: true
      - name: unreceivedAmount
        label: 未收款金额
        dataType: Decimal
        sourceExpression: "MAX(contract.totalAmount) - COALESCE(MAX(paymentAgg.receivedAmount), 0)"
        aggregateFunction: NONE
        format: "#,##0.00"
        nullable: false
        visible: true
        sortable: true
      - name: receiptRate
        label: 收款完成率
        dataType: Decimal
        sourceExpression: "COALESCE(MAX(paymentAgg.receivedAmount), 0) / NULLIF(MAX(contract.totalAmount), 0)"
        aggregateFunction: NONE
        format: "0.00%"
        nullable: false
        visible: true
        sortable: true
    groupBy:
      - contract.contractId
      - contract.contractName
    having: []
    orderBy:
      - expression: contract.signDate
        direction: DESC
    pagination:
      enabled: true
      defaultPageSize: 20
      maxPageSize: 200
    reportOptions: null
    referenceSql:
      dialect: PostgreSQL
      sql: |
        WITH invoice_agg AS (
          SELECT contract_id, SUM(invoice_amount) AS invoiced_amount
          FROM invoice
          GROUP BY contract_id
        ), payment_agg AS (
          SELECT contract_id, SUM(received_amount) AS received_amount
          FROM payment
          GROUP BY contract_id
        )
        SELECT c.contract_id,
               c.contract_name,
               MAX(c.total_amount) AS contract_amount,
               COALESCE(MAX(i.invoiced_amount), 0) AS invoiced_amount,
               COALESCE(MAX(p.received_amount), 0) AS received_amount,
               MAX(c.total_amount) - COALESCE(MAX(p.received_amount), 0) AS unreceived_amount,
               COALESCE(MAX(p.received_amount), 0) / NULLIF(MAX(c.total_amount), 0) AS receipt_rate
        FROM contract c
        LEFT JOIN invoice_agg i ON i.contract_id = c.contract_id
        LEFT JOIN payment_agg p ON p.contract_id = c.contract_id
        WHERE (:departmentId IS NULL OR c.department_id = :departmentId)
          AND (:dateStart IS NULL OR c.sign_date >= :dateStart)
          AND (:dateEnd IS NULL OR c.sign_date <= :dateEnd)
        GROUP BY c.contract_id, c.contract_name, c.sign_date
        ORDER BY c.sign_date DESC
      parameterBindings:
        - parameter: departmentId
          placeholder: ":departmentId"
        - parameter: dateStart
          placeholder: ":dateStart"
        - parameter: dateEnd
          placeholder: ":dateEnd"
      resultMappings:
        - sqlColumn: contract_id
          resultColumn: contractId
        - sqlColumn: contract_name
          resultColumn: contractName
        - sqlColumn: contract_amount
          resultColumn: contractAmount
        - sqlColumn: invoiced_amount
          resultColumn: invoicedAmount
        - sqlColumn: received_amount
          resultColumn: receivedAmount
        - sqlColumn: unreceived_amount
          resultColumn: unreceivedAmount
        - sqlColumn: receipt_rate
          resultColumn: receiptRate
    version: "1.0"

  - id: QR-CONTRACT-UNRECEIVED-001
    name: 已开票未收款合同分析
    alias: invoicedUnreceivedContracts
    objectType: LIST_QUERY
    description: 查询已产生开票记录但累计收款金额小于累计开票金额的合同
    behaviorRef: Contract_QueryInvoicedUnreceived
    sourceObjects:
      - objectRef: OBJ-CONTRACT-001
        alias: contract
        primary: true
      - objectRef: OBJ-INVOICE-001
        alias: invoiceAgg
        primary: false
        preAggregation:
          groupBy: [contractId]
          columns:
            - name: invoicedAmount
              sourceExpression: invoiceAmount
              aggregateFunction: SUM
      - objectRef: OBJ-PAYMENT-001
        alias: paymentAgg
        primary: false
        preAggregation:
          groupBy: [contractId]
          columns:
            - name: receivedAmount
              sourceExpression: receivedAmount
              aggregateFunction: SUM
    joins:
      - joinId: J01
        joinType: INNER
        leftSource: contract
        rightSource: invoiceAgg
        relationRef: ASSOC-CONTRACT-INVOICE
        conditionExpression: "invoiceAgg.contractId == contract.contractId"
      - joinId: J02
        joinType: LEFT
        leftSource: contract
        rightSource: paymentAgg
        relationRef: ASSOC-CONTRACT-PAYMENT
        conditionExpression: "paymentAgg.contractId == contract.contractId"
    parameters:
      - name: departmentId
        label: 所属部门
        dataType: String
        required: false
        defaultValue: null
        allowedOperators: [EQ]
        sourceField: contract.departmentId
    conditions:
      - conditionId: C01
        leftExpression: contract.departmentId
        operator: EQ
        parameterRef: departmentId
        fixedValue: null
        logicalConnector: AND
        group: BASE
        skipWhenParameterEmpty: true
    resultColumns:
      - name: contractId
        label: 合同编号
        dataType: String
        sourceExpression: contract.contractId
        aggregateFunction: NONE
        format: null
        nullable: false
        visible: true
        sortable: true
      - name: contractName
        label: 合同名称
        dataType: String
        sourceExpression: contract.contractName
        aggregateFunction: NONE
        format: null
        nullable: false
        visible: true
        sortable: true
      - name: invoicedAmount
        label: 累计开票金额
        dataType: Decimal
        sourceExpression: invoiceAgg.invoicedAmount
        aggregateFunction: MAX
        format: "#,##0.00"
        nullable: false
        visible: true
        sortable: true
      - name: receivedAmount
        label: 累计收款金额
        dataType: Decimal
        sourceExpression: paymentAgg.receivedAmount
        aggregateFunction: MAX
        format: "#,##0.00"
        nullable: false
        visible: true
        sortable: true
      - name: outstandingAmount
        label: 已开票未收款金额
        dataType: Decimal
        sourceExpression: "MAX(invoiceAgg.invoicedAmount) - COALESCE(MAX(paymentAgg.receivedAmount), 0)"
        aggregateFunction: NONE
        format: "#,##0.00"
        nullable: false
        visible: true
        sortable: true
    groupBy:
      - contract.contractId
      - contract.contractName
    having:
      - conditionId: H01
        leftExpression: "MAX(invoiceAgg.invoicedAmount)"
        operator: GT
        parameterRef: null
        fixedValue: "COALESCE(MAX(paymentAgg.receivedAmount), 0)"
        logicalConnector: AND
        group: HAVING
        skipWhenParameterEmpty: false
    orderBy:
      - expression: outstandingAmount
        direction: DESC
    pagination:
      enabled: true
      defaultPageSize: 20
      maxPageSize: 200
    reportOptions: null
    referenceSql:
      dialect: PostgreSQL
      sql: |
        WITH invoice_agg AS (
          SELECT contract_id, SUM(invoice_amount) AS invoiced_amount
          FROM invoice
          GROUP BY contract_id
        ), payment_agg AS (
          SELECT contract_id, SUM(received_amount) AS received_amount
          FROM payment
          GROUP BY contract_id
        )
        SELECT c.contract_id,
               c.contract_name,
               MAX(i.invoiced_amount) AS invoiced_amount,
               COALESCE(MAX(p.received_amount), 0) AS received_amount,
               MAX(i.invoiced_amount) - COALESCE(MAX(p.received_amount), 0) AS outstanding_amount
        FROM contract c
        JOIN invoice_agg i ON i.contract_id = c.contract_id
        LEFT JOIN payment_agg p ON p.contract_id = c.contract_id
        WHERE (:departmentId IS NULL OR c.department_id = :departmentId)
        GROUP BY c.contract_id, c.contract_name
        HAVING MAX(i.invoiced_amount) > COALESCE(MAX(p.received_amount), 0)
        ORDER BY outstanding_amount DESC
      parameterBindings:
        - parameter: departmentId
          placeholder: ":departmentId"
      resultMappings:
        - sqlColumn: contract_id
          resultColumn: contractId
        - sqlColumn: contract_name
          resultColumn: contractName
        - sqlColumn: invoiced_amount
          resultColumn: invoicedAmount
        - sqlColumn: received_amount
          resultColumn: receivedAmount
        - sqlColumn: outstanding_amount
          resultColumn: outstandingAmount
    version: "1.0"

  - id: RPT-CONTRACT-DEPARTMENT-001
    name: 部门合同执行汇总报表
    alias: departmentContractSummaryReport
    objectType: REPORT
    description: 按部门汇总合同数量、合同金额、开票金额、收款金额和未收款金额
    behaviorRef: Contract_GenerateDepartmentSummaryReport
    sourceObjects:
      - objectRef: OBJ-CONTRACT-001
        alias: contract
        primary: true
      - objectRef: OBJ-DEPARTMENT-001
        alias: department
        primary: false
      - objectRef: OBJ-INVOICE-001
        alias: invoiceAgg
        primary: false
        preAggregation:
          groupBy: [contractId]
          columns:
            - name: invoicedAmount
              sourceExpression: invoiceAmount
              aggregateFunction: SUM
      - objectRef: OBJ-PAYMENT-001
        alias: paymentAgg
        primary: false
        preAggregation:
          groupBy: [contractId]
          columns:
            - name: receivedAmount
              sourceExpression: receivedAmount
              aggregateFunction: SUM
    joins:
      - joinId: J01
        joinType: INNER
        leftSource: contract
        rightSource: department
        relationRef: ASSOC-CONTRACT-DEPARTMENT
        conditionExpression: "contract.departmentId == department.departmentId"
      - joinId: J02
        joinType: LEFT
        leftSource: contract
        rightSource: invoiceAgg
        relationRef: ASSOC-CONTRACT-INVOICE
        conditionExpression: "invoiceAgg.contractId == contract.contractId"
      - joinId: J03
        joinType: LEFT
        leftSource: contract
        rightSource: paymentAgg
        relationRef: ASSOC-CONTRACT-PAYMENT
        conditionExpression: "paymentAgg.contractId == contract.contractId"
    parameters:
      - name: dateStart
        label: 统计开始日期
        dataType: Date
        required: true
        defaultValue: null
        allowedOperators: [GE]
        sourceField: contract.signDate
      - name: dateEnd
        label: 统计结束日期
        dataType: Date
        required: true
        defaultValue: null
        allowedOperators: [LE]
        sourceField: contract.signDate
    conditions:
      - conditionId: C01
        leftExpression: contract.signDate
        operator: GE
        parameterRef: dateStart
        fixedValue: null
        logicalConnector: AND
        group: BASE
        skipWhenParameterEmpty: false
      - conditionId: C02
        leftExpression: contract.signDate
        operator: LE
        parameterRef: dateEnd
        fixedValue: null
        logicalConnector: AND
        group: BASE
        skipWhenParameterEmpty: false
    resultColumns:
      - name: departmentName
        label: 部门
        dataType: String
        sourceExpression: department.departmentName
        aggregateFunction: NONE
        format: null
        nullable: false
        visible: true
        sortable: true
      - name: contractCount
        label: 合同数量
        dataType: Integer
        sourceExpression: contract.contractId
        aggregateFunction: COUNT_DISTINCT
        format: "#,##0"
        nullable: false
        visible: true
        sortable: true
      - name: contractAmount
        label: 合同金额
        dataType: Decimal
        sourceExpression: contract.totalAmount
        aggregateFunction: SUM
        format: "#,##0.00"
        nullable: false
        visible: true
        sortable: true
      - name: invoicedAmount
        label: 开票金额
        dataType: Decimal
        sourceExpression: invoiceAgg.invoicedAmount
        aggregateFunction: SUM
        format: "#,##0.00"
        nullable: false
        visible: true
        sortable: true
      - name: receivedAmount
        label: 收款金额
        dataType: Decimal
        sourceExpression: paymentAgg.receivedAmount
        aggregateFunction: SUM
        format: "#,##0.00"
        nullable: false
        visible: true
        sortable: true
      - name: unreceivedAmount
        label: 未收款金额
        dataType: Decimal
        sourceExpression: "SUM(contract.totalAmount) - COALESCE(SUM(paymentAgg.receivedAmount), 0)"
        aggregateFunction: NONE
        format: "#,##0.00"
        nullable: false
        visible: true
        sortable: true
    groupBy:
      - department.departmentId
      - department.departmentName
    having: []
    orderBy:
      - expression: contractAmount
        direction: DESC
    pagination:
      enabled: false
      defaultPageSize: 0
      maxPageSize: 0
    reportOptions:
      title: 部门合同执行汇总报表
      layout: TABLE
      groupFields: [departmentName]
      subtotalFields: []
      totalFields: [contractCount, contractAmount, invoicedAmount, receivedAmount, unreceivedAmount]
      exportFormats: [XLSX, CSV, PDF]
      emptyValueDisplay: "-"
    referenceSql:
      dialect: PostgreSQL
      sql: |
        WITH invoice_agg AS (
          SELECT contract_id, SUM(invoice_amount) AS invoiced_amount
          FROM invoice
          GROUP BY contract_id
        ), payment_agg AS (
          SELECT contract_id, SUM(received_amount) AS received_amount
          FROM payment
          GROUP BY contract_id
        )
        SELECT d.department_name,
               COUNT(DISTINCT c.contract_id) AS contract_count,
               SUM(c.total_amount) AS contract_amount,
               COALESCE(SUM(i.invoiced_amount), 0) AS invoiced_amount,
               COALESCE(SUM(p.received_amount), 0) AS received_amount,
               SUM(c.total_amount) - COALESCE(SUM(p.received_amount), 0) AS unreceived_amount
        FROM contract c
        JOIN department d ON d.department_id = c.department_id
        LEFT JOIN invoice_agg i ON i.contract_id = c.contract_id
        LEFT JOIN payment_agg p ON p.contract_id = c.contract_id
        WHERE c.sign_date BETWEEN :dateStart AND :dateEnd
        GROUP BY d.department_id, d.department_name
        ORDER BY contract_amount DESC
      parameterBindings:
        - parameter: dateStart
          placeholder: ":dateStart"
        - parameter: dateEnd
          placeholder: ":dateEnd"
      resultMappings:
        - sqlColumn: department_name
          resultColumn: departmentName
        - sqlColumn: contract_count
          resultColumn: contractCount
        - sqlColumn: contract_amount
          resultColumn: contractAmount
        - sqlColumn: invoiced_amount
          resultColumn: invoicedAmount
        - sqlColumn: received_amount
          resultColumn: receivedAmount
        - sqlColumn: unreceived_amount
          resultColumn: unreceivedAmount
    version: "1.0"
```

对应的M2查询报表行为必须一对一引用上述对象：

```yaml
# m2-behavior-model.yaml 对应片段
behaviors:
  - id: Contract_QueryExecutionAnalysis
    name: 查询合同执行情况分析
    ownerEntity: OBJ-CONTRACT-001
    behaviorType: QUERY
    triggerType: USER_ACTION
    preconditions: []
    postconditions: []
    appliedRules: []
    requiredPermissions: [PERM-CONTRACT-ANALYSIS]
    producedEvents: []
    queryReportRef: QR-CONTRACT-EXECUTION-001

  - id: Contract_QueryInvoicedUnreceived
    name: 查询已开票未收款合同
    ownerEntity: OBJ-CONTRACT-001
    behaviorType: QUERY
    triggerType: USER_ACTION
    preconditions: []
    postconditions: []
    appliedRules: []
    requiredPermissions: [PERM-CONTRACT-ANALYSIS]
    producedEvents: []
    queryReportRef: QR-CONTRACT-UNRECEIVED-001

  - id: Contract_GenerateDepartmentSummaryReport
    name: 生成部门合同执行汇总报表
    ownerEntity: OBJ-CONTRACT-001
    behaviorType: QUERY
    triggerType: USER_ACTION
    preconditions: []
    postconditions: []
    appliedRules: []
    requiredPermissions: [PERM-CONTRACT-REPORT]
    producedEvents: []
    queryReportRef: RPT-CONTRACT-DEPARTMENT-001
```

## 9.6  依赖与一致性约束

1. M7 `sourceObjects.objectRef` 必须引用M1已存在的聚合根；`entityPath`、参数字段、Join字段、条件字段和结果字段必须可解析到M1。
2. 每个M7对象必须填写唯一 `behaviorRef`，目标必须是M2 `behaviorType=QUERY` 的行为。
3. M2 `queryReportRef` 与M7 `behaviorRef` 必须双向一致。
4. 一个M7对象只能绑定一个M2行为，一个M2行为也只能绑定一个M7对象；任何一端重复引用均为错误。
5. 普通不依赖M7的M2查询行为可以不填写 `queryReportRef`。
6. M7不得出现 `ruleRefs`、`requiredPermissions`、`roleRefs`、`eventRefs`、`scenarioRefs` 或 `flowRefs` 等跨模型字段。
7. M7的权限不单独建模。访问控制继续由M2行为的 `requiredPermissions` 与M5维护，M7不感知角色和权限。
8. 查询条件和聚合表达式是查询定义的一部分，不允许为了复用而外挂M3规则。
9. 每个查询来源别名必须唯一，且必须有且仅有一个 `primary=true` 的主对象。
10. Join两端别名必须存在；`relationRef` 存在时必须引用M1聚合关联，条件表达式中的字段必须存在。
11. 参数名、结果列名、Join ID和条件ID必须在当前M7对象内唯一。
12. `parameterBindings` 必须覆盖参考SQL中使用的全部命名参数；不得存在未声明参数。
13. `resultMappings.resultColumn` 必须引用当前对象已定义的结果列；SQL返回列与结果映射必须一致。
14. `groupBy`、聚合结果和 `having` 必须在语义定义与参考SQL之间保持一致。
15. `objectType=REPORT` 必须填写 `reportOptions`；其他类型的 `reportOptions` 应为空。
16. 参考SQL允许使用实际物理表名，但不得成为唯一语义定义；数据库结构变化时应更新SQL，不得改变M7业务口径。
17. 两个或更多一对多来源同时参与聚合时，必须通过`preAggregation`分别按Join键汇总后再关联；禁止以`SUM(DISTINCT amount)`代替正确的预聚合。

---

# 第十章  传统需求覆盖度分析

## 10.1  与软件需求规格说明书的映射

| 需求维度 | 覆盖程度 | 承载模型或配套载体 |
|----------|----------|--------------------|
| 领域概念、数据及聚合边界 | 完整覆盖 | M1对象模型 |
| 原子功能、命令与查询入口 | 完整覆盖 | M2行为模型 |
| 业务规则与一致性约束 | 完整覆盖 | M1内置约束、`refRules`、`invariants` + M3规则模型 |
| 领域事件、消息载荷及订阅 | 完整覆盖 | ME事件模型 |
| 跨对象事件协同 | 完整覆盖 | M4场景模型 |
| 角色、权限与外部参与方 | 完整覆盖 | M5主体模型；权限通过M2行为授权 |
| 端到端业务协同流 | 完整覆盖 | M6 `COLLABORATION` 流程 |
| 人工审批流 | 完整覆盖 | M6 `APPROVAL` 流程 + M5角色 |
| 跨对象查询、统计分析 | 完整覆盖 | M7查询统计对象 + 一对一M2 `QUERY`行为 |
| 固定业务报表 | 完整覆盖 | M7 `REPORT`对象 + 一对一M2 `QUERY`行为 |
| 外部系统调用 | 主要语义覆盖 | M5 `ExternalEntity` + M2 `EXTERNAL_CALL`；详细协议契约可配套OpenAPI/AsyncAPI |
| 异常、驳回、退回和超时路径 | 主要语义覆盖 | M2前后置条件、M6分支与终止路径；技术重试和分布式补偿由实现规格承载 |
| 并发与事务控制 | 部分覆盖 | M1不变性描述业务边界；锁、隔离级别和事务策略由技术设计承载 |
| UI/UX、页面布局与交互细节 | 框架外 | 产品原型、交互说明、设计系统 |
| 性能、容量、可用性、安全基线等NFR | 框架外 | 非功能需求规格、SLA/SLO和安全规范 |
| 部署拓扑、运维、监控与灾备 | 框架外 | 架构设计、部署与运维规范 |
| BI数仓、自助分析和指标平台 | 明确排除 | 事实表、维度表、宽表、Cube、ETL及指标口径平台不属于M7 |

## 10.2  覆盖结论

对于CRM、资产管理、合同管理等中大型业务系统，本框架能够承载软件需求中的核心业务语义：领域对象及聚合边界、原子用例、业务规则、事件驱动协同、角色权限、端到端流程、审批流程、跨对象查询统计和固定报表。八个模型之间的引用关系还能形成从需求到设计及代码的可追踪链路。

本框架不应被视为一份完整SRS的唯一物理载体。完整交付仍应把八模型与UI原型、非功能需求、接口详细契约、数据迁移、部署运维和测试验收标准组合起来。换言之，八模型完整覆盖“系统做什么以及核心业务为何如此运转”，配套规格覆盖“界面如何呈现、达到什么质量目标以及如何部署运行”。

## 10.3  DDD与事件驱动架构体现

- M1以聚合根、不变性、值对象和聚合间ID引用落实DDD领域边界。
- M2把应用能力拆成原子行为，区分命令、查询、事件处理和外部调用。
- ME把已发生事实定义为独立事件契约，以生产者、载荷和订阅者建立异步边界。
- M4只描述跨对象的事件协同，避免把顺序流程与事件语义混合。
- M6承担显式流程控制，并通过角色、规则、行为、场景及子流程组合业务旅程。
- M7将跨聚合读模型与写侧聚合解耦，体现CQRS式读写职责分离，但不引入独立BI数据模型。

## 10.4  已知边界

- 不独立建模UI/UX、国际化、无障碍和终端适配。
- 不独立建模性能、容量、可靠性、安全合规等可量化质量属性。
- 不定义Saga补偿协议、消息中间件配置、事件存储或事件溯源实现。
- 不定义数据仓库、指标平台、即席分析或可视化仪表盘。
- 不替代OpenAPI、AsyncAPI、数据库迁移脚本、部署清单和验收测试用例。

---

# 第十一章  实施指南与最佳实践

## 11.1  建模顺序推荐

八个模型之间存在依赖，建议按以下顺序建模。M5分两次完善，因此阶段数多于模型数：

| 阶段 | 建模对象 | 说明 |
|------|----------|------|
| 1 | M1对象模型 | 识别聚合、属性、关联、内置约束、`refRules`和`invariants` |
| 2 | M5主体模型（参与方和角色） | 识别内部角色、外部参与方及职责边界，权限可稍后补齐 |
| 3 | M3规则模型 | 梳理跨对象、跨行为、事件驱动、外部决策或需要独立复用的规则 |
| 4 | M2行为模型 | 定义对象行为、规则调用、事件引用及查询行为入口 |
| 5 | M7查询统计与报表模型 | 在M1字段和M2查询行为稳定后，建立严格一对一查询报表定义 |
| 6 | ME事件模型 | 定义事件生产者、订阅者、载荷和顺序语义，构建事件链 |
| 7 | M5主体模型（权限） | 定义权限并绑定M2行为；M7不直接绑定权限 |
| 8 | M4场景模型 | 组装真实存在的跨对象事件协同链 |
| 9 | M6流程模型 | 定义端到端协同流和审批流，引用角色、行为、规则、场景及子流程 |

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
- [ ] 必填、唯一、类型、枚举和数据字典约束是否优先使用属性内置字段？
- [ ] 只依赖当前属性值的扩展约束是否放入refRules，并且表达式只引用value？
- [ ] 依赖同一聚合多个属性或子实体的约束是否放入invariants？

### M2 行为模型评审
- [ ] 每个行为是否真正原子化，只操作一个对象？
- [ ] 前置条件是否完整覆盖了行为可执行的业务前提？
- [ ] 后置状态变更是否完整描述了行为的全部副作用？
- [ ] producedEvents字段是否只包含事件ID引用（完整定义在事件模型中）？
- [ ] `queryReportRef`是否只出现在`behaviorType=QUERY`的行为上？
- [ ] 引用M7的查询行为是否与M7 `behaviorRef`双向一致且严格一对一？

### M3 规则模型评审
- [ ] 是否错误包含了可以由属性内置字段、refRules或invariants表达的对象内部规则？
- [ ] 每条规则是否至少涉及跨对象、跨行为、事件驱动、外部决策或独立复用中的一种？
- [ ] 规则表达式是否无副作用（不改变系统状态）？
- [ ] 规则版本是否独立管理，与行为版本解耦？

### ME 事件模型评审
- [ ] 每个事件是否有明确的单一生产者行为？
- [ ] 事件载荷是否包含订阅者所需的核心业务数据？
- [ ] 事件命名是否使用过去时态（如Order.Created而非Order.Create）？
- [ ] 订阅者行为或规则之间是否相互独立，无隐式依赖？
- [ ] 跨聚合属性新增、修改、删除或状态变化是否均已通过事件解耦，未由源行为直接写入目标聚合？
- [ ] 无条件下游操作是否由行为直接订阅，需要条件判断的下游操作是否由规则订阅并通过triggeredBehaviors触发行为？
- [ ] 对于AT_LEAST_ONCE和EXACTLY_ONCE语义，订阅者是否实现了幂等性？
- [ ] 事件链路是否清晰可追溯（生产者→事件→订阅者）？

### M4 场景模型评审
- [ ] 每个场景是否确实存在跨对象状态影响和事件解耦，而非普通顺序流程？
- [ ] 根集合是否使用event_scenarios，且不存在business_processes或use_cases层？
- [ ] 每个BEHAVIOR_CALL是否引用已存在的M2行为？
- [ ] 每个EVENT_EMIT是否引用已存在的ME事实事件，且紧跟对应生产行为？
- [ ] 每个RULE_EVALUATE是否引用已存在的M3规则？
- [ ] 场景是否未包含角色任务、审批活动、通用网关或端到端业务阶段？
- [ ] 场景前置/后置条件是否与M2行为的前后置条件一致？
- [ ] 事件链编排是否与事件模型中的订阅关系一致？

### M5 主体模型评审
- [ ] 外部系统是否定义了接口契约（协议、超时、重试）？
- [ ] ABAC条件是否覆盖了数据隔离需求（如多租户）？
- [ ] 角色继承关系是否符合最小权限原则？

### M6 流程模型评审
- [ ] 每条流程是否明确标记为COLLABORATION或APPROVAL？
- [ ] 是否有且仅有一个开始活动，并至少有一个结束活动？
- [ ] USER_TASK和APPROVAL_TASK是否只引用已存在的M5 roleId？
- [ ] 活动roleRef是否同时包含在流程roleRefs中？
- [ ] BEHAVIOR_CALL、SCENARIO_CALL和SUB_FLOW_CALL引用是否存在且类型正确？
- [ ] 协同流调用审批流时是否通过subFlowRef引用，而非复制审批活动？
- [ ] 网关是否至少两个分支、最多一个默认分支，且非默认分支存在判断条件？
- [ ] 审批流是否覆盖通过、驳回、退回等真实结果？
- [ ] 子流程调用图是否无循环？
- [ ] 所有非结束活动是否可从开始节点到达，并最终存在到达结束节点的路径？

### M7 查询统计与报表模型评审
- [ ] 每个对象是否仅直接引用M1对象/字段及唯一M2 `QUERY`行为？
- [ ] 是否未定义权限、角色、规则、事件、场景或流程引用？
- [ ] 是否有且仅有一个主查询来源，且所有来源别名和Join引用均有效？
- [ ] 参数、条件、结果列、分组、排序和分页是否完整表达业务查询口径？
- [ ] 多个一对多来源同时参与聚合时，是否先按关联键预聚合，避免行数膨胀和重复计数？
- [ ] `REPORT`是否定义固定列、分组/合计及导出格式，其他类型是否不携带`reportOptions`？
- [ ] 参考SQL的参数及结果映射是否与语义定义一致？
- [ ] M7是否保持业务查询边界，未引入事实表、维度表、宽表、Cube或ETL语义？

---

## 11.3  文件存储、导入与版本管理

平台源代码、数据库和模型文件统一位于 `onto-cs` 目录。导入外部模型文件夹时，平台必须将文件复制到托管项目目录，不直接修改外部源文件。目录约定如下：

```
onto-cs/
├── model/
│   └── {projectId}/
│       └── yaml/
│           ├── m1-object-model.yaml
│           ├── m2-behavior-model.yaml
│           ├── m3-rule-model.yaml
│           ├── me-event-model.yaml
│           ├── m4-scenario-model.yaml
│           ├── m5-actor-model.yaml
│           ├── m6-flow-model.yaml
│           ├── m7-report-model.yaml
│           └── manifest.json       # 可选，路径必须为相对路径
├── onto-cs.db                      # SQLite项目与应用元数据
└── ...                             # 后续平台源代码与工程文件
```

**版本管理约定**：

- 每个元文件内部维护自己的 `version` 字段
- 跨模型的破坏性变更（如实体删除、行为签名变更、事件载荷变更）需要在 `CHANGELOG.md` 中记录
- 规则模型和事件模型支持独立版本，允许热更新而不触发其他模型变更
- 事件载荷的字段变更需要特别注意向后兼容性，建议采用增量添加而非修改现有字段
- 导入时先校验八类固定文件；通过后复制到 `onto-cs/model/{projectId}/yaml/`
- 导入源目录仅作为输入，不与托管项目建立文件链接，也不在原目录回写
- `manifest.json` 为可选文件；缺失时按八个固定文件名识别，存在时不得保存绝对路径

---

## 11.4  工具链建议

| 工具场景 | 建议方案 |
|----------|----------|
| 模型编辑 | VS Code + YAML插件 + 自定义JSON Schema校验，保证模型文件的结构合规性 |
| 可视化 | 分别生成事件协同图和流程图；M6流程图应展示角色泳道、网关、子流程和审批结果 |
| 一致性检查 | 检查行为ownerEntity、事件生产消费、规则双向引用、M4场景链、M6流程引用以及M2/M7一对一绑定 |
| 代码生成 | 生成实体骨架、接口签名、事件DTO、权限枚举、查询DTO、Query Service及报表导出骨架 |
| 变更影响分析 | M1变更时分析M2、M3、ME、M4、M6和M7；M2查询行为变更时检查M7；M5角色变更时检查M6人工活动 |
| 模型验证CI | 校验事件链、流程可达性、角色引用、网关分支、子流程调用环、M7来源字段及M2/M7一对一关系 |
| 事件链追踪 | 开发工具支持从任意行为追踪其产生的事件链路，以及从任意事件追踪其生产者和所有订阅者 |
| 查询预览 | 根据M7生成参数表单和结果预览，并对参考SQL执行只读语法及结果映射检查 |

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
| M4 事件协同场景 | Event Choreography / Process Manager / Saga Coordinator |
| M5 主体模型 | Spring Security / Casbin / OPA Policy |
| M6 端到端协同流 | BPMN Process / Workflow Engine / Orchestration Service |
| M6 审批流 | Human Task Engine / Approval Workflow / BPMN User Task |
| M7 查询统计对象 | Query Service / Read Repository / CQRS Read Model / SQL Repository |
| M7 固定报表对象 | Report Application Service / Tabular Renderer / XLSX、CSV、PDF Exporter |
| M7 参考SQL | 可审查SQL模板 / Named Parameter Query / 数据库方言适配器 |

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
| 事件（Event） | 已经发生的业务事实通知，由生产者行为或规则触发，被订阅者行为或规则消费 |
| 事件链（Event Chain） | 由行为或规则产生事件，事件再触发行为或规则形成的异步处理链路 |
| 事件协同场景（Event Collaboration Scenario） | M4中跨对象的行为、事实事件、规则和目标行为语义链，不承载人工流程控制 |
| 端到端协同流（Collaboration Flow） | M6中跨业务阶段从开始到结束的完整业务流程，可调用审批子流程和事件协同场景 |
| 审批流（Approval Flow） | M6中由角色承担审批任务，并显式描述通过、驳回、退回及条件网关的流程 |
| 人工任务（Human Task） | 由M5角色承担的USER_TASK或APPROVAL_TASK，不直接绑定具体用户或Actor |
| 子流程（Sub Flow） | 被另一条M6流程通过subFlowRef调用的独立流程定义 |
| 查询报表对象（Query Report Object） | M7中定义跨对象查询、统计分析或固定报表业务语义的对象，与一个M2 QUERY行为严格一对一 |
| 统计查询（Statistical Query） | 在具体业务查询内部进行分组和聚合的M7对象，不是可独立复用的BI指标模型 |
| 固定业务报表（Fixed Business Report） | 具有预定义结果列、分组、合计和导出格式的M7 REPORT对象 |
| 参考SQL（Reference SQL） | M7语义定义的实现参考，通过命名参数和结果映射接受校验，但不是业务语义的唯一权威来源 |
| 生产者（Producer） | 触发事件的行为或规则，每个事件有且仅有一个生产者 |
| 订阅者（Subscriber） | 消费事件的行为或规则，一个事件可以有多个订阅者 |
| 事件载荷（Event Payload） | 事件携带的业务数据字段，供订阅者使用 |
| 事件顺序语义（Event Ordering） | 事件投递的保证级别：至少一次、恰好一次、尽力而为 |
| EDA（Event-Driven Architecture） | 事件驱动架构，通过消息事件实现组件间的松耦合 |
| RBAC | 基于角色的访问控制（Role-Based Access Control） |
| ABAC | 基于属性的访问控制（Attribute-Based Access Control），比RBAC更细粒度 |
| 幂等性（Idempotency） | 操作执行一次与执行多次产生相同结果的特性，分布式系统的关键要求 |
| 幂等键（Idempotency Key） | 用于保证操作幂等性的唯一标识，通常由业务字段组合而成 |
| AND-Join | 流程编排中等待多个并行分支全部完成的聚合节点 |
| XOR-Split | 流程编排中根据条件选择唯一一个分支执行的网关节点 |
| Dead Letter | 无法处理的消息被放入死信队列，等待人工干预或后续处理 |
| Dangling Reference | 悬空引用，模型中引用了不存在的目标，需通过CI校验防止 |
| OWL | 网络本体语言（Web Ontology Language），W3C标准的语义本体描述语言 |
| 消息中间件（Message Broker） | 实现事件传递的基础设施，如Kafka、RabbitMQ、Pulsar |
| 事件总线（Event Bus） | 应用内的事件分发机制，支持发布-订阅模式 |
| 事件溯源（Event Sourcing） | 将所有状态变更存储为事件序列，而非仅存储当前状态 |
| 数据冗余（Data Redundancy） | 在聚合内复制其他聚合的关键信息，避免跨聚合查询 |
| 最终一致性（Eventual Consistency） | 分布式系统中，数据在一段时间后达到一致状态 |
| 事务边界（Transaction Boundary） | 一个事务的范围，在DDD中通常是一个聚合根 |

---

*© 2026  Ontology-Driven Software Modeling Framework  v4.0*
