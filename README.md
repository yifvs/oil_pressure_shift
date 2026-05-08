# 7B 发动机滑油压力漂移监控系统

## 项目简介

本系统借鉴 **OEM 监控逻辑**，实现对发动机滑油压力漂移的自动化监测与早期预警。系统通过消除 N2 转速、油温、高度等物理干扰因素，提取纯残差信号，结合长短均值偏差指标与变点检测算法，精准识别滑油系统异常漂移趋势。

---

## 系统架构

```
┌─────────────────────┐     ┌──────────────────────────┐     ┌─────────────────────────┐
│  原始 QAR/ACARS 数据 │ ──> │  calculate_residuals.py  │ ──> │ snapshots_with_residuals │
│  (按航段 CSV 文件)   │     │  残差计算 & 基准建模      │     │        .csv             │
└─────────────────────┘     └──────────────────────────┘     └──────────┬──────────────┘
                                                                      │
                                      ┌───────────────────────────────┼───────────────────────────┐
                                      v                               v                           v
                             ┌──────────────────────────┐  ┌────────────────────────┐   ┌─────────┐
                             │ analyze_residual_trend.py│  │detect_coking_blockage   │  │(并行占位)│
                             │  趋势分析 & RUL 预估      │  │  结焦/堵塞物理极限监控    │  │         │
                             │  基于 ZPOIL_NORM 等效油压 │  │  基于 ACTUAL_OILPRS 绝对值│ │         │
                             └──────────┬───────────────┘  └──────────┬─────────────┘   └─────────┘
                                        │                            │
                        ┌───────────────┼──────────────┐             │
                        v               v              v             v
              ┌────────────────┐ ┌──────────┐ ┌──────────────┐ ┌────────────────────────┐
              │ ENG*_Trend_    │ │ fleet_*  │ │ENG*_Coking_  │ │ coking_blockage_*      │
              │ Diagnostic.png │ │ .csv     │ │ Alert.png    │ │ summary.csv            │
              └────────────────┘ └──────────┘ └──────────────┘ └────────────────────────┘
```

> **设计原则**: 本系统包含两条独立的监控链路：
> - **趋势漂移链路** (`analyze_residual_trend.py`): 使用等效油压 (ZPOIL_NORM)，消除工况干扰后检测缓慢漂移趋势，适合早期预警与 RUL 预估。
> - **结焦/堵塞链路** (`detect_coking_blockage.py`): **直接使用原始绝对值 (ACTUAL_OILPRS)**，因为 70 psi 是机械物理安全极限（超出可能导致油管或油封爆裂），这是一个不可妥协的死命令。

---

## 文件说明

| 文件 | 功能 |
|------|------|
| `calculate_residuals.py` | 数据预处理、巡航快照提取、Ridge 基准建模、残差计算 |
| `analyze_residual_trend.py` | OEM Phase II 指标计算、健康状态评估、变点检测、趋势漂移可视化报告生成（基于 ZPOIL_NORM） |
| `detect_coking_blockage.py` | **结焦/堵塞物理极限监控**，直接使用 ACTUAL_OILPRS 绝对值对照 70psi 红线，结合油温隔离判定排除物理干扰 |

---

## 环境依赖

```bash
pip install numpy pandas scikit-learn scipy ruptures matplotlib
```

---

## 使用方法

### 第一步：计算残差

编辑 `calculate_residuals.py` 中的配置参数：

```python
CONFIG = {
    "csv_folder": "G:\\B-XXXX左发滑油压力高",   # 原始数据文件夹路径
    "output_csv": "snapshots_with_residuals.csv", # 输出文件名
}
```

运行：
```bash
python calculate_residuals.py
```

**输出**: `snapshots_with_residuals.csv`，包含以下字段：

| 字段 | 说明 |
|------|------|
| `SNAPSHOT_TIME` | 巡航段中位时间戳 |
| `SEGMENT_ID` | 航段标识符（来源文件名） |
| `ENGINE` | 发动机编号 (ENG1 / ENG2) |
| `ACTUAL_OILPRS` | 实际滑油压力中位数 (psid) |
| `N2` | N2 转速中位数 (%) |
| `OILTMP` | 滑油温度中位数 (°C) |
| `ALT` | 高度中位数 (ft) |
| `EXPECTED_OILPRS` | Ridge 模型预测的期望油压 |
| `RESIDUAL` | 残差 = 实际值 - 期望值 |
| `ZPOIL_NORM` | 归一化等效油压 (基准截距 + 残差) |

### 第二步：趋势分析与告警

```bash
python analyze_residual_trend.py
```

**输出**:
- `oil_pressure_reports/ENG1_Trend_Diagnostic.png` — ENG1 诊断面板图
- `oil_pressure_reports/ENG2_Trend_Diagnostic.png` — ENG2 诊断面板图
- `oil_pressure_reports/fleet_risk_summary.csv` — 机队风险汇总表

