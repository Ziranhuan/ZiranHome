---
title: 基于 MIMIC 数据库的脓毒症队列构建与数据清洗
date: 2026-07-11 10:00:00
tags:
  - MIMIC数据库
  - 脓毒症
  - 数据清洗
  - 医疗人工智能
categories:
  - 科研
---

## 项目背景

脓毒症是 ICU 中最常见的致死性疾病之一，全球每年影响数千万人，死亡率高达 10%-30%。早期识别和动态分层对降低死亡率至关重要。

本项目基于 MIMIC-III 和 MIMIC-IV 临床数据库，构建符合 Sepsis-3 国际标准的脓毒症队列，提取 SMART 8 项动态标志物，为后续的动态轨迹建模（GBTM + M0-M3 双任务）提供高质量数据基础。

## 一、MIMIC 数据库申请与资格审查

### 1.1 什么是 MIMIC？

MIMIC（重症监护医学信息库）是由 MIT 计算生理学实验室与 Beth Israel Deaconess 医疗中心联合维护的重症监护数据库：

- **MIMIC-III**：2001-2012 年，BIDMC ICU 收治的 4 万余例患者数据
- **MIMIC-IV**：2008-2019 年，更新版本的数据库，数据结构更规范

### 1.2 申请流程

MIMIC 数据库包含真实的患者隐私数据，访问需要严格的资格审查：

1. **注册 PhysioNetWorks 账号**：在 physionet.org 注册并完成身份验证
2. **完成 CITI 培训课程**：通过人类受试者保护在线培训，涵盖伦理原则、隐私保护、知情同意等模块
3. **通过资格考试**：完成 MIMIC 数据库的资格审查考试，内容包括数据使用协议、患者隐私保护规范、数据安全要求等
4. **签署数据使用协议**：承诺仅用于研究目的，不得尝试重新识别患者身份

整个申请过程历时约两周，最终获得了 MIMIC-III 和 MIMIC-IV 的完整访问权限。

## 二、数据提取流程

### 2.1 技术环境搭建

数据源：MIMIC-IV 3.1 + MIMIC-III，提取工具采用 MIMIC-Code 官方 concepts_duckdb 衍生表，中间库为 DuckDB（mimic_derived.db），脚本语言为 Python + SQL。

采用 MIMIC-Code 官方概念库构建衍生表，确保数据口径可引用、可复现，避免自行编写 SQL 造成的口径偏差。

```python
# 10_setup_views.py — 建库 + schema + 视图指向 CSV.gz（零拷贝）
import duckdb

SRC = 'D:/BaiduNetdiskDownload/mimic-iv-3.1'
DB  = 'D:/mimic_derived.db'
con = duckdb.connect(DB)

HOSP = ['admissions', 'diagnoses_icd', 'labevents', 'microbiologyevents',
        'patients', 'poe', 'poe_detail', 'prescriptions', 'services']
ICU  = ['chartevents', 'd_items', 'icustays', 'inputevents',
        'outputevents', 'procedureevents']

for sch in ['mimiciv_hosp', 'mimiciv_icu', 'mimiciv_derived']:
    con.execute(f"CREATE SCHEMA IF NOT EXISTS {sch}")

# 视图：read_csv_auto 大采样确保类型推断准确
def mkview(sch, tbl, folder):
    path = f'{SRC}/{folder}/{tbl}.csv.gz'
    con.execute(f"""CREATE OR REPLACE VIEW {sch}.{tbl} AS
        SELECT * FROM read_csv_auto('{path}', sample_size=-1)""")

for t in HOSP: mkview('mimiciv_hosp', t, 'hosp')
for t in ICU:  mkview('mimiciv_icu',  t, 'icu')
con.close()
```

### 2.2 队列筛选（Sepsis-3 标准）

队列定义严格对齐 Sepsis-3 国际共识，筛选漏斗如下：

```
全 MIMIC-IV ICU 患者
    ↓ 41,296 例
首次 ICU 入院（每病人去重，取最早 ICU stay）
    ↓ 37,512 例
ICU 住院时长 ≥ 24 小时
    ↓ 33,746 例
排除外院转入
    ↓ 24,616 例  ← MIMIC-IV 最终队列
```

