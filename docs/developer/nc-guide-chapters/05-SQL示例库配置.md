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

### 5.5 财务报表类示例

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

### 5.6 财务比率分析示例

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

### 5.7 同比环比分析示例

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

### 5.8 辅助核算分析示例

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

### 5.9 日常查询示例

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

### 5.10 多组织对比示例

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