### 第三步：结焦/堵塞物理极限监控（可选，与第二步并行）

```bash
python detect_coking_blockage.py
```

> **注意**: 此步骤与第二步完全独立，可单独执行或与趋势分析并行运行。两者使用同一输入文件 (`snapshots_with_residuals.csv`) 但监控逻辑不同。

**核心设计理念**:

结焦/堵塞监控 **不能** 使用等效油压 (ZPOIL_NORM)，必须直接使用 **原始绝对值 (ACTUAL_OILPRS)** 对照 70 psi 红线。原因如下：

| 维度 | 趋势漂移监控 (ZPOIL_NORM) | 结焦/堵塞监控 (ACTUAL_OILPRS) |
|------|--------------------------|------------------------------|
| 监控目标 | 缓慢漂移趋势、早期退化信号 | 机械物理安全极限 |
| 红线含义 | 统计意义上的异常边界 | **硬性物理极限**（超出可能导致油管/油封爆裂） |
| 数据处理 | 消除 N2/温度/高度干扰后看残差 | 直接看原始值，不进行工况归一化 |
| 判定逻辑 | Delta 长短均值偏差 + 斜率外推 | 压力阶跃上升 + **油温隔离判定** |

**输出**:
- `coking_blockage_alerts/ENG1_Coking_Alert.png` — ENG1 结焦/堵塞监控图
- `coking_blockage_alerts/ENG2_Coking_Alert.png` — ENG2 结焦/堵塞监控图
- `coking_blockage_alerts/coking_blockage_summary.csv` — 结焦报警汇总表

---

## 核心算法

### 1. 残差计算模型 (`calculate_residuals.py`)

采用 **Ridge 回归** 建立多变量基准模型：

$$
\text{OILPRS}_{\text{pred}} = a_1 \cdot \Delta N2 + a_2 \cdot \Delta T_{\text{oil}} + a_3 \cdot \Delta ALT + b
$$

其中：
- $\Delta N2 = N2 - 92.0\%$ （参考转速）
- $\Delta T_{\text{oil}} = T_{\text{oil}} - 100.0°C$ （参考油温）
- $\Delta ALT = ALT - 30000\,\text{ft}$ （参考高度）
- 前 30% 航段作为基线训练集，后续所有点用同一模型预测并求残差

### 2. 长短均值偏差 (`analyze_residual_trend.py`)

$$
\Delta_{\text{LT-ST}} = \text{Mean}_{\text{short}}(R) - \text{Mean}_{\text{long}}(R)
$$

- **短期均值窗口**: 5 个航段
- **长期均值窗口**: 30 个航段
- **告警阈值**: $|\Delta_{\text{LT-ST}}| > 1.5\,\text{psi}$

### 3. 趋势检测

| 方法 | 用途 |
|------|------|
| Theil-Sen Slope | 估算近期漂移速率（鲁棒回归，抗离群值）|
| PELT 变点检测 (ruptures) | 识别系统状态突变点，用于分段评估 RUL |

### 4. 健康状态评级（趋势漂移）

| 等级 | 判定条件 | 含义 |
|------|----------|------|
| 🟢 GREEN | \|Delta\| < 1.5 且斜率在安全带内 | 正常，无漂移 |
| 🟡 YELLOW | \|Delta\| >= 1.5 或接近红线 | 观察期，需持续监控 |
| 🔴 RED | 突破绝对红线 (70 psi) 或严重偏低 | 立即维护干预 |

> **注**: `Delta` = LT_ST_DELTA，即短期均值与长期均值的偏差值 (psi)。

### 5. 结焦/堵塞物理极限检测 (`detect_coking_blockage.py`)

**核心思想**: 当油温平稳时，压力发生阶跃式上升 → 高度怀疑供油管结焦或油封故障。

**判定流程**:

```
                    ┌──────────────────────────────┐
                    │ 计算 ACTUAL_OILPRS 的长短期均值 │
                    │   ST_MEAN (近3航段)           │
                    │   LT_MEAN (过去20航段)         │
                    └──────────────┬───────────────┘
                                   │
                    ┌──────────────▼───────────────┐
                    │  计算油温 OILTMP 的长短期均值  │
                    └──────────────┬───────────────┘
                                   │
              ┌────────────────────▼────────────────────┐
              │         油温是否平稳？                   │
              │        |ΔTMP| ≤ 4℃ ?                   │
              └────┬───────────────┬───────────────────┘
                   │ YES           │ NO
                   ▼               ▼
          ┌────────────────┐  ┌────────────────────────┐
          │ 压力 ≥ 70psi ?  │  │ 排除：温度变化导致      │
          │ 阶跃 ≥ 6psi?    │  │ 压力变化符合流体物理特性 │
          └──┬──────────┬───┘  └───────────┬────────────┘
            RED        YELLOW              GREEN
             ▼           ▼                  ▼
       紧急告警    注意(早期风险)      安全(正常物理现象)
```

