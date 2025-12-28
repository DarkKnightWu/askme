# 用友 NC 财务系统 SQLBot 调优指南

> 本指南详细说明如何针对用友 NC/NCC/BIP 财务系统的后台数据库，在 SQLBot 中进行专业化配置，以提升 AI 问答的准确度。

---

## 目录

1. [NC 财务系统数据库特点分析](#1-nc-财务系统数据库特点分析)
2. [数据库连接与元数据同步](#2-数据库连接与元数据同步)
3. [核心业务表描述配置](#3-核心业务表描述配置)
4. [财务术语库配置](#4-财务术语库配置)
5. [SQL 示例库配置](#5-sql-示例库配置)
6. [自定义提示词配置](#6-自定义提示词配置)
7. [表关系配置](#7-表关系配置)
8. [常见问题场景优化](#8-常见问题场景优化)
9. [附录：NC 常用表速查](#附录nc-常用表速查)

---

## 1. NC 财务系统数据库特点分析

### 1.1 数据库架构层次

用友 NC 系统通常采用三层数据架构：

| 层次 | 说明 | 特点 |
| :--- | :--- | :--- |
| **ODS 层** | 原始数据层 | 包含所有原始字段，保留业务系统原始数据 |
| **DW 层** | 数据仓库层 | 经过清洗和处理的宽表，用于分析 |
| **DIM 层** | 维度数据层 | 参照数据、编码表等维度信息 |

> **建议**：在 SQLBot 中主要使用 **DW 层** 和 **DIM 层** 的数据，避免直接使用 ODS 层原始表。

### 1.2 表命名规范

NC 系统表名采用 `模块前缀_业务名称` 的格式：

| 模块前缀 | 业务模块 | 示例表名 |
| :--- | :--- | :--- |
| `gl_` | 总账（General Ledger） | `gl_voucher`, `gl_detail` |
| `ap_` | 应付（Accounts Payable） | `ap_payplan`, `ap_paybill` |
| `ar_` | 应收（Accounts Receivable） | `ar_recbill`, `ar_recplan` |
| `fa_` | 固定资产（Fixed Assets） | `fa_card`, `fa_assetreduce` |
| `bd_` | 基础数据（Base Data） | `bd_accsubj`, `bd_stordoc` |
| `po_` | 采购（Purchase Order） | `po_praybill`, `po_order` |
| `so_` | 销售（Sales Order） | `so_saleorder`, `so_invoiceh` |
| `ic_` | 库存（Inventory Control） | `ic_general_b`, `ic_freeze` |
| `sm_` | 系统管理（System Management） | `sm_busicenter`, `sm_user` |
| `pub_` | 公共模块 | `pub_wf_def`, `pub_billtemplet` |
| `md_` | 元数据（Metadata） | `md_class`, `md_table` |

### 1.3 字段命名规范

| 前缀/后缀 | 含义 | 示例 |
| :--- | :--- | :--- |
| `pk_` | 主键（Primary Key） | `pk_voucher`, `pk_detail` |
| `fk_` | 外键（Foreign Key） | `fk_org`, `fk_accsubj` |
| `dr` | 删除标记（Delete Record） | `dr = 0` 表示未删除 |
| `ts` | 时间戳（Timestamp） | 记录修改时间 |
| `def1` ~ `def50` | 自定义字段 | 用户扩展字段 |
| `_b` | 表体（Body） | `gl_detail_b` 表示凭证分录子表 |
| `_h` | 表头（Head） | `so_invoiceh` 表示销售发票头表 |

### 1.4 常用业务字段

| 字段名 | 含义 | 取值说明 |
| :--- | :--- | :--- |
| `pk_org` | 组织主键 | 财务组织的唯一标识 |
| `pk_group` | 集团主键 | 集团法人的唯一标识 |
| `cyear` | 会计年度 | 四位数字，如 2024 |
| `cmonth` | 会计期间 | 1-12 或 00-12（含调整期） |
| `vouchkind` | 凭证类别 | 记账凭证、收款凭证、付款凭证等 |
| `billstatus` | 单据状态 | 0=保存, 1=已提交, 2=审核中, 3=已审核 |
| `balanorient` | 科目方向 | 0=借方, 1=贷方 |
| `debitamount` | 借方金额 | 原币借方发生额 |
| `creditamount` | 贷方金额 | 原币贷方发生额 |
| `localdebitamount` | 本位币借方 | 本位币借方金额 |
| `localcreditamount` | 本位币贷方 | 本位币贷方金额 |

---

## 2. 数据库连接与元数据同步

### 2.1 数据源配置

在 SQLBot 中添加 NC 数据源时的关键配置：

```yaml
数据源名称: NC财务系统-生产库
数据库类型: Oracle / SQL Server / PostgreSQL  # 根据实际选择
数据库地址: <NC数据库服务器地址>
端口: 1521 / 1433 / 5432
数据库名: ncprod / nccdb
用户名: <只读账号>  # 强烈建议使用只读账号
密码: <密码>
```

> **安全建议**：为 SQLBot 创建专用的**只读数据库账号**，仅授予 SELECT 权限。

### 2.2 元数据同步策略

由于 NC 系统表数量众多（通常 500+），建议**选择性同步**：

1. **核心业务表**：总账、应收、应付、固定资产等核心表
2. **基础档案表**：科目、组织、供应商、客户等
3. **排除系统表**：排除 `md_`、`pub_`、`sm_` 等系统管理表

### 2.3 推荐同步的表清单

```
# 总账模块
gl_voucher          # 凭证主表
gl_detail           # 凭证分录表
gl_balance          # 总账余额表
gl_freevalue        # 辅助核算值
gl_accass           # 辅助核算余额

# 科目与组织
bd_accsubj          # 会计科目表
bd_accassitem       # 辅助核算项
org_orgs            # 组织表
org_financeorg      # 财务组织

# 应收应付
ap_payplan          # 付款计划
ap_paybill          # 付款单
ar_recplan          # 收款计划
ar_recbill          # 收款单

# 固定资产
fa_card             # 资产卡片
fa_depreciation     # 折旧信息
fa_assetreduce      # 减值信息

# 基础档案
bd_supplier         # 供应商
bd_customer         # 客户
bd_bankaccbas       # 银行账户
bd_currtype         # 币种
bd_invcl            # 存货分类
bd_invbasdoc        # 存货档案
```

---

## 3. 核心业务表描述配置

### 3.1 总账模块表描述

#### gl_voucher（凭证主表）

```
表名: gl_voucher
业务描述: 记账凭证主表，存储凭证头信息。每条记录代表一张完整的记账凭证。

关键字段说明:
- pk_voucher: 凭证主键，凭证的唯一标识
- pk_org: 记账组织主键，关联 org_orgs 表
- pk_accountingbook: 核算账簿主键
- vouchkind: 凭证类别（记账凭证/收款凭证/付款凭证）
- num: 凭证号，格式通常为"类别-序号"
- prepareddate: 制单日期
- signdate: 签字日期
- tallydate: 记账日期
- cyear: 会计年度（4位数字）
- cmonth: 会计期间（1-12）
- vouchstatus: 凭证状态（0=暂存, 1=提交, 2=审核, 3=记账）
- dr: 删除标记，0=有效，非0=已删除

注意事项:
- 查询有效凭证时必须加 dr = 0 条件
- 查询已记账凭证需加 vouchstatus >= 3
- 通常需要关联 gl_detail 获取分录明细
```

#### gl_detail（凭证分录表）

```
表名: gl_detail
业务描述: 凭证分录表（凭证明细），存储凭证的借贷分录。与 gl_voucher 是多对一关系。

关键字段说明:
- pk_detail: 分录主键
- pk_voucher: 关联的凭证主键（外键）
- pk_accsubj: 会计科目主键，关联 bd_accsubj 表
- subjcode: 科目编码
- subjname: 科目名称
- balanorient: 科目方向，0=借方，1=贷方
- debitamount: 原币借方金额
- creditamount: 原币贷方金额
- localdebitamount: 本位币借方金额
- localcreditamount: 本位币贷方金额
- explanation: 摘要
- pk_currtype: 币种主键
- pk_checktype: 结算方式
- checkno: 结算号

注意事项:
- 借贷金额是互斥的，借方有值则贷方为0，反之亦然
- 金额字段可能为 NULL，计算时需用 COALESCE 处理
- 辅助核算信息需关联 gl_freevalue 表
```

#### gl_balance（总账余额表）

```
表名: gl_balance
业务描述: 科目余额表，存储各科目期初、本期发生、期末余额等数据。

关键字段说明:
- pk_org: 组织主键
- pk_accsubj: 科目主键
- cyear: 会计年度
- cmonth: 会计期间
- beginbalance: 期初余额
- monthdebit: 本期借方发生额
- monthcredit: 本期贷方发生额
- yeardebit: 本年累计借方
- yearcredit: 本年累计贷方
- endbalance: 期末余额

计算公式:
- 期末余额 = 期初余额 + 本期借方 - 本期贷方（资产类）
- 期末余额 = 期初余额 - 本期借方 + 本期贷方（负债/权益类）
```

### 3.2 科目与组织表描述

#### bd_accsubj（会计科目表）

```
表名: bd_accsubj
业务描述: 会计科目档案表，存储科目编码、名称、类型等信息。

关键字段说明:
- pk_accsubj: 科目主键
- accsubjcode: 科目编码（如 1001, 1001.01）
- accsubjname: 科目名称
- pk_org: 所属组织
- subjlevel: 科目级次（1=一级科目，2=二级科目，以此类推）
- balanorient: 余额方向（0=借方，1=贷方）
- isskillsubj: 是否启用辅助核算（Y/N）
- asssubj: 辅助项编码组合
- enablestate: 启用状态（1=启用，2=停用）
- isleaf: 是否末级科目（Y=末级，可做账；N=非末级）

注意事项:
- 只有末级科目（isleaf='Y'）才能进行账务处理
- 科目编码层级通过长度或分隔符判断（如 4-2-2 格式）
- 辅助核算科目需关联 bd_accassitem 表
```

#### org_orgs（组织表）

```
表名: org_orgs
业务描述: 组织机构表，存储集团及各级组织信息。

关键字段说明:
- pk_org: 组织主键
- code: 组织编码
- name: 组织名称
- pk_group: 所属集团
- pk_father: 上级组织
- orglevel: 组织层级
- enablestate: 启用状态

注意事项:
- 财务核算通常使用 org_financeorg 表中的财务组织
- 需要关联获取组织的中文名称
```

### 3.3 应收应付表描述

#### ap_paybill（付款单）

```
表名: ap_paybill
业务描述: 付款单主表，记录企业对供应商的付款信息。

关键字段说明:
- pk_paybill: 付款单主键
- pk_org: 付款组织
- billno: 单据编号
- pk_supplier: 供应商主键
- billstatus: 单据状态（1=保存, 2=审核, 3=确认）
- paydate: 付款日期
- payamount: 付款金额
- pk_currtype: 币种
- pk_balatype: 结算方式（现金/银行等）

注意事项:
- 有效付款单需判断 billstatus >= 2（已审核）
- 付款明细需关联 ap_paybill_b 表
```

#### ar_recbill（收款单）

```
表名: ar_recbill
业务描述: 收款单主表，记录企业向客户的收款信息。

关键字段说明:
- pk_recbill: 收款单主键
- pk_org: 收款组织
- billno: 单据编号
- pk_customer: 客户主键
- billstatus: 单据状态
- recdate: 收款日期
- recamount: 收款金额
- pk_currtype: 币种

注意事项:
- 有效收款单需判断 billstatus >= 2（已审核）
- 收款明细需关联 ar_recbill_b 表
```

---

## 4. 财务术语库配置

### 4.1 核心财务术语

以下是 NC 财务系统常用的业务术语配置：

#### 会计科目类术语

```yaml
术语: 资产类科目
同义词: 资产科目
描述: |
  科目编码以 1 开头的科目，包括现金、银行存款、应收账款、存货、固定资产等。
  判断条件：bd_accsubj.accsubjcode LIKE '1%'
  余额方向：借方（balanorient = 0）
生效数据源: NC财务系统

术语: 负债类科目
同义词: 负债科目
描述: |
  科目编码以 2 开头的科目，包括短期借款、应付账款、预收账款等。
  判断条件：bd_accsubj.accsubjcode LIKE '2%'
  余额方向：贷方（balanorient = 1）
生效数据源: NC财务系统

术语: 权益类科目
同义词: 所有者权益科目、净资产科目
描述: |
  科目编码以 3 开头的科目，包括实收资本、资本公积、盈余公积、未分配利润等。
  判断条件：bd_accsubj.accsubjcode LIKE '3%'
  余额方向：贷方（balanorient = 1）
生效数据源: NC财务系统

术语: 成本类科目
同义词: 成本科目
描述: |
  科目编码以 4 开头的科目，包括生产成本、制造费用等。
  判断条件：bd_accsubj.accsubjcode LIKE '4%'
生效数据源: NC财务系统

术语: 损益类科目
同义词: 损益科目
描述: |
  科目编码以 5 或 6 开头的科目。
  收入类以 5 开头：主营业务收入、其他业务收入等
  费用类以 6 开头：管理费用、销售费用、财务费用等
  判断条件：bd_accsubj.accsubjcode LIKE '5%' OR bd_accsubj.accsubjcode LIKE '6%'
生效数据源: NC财务系统
```

#### 凭证类术语

```yaml
术语: 已记账凭证
同义词: 入账凭证、过账凭证
描述: |
  凭证状态为已记账的凭证，可用于财务报表统计。
  判断条件：gl_voucher.vouchstatus >= 3
  注意：同时需要 dr = 0 排除已删除凭证
生效数据源: NC财务系统

术语: 有效凭证
同义词: 正常凭证
描述: |
  未被删除的凭证记录。
  判断条件：gl_voucher.dr = 0
生效数据源: NC财务系统

术语: 期初余额
同义词: 年初余额、初始余额
描述: |
  会计期间初的科目余额。
  数据来源：gl_balance.beginbalance
  如查询年初余额，使用 cmonth = 0 或 cmonth = 1 的期初值
生效数据源: NC财务系统

术语: 本期发生额
同义词: 当期发生额、月发生额
描述: |
  当期借方和贷方发生额的合计。
  借方发生额：gl_balance.monthdebit 或 SUM(gl_detail.localdebitamount)
  贷方发生额：gl_balance.monthcredit 或 SUM(gl_detail.localcreditamount)
生效数据源: NC财务系统

术语: 本年累计
同义词: 年累计、年度累计
描述: |
  从年初到当前期间的累计发生额。
  借方累计：gl_balance.yeardebit
  贷方累计：gl_balance.yearcredit
生效数据源: NC财务系统
```

#### 应收应付类术语

```yaml
术语: 应收账款余额
同义词: 应收余额、客户欠款
描述: |
  应收账款科目的期末余额，表示客户尚未支付的款项。
  数据来源：gl_balance 表，科目编码 1122（应收账款）
  或通过 ar_recbill 收款单统计未核销金额
生效数据源: NC财务系统

术语: 应付账款余额
同义词: 应付余额、欠供应商款项
描述: |
  应付账款科目的期末余额，表示企业尚未支付给供应商的款项。
  数据来源：gl_balance 表，科目编码 2202（应付账款）
  或通过 ap_paybill 付款单统计未付金额
生效数据源: NC财务系统

术语: 账龄分析
同义词: 账龄、逾期分析
描述: |
  按照账款发生时间进行的分类统计，通常分为：
  - 0-30天
  - 31-60天
  - 61-90天
  - 91-180天
  - 180天以上
  计算方式：当前日期 - 单据日期
生效数据源: NC财务系统
```

#### 固定资产类术语

```yaml
术语: 固定资产原值
同义词: 资产原值、入账价值
描述: |
  固定资产购买或建造时的初始成本。
  数据来源：fa_card.originalvalue
生效数据源: NC财务系统

术语: 累计折旧
同义词: 已提折旧、折旧累计额
描述: |
  固定资产从入账到当前累计计提的折旧金额。
  数据来源：fa_card.accudepre 或 fa_depreciation 折旧表
生效数据源: NC财务系统

术语: 固定资产净值
同义词: 资产净值、账面净值
描述: |
  固定资产原值减去累计折旧后的金额。
  计算公式：净值 = 原值 - 累计折旧 - 减值准备
  数据来源：fa_card.netvalue 或计算得出
生效数据源: NC财务系统
```

### 4.2 组织与期间术语

```yaml
术语: 当期
同义词: 本期、本月、当月
描述: |
  当前会计期间，由 cyear 和 cmonth 共同确定。
  例如：cyear = 2024 AND cmonth = 12 表示2024年12月
生效数据源: NC财务系统

术语: 上期
同义词: 上月、前期
描述: |
  当前期间的前一个会计期间。
  注意跨年处理：如当期是1月，则上期是上一年的12月
生效数据源: NC财务系统

术语: 本年
同义词: 当年、年度
描述: |
  当前会计年度内的所有期间。
  条件：cyear = <当前年份> AND cmonth BETWEEN 1 AND 12
生效数据源: NC财务系统

术语: 财务组织
同义词: 核算组织、账务组织
描述: |
  进行独立财务核算的组织单元，可能是法人公司或核算单位。
  数据来源：org_financeorg 或 org_orgs 表
  关联字段：pk_org
生效数据源: NC财务系统
```

---

## 5. SQL 示例库配置

### 5.1 凭证查询示例

```yaml
问题: 查询本月所有已记账凭证
SQL: |
  SELECT 
      v.num AS 凭证号,
      v.prepareddate AS 制单日期,
      d.subjcode AS 科目编码,
      d.subjname AS 科目名称,
      d.explanation AS 摘要,
      COALESCE(d.localdebitamount, 0) AS 借方金额,
      COALESCE(d.localcreditamount, 0) AS 贷方金额
  FROM gl_voucher v
  INNER JOIN gl_detail d ON v.pk_voucher = d.pk_voucher
  WHERE v.dr = 0 
    AND d.dr = 0
    AND v.vouchstatus >= 3
    AND v.cyear = EXTRACT(YEAR FROM CURRENT_DATE)
    AND v.cmonth = EXTRACT(MONTH FROM CURRENT_DATE)
  ORDER BY v.num, d.pk_detail
数据源: NC财务系统

问题: 查询指定科目的凭证明细
SQL: |
  SELECT 
      v.prepareddate AS 制单日期,
      v.num AS 凭证号,
      d.explanation AS 摘要,
      COALESCE(d.localdebitamount, 0) AS 借方金额,
      COALESCE(d.localcreditamount, 0) AS 贷方金额,
      s.accsubjcode AS 科目编码,
      s.accsubjname AS 科目名称
  FROM gl_voucher v
  INNER JOIN gl_detail d ON v.pk_voucher = d.pk_voucher
  INNER JOIN bd_accsubj s ON d.pk_accsubj = s.pk_accsubj
  WHERE v.dr = 0 
    AND d.dr = 0
    AND v.vouchstatus >= 3
    AND s.accsubjcode LIKE '1001%'  -- 可替换为具体科目
  ORDER BY v.prepareddate, v.num
数据源: NC财务系统

问题: 统计本月凭证数量和金额
SQL: |
  SELECT 
      COUNT(DISTINCT v.pk_voucher) AS 凭证数量,
      SUM(COALESCE(d.localdebitamount, 0)) AS 借方合计,
      SUM(COALESCE(d.localcreditamount, 0)) AS 贷方合计
  FROM gl_voucher v
  INNER JOIN gl_detail d ON v.pk_voucher = d.pk_voucher
  WHERE v.dr = 0 
    AND d.dr = 0
    AND v.vouchstatus >= 3
    AND v.cyear = EXTRACT(YEAR FROM CURRENT_DATE)
    AND v.cmonth = EXTRACT(MONTH FROM CURRENT_DATE)
数据源: NC财务系统
```

### 5.2 余额查询示例

```yaml
问题: 查询某科目期末余额
SQL: |
  SELECT 
      s.accsubjcode AS 科目编码,
      s.accsubjname AS 科目名称,
      b.cyear AS 年度,
      b.cmonth AS 期间,
      b.beginbalance AS 期初余额,
      b.monthdebit AS 本期借方,
      b.monthcredit AS 本期贷方,
      b.endbalance AS 期末余额
  FROM gl_balance b
  INNER JOIN bd_accsubj s ON b.pk_accsubj = s.pk_accsubj
  WHERE b.cyear = EXTRACT(YEAR FROM CURRENT_DATE)
    AND b.cmonth = EXTRACT(MONTH FROM CURRENT_DATE)
    AND s.accsubjcode = '1001'  -- 银行存款科目
    AND s.dr = 0
数据源: NC财务系统

问题: 查询各一级科目余额汇总
SQL: |
  SELECT 
      SUBSTR(s.accsubjcode, 1, 4) AS 科目大类,
      s.accsubjname AS 科目名称,
      SUM(b.endbalance) AS 期末余额合计
  FROM gl_balance b
  INNER JOIN bd_accsubj s ON b.pk_accsubj = s.pk_accsubj
  WHERE b.cyear = EXTRACT(YEAR FROM CURRENT_DATE)
    AND b.cmonth = EXTRACT(MONTH FROM CURRENT_DATE)
    AND s.subjlevel = 1  -- 一级科目
    AND s.dr = 0
  GROUP BY SUBSTR(s.accsubjcode, 1, 4), s.accsubjname
  ORDER BY 科目大类
数据源: NC财务系统

问题: 查询资产负债率
SQL: |
  WITH asset_total AS (
      SELECT SUM(b.endbalance) AS total_asset
      FROM gl_balance b
      INNER JOIN bd_accsubj s ON b.pk_accsubj = s.pk_accsubj
      WHERE b.cyear = EXTRACT(YEAR FROM CURRENT_DATE)
        AND b.cmonth = EXTRACT(MONTH FROM CURRENT_DATE)
        AND s.accsubjcode LIKE '1%'  -- 资产类
        AND s.isleaf = 'Y'
        AND s.dr = 0
  ),
  liability_total AS (
      SELECT SUM(b.endbalance) AS total_liability
      FROM gl_balance b
      INNER JOIN bd_accsubj s ON b.pk_accsubj = s.pk_accsubj
      WHERE b.cyear = EXTRACT(YEAR FROM CURRENT_DATE)
        AND b.cmonth = EXTRACT(MONTH FROM CURRENT_DATE)
        AND s.accsubjcode LIKE '2%'  -- 负债类
        AND s.isleaf = 'Y'
        AND s.dr = 0
  )
  SELECT 
      a.total_asset AS 资产总额,
      l.total_liability AS 负债总额,
      ROUND(l.total_liability / NULLIF(a.total_asset, 0) * 100, 2) AS 资产负债率
  FROM asset_total a, liability_total l
数据源: NC财务系统
```

### 5.3 应收应付示例

```yaml
问题: 查询应收账款账龄分析
SQL: |
  SELECT 
      c.name AS 客户名称,
      COUNT(*) AS 单据数量,
      SUM(CASE WHEN CURRENT_DATE - r.billdate <= 30 THEN r.recamount ELSE 0 END) AS "0-30天",
      SUM(CASE WHEN CURRENT_DATE - r.billdate BETWEEN 31 AND 60 THEN r.recamount ELSE 0 END) AS "31-60天",
      SUM(CASE WHEN CURRENT_DATE - r.billdate BETWEEN 61 AND 90 THEN r.recamount ELSE 0 END) AS "61-90天",
      SUM(CASE WHEN CURRENT_DATE - r.billdate > 90 THEN r.recamount ELSE 0 END) AS "90天以上",
      SUM(r.recamount) AS 合计金额
  FROM ar_recbill r
  INNER JOIN bd_customer c ON r.pk_customer = c.pk_customer
  WHERE r.dr = 0
    AND r.billstatus >= 2  -- 已审核
  GROUP BY c.name
  ORDER BY 合计金额 DESC
数据源: NC财务系统

问题: 查询本月付款明细
SQL: |
  SELECT 
      p.billno AS 单据编号,
      p.paydate AS 付款日期,
      s.name AS 供应商名称,
      p.payamount AS 付款金额,
      cur.name AS 币种
  FROM ap_paybill p
  INNER JOIN bd_supplier s ON p.pk_supplier = s.pk_supplier
  LEFT JOIN bd_currtype cur ON p.pk_currtype = cur.pk_currtype
  WHERE p.dr = 0
    AND p.billstatus >= 2
    AND EXTRACT(YEAR FROM p.paydate) = EXTRACT(YEAR FROM CURRENT_DATE)
    AND EXTRACT(MONTH FROM p.paydate) = EXTRACT(MONTH FROM CURRENT_DATE)
  ORDER BY p.paydate DESC
数据源: NC财务系统
```

### 5.4 固定资产示例

```yaml
问题: 查询固定资产明细及净值
SQL: |
  SELECT 
      fa.assetcode AS 资产编码,
      fa.assetname AS 资产名称,
      fa.originalvalue AS 原值,
      fa.accudepre AS 累计折旧,
      fa.originalvalue - fa.accudepre AS 净值,
      fa.startdate AS 入账日期,
      fa.deptname AS 使用部门
  FROM fa_card fa
  WHERE fa.dr = 0
    AND fa.cardstatus = 1  -- 在用资产
  ORDER BY fa.originalvalue DESC
数据源: NC财务系统

问题: 统计各部门固定资产总值
SQL: |
  SELECT 
      fa.deptname AS 使用部门,
      COUNT(*) AS 资产数量,
      SUM(fa.originalvalue) AS 资产原值合计,
      SUM(fa.accudepre) AS 累计折旧合计,
      SUM(fa.originalvalue - fa.accudepre) AS 净值合计
  FROM fa_card fa
  WHERE fa.dr = 0
    AND fa.cardstatus = 1
  GROUP BY fa.deptname
  ORDER BY 净值合计 DESC
数据源: NC财务系统
```

---

## 6. 自定义提示词配置

### 6.1 NC 系统通用规则

```yaml
提示词名称: NC系统通用查询规则
类型: 问 SQL
生效数据源: NC财务系统

提示词内容: |
  在生成查询 NC 财务系统的 SQL 时，请遵循以下规则：
  
  ## 数据有效性
  1. 所有表查询必须加 dr = 0 条件，排除已删除记录
  2. 单据类查询需判断 billstatus 状态，通常 >= 2 表示已审核
  3. 凭证查询需判断 vouchstatus 状态，>= 3 表示已记账
  
  ## 字段处理
  4. 金额字段可能为 NULL，使用 COALESCE(字段, 0) 处理
  5. 主键字段以 pk_ 开头，外键关联时注意字段名匹配
  6. 日期字段格式为 DATE 类型，可直接进行日期运算
  
  ## 期间处理
  7. 会计年度使用 cyear（4位数字），会计期间使用 cmonth（1-12）
  8. 查询"本月"时使用 EXTRACT(YEAR FROM CURRENT_DATE) 和 EXTRACT(MONTH FROM CURRENT_DATE)
  9. 年初余额使用 cmonth = 0 或查询 cmonth = 1 的期初值
  
  ## 表关联
  10. 凭证明细需关联 gl_voucher 和 gl_detail
  11. 科目信息需关联 bd_accsubj 表
  12. 组织信息需关联 org_orgs 或 org_financeorg 表
```

### 6.2 科目查询规则

```yaml
提示词名称: 科目余额查询规则
类型: 问 SQL
生效数据源: NC财务系统

提示词内容: |
  查询科目余额相关数据时，请遵循以下规则：
  
  1. 科目余额优先从 gl_balance 表查询，性能更好
  2. 如需凭证级明细，则从 gl_detail 表汇总
  3. 科目编码规则：
     - 1xxx: 资产类（借方余额为正）
     - 2xxx: 负债类（贷方余额为正）
     - 3xxx: 权益类（贷方余额为正）
     - 4xxx: 成本类
     - 5xxx: 收入类
     - 6xxx: 费用类
  4. 只统计末级科目（isleaf = 'Y'）的数据，非末级科目是汇总值
  5. 期末余额计算：
     - 资产类：期初 + 借方 - 贷方
     - 负债/权益类：期初 - 借方 + 贷方
```

### 6.3 期间处理规则

```yaml
提示词名称: 会计期间处理规则
类型: 问 SQL
生效数据源: NC财务系统

提示词内容: |
  处理会计期间相关查询时，请遵循以下规则：
  
  1. "本月" = 当前系统日期所在月份
     条件：cyear = EXTRACT(YEAR FROM CURRENT_DATE) AND cmonth = EXTRACT(MONTH FROM CURRENT_DATE)
  
  2. "上月" 需考虑跨年：
     - 如果当前是1月，上月是去年12月
     - 使用 CASE WHEN 或日期函数处理
  
  3. "本季度" 计算：
     - Q1: cmonth IN (1, 2, 3)
     - Q2: cmonth IN (4, 5, 6)
     - Q3: cmonth IN (7, 8, 9)
     - Q4: cmonth IN (10, 11, 12)
  
  4. "本年累计" 条件：
     cyear = EXTRACT(YEAR FROM CURRENT_DATE) AND cmonth BETWEEN 1 AND 当前月
  
  5. "去年同期" 计算：
     cyear = EXTRACT(YEAR FROM CURRENT_DATE) - 1 AND cmonth = EXTRACT(MONTH FROM CURRENT_DATE)
```

---

## 7. 表关系配置

### 7.1 核心表关联关系

在 SQLBot 中配置以下表关系：

#### 凭证相关

```yaml
# 凭证主表 - 凭证分录
主表: gl_voucher
关联表: gl_detail
主表字段: pk_voucher
关联表字段: pk_voucher
关系类型: INNER JOIN
说明: 一张凭证包含多条分录

# 凭证分录 - 科目
主表: gl_detail
关联表: bd_accsubj
主表字段: pk_accsubj
关联表字段: pk_accsubj
关系类型: INNER JOIN
说明: 每条分录对应一个科目
```

#### 余额相关

```yaml
# 余额表 - 科目
主表: gl_balance
关联表: bd_accsubj
主表字段: pk_accsubj
关联表字段: pk_accsubj
关系类型: INNER JOIN
说明: 余额按科目存储

# 余额表 - 组织
主表: gl_balance
关联表: org_orgs
主表字段: pk_org
关联表字段: pk_org
关系类型: LEFT JOIN
说明: 余额按组织存储
```

#### 应收应付相关

```yaml
# 收款单 - 客户
主表: ar_recbill
关联表: bd_customer
主表字段: pk_customer
关联表字段: pk_customer
关系类型: INNER JOIN
说明: 收款单关联客户

# 付款单 - 供应商
主表: ap_paybill
关联表: bd_supplier
主表字段: pk_supplier
关联表字段: pk_supplier
关系类型: INNER JOIN
说明: 付款单关联供应商
```

---

## 8. 常见问题场景优化

### 8.1 问题："本月收入是多少"

**优化配置**：

```yaml
术语:
  名称: 收入
  同义词: 营收、营业收入、销售收入
  描述: |
    收入类科目的发生额合计，通常指主营业务收入和其他业务收入。
    科目编码：5001（主营业务收入）、5051（其他业务收入）
    计算：SUM(贷方发生额 - 借方发生额)
    数据来源：gl_detail 或 gl_balance

SQL示例:
  问题: 本月收入是多少
  SQL: |
    SELECT 
        SUM(COALESCE(d.localcreditamount, 0) - COALESCE(d.localdebitamount, 0)) AS 本月收入
    FROM gl_voucher v
    INNER JOIN gl_detail d ON v.pk_voucher = d.pk_voucher
    INNER JOIN bd_accsubj s ON d.pk_accsubj = s.pk_accsubj
    WHERE v.dr = 0 AND d.dr = 0
      AND v.vouchstatus >= 3
      AND v.cyear = EXTRACT(YEAR FROM CURRENT_DATE)
      AND v.cmonth = EXTRACT(MONTH FROM CURRENT_DATE)
      AND (s.accsubjcode LIKE '5001%' OR s.accsubjcode LIKE '5051%')
```

### 8.2 问题："现金余额"

**优化配置**：

```yaml
术语:
  名称: 现金余额
  同义词: 库存现金、现金
  描述: |
    库存现金科目的期末余额。
    科目编码：1001（库存现金）
    数据来源：gl_balance.endbalance

SQL示例:
  问题: 现金余额是多少
  SQL: |
    SELECT 
        s.accsubjcode AS 科目编码,
        s.accsubjname AS 科目名称,
        b.endbalance AS 现金余额
    FROM gl_balance b
    INNER JOIN bd_accsubj s ON b.pk_accsubj = s.pk_accsubj
    WHERE b.cyear = EXTRACT(YEAR FROM CURRENT_DATE)
      AND b.cmonth = EXTRACT(MONTH FROM CURRENT_DATE)
      AND s.accsubjcode = '1001'
      AND s.dr = 0
```

### 8.3 问题："利润表"

**优化配置**：

```yaml
术语:
  名称: 利润表
  同义词: 损益表、收益表
  描述: |
    反映企业一定期间经营成果的财务报表。
    主要项目：营业收入、营业成本、期间费用、利润总额、净利润
    计算逻辑：
    - 营业收入 = 5001（主营业务收入）+ 5051（其他业务收入）
    - 营业成本 = 6001（主营业务成本）+ 6051（其他业务成本）
    - 期间费用 = 6601（销售费用）+ 6602（管理费用）+ 6603（财务费用）
    - 利润总额 = 营业收入 - 营业成本 - 期间费用 + 营业外收支
    - 净利润 = 利润总额 - 所得税费用

SQL示例:
  问题: 本月利润表
  SQL: |
    WITH income AS (
        SELECT SUM(COALESCE(d.localcreditamount, 0) - COALESCE(d.localdebitamount, 0)) AS amount
        FROM gl_voucher v
        INNER JOIN gl_detail d ON v.pk_voucher = d.pk_voucher
        INNER JOIN bd_accsubj s ON d.pk_accsubj = s.pk_accsubj
        WHERE v.dr = 0 AND d.dr = 0 AND v.vouchstatus >= 3
          AND v.cyear = EXTRACT(YEAR FROM CURRENT_DATE)
          AND v.cmonth = EXTRACT(MONTH FROM CURRENT_DATE)
          AND s.accsubjcode LIKE '5%'
    ),
    cost AS (
        SELECT SUM(COALESCE(d.localdebitamount, 0) - COALESCE(d.localcreditamount, 0)) AS amount
        FROM gl_voucher v
        INNER JOIN gl_detail d ON v.pk_voucher = d.pk_voucher
        INNER JOIN bd_accsubj s ON d.pk_accsubj = s.pk_accsubj
        WHERE v.dr = 0 AND d.dr = 0 AND v.vouchstatus >= 3
          AND v.cyear = EXTRACT(YEAR FROM CURRENT_DATE)
          AND v.cmonth = EXTRACT(MONTH FROM CURRENT_DATE)
          AND s.accsubjcode LIKE '6%'
    )
    SELECT 
        i.amount AS 营业收入,
        c.amount AS 营业成本及费用,
        i.amount - c.amount AS 净利润
    FROM income i, cost c
```

---

## 9. 补充术语库（财务分析类）

### 9.1 财务比率分析术语

```yaml
术语: 流动比率
同义词: 流动资金比率
描述: |
  衡量企业短期偿债能力的指标。
  计算公式：流动比率 = 流动资产 / 流动负债
  流动资产：科目编码 1001-1299（现金、应收、存货等）
  流动负债：科目编码 2001-2299（短期借款、应付等）
  参考值：一般认为 2:1 较为合理
生效数据源: NC财务系统

术语: 速动比率
同义词: 酸性测试比率
描述: |
  更严格的短期偿债能力指标，排除存货。
  计算公式：速动比率 = (流动资产 - 存货) / 流动负债
  存货科目：1401-1499
  参考值：一般认为 1:1 较为合理
生效数据源: NC财务系统

术语: 资产负债率
同义词: 负债比率、举债经营比率
描述: |
  衡量企业负债水平和风险程度的指标。
  计算公式：资产负债率 = 负债总额 / 资产总额 × 100%
  资产总额：科目编码 1xxx 的期末余额合计
  负债总额：科目编码 2xxx 的期末余额合计
  参考值：一般不超过 70%
生效数据源: NC财务系统

术语: 净资产收益率
同义词: ROE、权益报酬率、股东权益报酬率
描述: |
  衡量股东权益获利能力的指标。
  计算公式：ROE = 净利润 / 平均净资产 × 100%
  净利润：损益类科目的本年累计
  净资产：科目编码 3xxx 的期末余额
  平均净资产 = (期初净资产 + 期末净资产) / 2
生效数据源: NC财务系统

术语: 总资产周转率
同义词: 资产周转率
描述: |
  衡量企业资产利用效率的指标。
  计算公式：总资产周转率 = 营业收入 / 平均总资产
  营业收入：科目 5001（主营业务收入）的贷方发生额
  平均总资产 = (期初资产 + 期末资产) / 2
  单位：次/年
生效数据源: NC财务系统

术语: 应收账款周转率
同义词: 应收周转率
描述: |
  衡量应收账款回收速度的指标。
  计算公式：应收账款周转率 = 营业收入 / 平均应收账款
  应收账款科目：1122
  周转天数 = 365 / 周转率
生效数据源: NC财务系统

术语: 存货周转率
同义词: 库存周转率
描述: |
  衡量存货周转速度的指标。
  计算公式：存货周转率 = 营业成本 / 平均存货
  存货科目：1401-1499
  营业成本科目：6001
  周转天数 = 365 / 周转率
生效数据源: NC财务系统

术语: 毛利率
同义词: 销售毛利率、毛利润率
描述: |
  衡量产品盈利能力的指标。
  计算公式：毛利率 = (营业收入 - 营业成本) / 营业收入 × 100%
  营业收入：科目 5001 贷方发生
  营业成本：科目 6001 借方发生
生效数据源: NC财务系统

术语: 净利率
同义词: 销售净利率、净利润率
描述: |
  衡量企业最终盈利能力的指标。
  计算公式：净利率 = 净利润 / 营业收入 × 100%
  净利润 = 收入类科目贷方 - 成本费用类科目借方
生效数据源: NC财务系统

术语: 经营活动现金流
同义词: 经营现金流、OCF
描述: |
  企业日常经营活动产生的现金净流入。
  计算来源：需要从现金流量表相关凭证统计
  或通过银行存款科目的经营类收支归集
生效数据源: NC财务系统
```

### 9.2 辅助核算类术语

```yaml
术语: 辅助核算
同义词: 辅助账、明细核算
描述: |
  对会计科目进行更细致维度的核算管理。
  常见辅助核算项目：
  - 客户：应收账款的客户维度分析
  - 供应商：应付账款的供应商维度分析
  - 部门：费用的部门归属分析
  - 项目：按项目归集收入和成本
  - 人员：个人借款、工资等
  数据来源：gl_freevalue 和 gl_accass 表
生效数据源: NC财务系统

术语: 客户辅助
同义词: 客户往来、客户核算
描述: |
  按客户维度进行的应收款项明细核算。
  关联表：bd_customer（客户档案）
  常用于：应收账款、预收账款、合同资产等科目
  查询：gl_freevalue.pk_customer 或辅助核算余额表
生效数据源: NC财务系统

术语: 供应商辅助
同义词: 供应商往来、供应商核算
描述: |
  按供应商维度进行的应付款项明细核算。
  关联表：bd_supplier（供应商档案）
  常用于：应付账款、预付账款、合同负债等科目
  查询：gl_freevalue.pk_supplier 或辅助核算余额表
生效数据源: NC财务系统

术语: 部门辅助
同义词: 部门核算、成本中心
描述: |
  按部门维度进行的费用归集核算。
  关联表：org_dept（部门档案）
  常用于：管理费用、销售费用、制造费用等
  用途：成本中心核算、费用预算控制
生效数据源: NC财务系统

术语: 项目辅助
同义词: 项目核算、在建工程核算
描述: |
  按项目维度进行的收支归集核算。
  关联表：bd_project（项目档案）
  常用于：在建工程、研发支出、工程施工等
  用途：项目成本归集、项目利润分析
生效数据源: NC财务系统
```

### 9.3 期间与比较类术语

```yaml
术语: 同比
同义词: 同期比较、与去年同期比
描述: |
  与去年同一期间进行比较分析。
  本期同比增长率 = (本期数 - 去年同期数) / 去年同期数 × 100%
  去年同期条件：cyear = 当前年 - 1 AND cmonth = 当前月
生效数据源: NC财务系统

术语: 环比
同义词: 与上期比、与上月比
描述: |
  与上一个期间进行比较分析。
  本期环比增长率 = (本期数 - 上期数) / 上期数 × 100%
  上期条件：需考虑跨年（1月的上期是去年12月）
生效数据源: NC财务系统

术语: 累计
同义词: 年累计、本年累计
描述: |
  从年初到本期的累计发生额。
  数据来源：gl_balance.yeardebit / yearcredit
  或 cmonth BETWEEN 1 AND 当前月 的 SUM
生效数据源: NC财务系统

术语: 调整期
同义词: 调整期间、第13期
描述: |
  会计年度结束后的调整期间，用于年末调账。
  NC系统中通常 cmonth = 0 或 cmonth = 13 表示调整期
  注意：正常月份汇总时可能需要排除调整期
生效数据源: NC财务系统
```

### 9.4 单据状态类术语

```yaml
术语: 草稿状态
同义词: 暂存、保存
描述: |
  单据已创建但未提交审批。
  状态值：billstatus = 0 或 1
  特点：可修改、可删除，不进入业务流程
生效数据源: NC财务系统

术语: 已提交
同义词: 待审核、审批中
描述: |
  单据已提交等待审批。
  状态值：billstatus = 2 或 审批流程中
  特点：不可修改，等待上级审批
生效数据源: NC财务系统

术语: 已审核
同义词: 已批准、审批通过
描述: |
  单据已通过审批，具有业务效力。
  状态值：billstatus >= 3 或 vouchstatus >= 2
  特点：已审核单据可参与业务计算
生效数据源: NC财务系统

术语: 已记账
同义词: 已过账、已入账
描述: |
  凭证已完成记账处理，正式进入账务系统。
  状态值：vouchstatus >= 3
  特点：已记账凭证会更新科目余额
生效数据源: NC财务系统

术语: 已结账
同义词: 已关账、期末结转
描述: |
  会计期间已完成结账处理。
  检查方式：查询期间状态表或结账日志
  特点：已结账期间不能再录入凭证
生效数据源: NC财务系统
```

---

## 10. 补充 SQL 示例库

### 10.1 财务报表类示例

```yaml
问题: 查询资产负债表主要项目
SQL: |
  WITH asset_items AS (
      SELECT 
          CASE 
              WHEN s.accsubjcode LIKE '1001%' THEN '货币资金'
              WHEN s.accsubjcode LIKE '1002%' THEN '货币资金'
              WHEN s.accsubjcode LIKE '1122%' THEN '应收账款'
              WHEN s.accsubjcode LIKE '1123%' THEN '预付账款'
              WHEN s.accsubjcode LIKE '1401%' THEN '存货'
              WHEN s.accsubjcode LIKE '1601%' THEN '固定资产'
              WHEN s.accsubjcode LIKE '1701%' THEN '无形资产'
              ELSE '其他资产'
          END AS 项目名称,
          SUM(b.endbalance) AS 期末余额
      FROM gl_balance b
      INNER JOIN bd_accsubj s ON b.pk_accsubj = s.pk_accsubj
      WHERE b.cyear = EXTRACT(YEAR FROM CURRENT_DATE)
        AND b.cmonth = EXTRACT(MONTH FROM CURRENT_DATE)
        AND s.accsubjcode LIKE '1%'
        AND s.isleaf = 'Y'
        AND s.dr = 0
      GROUP BY CASE 
          WHEN s.accsubjcode LIKE '1001%' THEN '货币资金'
          WHEN s.accsubjcode LIKE '1002%' THEN '货币资金'
          WHEN s.accsubjcode LIKE '1122%' THEN '应收账款'
          WHEN s.accsubjcode LIKE '1123%' THEN '预付账款'
          WHEN s.accsubjcode LIKE '1401%' THEN '存货'
          WHEN s.accsubjcode LIKE '1601%' THEN '固定资产'
          WHEN s.accsubjcode LIKE '1701%' THEN '无形资产'
          ELSE '其他资产'
      END
  )
  SELECT 项目名称, 期末余额
  FROM asset_items
  ORDER BY 期末余额 DESC
数据源: NC财务系统

问题: 生成本月利润表
SQL: |
  SELECT 
      '一、营业收入' AS 项目,
      SUM(CASE WHEN s.accsubjcode LIKE '5001%' 
          THEN COALESCE(d.localcreditamount, 0) - COALESCE(d.localdebitamount, 0) 
          ELSE 0 END) AS 本期金额
  FROM gl_voucher v
  INNER JOIN gl_detail d ON v.pk_voucher = d.pk_voucher
  INNER JOIN bd_accsubj s ON d.pk_accsubj = s.pk_accsubj
  WHERE v.dr = 0 AND d.dr = 0 AND v.vouchstatus >= 3
    AND v.cyear = EXTRACT(YEAR FROM CURRENT_DATE)
    AND v.cmonth = EXTRACT(MONTH FROM CURRENT_DATE)
  
  UNION ALL
  
  SELECT 
      '二、营业成本' AS 项目,
      SUM(CASE WHEN s.accsubjcode LIKE '6001%' 
          THEN COALESCE(d.localdebitamount, 0) - COALESCE(d.localcreditamount, 0) 
          ELSE 0 END) AS 本期金额
  FROM gl_voucher v
  INNER JOIN gl_detail d ON v.pk_voucher = d.pk_voucher
  INNER JOIN bd_accsubj s ON d.pk_accsubj = s.pk_accsubj
  WHERE v.dr = 0 AND d.dr = 0 AND v.vouchstatus >= 3
    AND v.cyear = EXTRACT(YEAR FROM CURRENT_DATE)
    AND v.cmonth = EXTRACT(MONTH FROM CURRENT_DATE)
  
  UNION ALL
  
  SELECT 
      '三、销售费用' AS 项目,
      SUM(CASE WHEN s.accsubjcode LIKE '6601%' 
          THEN COALESCE(d.localdebitamount, 0) - COALESCE(d.localcreditamount, 0) 
          ELSE 0 END) AS 本期金额
  FROM gl_voucher v
  INNER JOIN gl_detail d ON v.pk_voucher = d.pk_voucher
  INNER JOIN bd_accsubj s ON d.pk_accsubj = s.pk_accsubj
  WHERE v.dr = 0 AND d.dr = 0 AND v.vouchstatus >= 3
    AND v.cyear = EXTRACT(YEAR FROM CURRENT_DATE)
    AND v.cmonth = EXTRACT(MONTH FROM CURRENT_DATE)
  
  UNION ALL
  
  SELECT 
      '四、管理费用' AS 项目,
      SUM(CASE WHEN s.accsubjcode LIKE '6602%' 
          THEN COALESCE(d.localdebitamount, 0) - COALESCE(d.localcreditamount, 0) 
          ELSE 0 END) AS 本期金额
  FROM gl_voucher v
  INNER JOIN gl_detail d ON v.pk_voucher = d.pk_voucher
  INNER JOIN bd_accsubj s ON d.pk_accsubj = s.pk_accsubj
  WHERE v.dr = 0 AND d.dr = 0 AND v.vouchstatus >= 3
    AND v.cyear = EXTRACT(YEAR FROM CURRENT_DATE)
    AND v.cmonth = EXTRACT(MONTH FROM CURRENT_DATE)
  
  UNION ALL
  
  SELECT 
      '五、财务费用' AS 项目,
      SUM(CASE WHEN s.accsubjcode LIKE '6603%' 
          THEN COALESCE(d.localdebitamount, 0) - COALESCE(d.localcreditamount, 0) 
          ELSE 0 END) AS 本期金额
  FROM gl_voucher v
  INNER JOIN gl_detail d ON v.pk_voucher = d.pk_voucher
  INNER JOIN bd_accsubj s ON d.pk_accsubj = s.pk_accsubj
  WHERE v.dr = 0 AND d.dr = 0 AND v.vouchstatus >= 3
    AND v.cyear = EXTRACT(YEAR FROM CURRENT_DATE)
    AND v.cmonth = EXTRACT(MONTH FROM CURRENT_DATE)
数据源: NC财务系统
```

### 10.2 财务比率分析示例

```yaml
问题: 计算流动比率和速动比率
SQL: |
  WITH current_asset AS (
      SELECT SUM(b.endbalance) AS amount
      FROM gl_balance b
      INNER JOIN bd_accsubj s ON b.pk_accsubj = s.pk_accsubj
      WHERE b.cyear = EXTRACT(YEAR FROM CURRENT_DATE)
        AND b.cmonth = EXTRACT(MONTH FROM CURRENT_DATE)
        AND s.accsubjcode >= '1001' AND s.accsubjcode < '1500'
        AND s.isleaf = 'Y' AND s.dr = 0
  ),
  inventory AS (
      SELECT SUM(b.endbalance) AS amount
      FROM gl_balance b
      INNER JOIN bd_accsubj s ON b.pk_accsubj = s.pk_accsubj
      WHERE b.cyear = EXTRACT(YEAR FROM CURRENT_DATE)
        AND b.cmonth = EXTRACT(MONTH FROM CURRENT_DATE)
        AND s.accsubjcode >= '1401' AND s.accsubjcode < '1500'
        AND s.isleaf = 'Y' AND s.dr = 0
  ),
  current_liability AS (
      SELECT SUM(b.endbalance) AS amount
      FROM gl_balance b
      INNER JOIN bd_accsubj s ON b.pk_accsubj = s.pk_accsubj
      WHERE b.cyear = EXTRACT(YEAR FROM CURRENT_DATE)
        AND b.cmonth = EXTRACT(MONTH FROM CURRENT_DATE)
        AND s.accsubjcode >= '2001' AND s.accsubjcode < '2500'
        AND s.isleaf = 'Y' AND s.dr = 0
  )
  SELECT 
      a.amount AS 流动资产,
      i.amount AS 存货,
      l.amount AS 流动负债,
      ROUND(a.amount / NULLIF(l.amount, 0), 2) AS 流动比率,
      ROUND((a.amount - COALESCE(i.amount, 0)) / NULLIF(l.amount, 0), 2) AS 速动比率
  FROM current_asset a, inventory i, current_liability l
数据源: NC财务系统

问题: 计算应收账款周转率
SQL: |
  WITH revenue AS (
      SELECT SUM(COALESCE(d.localcreditamount, 0)) AS amount
      FROM gl_voucher v
      INNER JOIN gl_detail d ON v.pk_voucher = d.pk_voucher
      INNER JOIN bd_accsubj s ON d.pk_accsubj = s.pk_accsubj
      WHERE v.dr = 0 AND d.dr = 0 AND v.vouchstatus >= 3
        AND v.cyear = EXTRACT(YEAR FROM CURRENT_DATE)
        AND s.accsubjcode LIKE '5001%'
  ),
  ar_begin AS (
      SELECT SUM(b.beginbalance) AS amount
      FROM gl_balance b
      INNER JOIN bd_accsubj s ON b.pk_accsubj = s.pk_accsubj
      WHERE b.cyear = EXTRACT(YEAR FROM CURRENT_DATE)
        AND b.cmonth = 1
        AND s.accsubjcode LIKE '1122%'
        AND s.isleaf = 'Y' AND s.dr = 0
  ),
  ar_end AS (
      SELECT SUM(b.endbalance) AS amount
      FROM gl_balance b
      INNER JOIN bd_accsubj s ON b.pk_accsubj = s.pk_accsubj
      WHERE b.cyear = EXTRACT(YEAR FROM CURRENT_DATE)
        AND b.cmonth = EXTRACT(MONTH FROM CURRENT_DATE)
        AND s.accsubjcode LIKE '1122%'
        AND s.isleaf = 'Y' AND s.dr = 0
  )
  SELECT 
      r.amount AS 营业收入,
      ab.amount AS 期初应收,
      ae.amount AS 期末应收,
      (COALESCE(ab.amount, 0) + COALESCE(ae.amount, 0)) / 2 AS 平均应收,
      ROUND(r.amount / NULLIF((COALESCE(ab.amount, 0) + COALESCE(ae.amount, 0)) / 2, 0), 2) AS 应收周转率,
      ROUND(365 / NULLIF(r.amount / NULLIF((COALESCE(ab.amount, 0) + COALESCE(ae.amount, 0)) / 2, 0), 0), 0) AS 周转天数
  FROM revenue r, ar_begin ab, ar_end ae
数据源: NC财务系统

问题: 计算毛利率和净利率
SQL: |
  WITH financial_data AS (
      SELECT 
          SUM(CASE WHEN s.accsubjcode LIKE '5001%' 
              THEN COALESCE(d.localcreditamount, 0) - COALESCE(d.localdebitamount, 0) 
              ELSE 0 END) AS revenue,
          SUM(CASE WHEN s.accsubjcode LIKE '6001%' 
              THEN COALESCE(d.localdebitamount, 0) - COALESCE(d.localcreditamount, 0) 
              ELSE 0 END) AS cost,
          SUM(CASE WHEN s.accsubjcode LIKE '6%' 
              THEN COALESCE(d.localdebitamount, 0) - COALESCE(d.localcreditamount, 0) 
              ELSE 0 END) AS total_expense
      FROM gl_voucher v
      INNER JOIN gl_detail d ON v.pk_voucher = d.pk_voucher
      INNER JOIN bd_accsubj s ON d.pk_accsubj = s.pk_accsubj
      WHERE v.dr = 0 AND d.dr = 0 AND v.vouchstatus >= 3
        AND v.cyear = EXTRACT(YEAR FROM CURRENT_DATE)
        AND v.cmonth = EXTRACT(MONTH FROM CURRENT_DATE)
  )
  SELECT 
      revenue AS 营业收入,
      cost AS 营业成本,
      revenue - cost AS 毛利润,
      ROUND((revenue - cost) / NULLIF(revenue, 0) * 100, 2) AS 毛利率,
      revenue - total_expense AS 净利润,
      ROUND((revenue - total_expense) / NULLIF(revenue, 0) * 100, 2) AS 净利率
  FROM financial_data
数据源: NC财务系统
```

### 10.3 同比环比分析示例

```yaml
问题: 收入同比分析
SQL: |
  WITH current_month AS (
      SELECT SUM(COALESCE(d.localcreditamount, 0) - COALESCE(d.localdebitamount, 0)) AS amount
      FROM gl_voucher v
      INNER JOIN gl_detail d ON v.pk_voucher = d.pk_voucher
      INNER JOIN bd_accsubj s ON d.pk_accsubj = s.pk_accsubj
      WHERE v.dr = 0 AND d.dr = 0 AND v.vouchstatus >= 3
        AND v.cyear = EXTRACT(YEAR FROM CURRENT_DATE)
        AND v.cmonth = EXTRACT(MONTH FROM CURRENT_DATE)
        AND s.accsubjcode LIKE '5001%'
  ),
  last_year_month AS (
      SELECT SUM(COALESCE(d.localcreditamount, 0) - COALESCE(d.localdebitamount, 0)) AS amount
      FROM gl_voucher v
      INNER JOIN gl_detail d ON v.pk_voucher = d.pk_voucher
      INNER JOIN bd_accsubj s ON d.pk_accsubj = s.pk_accsubj
      WHERE v.dr = 0 AND d.dr = 0 AND v.vouchstatus >= 3
        AND v.cyear = EXTRACT(YEAR FROM CURRENT_DATE) - 1
        AND v.cmonth = EXTRACT(MONTH FROM CURRENT_DATE)
        AND s.accsubjcode LIKE '5001%'
  )
  SELECT 
      c.amount AS 本月收入,
      l.amount AS 去年同月收入,
      c.amount - COALESCE(l.amount, 0) AS 同比增减额,
      ROUND((c.amount - COALESCE(l.amount, 0)) / NULLIF(l.amount, 0) * 100, 2) AS 同比增长率
  FROM current_month c, last_year_month l
数据源: NC财务系统

问题: 费用环比分析
SQL: |
  WITH current_month AS (
      SELECT 
          CASE 
              WHEN s.accsubjcode LIKE '6601%' THEN '销售费用'
              WHEN s.accsubjcode LIKE '6602%' THEN '管理费用'
              WHEN s.accsubjcode LIKE '6603%' THEN '财务费用'
          END AS 费用类型,
          SUM(COALESCE(d.localdebitamount, 0) - COALESCE(d.localcreditamount, 0)) AS amount
      FROM gl_voucher v
      INNER JOIN gl_detail d ON v.pk_voucher = d.pk_voucher
      INNER JOIN bd_accsubj s ON d.pk_accsubj = s.pk_accsubj
      WHERE v.dr = 0 AND d.dr = 0 AND v.vouchstatus >= 3
        AND v.cyear = EXTRACT(YEAR FROM CURRENT_DATE)
        AND v.cmonth = EXTRACT(MONTH FROM CURRENT_DATE)
        AND s.accsubjcode LIKE '66%'
      GROUP BY CASE 
          WHEN s.accsubjcode LIKE '6601%' THEN '销售费用'
          WHEN s.accsubjcode LIKE '6602%' THEN '管理费用'
          WHEN s.accsubjcode LIKE '6603%' THEN '财务费用'
      END
  ),
  last_month AS (
      SELECT 
          CASE 
              WHEN s.accsubjcode LIKE '6601%' THEN '销售费用'
              WHEN s.accsubjcode LIKE '6602%' THEN '管理费用'
              WHEN s.accsubjcode LIKE '6603%' THEN '财务费用'
          END AS 费用类型,
          SUM(COALESCE(d.localdebitamount, 0) - COALESCE(d.localcreditamount, 0)) AS amount
      FROM gl_voucher v
      INNER JOIN gl_detail d ON v.pk_voucher = d.pk_voucher
      INNER JOIN bd_accsubj s ON d.pk_accsubj = s.pk_accsubj
      WHERE v.dr = 0 AND d.dr = 0 AND v.vouchstatus >= 3
        AND ((v.cyear = EXTRACT(YEAR FROM CURRENT_DATE) AND v.cmonth = EXTRACT(MONTH FROM CURRENT_DATE) - 1)
             OR (v.cyear = EXTRACT(YEAR FROM CURRENT_DATE) - 1 AND v.cmonth = 12 AND EXTRACT(MONTH FROM CURRENT_DATE) = 1))
        AND s.accsubjcode LIKE '66%'
      GROUP BY CASE 
          WHEN s.accsubjcode LIKE '6601%' THEN '销售费用'
          WHEN s.accsubjcode LIKE '6602%' THEN '管理费用'
          WHEN s.accsubjcode LIKE '6603%' THEN '财务费用'
      END
  )
  SELECT 
      c.费用类型,
      c.amount AS 本月金额,
      COALESCE(l.amount, 0) AS 上月金额,
      c.amount - COALESCE(l.amount, 0) AS 环比增减,
      ROUND((c.amount - COALESCE(l.amount, 0)) / NULLIF(l.amount, 0) * 100, 2) AS 环比增长率
  FROM current_month c
  LEFT JOIN last_month l ON c.费用类型 = l.费用类型
  ORDER BY c.amount DESC
数据源: NC财务系统

问题: 各月收入趋势分析
SQL: |
  SELECT 
      v.cmonth AS 月份,
      SUM(COALESCE(d.localcreditamount, 0) - COALESCE(d.localdebitamount, 0)) AS 月收入,
      SUM(SUM(COALESCE(d.localcreditamount, 0) - COALESCE(d.localdebitamount, 0))) 
          OVER (ORDER BY v.cmonth) AS 累计收入
  FROM gl_voucher v
  INNER JOIN gl_detail d ON v.pk_voucher = d.pk_voucher
  INNER JOIN bd_accsubj s ON d.pk_accsubj = s.pk_accsubj
  WHERE v.dr = 0 AND d.dr = 0 AND v.vouchstatus >= 3
    AND v.cyear = EXTRACT(YEAR FROM CURRENT_DATE)
    AND s.accsubjcode LIKE '5001%'
  GROUP BY v.cmonth
  ORDER BY v.cmonth
数据源: NC财务系统
```

### 10.4 辅助核算分析示例

```yaml
问题: 按客户统计应收账款
SQL: |
  SELECT 
      c.code AS 客户编码,
      c.name AS 客户名称,
      SUM(b.endbalance) AS 应收余额
  FROM gl_accass b
  INNER JOIN bd_customer c ON b.pk_assid = c.pk_customer
  INNER JOIN bd_accsubj s ON b.pk_accsubj = s.pk_accsubj
  WHERE b.cyear = EXTRACT(YEAR FROM CURRENT_DATE)
    AND b.cmonth = EXTRACT(MONTH FROM CURRENT_DATE)
    AND s.accsubjcode LIKE '1122%'
    AND b.dr = 0
  GROUP BY c.code, c.name
  HAVING SUM(b.endbalance) <> 0
  ORDER BY 应收余额 DESC
数据源: NC财务系统

问题: 按供应商统计应付账款
SQL: |
  SELECT 
      s.code AS 供应商编码,
      s.name AS 供应商名称,
      SUM(b.endbalance) AS 应付余额
  FROM gl_accass b
  INNER JOIN bd_supplier s ON b.pk_assid = s.pk_supplier
  INNER JOIN bd_accsubj subj ON b.pk_accsubj = subj.pk_accsubj
  WHERE b.cyear = EXTRACT(YEAR FROM CURRENT_DATE)
    AND b.cmonth = EXTRACT(MONTH FROM CURRENT_DATE)
    AND subj.accsubjcode LIKE '2202%'
    AND b.dr = 0
  GROUP BY s.code, s.name
  HAVING SUM(b.endbalance) <> 0
  ORDER BY 应付余额 DESC
数据源: NC财务系统

问题: 按部门统计管理费用
SQL: |
  SELECT 
      dept.name AS 部门名称,
      SUM(COALESCE(d.localdebitamount, 0) - COALESCE(d.localcreditamount, 0)) AS 管理费用
  FROM gl_voucher v
  INNER JOIN gl_detail d ON v.pk_voucher = d.pk_voucher
  INNER JOIN gl_freevalue fv ON d.pk_detail = fv.pk_detail
  INNER JOIN org_dept dept ON fv.pk_dept = dept.pk_dept
  INNER JOIN bd_accsubj s ON d.pk_accsubj = s.pk_accsubj
  WHERE v.dr = 0 AND d.dr = 0 AND v.vouchstatus >= 3
    AND v.cyear = EXTRACT(YEAR FROM CURRENT_DATE)
    AND v.cmonth = EXTRACT(MONTH FROM CURRENT_DATE)
    AND s.accsubjcode LIKE '6602%'
  GROUP BY dept.name
  ORDER BY 管理费用 DESC
数据源: NC财务系统

问题: 项目成本归集分析
SQL: |
  SELECT 
      p.code AS 项目编码,
      p.name AS 项目名称,
      SUM(CASE WHEN s.accsubjcode LIKE '6401%' THEN COALESCE(d.localdebitamount, 0) ELSE 0 END) AS 直接材料,
      SUM(CASE WHEN s.accsubjcode LIKE '6402%' THEN COALESCE(d.localdebitamount, 0) ELSE 0 END) AS 直接人工,
      SUM(CASE WHEN s.accsubjcode LIKE '6403%' THEN COALESCE(d.localdebitamount, 0) ELSE 0 END) AS 制造费用,
      SUM(COALESCE(d.localdebitamount, 0)) AS 成本合计
  FROM gl_voucher v
  INNER JOIN gl_detail d ON v.pk_voucher = d.pk_voucher
  INNER JOIN gl_freevalue fv ON d.pk_detail = fv.pk_detail
  INNER JOIN bd_project p ON fv.pk_project = p.pk_project
  INNER JOIN bd_accsubj s ON d.pk_accsubj = s.pk_accsubj
  WHERE v.dr = 0 AND d.dr = 0 AND v.vouchstatus >= 3
    AND v.cyear = EXTRACT(YEAR FROM CURRENT_DATE)
    AND s.accsubjcode LIKE '64%'
  GROUP BY p.code, p.name
  ORDER BY 成本合计 DESC
数据源: NC财务系统
```

### 10.5 日常查询示例

```yaml
问题: 查询今日录入的凭证
SQL: |
  SELECT 
      v.num AS 凭证号,
      v.prepareddate AS 制单日期,
      COUNT(d.pk_detail) AS 分录数,
      SUM(COALESCE(d.localdebitamount, 0)) AS 借方合计
  FROM gl_voucher v
  INNER JOIN gl_detail d ON v.pk_voucher = d.pk_voucher
  WHERE v.dr = 0 AND d.dr = 0
    AND v.prepareddate = CURRENT_DATE
  GROUP BY v.pk_voucher, v.num, v.prepareddate
  ORDER BY v.num
数据源: NC财务系统

问题: 查询待审核的凭证
SQL: |
  SELECT 
      v.num AS 凭证号,
      v.prepareddate AS 制单日期,
      v.explanation AS 摘要,
      u.user_name AS 制单人
  FROM gl_voucher v
  LEFT JOIN sm_user u ON v.pk_prepared = u.pk_user
  WHERE v.dr = 0
    AND v.vouchstatus < 2  -- 未审核
    AND v.cyear = EXTRACT(YEAR FROM CURRENT_DATE)
    AND v.cmonth = EXTRACT(MONTH FROM CURRENT_DATE)
  ORDER BY v.prepareddate DESC
数据源: NC财务系统

问题: 查询大额凭证（金额超过100万）
SQL: |
  SELECT 
      v.num AS 凭证号,
      v.prepareddate AS 制单日期,
      s.accsubjcode AS 科目编码,
      s.accsubjname AS 科目名称,
      d.explanation AS 摘要,
      COALESCE(d.localdebitamount, 0) AS 借方金额,
      COALESCE(d.localcreditamount, 0) AS 贷方金额
  FROM gl_voucher v
  INNER JOIN gl_detail d ON v.pk_voucher = d.pk_voucher
  INNER JOIN bd_accsubj s ON d.pk_accsubj = s.pk_accsubj
  WHERE v.dr = 0 AND d.dr = 0 AND v.vouchstatus >= 3
    AND v.cyear = EXTRACT(YEAR FROM CURRENT_DATE)
    AND (COALESCE(d.localdebitamount, 0) >= 1000000 
         OR COALESCE(d.localcreditamount, 0) >= 1000000)
  ORDER BY GREATEST(COALESCE(d.localdebitamount, 0), COALESCE(d.localcreditamount, 0)) DESC
数据源: NC财务系统

问题: 银行存款日记账
SQL: |
  SELECT 
      v.prepareddate AS 日期,
      v.num AS 凭证号,
      d.explanation AS 摘要,
      COALESCE(d.localdebitamount, 0) AS 借方金额,
      COALESCE(d.localcreditamount, 0) AS 贷方金额,
      SUM(COALESCE(d.localdebitamount, 0) - COALESCE(d.localcreditamount, 0)) 
          OVER (ORDER BY v.prepareddate, v.num) AS 余额
  FROM gl_voucher v
  INNER JOIN gl_detail d ON v.pk_voucher = d.pk_voucher
  INNER JOIN bd_accsubj s ON d.pk_accsubj = s.pk_accsubj
  WHERE v.dr = 0 AND d.dr = 0 AND v.vouchstatus >= 3
    AND v.cyear = EXTRACT(YEAR FROM CURRENT_DATE)
    AND v.cmonth = EXTRACT(MONTH FROM CURRENT_DATE)
    AND s.accsubjcode LIKE '1002%'  -- 银行存款
  ORDER BY v.prepareddate, v.num
数据源: NC财务系统

问题: 查询未核销的预付账款
SQL: |
  SELECT 
      s.name AS 供应商名称,
      b.endbalance AS 预付余额,
      CASE WHEN b.endbalance > 0 THEN '预付款待核销'
           WHEN b.endbalance < 0 THEN '需补付款'
           ELSE '已核销' END AS 状态
  FROM gl_accass b
  INNER JOIN bd_supplier s ON b.pk_assid = s.pk_supplier
  INNER JOIN bd_accsubj subj ON b.pk_accsubj = subj.pk_accsubj
  WHERE b.cyear = EXTRACT(YEAR FROM CURRENT_DATE)
    AND b.cmonth = EXTRACT(MONTH FROM CURRENT_DATE)
    AND subj.accsubjcode LIKE '1123%'  -- 预付账款
    AND b.dr = 0
    AND b.endbalance <> 0
  ORDER BY ABS(b.endbalance) DESC
数据源: NC财务系统
```

### 10.6 多组织对比示例

```yaml
问题: 各公司收入对比
SQL: |
  SELECT 
      o.name AS 公司名称,
      SUM(COALESCE(d.localcreditamount, 0) - COALESCE(d.localdebitamount, 0)) AS 收入金额,
      ROUND(SUM(COALESCE(d.localcreditamount, 0) - COALESCE(d.localdebitamount, 0)) * 100.0 / 
            SUM(SUM(COALESCE(d.localcreditamount, 0) - COALESCE(d.localdebitamount, 0))) OVER (), 2) AS 占比
  FROM gl_voucher v
  INNER JOIN gl_detail d ON v.pk_voucher = d.pk_voucher
  INNER JOIN bd_accsubj s ON d.pk_accsubj = s.pk_accsubj
  INNER JOIN org_orgs o ON v.pk_org = o.pk_org
  WHERE v.dr = 0 AND d.dr = 0 AND v.vouchstatus >= 3
    AND v.cyear = EXTRACT(YEAR FROM CURRENT_DATE)
    AND v.cmonth = EXTRACT(MONTH FROM CURRENT_DATE)
    AND s.accsubjcode LIKE '5001%'
  GROUP BY o.name
  ORDER BY 收入金额 DESC
数据源: NC财务系统

问题: 集团合并资产负债
SQL: |
  SELECT 
      CASE 
          WHEN s.accsubjcode LIKE '1%' THEN '资产'
          WHEN s.accsubjcode LIKE '2%' THEN '负债'
          WHEN s.accsubjcode LIKE '3%' THEN '所有者权益'
      END AS 类别,
      SUM(b.endbalance) AS 合并金额
  FROM gl_balance b
  INNER JOIN bd_accsubj s ON b.pk_accsubj = s.pk_accsubj
  WHERE b.cyear = EXTRACT(YEAR FROM CURRENT_DATE)
    AND b.cmonth = EXTRACT(MONTH FROM CURRENT_DATE)
    AND s.subjlevel = 1  -- 一级科目
    AND s.dr = 0
  GROUP BY CASE 
      WHEN s.accsubjcode LIKE '1%' THEN '资产'
      WHEN s.accsubjcode LIKE '2%' THEN '负债'
      WHEN s.accsubjcode LIKE '3%' THEN '所有者权益'
  END
  ORDER BY 类别
数据源: NC财务系统
```

---

## 附录：NC 常用表速查

### 总账模块

| 表名 | 说明 | 主键 |
| :--- | :--- | :--- |
| `gl_voucher` | 凭证主表 | pk_voucher |
| `gl_detail` | 凭证分录表 | pk_detail |
| `gl_balance` | 科目余额表 | pk_balance |
| `gl_freevalue` | 辅助核算值 | pk_freevalue |
| `gl_accass` | 辅助核算余额 | pk_accass |

### 基础档案模块

| 表名 | 说明 | 主键 |
| :--- | :--- | :--- |
| `bd_accsubj` | 会计科目 | pk_accsubj |
| `bd_accassitem` | 辅助核算项 | pk_accassitem |
| `bd_supplier` | 供应商 | pk_supplier |
| `bd_customer` | 客户 | pk_customer |
| `bd_currtype` | 币种 | pk_currtype |
| `bd_bankaccbas` | 银行账户 | pk_bankaccbas |
| `bd_stordoc` | 仓库档案 | pk_stordoc |
| `bd_invbasdoc` | 存货档案 | pk_invbasdoc |

### 组织模块

| 表名 | 说明 | 主键 |
| :--- | :--- | :--- |
| `org_orgs` | 组织表 | pk_org |
| `org_financeorg` | 财务组织 | pk_financeorg |
| `org_dept` | 部门 | pk_dept |

### 应收应付模块

| 表名 | 说明 | 主键 |
| :--- | :--- | :--- |
| `ar_recbill` | 收款单 | pk_recbill |
| `ar_recbill_b` | 收款单明细 | pk_recbill_b |
| `ap_paybill` | 付款单 | pk_paybill |
| `ap_paybill_b` | 付款单明细 | pk_paybill_b |

### 固定资产模块

| 表名 | 说明 | 主键 |
| :--- | :--- | :--- |
| `fa_card` | 资产卡片 | pk_card |
| `fa_depreciation` | 折旧信息 | pk_depreciation |
| `fa_assetreduce` | 资产减值 | pk_assetreduce |

---

## 检查清单

### NC 财务系统接入 SQLBot 检查清单

- [ ] 创建只读数据库账号
- [ ] 配置数据源连接
- [ ] 同步核心业务表元数据
- [ ] 添加表级业务描述（至少覆盖 20 个核心表）
- [ ] 添加关键字段描述（状态字段、金额字段、日期字段）
- [ ] 配置财务术语库（至少 30 个核心术语）
- [ ] 添加 SQL 示例（至少 50 个高频问题场景）
- [ ] 配置自定义提示词（通用规则、科目规则、期间规则）
- [ ] 配置核心表关联关系
- [ ] 测试验证常见问题场景
- [ ] 收集用户反馈，持续优化