```sql
-- 12_cohort_sepsis3.py — 官方 Sepsis-3 队列 + 论文纳排规则
-- 纳入: sepsis3=true, 成人≥18, 首次ICU, ICU LOS≥24h
-- 排除: 外院转入(TRANSFER FROM HOSPITAL)
CREATE OR REPLACE TABLE mimiciv_derived.cohort_s3 AS
SELECT d.subject_id, d.hadm_id, d.stay_id,
  d.icu_intime, d.icu_outtime, d.los_icu AS icu_los_days,
  d.gender, d.admission_age AS age, d.dod,
  d.hospital_expire_flag AS hosp_death,
  CASE WHEN d.dod IS NOT NULL
       AND date_diff('day', d.icu_intime, d.dod) BETWEEN 0 AND 28
       THEN 1 ELSE 0 END AS death_28d,
  s.suspected_infection_time, s.sofa_score AS sofa_at_onset
FROM mimiciv_derived.icustay_detail d
JOIN mimiciv_derived.sepsis3 s USING(stay_id)
JOIN mimiciv_hosp.admissions a USING(hadm_id)
WHERE s.sepsis3 = true
  AND d.admission_age >= 18
  AND d.first_icu_stay = true
  AND d.los_icu >= 1.0
  AND a.admission_location != 'TRANSFER FROM HOSPITAL'
```

**Sepsis-3 判定标准**：
- 疑似感染：抗生素使用 + 微生物培养时间窗匹配
- SOFA 急性升高 ≥ 2 分（相比基线）

> **论文原文（ML, npj Digital Medicine 2026）**
>
> "6822 excluded due to missing data (>20%, n = 2541), ICU stays <24 h (n = 1873), transfers from external hospitals (n = 1248), or pre-existing limitations of care (n = 1160)."
>
> —— Zhang R. et al., *Machine learning predicts sepsis deterioration trajectories*

**MIMIC-III 处理**：仅保留 CareVue 系统（2001-2008）数据，与 MIMIC-IV（MetaVision，2008-2019）零时间重叠，避免同一患者重复纳入。

### 2.3 变量提取

#### SMART 8 项核心标志物（日频）

| 指标 | 编号 | 临床意义 |
|------|------|---------|
| 白细胞（WBC） | 51301 | 全身炎症反应 |
| 淋巴细胞绝对计数 | 51133/52075 | 免疫抑制标志 |
| 中性粒细胞绝对计数 | 51133/52074 | 感染严重度 |
| 单核细胞绝对计数 | 52074 | 炎症调控 |
| 血小板 | 51265 | 凝血与消耗 |
| 凝血酶原时间（PT） | 51274 | 外凝途径 |
| 国际标准化比值（INR） | 51237 | 凝血标准化 |
| 活化部分凝血活酶时间（APTT） | 51275 | 内凝途径 |

> 五分类缺失时，用白细胞计数 × 百分比换算绝对计数，确保数据完整性。

> **论文原文（ZL, Nature Communications 2025）**
>
> "Eight markers (APTT, INR, lymphocyte, monocyte, neutrophil, WBC, PLT counts, and patient age) were identified and validated to delineate risk stratification (mild, moderate, severe, and dangerous) and sepsis subphenotypes (CIS1 and CIS2)."
>
> "Routine clinical tests (e.g., activated partial thromboplastin time (APTT), platelet count (PLT), international normalized ratio (INR), and white blood cell count (WBC)) offer fast, affordable, and practical alternatives for assessing sepsis heterogeneity."
>
> —— Zhu L. et al., *Explainable AI unravels sepsis heterogeneity via coagulation-inflammation profiles*

```sql
-- 13_smart_on_s3.py — SMART 8项逐日提取（Day1-7首值）
-- 从 labevents 按天取首值，五分类绝对计数缺失时用 WBC×百分比换算
CREATE OR REPLACE TABLE mimiciv_derived.smart_daily AS
WITH p AS (
  SELECT stay_id, icu_day,
    MAX(CASE WHEN itemid=51301 THEN valuenum END) wbc,
    MAX(CASE WHEN itemid=51265 THEN valuenum END) plt,
    MAX(CASE WHEN itemid=51237 THEN valuenum END) inr,
    MAX(CASE WHEN itemid=51275 THEN valuenum END) aptt,
    MAX(CASE WHEN itemid=51274 THEN valuenum END) pt,
    MAX(CASE WHEN itemid=51133 THEN valuenum END) lym_abs,
    MAX(CASE WHEN itemid=52075 THEN valuenum END) neut_abs,
    MAX(CASE WHEN itemid=52074 THEN valuenum END) mono_abs,
    MAX(CASE WHEN itemid=51244 THEN valuenum END) lym_pct,
    MAX(CASE WHEN itemid=51256 THEN valuenum END) neut_pct,
    MAX(CASE WHEN itemid=51254 THEN valuenum END) mono_pct
  FROM mimiciv_derived.smart_firstday
  GROUP BY stay_id, icu_day)
SELECT stay_id, icu_day, wbc, plt, inr, aptt, pt,
  COALESCE(lym_abs,  wbc*lym_pct/100.0)  AS lym,
  COALESCE(neut_abs, wbc*neut_pct/100.0) AS neut,
  COALESCE(mono_abs, wbc*mono_pct/100.0) AS mono
FROM p
```