**关键参数说明**:

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `st_window` | 3 | 短期均值窗口（最近航班数） |
| `lt_window` | 20 | 长期均值窗口（历史习惯） |
| `prs_redline` | 70.0 | **机械物理极限红线 (psi)** — 死命令，不可妥协 |
| `prs_shift_alert` | 6.0 | 压力突增阶跃阈值 (psi) |
| `temp_steady_limit` | 4.0 | 油温变化阈值 (°C) — 超出此值视为温度不稳，排除误报 |

**油温隔离判定的必要性**: 冬季低温环境下油压自然升高是正常的流体物理行为。只有当 **油温保持平稳 (|ΔT| ≤ 4℃)** 时检测到压力突增，才能确认为结焦/堵塞的机械故障信号。

---

## 配置参数详解

### `calculate_residuals.py` 关键参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `csv_folder` | `G:\1505左发滑油压力高` | 原始 CSV 数据目录 |
| `resample_rule` | `5S` | 重采样间隔（秒） |
| `cruise_n2_min/max` | 85 / 98 | 巡航 N2 有效范围 (%) |
| `max_dn2` | 0.3 | 最大 N2 变化率阈值 |
| `max_dalt` | 30 | 最大高度变化率阈值 (ft) |
| `min_oiltmp/max_oiltmp` | 60 / 140 | 油温有效范围 (°C) |
| `min_cruise_points` | 60 | 最少巡航数据点数 |
| `baseline_fraction` | 0.3 | 用于训练基准模型的航段比例 |
| `n2_ref` | 92.0 | N2 参考值 (%) |
| `t_ref` | 100.0 | 油温参考值 (°C) |
| `alt_ref` | 30000.0 | 高度参考值 (ft) |

### `analyze_residual_trend.py` 关键参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `median_window` | 3 | 中位数平滑窗口 |
| `lt_window` | 30 | 长周期均值窗口（航段数） |
| `st_window` | 5 | 短周期均值窗口（航段数） |
| `cfm_warning_limit` | 60.0 | 高压观察线 (psi) |
| `cfm_redline_limit` | 70.0 | 绝对红线 (psi) |
| `delta_alert_limit` | 1.5 | Delta 漂移告警界限 (psi) |
| `changepoint_penalty` | 10 | PELT 变点检测惩罚因子 |
| `safe_slope_band` | 0.005 | 安全斜率带宽 |

### `detect_coking_blockage.py` 关键参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `st_window` | 3 | 短期均值窗口（航段数）|
| `lt_window` | 20 | 长期均值窗口（航段数）|
| `prs_redline` | 70.0 | **机械物理绝对红线 (psi)** |
| `prs_shift_alert` | 6.0 | 压力阶跃上升告警阈值 (psi) |
| `temp_steady_limit` | 4.0 | 油温平稳判定阈值 (°C)，超出则排除误报 |

---

## 输出报告解读

### 趋势漂移诊断面板图（上下双屏）— `analyze_residual_trend.py`

**上图 — 等效油压趋势**:
- 灰色散点: 各航段等效油压原始值
- 彩色曲线: EWM 平滑后的慢趋势
- 橙色虚线: 60 psi 观察线
- 红色实线: 70 psi 绝对红线
- 紫色竖线: 检出的状态突变点

**下图 — OEM Delta 漂移量**:
- 黑色曲线: 短长期均值偏差随时间变化
- 红色水平线: ±1.5 psi 告警界限
- 红色填充区域: 超限告警时段

### 结焦/堵塞监控图（上下对照）— `detect_coking_blockage.py`

**上图 — 实际绝对油压 (ACTUAL_OILPRS)**:
- 彩色散点: 各航段原始绝对油压
- 灰色虚线: 历史习惯均值 (LT_MEAN)
- **红色实线: 70 psi 机械物理极限红线（死命令）**
- 红色箭头标注: 检测到压力阶跃突增时自动标注

**下图 — 滑油温度隔离监控 (OILTMP)**:
- 绿色散点: 实际油温数据
- 灰色虚线: 历史温度均值
- 红色水平箭头: 当触发告警时，标注 "Oil Temperature remains steady" 佐证油温未变

---

## 典型工作流程

```bash
# 1. 将 QAR 导出的航段 CSV 文件放入指定文件夹
# 2. 修改 CONFIG 中的 csv_folder 路径

# 3. 运行残差计算（生成中间数据）
python calculate_residuals.py

# 4a. 运行趋势漂移分析（基于等效油压 ZPOIL_NORM）
python analyze_residual_trend.py
# 输出 → oil_pressure_reports/

# 4b. [可选/并行] 运行结焦/堵塞物理极限监控（基于原始绝对值 ACTUAL_OILPRS）
python detect_coking_blockage.py
# 输出 → coking_blockage_alerts/

# 5. 查看各目录下的图表与汇总报告
```
