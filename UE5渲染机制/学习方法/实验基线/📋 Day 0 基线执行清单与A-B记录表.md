# 📋 Day 0 基线执行清单与A/B记录表

主题：UE5 GPU Crowd 性能工程

用途：
在进入 Phase 1 前，先把测试口径和基线数据固定下来，保证后续每次优化都有可复现对比。

---

## ✅ Day 0 一次性执行清单

### 1) 环境固定

- [ ] UE5 版本固定（记录完整版本号）。
- [ ] 项目分支固定（记录分支名和 Commit）。
- [ ] 测试机器固定（CPU/GPU/内存/驱动版本）。
- [ ] 分辨率固定（例如 1920x1080）。
- [ ] 画质档固定（Scalability 级别）。
- [ ] VSync 固定（开或关，必须一致）。
- [ ] 后台程序控制到最少。

### 2) 场景固定

- [ ] 固定测试地图（名称唯一）。
- [ ] 固定相机路径（长度和时长一致）。
- [ ] 固定角色数量和出生分布。
- [ ] 固定角色行为脚本（Idle/Walk/Run 等）。
- [ ] 固定运行时长（建议每次 60 秒）。

### 3) 指标采集流程固定

- [ ] 运行前先预热 10-20 秒。
- [ ] 采集 `stat unit`。
- [ ] 采集 `stat anim`。
- [ ] 采集 `profilegpu`。
- [ ] 采集 Unreal Insights（30-60 秒）。
- [ ] 每个配置至少跑 3 次取均值。

### 4) 数据输出固定

- [ ] 保存截图命名规范统一。
- [ ] 保存 CSV 命名规范统一。
- [ ] 记录当次开关矩阵（Budget/Sharing/Mass/VAT）。
- [ ] 记录异常现象（卡顿、抖动、错帧）。

---

## 🧪 A/B 测试矩阵（先填这个）

| Case ID | Budget | Sharing | Mass | VAT | 说明 |
|---|---|---|---|---|---|
| A0 | Off | Off | Off | Off | Baseline 全原生 |
| A1 | On | Off | Off | Off | 仅 Budget |
| A2 | On | On | Off | Off | Budget + Sharing |
| A3 | On | On | On | Off | Budget + Sharing + Mass |
| A4 | On | On | On | On | Budget + Sharing + Mass + VAT |

---

## 📊 结果记录表（每次测试都填）

| 日期 | Case ID | 次数 | GT(ms) | RT(ms) | GPU(ms) | DrawCall | Visible Crowd | Anim(ms) | 结论 |
|---|---|---|---:|---:|---:|---:|---:|---:|---|
| YYYY-MM-DD | A0 | Run1 |  |  |  |  |  |  |  |
| YYYY-MM-DD | A0 | Run2 |  |  |  |  |  |  |  |
| YYYY-MM-DD | A0 | Run3 |  |  |  |  |  |  |  |
| YYYY-MM-DD | A1 | Run1 |  |  |  |  |  |  |  |
| YYYY-MM-DD | A1 | Run2 |  |  |  |  |  |  |  |
| YYYY-MM-DD | A1 | Run3 |  |  |  |  |  |  |  |

建议：同一 Case 跑 3 次，保留均值和 P95。

---

## 📈 汇总表（写报告用）

| 指标 | Baseline (A0) | 当前最佳 (Ax) | 改善幅度 |
|---|---:|---:|---:|
| GT(ms) |  |  |  |
| RT(ms) |  |  |  |
| GPU(ms) |  |  |  |
| DrawCall |  |  |  |
| Visible Crowd |  |  |  |
| Anim(ms) |  |  |  |

改善幅度计算：

- 时间类（越低越好）：`(Baseline - Current) / Baseline * 100%`
- 数量类（越高越好，例如 Visible Crowd）：`(Current - Baseline) / Baseline * 100%`

---

## 🗂️ 证据清单（面试/复盘必备）

- [ ] `stat unit` 对比截图（同视角同时段）。
- [ ] `profilegpu` 对比截图。
- [ ] `stat anim` 对比截图。
- [ ] Insights 截图（GT/RT 关键段）。
- [ ] 开关矩阵截图或配置文件。
- [ ] Demo 录屏（Baseline vs 当前最佳）。

---

## 🧭 每日执行模板（复制粘贴）

```markdown
## YYYY-MM-DD 日报

### 今日目标
- 

### 今日改动
- 分支：
- Commit：
- 模块：
- 开关矩阵：Budget( ) Sharing( ) Mass( ) VAT( )

### 测试设置
- 地图：
- 分辨率：
- 画质：
- 角色数：
- 运行时长：

### 结果
- GT(ms)：
- RT(ms)：
- GPU(ms)：
- DrawCall：
- Visible Crowd：
- Anim(ms)：

### 现象与问题
- 

### 明日计划
- 
```

---

## 🧱 判定规则（避免“优化幻觉”）

1. 只改一个变量：每次实验只改一个核心开关。
2. 保持同口径：地图、路径、时长、分辨率必须一致。
3. 至少三次：单次好看不算结论。
4. 先看稳定性：先看 P95，再看平均值。
5. 有回退路径：任何优化都要能一键关闭。

---

## ✅ Day 0 完成定义

满足以下条件即完成：

- [ ] Baseline（A0）数据完整。
- [ ] 至少一个优化 Case（A1/A2）数据完整。
- [ ] 三次重复测试完成并有均值。
- [ ] 证据材料归档完成。
- [ ] 可在主计划中开始 Phase 1。
