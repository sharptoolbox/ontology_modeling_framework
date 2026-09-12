# 本体驱动的软件建模方案
**Ontology-Driven Software Modeling Framework**

> 完整建模规范 · 七大模型元文件 · 实施指南
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

## 2.1  设计目标

对象模型是整个本体体系的基础，负责描述业务域中的核心数据实体及其关系，对应领域驱动设计（DDD）中的领域模型层。其范围包含三个核心关切：

- **实体定义**：业务实体的属性、标识、生命周期状态
- **关联关系**：实体之间的双向语义关系（聚合、组合、关联、依赖）
- **参照完整性**：跨实体的完整性约束，内嵌于对象模型而非规则模型

---

## 2.2  模型元素规范

### 2.2.1  实体（Entity）

| 属性名 | 类型 | 说明 |
|--------|------|------|
| id | String / UUID | 实体唯一标识，建议使用UUID v4 |
| name | String | 实体业务名称（中文，需唯一） |
| alias | String | 英文标识符，用于代码映射 |
| description | String | 业务含义说明 |
| lifecycle | Enum[] | 实体生命周期状态列表，如 [草稿, 生效, 注销] |
| attributes | Attribute[] | 属性集合，见下方属性规范 |
| constraints | Constraint[] | 参照完整性约束集合 |
| tags | String[] | 分类标签，如 [核心域, 支撑域] |

### 2.2.2  属性（Attribute）

| 属性名 | 类型 | 说明 |
|--------|------|------|
| name | String | 属性名（英文camelCase） |
| label | String | 业务展示名（中文） |
| type | DataType | 基础类型：String / Integer / Decimal / Date / DateTime / Boolean / Enum / Reference |
| required | Boolean | 是否必填 |
| unique | Boolean | 是否唯一 |
| defaultValue | Any | 默认值 |
| enumValues | String[] | 当type=Enum时的枚举值列表 |
| refEntity | EntityRef | 当type=Reference时引用的实体 |
| refCardinality | Enum | 引用基数：ONE_TO_ONE / ONE_TO_MANY / MANY_TO_MANY |

### 2.2.3  关联关系（Relation）

关联关系在对象模型中独立定义，支持双向语义描述（参考OWL的ObjectProperty）：

| 属性名 | 类型 | 说明 |
|--------|------|------|
| id | String | 关系唯一标识 |
| type | Enum | AGGREGATION（聚合）/ COMPOSITION（组合）/ ASSOCIATION（关联）/ DEPENDENCY（依赖） |
| sourceEntity | EntityRef | 来源实体 |
| targetEntity | EntityRef | 目标实体 |
| sourceRole | String | 来源端角色名（如"所属订单"） |
| targetRole | String | 目标端角色名（如"订单明细"） |
| sourceCardinality | Enum | 来源端基数：ONE / ZERO_OR_ONE / MANY / ONE_OR_MORE |
| targetCardinality | Enum | 目标端基数（同上） |
| inverseOf | RelationRef | 反向关系引用（实现双向） |
| cascadeDelete | Boolean | 是否级联删除 |

### 2.2.4  参照完整性约束（ReferentialConstraint）

| 属性名 | 类型 | 说明 |
|--------|------|------|
| constraintType | Enum | FOREIGN_KEY / UNIQUE_COMPOSITE / CHECK / NOT_NULL |
| scope | Enum | 约束生效范围：ENTITY（实体内）/ RELATION（跨实体） |
| expression | String | 约束表达式，使用受限的谓词逻辑语法 |
| violationMessage | String | 违反约束时的业务错误消息 |
| enforcedAt | Enum | 执行时机：PERSIST（持久化前）/ QUERY（查询时） |

---

## 2.3  YAML 元文件模板

