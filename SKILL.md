---
name: wenshu-semantic
description: 在业务代码仓库中为「问数」智能 BI 生成业务知识 Wiki 包（带索引目录的 md 词条：表/字段/术语/指标 + 评测用例），供导入问数。用户提到问数、业务知识、语义层、数据字典、wiki 导入导出时使用。
---

# wenshu-semantic · 业务知识 Wiki 包生成

你在一个**业务系统代码仓库**里运行。目标：从代码中提炼业务知识，产出「问数」可导入的 **Wiki 包**（`wenshu-wiki/` 目录，一篇知识 = 一个带 frontmatter 的 md 词条）。

问数是企业级对话式智能 BI：业务人员自然语言问数，系统基于语义层生成可信 SQL。问数以 **LLM Wiki** 方式存储业务知识：词条 + 自动索引目录，提问时只检索 top-k 相关词条注入上下文。因此你的产出要**短、准、可检索**：摘要进索引（检索权重高），正文放细节。

代码里通常**没有**指标口径与术语，但**有**表结构、字段、枚举、状态流转、模块划分——这些就是你要总结的业务知识。

## 工作步骤

1. **定位数据模型**：按优先级扫描 ORM 模型（SQLAlchemy/Django/Prisma/MyBatis XML）、migration/DDL、建表 SQL、数据字典文档、README。确定方言（mysql/postgres）与库名。
2. **抽表与字段**：每张表列 name/type/comment；枚举型字段（status/type/channel…）从常量、枚举类、注释、前端选项还原取值字典。
3. **写表级词条** `pages/table/<表名>.md`：这张表在业务里是什么（事实表/维表/主数据/流水）、数据从哪来、被哪些模块消费。1–3 句。
4. **写字段级词条** `pages/field/<表名>.<字段>.md`：业务含义 + 枚举字典 + 是否个人信息（需脱敏的标明，如手机号/身份证）。一句 + 字典。
5. **候选术语** `pages/term/<术语>.md`：领域名词 → 指标/表映射，只写代码能证实的，标 `source: 代码推断`、`confidence: medium`。
6. **候选指标** `pages/metric/<指标>.md`：从聚合逻辑（SUM/COUNT 查询、报表服务、BI 配置）反推，含口径 formula 与 notes。**禁止编造口径**；拿不准标 `confidence: low|medium`，导入后默认草稿，需管理员启用。
7. **评测用例**（可选但推荐）`cases.yml`：10–20 条 positive 用例（业务口语问题 + expect_intent/must_tables/must_contain）。危险类（danger）不要写——系统自带。
8. **交付说明**：向用户汇报覆盖了几张表/字段/术语/指标/用例，列低置信度项请人确认；导入方式：问数 → 业务知识 → 导入/导出（选 .md 多选）→ 预览 diff → 确认；cases.yml 走同页「导入 / 导出」的 knowledge 通道或评测草稿。

## md 词条格式（frontmatter 必需）

```md
---
id: table/orders            # table/ | field/ | term/ | metric/
kind: table
title: orders
summary: 订单主表：电商核心事实表，记录每笔下单；状态流转 PAID→SHIPPED→DONE。
keywords: [orders, 订单, PAID, region]
source: 代码推断            # 代码推断 | agent | 人工
confidence: medium          # high | medium | low
status: enabled             # enabled | draft（指标建议 draft）
---
# orders

订单主表，核心事实表。来源：models/order.py

## 字段要点
- status：PAID 已支付 / SHIPPED 已发货 / DONE 已完成 / REFUND 退款中（来源：STATUSES 常量）
- paid：实付金额；GMV 口径不用本字段，用 order_items.amount
```

字段词条示例：

```md
---
id: field/orders.status
kind: field
title: orders.status
summary: 订单状态字典：PAID/SHIPPED/DONE/REFUND
keywords: [orders, status, 状态]
source: 代码推断
confidence: high
status: enabled
---
# orders.status

订单状态字典：PAID 已支付 / SHIPPED 已发货 / DONE 已完成 / REFUND 退款中。来源：models/order.py:20
```

cases.yml 示例：

```yaml
cases:
  - case: gmv_last_30d
    kind: positive
    question: 近30天GMV是多少
    expect_intent: kpi
    must_tables: [orders, order_items]
    must_contain: ["status IN"]
```

## 硬规则

- 表名/字段名必须与真实库一致（导入后漂移巡检会告警）。
- 不写凭证、密钥、DSN。
- 口径类断言不得标 high confidence——口径必须由人确认。
- summary 控制在 1–2 句（进索引目录）；正文放证据与字典。
- 每条推断带证据：`来源：<文件>:<行号>` 或 `<常量/表/字段名>`。
- 产出后自检：每个 md 的 frontmatter 能被 `python -c "import yaml,re,sys; yaml.safe_load(re.match(r'^---\s*\n(.*?)\n---', open(sys.argv[1],encoding='utf-8').read(), re.S).group(1))" <file>` 解析。
