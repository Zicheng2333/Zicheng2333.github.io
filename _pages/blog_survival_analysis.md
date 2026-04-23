---
permalink: /blog/survival-analysis/
title: "Blog | Survival Analysis 实战复现（Spark + Lifelines + SQL）"
author_profile: false
---

# Survival Analysis 实战复现

这篇博客基于我在课程项目中的完整复现实验，内容覆盖：

1. 官方 Spark 生存分析案例复现（Kaplan-Meier / CoxPH / AFT / CLV）。
2. 本地可运行 notebook 与脚本化结果导出。
3. MySQL 导入与 text-to-SQL 失败案例分析。

---

## 1. 项目与数据

- 官方案例仓库：<https://github.com/databricks-industry-solutions/survival-analysis/tree/main>
- 原始数据：IBM Telco Customer Churn
- 本地复现环境：PySpark + Lifelines + MariaDB (MySQL compatible)

数据处理后规模：

- Bronze：7043
- Silver（用于生存分析）：3351

---

## 2. Kaplan-Meier：生存曲线与组间差异

总体中位生存期（median survival time）约为 **34.0 月**。

![Kaplan-Meier Population Curve](/images/blog/survival-analysis/km_population_curve.png)

图表说明：

1. 生存概率随时间递减，符合“流失风险随服务时长累积”的业务直觉。
2. 曲线在前期下降较快，提示早期留存运营更关键。
3. 中位生存期约 34 个月，可作为策略评估中的基准时间点。

分组示例（gender 与 onlineSecurity）：

![KM by Gender](/images/blog/survival-analysis/km_by_gender.png)

图表说明：

1. Gender 的两条曲线整体接近，肉眼上差异有限。
2. 与统计检验结果一致（log-rank p=0.153317，不显著）。

![KM by OnlineSecurity](/images/blog/survival-analysis/km_by_onlineSecurity.png)

图表说明：

1. OnlineSecurity 分组曲线分离明显。
2. 与 log-rank 检验一致（p=1.188e-32，显著），说明该变量与生存差异相关。

---

## 3. Cox Proportional Hazards：风险比建模

关键指标：

- Concordance：0.6409
- Partial AIC：22639.905

![Cox Hazard Ratios](/images/blog/survival-analysis/cox_hazard_ratios.png)

图表说明：

1. 多个变量的 HR < 1（如 `onlineBackup_Yes`, `techSupport_Yes`），对应风险下降方向。
2. 置信区间反映参数不确定性，可用于判断估计稳定性。
3. 模型可用于风险排序，但需要结合 PH 假设检验解释。

核心变量结果：

| Covariate | HR (exp(coef)) | p-value |
|---|---:|---:|
| dependents_Yes | 0.7199 | 3.520e-06 |
| internetService_DSL | 0.8047 | 2.321e-04 |
| onlineBackup_Yes | 0.4600 | 2.277e-39 |
| techSupport_Yes | 0.5277 | 2.163e-17 |

PH 假设检验摘要：

| Variable | p-value | PH assumption violated? |
|---|---:|---|
| dependents_Yes | 3.680e-01 | No |
| internetService_DSL | 2.369e-07 | Yes |
| onlineBackup_Yes | 2.912e-05 | Yes |
| techSupport_Yes | 2.077e-04 | Yes |

结论：该 Cox 规格在多个变量上违反 PH 假设，做因果机制解释时需谨慎。

---

## 4. CLV：由生存概率到经营价值

基于生存函数计算期望月利润、折现后利润与累计 NPV，关键节点：

- NPV@12：251.2126
- NPV@24：405.7006
- NPV@36：508.2567

![CLV Cumulative NPV](/images/blog/survival-analysis/clv_cumulative_npv_bar.png)

图表说明：

1. 累计 NPV 随月数增长，长期留存价值更高。
2. 12 -> 24 月提升较明显，表明中期留存管理收益突出。
3. 36 月继续增长，支持长期经营视角下的用户价值评估。

![CLV Survival Curve](/images/blog/survival-analysis/clv_survival_curve.png)

图表说明：

1. 该曲线由 Cox 预测生存函数得到，是 CLV 估算的核心输入。
2. 曲线前段形态决定短期收益回收速度。
3. 后段尾部决定长期价值天花板。

---

## 5. MySQL + text-to-SQL：三个典型失败案例

数据库：`survival_analysis_db`  
核心表：`fact_subscriptions`, `dim_contract`, `dim_internet_service`

### Case 1: 分母范围错误

- 问题：按合同类型统计流失率。
- 常见错误：先 `WHERE churn=1` 再计算比例，导致结果全 100%。
- 正确思路：使用 `AVG(CASE WHEN churn=1 THEN 1 ELSE 0 END)`。

### Case 2: 业务条件遗漏

- 问题：仅未流失用户中，哪个互联网服务类型平均 tenure 最高？
- 常见错误：遗漏 `WHERE churn=0`。
- 结果差异：错误 SQL 得到 32.92，正确 SQL 得到 42.09。

### Case 3: 窗口函数分区缺失

- 问题：每种合同类型内月费前20%用户数量。
- 常见错误：`NTILE(5)` 未写 `PARTITION BY contract_id`，把组内前20%做成全局前20%。

---

## 6. 小结

1. 生存分析链路（KM -> CoxPH -> AFT -> CLV）可以从统计层直接延伸到业务决策层。
2. 在这个数据集上，`onlineSecurity` 等变量与生存差异高度相关。
3. LLM 在 text-to-SQL 中仍容易在条件约束、分母定义和窗口分区上出错，必须做结果校验。

如果你希望，我可以继续把这篇博客扩展成“可复现实验教程版”（附完整 SQL/脚本片段与目录结构）。