#### SOFA 评分（6 小时粒度）

SOFA（序贯器官衰竭评估）是脓毒症器官功能评估的核心指标，按六大系统逐时计算：

- **呼吸**：氧合指数 + 机械通气
- **凝血**：血小板计数
- **肝**：胆红素
- **心血管**：平均动脉压 + 血管活性药剂量
- **神经**：格拉斯哥昏迷评分
- **肾**：肌酐 + 尿量

对齐论文方法，将官方逐时 SOFA 分箱为 6 小时粒度（前 72 小时，12 段），捕捉器官衰竭的动态变化。

```sql
-- 15_sofa_6h.py — 官方逐时 SOFA → 6h 分箱（前 72h = 12 个 6h 段）
-- 用 sofa_24hours（24h 滚动窗标准 SOFA），每 6h 段取该段内最大值
CREATE OR REPLACE TABLE mimiciv_derived.sofa_6h AS
SELECT s.stay_id,
  CAST(floor(s.hr/6.0) AS INT) AS bin6,   -- 0=0-6h, 1=6-12h, ...
  max(s.sofa_24hours) AS sofa_total,
  max(s.respiration_24hours) AS resp,
  max(s.coagulation_24hours) AS coag,
  max(s.liver_24hours) AS liver,
  max(s.cardiovascular_24hours) AS cardio,
  max(s.cns_24hours) AS cns,
  max(s.renal_24hours) AS renal
FROM mimiciv_derived.sofa s
JOIN mimiciv_derived.cohort_s3 c USING(stay_id)
WHERE s.hr >= 0 AND s.hr < 72
GROUP BY s.stay_id, bin6
```

> **论文原文（ML, npj Digital Medicine 2026）**
>
> "SOFA scores were calculated every 6 h. To capture the dynamic characteristics of time-series data, we extracted temporal features including raw and normalized values, rates of change (every 6, 12, and 24 h), variability indicators (standard deviation, coefficient of variation), trend features (linear regression slope), and fluctuation complexity indicators (sample entropy, approximate entropy)."
>
> —— Zhang R. et al., *Machine learning predicts sepsis deterioration trajectories*

#### 生命体征变异性（6 小时粒度）

提取心率、平均动脉压、呼吸频率等生命体征的标准差作为变异性指标（心率变异性、血压变异性、呼吸变异性），对齐论文的核心特征工程思路。

> **论文原文（ML, npj Digital Medicine 2026）**
>
> "The deterioration group displayed persistently elevated lactate (median peak: 3.4 mmol/L vs. 1.8 mmol/L in rapid recovery), reduced heart rate variability (HR SD: 8.4 bpm vs. 14.2 bpm)."
>
> "Reduced heart rate variability (SD < 10 bpm) predicted mortality (development: HR = 2.17; MIMIC-III: HR = 2.09; eICU: HR = 1.94)."
>
> "Patients in the clinical deterioration group showed progressive loss of heart rate variability 12-24 h before overt clinical deterioration."
>
> —— Zhang R. et al., *Machine learning predicts sepsis deterioration trajectories*

### 2.4 脚本流程

整个提取流程拆分为 27 个有序脚本（10-36 号），每个脚本完成一个独立任务：

```
10_setup_views.py      → 建立数据库视图
11_build_concepts.py   → 构建官方概念表
12_cohort_sepsis3.py   → Sepsis-3 队列筛选
13_smart_on_s3.py      → SMART 标志物提取
14_smart_score_s3.py   → SMART 评分计算
15_sofa_6h.py          → 6 小时粒度 SOFA
16_vitals_6h.py        → 6 小时粒度生命体征
17_assemble_s3.py      → 数据组装
18_analytic_cohort.py  → 分析队列构建
19_final_dataset.py    → 最终数据集
20-24                  → 模板导出与去重
30-37                  → MIMIC-III 独立流程
```