```yaml
# M1 对象模型元文件 - object-model.yaml
model_type: OBJECT
version: "1.0"
domain: "订单管理"

entities:
  - id: ORD-001
    name: 订单
    alias: Order
    lifecycle: [草稿, 待支付, 已支付, 已发货, 已完成, 已取消]
    attributes:
      - name: orderId
        label: 订单号
        type: String
        required: true
        unique: true
      - name: status
        label: 订单状态
        type: Enum
        enumValues: [草稿, 待支付, 已支付, 已发货, 已完成, 已取消]
      - name: totalAmount
        label: 订单总额
        type: Decimal
        required: true
    constraints:
      - constraintType: CHECK
        scope: ENTITY
        expression: "totalAmount >= 0"
        violationMessage: 订单总额不能为负数

  - id: ORD-002
    name: 订单明细
    alias: OrderItem
    attributes:
      - name: itemId
        label: 明细ID
        type: String
        required: true
        unique: true
      - name: productId
        label: 商品ID
        type: Reference
        refEntity: PRD-001
        refCardinality: ONE_TO_MANY
      - name: quantity
        label: 数量
        type: Integer
        required: true
      - name: unitPrice
        label: 单价
        type: Decimal
        required: true
    constraints:
      - constraintType: CHECK
        scope: ENTITY
        expression: "quantity > 0 AND unitPrice >= 0"
        violationMessage: 数量必须大于0，单价不能为负数

relations:
  - id: REL-001
    type: COMPOSITION
    sourceEntity: ORD-001
    targetEntity: ORD-002
    sourceRole: 所属订单
    targetRole: 订单明细
    sourceCardinality: ONE
    targetCardinality: ONE_OR_MORE
    inverseOf: REL-001-INV
    cascadeDelete: true
```

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

### 3.2.3  前置/后置条件（Condition / StateChange）

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
      - eventId: Order.PaymentConfirmed
        eventName: 订单支付已确认
        triggerCondition: "postcondition.success == true"
        payload: [orderId, totalAmount, paidAt, userId]
        ordering: EXACTLY_ONCE
    qualityAnnotation: QA-ORDER-001
    compensationRef: COMP-Order_ConfirmPayment

  - id: Inventory_DeductStock
    name: 扣减库存
    ownerEntity: INV-001
    behaviorType: COMMAND
    triggerType: EVENT
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
      - eventId: Inventory.StockDeducted
        eventName: 库存已扣减
        triggerCondition: "postcondition.success == true"
        payload: [productId, deductedQty, remainingQty]
        ordering: EXACTLY_ONCE
    compensationRef: COMP-Inventory_DeductStock
```

---

# 第四章  M3 规则模型

## 4.1  设计目标与边界

规则模型专注于可复用的解耦业务规则，是从行为逻辑中分离出来的独立关切。其核心价值在于：同一规则可以被多个行为引用，规则变更不影响行为定义。

> **重要边界说明**：参照完整性约束（外键约束、唯一约束等）不在规则模型中定义，而是归属M1对象模型。规则模型处理的是业务逻辑层面的约束，而非数据结构层面的约束。

---

## 4.2  规则分类体系

| 规则类型 | 说明 |
|----------|------|
| 验证规则（Validation Rule） | 判断输入数据是否符合业务要求，返回 true/false，不改变状态 |
| 计算规则（Calculation Rule） | 根据输入参数计算并返回结果值，如计算运费、折扣金额 |
| 推导规则（Derivation Rule） | 基于已知属性推导出其他属性值，如根据级别推导权限范围 |
| 转换规则（Transformation Rule） | 将一种数据格式转换为另一种，适用于外部系统集成场景 |
| 风控规则（Risk Rule） | 评估业务风险，返回风险等级或通过/拒绝决策（可接入外部规则引擎） |

---

## 4.3  模型元素规范

| 属性名 | 类型 | 说明 |
|--------|------|------|
| id | String | 规则唯一标识，建议格式：RULE-{Domain}-{Seq} |
| name | String | 规则业务名称 |
| ruleType | Enum | VALIDATION / CALCULATION / DERIVATION / TRANSFORMATION / RISK |
| description | String | 规则的业务逻辑说明 |
| inputParams | Param[] | 输入参数定义（名称、类型、来源实体字段） |
| outputType | DataType | 返回值类型 |
| expression | String | 规则表达式（支持伪代码或DSL，不限定具体语言） |
| reusedBy | BehaviorRef[] | 引用本规则的行为列表（反向追踪） |
| externalEngine | String | 若委托外部规则引擎，填写引擎名称（如 Drools, OPA） |
| version | String | 规则版本，支持规则的独立版本管理 |

---

## 4.4  YAML 元文件模板

```yaml
# M3 规则模型元文件 - rule-model.yaml
model_type: RULE
version: "1.0"
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
      - name: orderAmount
        type: Decimal
        sourceField: Order.totalAmount
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
      - name: destRegion
        type: String
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
      - name: paymentAmount
        type: Decimal
      - name: deviceFingerprint
        type: String
    outputType: Enum  # PASS / REVIEW / REJECT
    externalEngine: "RiskEngineService"
    version: "3.1"
