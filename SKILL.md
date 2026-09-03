---
name: wenshu-semantic
description: 在业务代码仓库中为「问数」智能 BI 生成业务说明包（表/字段业务说明、术语、候选指标），产出 knowledge.yaml 供导入。用户提到问数、业务说明包、语义层、数据字典导入导出时使用。
---

# wenshu-semantic · 业务说明包生成

你在一个**业务系统代码仓库**里运行。目标：从代码中提炼业务知识，产出「问数」可导入的**业务说明包**（`wenshu-knowledge/knowledge.yaml`）。

问数是企业级对话式智能 BI：业务人员用自然语言问数，系统基于语义层（指标口径/术语/表字段业务说明）生成可信 SQL。代码里通常**没有**指标口径与术语，但**有**表结构、字段、枚举、状态流转、模块划分——这些就是你要总结的业务知识。

## 工作步骤

1. **定位数据模型**：按优先级扫描 ORM 模型（SQLAlchemy/Django/Prisma/MyBatis XML）、migration/DDL、建表 SQL、数据字典文档、README。确定数据库方言（mysql/postgres）与库名。
2. **抽表与字段**：每张表列出 name/type/comment；枚举型字段（status/type/channel…）从常量、枚举类、注释、前端选项中还原**取值字典**。
3. **推断业务含义**（核心产出）：
   - 表级 `business`：这张表在业务里是什么（事实表/维表/主数据/流水），数据从哪来、被哪些模块消费。引用代码热度（被查询/写入的位置）作证据。
   - 字段级 `business`：业务含义 + 枚举字典 + 是否个人信息（需脱敏的标出来，如手机号/身份证）。
   - 每条推断在文本末尾带证据：`来源：<文件>:<行号>` 或 `来源：<表/字段/常量名>`。
4. **候选术语**：领域名词 → 指标/表的映射。只写代码能证实的（如报表字段名、页面文案），标 `source: 代码推断`、`confidence: medium`。
5. **候选指标**：从聚合逻辑（SUM/COUNT 查询、报表服务、BI 配置）反推指标草稿，含 display_name/formula/notes，一律 `confidence: medium|low`。**禁止编造口径**；拿不准的写进 notes 并标 low。
6. **产出文件**：`wenshu-knowledge/knowledge.yaml`，结构见下。可附 `wenshu-knowledge/docs/*.md` 人读文档（可选）。
7. **生成评测用例**（golden）：基于已挖出的指标/表，产出 10–20 条 `cases`：自然语言问题 + 期望（expect_intent / must_tables / must_contain）。问题用业务口语（含别名），期望必须与 formula/tables 一致；危险类用例（danger）不要写——系统自带。
8. **交付说明**：向用户汇报：覆盖了几张表、几条字段说明、几条候选术语/指标/用例；列出低置信度项请人确认；告知导入方式（问数 → 数据源 → 业务知识 → 导入/导出 → 预览 diff → 确认导入；指标与用例默认草稿，管理员启用后生效，用例进入 `wenshu eval`）。

## knowledge.yaml schema（version 1）

```yaml
version: 1
datasources:            # 可选，涉及的连接名
  - mall_mysql
tables:
  - table: orders       # 必须与库中真实表名一致
    business: 订单主表：电商核心事实表，记录每笔下单；状态流转 PAID→SHIPPED→DONE，REFUND 退款。来源：models/order.py:12
    source: agent       # agent | 代码推断 | 人工
    confidence: high    # high | medium | low
    fields:
      - name: status
        business: 订单状态字典：PAID 已支付 / SHIPPED 已发货 / DONE 已完成 / REFUND 退款中。来源：models/order.py:20
      - name: phone
        business: 手机号，个人信息，需脱敏 keep_last_4。来源：users 表
terms:                  # 业务名词 → 指标 id（或表名）
  - term: 单量
    maps_to: pay_orders
    source: 代码推断
metrics:                # 候选指标，导入后为草稿
  - metric: refund_rate
    display_name: 退款率
    formula: "COUNT(退款订单)/COUNT(支付订单)"
    notes: 代码推断草稿，待业务确认。来源：report/refund.py:44
    source: agent
    confidence: medium
cases:                  # 评测用例（positive），导入后为草稿，启用后进入 wenshu eval
  - case: gmv_last_30d
    kind: positive
    question: 近30天GMV是多少
    expect_intent: kpi
    must_tables: [orders, order_items]
    must_contain: ["status IN"]
  - case: sales_by_region
    kind: positive
    question: 近30天各地区销售额对比
    expect_intent: region
    must_tables: [orders]
    must_contain: ["region"]
```

## 硬规则

- 表名/字段名必须与真实库一致（导入会校验，未知表会告警）。
- 不写凭证、密钥、DSN。
- 不产出口径类断言为 high confidence——口径必须由人确认。
- 单表 business 控制在 1–3 句；字段 business 一句 + 字典。
- 产出后自检：`python -c "import yaml,sys;yaml.safe_load(open('wenshu-knowledge/knowledge.yaml',encoding='utf-8'))"` 能解析。

完整示例见同目录 `example.yaml`。