## 三、数据清洗

### 3.1 缺失值处理

SMART 8 项核心变量存在大量缺失（五分类原始缺失率 MIMIC-III 约 16%，MIMIC-IV 约 34%），采用两级插补策略：

- **第一级 LOCF**（末次观测值结转）：同一 ICU stay 内，用前次观测值填充后续缺失，五分类填充率提升至 96.8%
- **第二级 MICE**（链式方程多重插补）：补齐 LOCF 后残余缺失，最终核心变量 0% 缺失

> **论文原文（ML, npj Digital Medicine 2026）**
>
> "Missing data were handled using multiple imputation by chained equations for variables with <30% missingness. Variables with >30% missing data were excluded from the analysis."
>
> —— Zhang R. et al., *Machine learning predicts sepsis deterioration trajectories*

> **论文原文（ZL, Nature Communications 2025）**
>
> "Eight markers (APTT, INR, lymphocyte, monocyte, neutrophil, WBC, PLT counts, and patient age) were identified and validated to delineate risk stratification (mild, moderate, severe, and dangerous)."
>
> **Table 1 | SMART scoring system**（截断值与评分）
>
> | 标志物 | 截断值与评分 |
> |--------|------------|
> | APTT (s) | <24.0→0, <37.0→1, ≥37.0→2 |
> | 中性粒细胞 (×10⁹/L) | <1.8→2, <6.3→1, <25.0→2, ≥25.0→3 |
> | 年龄 | <46→-1, <58→1, <83→2, ≥83→3 |
> | 白细胞 (×10⁹/L) | <4.0→2, <10.0→1, <28.0→2, ≥28.0→3 |
> | 血小板 (×10⁹/L) | <125.0→2, ≥125.0→1 |
> | INR (PT) | ≤1.2→0, ≤1.4→1, ≤1.6→2, >1.6→3 |
> | 淋巴细胞 (×10⁹/L) | <1.1→2, ≥1.1→1 |
> | 单核细胞 (×10⁹/L) | <0.1→2, <0.6→1, ≥0.6→2 |
>
> 总分分级：Mild (≤8) / Moderate (9-12) / Severe (13-16) / Dangerous (≥17)
>
> —— Zhu L. et al., *Explainable AI unravels sepsis heterogeneity via coagulation-inflammation profiles*

```python
# 19_final_dataset.py — LOCF 核心补齐 + MICE 补残余 + 重算 SMART 评分
import pandas as pd, numpy as np
from sklearn.experimental import enable_iterative_imputer
from sklearn.impute import IterativeImputer

# 第一级：LOCF — 同一病人内按时间顺序前向填充
feats = ['wbc', 'plt', 'inr', 'aptt', 'lym', 'neut', 'mono']
df = df.sort_values(['stay_id', 'icu_day'])
df[feats] = df.groupby('stay_id')[feats].ffill()   # LOCF

# 第二级：MICE — 链式方程多重插补补齐残余缺失
print('MICE前缺失率:', {f: round(100*df[f].isna().mean(), 1) for f in feats})
imp = IterativeImputer(max_iter=10, random_state=0, min_value=0)
df[feats] = imp.fit_transform(df[feats])
print('MICE后缺失率:', {f: round(100*df[f].isna().mean(), 1) for f in feats})

# 在插补后数据上重算逐日 SMART 评分与等级
def smart(r):
    s  = 0 if r.aptt < 24 else (1 if r.aptt < 37 else 2)
    s += 2 if r.neut < 1.8 else (1 if r.neut < 6.3 else (2 if r.neut < 25 else 3))
    s += -1 if r.age < 46 else (1 if r.age < 58 else (2 if r.age < 83 else 3))
    s += 2 if r.wbc < 4 else (1 if r.wbc < 10 else (2 if r.wbc < 28 else 3))
    s += 2 if r.plt < 125 else 1
    s += 0 if r.inr <= 1.2 else (1 if r.inr <= 1.4 else (2 if r.inr <= 1.6 else 3))
    s += 2 if r.lym < 1.1 else 1
    s += 2 if r.mono < 0.1 else (1 if r.mono < 0.6 else 2)
    return s

df['smart_score'] = df.apply(smart, axis=1)
df['smart_level'] = pd.cut(df.smart_score, [-99, 8, 12, 16, 99],
                           labels=['Mild', 'Moderate', 'Severe', 'Dangerous'])

# 动态特征：每例 SMART 斜率（趋势）vs 死亡
g = df.sort_values('icu_day').groupby('stay_id')
def slope(x, y):
    return np.polyfit(x, y, 1)[0] if len(x) >= 2 else np.nan

feat_rows = []
for sid, grp in g:
    d1 = grp[grp.icu_day == 1]
    feat_rows.append(dict(
        stay_id=sid,
        smart_d1=d1.smart_score.iloc[0] if len(d1) else np.nan,
        smart_slope=slope(grp.icu_day.values, grp.smart_score.values),
        smart_max=grp.smart_score.max(),
        nlr_d1=(d1.neut.iloc[0] / d1.lym.iloc[0]) if len(d1) and d1.lym.iloc[0] > 0 else np.nan))
```