```

---

# 第五章  ME 事件模型

## 5.1  设计目标与边界

事件模型是从行为模型中独立出来的一等公民，专注于定义业务事件及其在系统中的传播路径。其核心价值在于：

- **松耦合架构**：事件生产者与订阅者之间无直接依赖，通过事件中介实现解耦
- **事件链可视化**：通过 `producerBehaviorRef` 和 `subscriberBehaviorRefs` 清晰表达事件驱动的业务流程
- **异步处理支持**：事件天然支持异步处理模式，适合长流程和跨服务协作
- **可追溯性**：每个事件都有明确的生产者和订阅者，便于业务流程追踪和问题排查

> **关键设计决策**：事件不再内嵌于行为模型的 `producedEvents` 字段中，而是作为独立模型存在。行为模型中的 `producedEvents` 仅保留事件ID引用，事件的完整定义（包括生产者、订阅者、载荷）统一在事件模型中维护。

---

## 5.2  事件模型的核心关切

| 关切维度 | 说明 |
|----------|------|
| 事件定义 | 业务事件的唯一标识、名称、业务含义 |
| 生产者 | 哪个行为触发了该事件（单一生产者原则） |
| 订阅者 | 哪些行为订阅了该事件（支持多个订阅者） |
| 事件载荷 | 事件携带的业务数据字段，供订阅者使用 |
| 顺序语义 | 事件的投递保证级别（至少一次、恰好一次、尽力而为） |
| 事件链构建 | 通过生产者和订阅者引用，构建完整的事件驱动流程链路 |

---

## 5.3  事件链的构建机制

事件模型通过以下机制构建完整的事件链：

### 5.3.1  单一生产者原则

每个事件有且仅有一个生产者行为（`producerBehaviorRef`），这确保了事件来源的明确性和可追溯性。生产者行为在执行成功后触发事件发布。

```
行为A (生产者) --触发--> 事件X --订阅--> 行为B (订阅者1)
                                  └--订阅--> 行为C (订阅者2)
```

### 5.3.2  多订阅者支持

一个事件可以被多个行为订阅（`subscriberBehaviorRefs`），实现一对多的事件广播模式。订阅者之间相互独立，互不影响。

### 5.3.3  事件链的传递

当订阅者行为执行后，可能产生新的事件，从而形成事件链：

```
合同录入行为 --触发--> Contract.Created事件 --订阅--> 付款条款批量创建行为
                                                    └--触发--> PaymentTerm.BatchCreated事件
```

通过这种方式，复杂的业务流程被分解为一系列松耦合的事件驱动步骤，每个步骤都可以独立演进和扩展。

### 5.3.4  跨对象协作

事件模型天然支持跨对象的业务协作。例如：

- 订单对象的"支付确认"行为触发 `Order.PaymentConfirmed` 事件
- 库存对象的"扣减库存"行为订阅该事件
- 通知对象的"发送支付成功通知"行为也订阅该事件

这种模式避免了订单对象直接调用库存和通知对象的方法，实现了真正的松耦合。

---

## 5.4  模型元素规范

### 5.4.1  事件（Event）

| 属性名 | 类型 | 说明 |
|--------|------|------|
| eventId | String | 事件唯一标识，建议格式：{Entity}.{State}_{Past}，如 Order.PaymentConfirmed |
| eventName | String | 事件业务名称（中文） |
| description | String | 事件的业务含义说明 |
| producerBehaviorRef | BehaviorRef | 生产该事件的行为引用（单一生产者） |
| producerEntityRef | EntityRef | 生产者行为所属的实体引用（用于可视化） |
| triggerCondition | String | 触发事件的条件表达式，通常为 "postcondition.success == true" |
| payload | PayloadField[] | 事件携带的数据字段列表 |
| subscriberBehaviorRefs | BehaviorRef[] | 订阅该事件的行为引用列表（支持多个订阅者） |
| ordering | Enum | 事件顺序语义：AT_LEAST_ONCE（至少一次）/ EXACTLY_ONCE（恰好一次）/ BEST_EFFORT（尽力而为） |
| version | String | 事件版本，支持事件schema的演进 |

### 5.4.2  事件载荷字段（PayloadField）

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
- [ ] 所有实体是否有明确的生命周期状态定义？
- [ ] 关联关系的基数是否符合业务实际？
- [ ] 参照完整性约束是否覆盖了所有跨实体约束？
- [ ] 是否存在循环依赖的实体关系？（如有，是否合理）
- [ ] 实体alias是否与代码命名保持一致？

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
| M1 实体 | JPA Entity / Prisma Model / Mongoose Schema |
| M1 关联关系 | ORM关联配置 / 数据库外键 |
| M2 行为（COMMAND） | Application Service Method / Use Case Handler |
| M2 行为（QUERY） | Query Service / Repository Query Method |
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

---

*© 2026  Ontology-Driven Software Modeling Framework  v1.0*