> 注意：">30% 剔除"规则仅应用于辅助特征池（如首日额外化验），不动核心暴露变量，保证 SMART 轨迹的连续性。

### 3.2 去重与合并

- **病人级去重**：同一患者仅保留最早 ICU 记录
- **跨库合并**：MIMIC-III（CareVue）与 MIMIC-IV（MetaVision）时间不重叠，可直接合并
- **来源标记**：合并数据集增加"来源医院"列区分数据来源

### 3.3 质量验证

数据集构建完成后，通过死亡率梯度验证数据可信度：

| SMART 分级 | 院内死亡率 | 趋势 |
|-----------|----------|------|
| 轻度 | 5.9% | 基线 |
| 中度 | 12.8% | ↑ |
| 重度 | 23.5% | ↑ |
| 危险 | 36.8% | ↑ |

| SOFA 评分区间 | 院内死亡率 |
|--------------|----------|
| 0-6 | 9.9% |
| 7-9 | 19.2% |
| 10-12 | 30.6% |
| 13+ | 51.7% |

死亡率梯度与临床认知和参考论文一致，验证了数据提取与清洗的正确性。

### 3.4 关键发现

生还组与死亡组的 SMART 斜率存在显著差异：

- **生还组**：SMART 斜率 -0.071（下降趋势，炎症逐步控制）
- **死亡组**：SMART 斜率 +0.034（上升趋势，炎症持续恶化）
- **第 1 天基线**：两组几乎无差异（12.6 vs 13.3）

这一发现支持了"动态趋势比静态基线更能区分预后"的核心假说，为后续轨迹建模提供了实证依据。

> **论文原文（ML, npj Digital Medicine 2026）**
>
> "SOFA slope analysis demonstrated that rapid decline (>2 points/24 h) correlated with reduced mortality (adjusted HR: 0.67, 95% CI: 0.58-0.78), while increases (>1 point/24 h) heightened risk (HR: 1.92, 95% CI: 1.65-2.24)."
>
> "Dynamic features and variability metrics consistently outranked static baseline characteristics in predictive importance."
>
> —— Zhang R. et al., *Machine learning predicts sepsis deterioration trajectories*

## 四、最终数据集

| 数据集 | 病人数 | 说明 |
|-------|-------|------|
| **MIMIC 合并** | **9,315** | III + IV 合并，主交付数据集 |
| MIMIC-IV | 6,847 | IV 单独 |
| MIMIC-III | 2,468 | III 单独（仅 CareVue） |

每个数据集包含 68 列：ICU 记录编号 + 患者编号 + 性别/年龄 + 第 1-7 天 × 8 项 SMART 指标 + 来源医院 + ICU 时长 + 住院死亡 + 死亡日期 + 28 天死亡 + 敏感性标记。

## 五、总结与展望

### 已完成

- MIMIC 数据库申请与资格审查
- MIMIC-III / MIMIC-IV 官方 Sepsis-3 队列构建
- SMART 8 项动态标志物提取（日频）
- SOFA 评分（6 小时粒度）与生命体征变异性提取
- 缺失值两级插补（LOCF + MICE），核心变量 0% 缺失
- 跨库合并与去重，交付 9,315 例主分析数据集
- 死亡率梯度与动态趋势验证

### 下一步

- GBTM（基于群的轨迹建模）轨迹建模
- M0-M3 双任务建模（动态分层 vs 静态预测）
- 样本熵/近似熵等非线性特征
- 与中国本地数据集的跨中心验证

---

> 本项目数据全部来源于 MIMIC-III 和 MIMIC-IV 数据库，已通过 PhysioNetWorks 资格审查，遵守相关数据使用协议。文中数据均为聚合统计，不涉及患者隐私。
