# PROJECT.md · 开发交接说明

> 这份文件是给「下一个接手的人（或 AI）」看的。读完它就能继续开发，不用重新摸一遍。

---

## 这是什么

**生活工作台** —— 单文件、零依赖、纯本地的个人生活管理工作台。

- 主文件：`生活工作台.html`（约 2900 行 / 190 KB，CSS + JS + 图标 + 图表全部内联）
- 备份版：`生活工作台-完整版.html`（额外含「记账理财」「日程统筹」，UI 停留在早期版本）
- 两版**共用同一 localStorage 键 `lifebench.v1`**，精简版保存时会把被移除模块的数据原样写回，不会丢

## 核心设计决策

**为什么要删掉记账和日程？**

手机系统（小米澎湃OS 4）已经覆盖了这两块：小米钱包的 AI 自动记账会抓微信/支付宝流水并分类出报表；系统日历 + 超级小爱灵感球能一句话加日程且带推送提醒。

所以本项目**主动砍掉这两个模块**，只保留系统做不到的：自定义打卡、习惯评分、待买清单、书影音收藏、跨模块汇总。

> 这是项目的立身之本。加功能前先问：系统是不是已经做了？

## 当前模块

| 模块 | 状态 | 说明 |
|---|---|---|
| 今日概览 | 启用 | 四维生活指数、时间进度、今天要处理、近 30 天节奏、本周小结、最近动态 |
| 习惯健康 | 启用 | 三种打卡方式、30 天热力图、加权近因评分 |
| 减脂健身 | 启用 | 体重体脂、7 日均线、BMI、目标进度、训练记录与容量趋势、动作进步、热量缺口、**减脂方案（BMR/TDEE/宏量）**、**饮食计划**、**训练模板一键套用**、周计划 |
| 待买清单 | 启用 | 优先级分组、金额合计、类别分布 |
| 书影音收藏 | 启用 | 状态/星级/短评、封面墙与列表、年度小结 |
| 数据与设置 | 启用 | 个人参数、模块状态、备份恢复 |
| 记账理财 | **已移除** | 数据保留，见完整版 |
| 日程统筹 | **已移除** | 数据保留，见完整版 |

## 数据模型

单一 localStorage 键 `lifebench.v1`：

```js
{
  v: 1,
  meta: {
    nick, height, tdee, budget, goalFrom, goalTo, weekplan[7], sinceBackup, lastBackup,
    // 减脂方案参数
    sex:'male'|'female', age:null|number, activity:1.2~1.9,
    goalMode:'cut'|'maintain'|'gain', deficit:kcal,
    proteinPerKg:2.0, fatPerKg:0.9, carbCycle:true
  },
  habits:   [{ id, name, type:'check'|'count'|'number', target, unit, color }],
  checkins: { [habitId]: { 'YYYY-MM-DD': number } },
  weights:  [{ id, date, weight, fat }],
  intake:   [{ id, date, kcal }],
  workouts: [{ id, date, spot, name, sets, reps, weight, dur, kcal, note }],
  templates:[{ id, name, note, builtin, days:[{ title, items:[{ spot, name, sets, reps, weight, dur }] }] }],
  shopping: [{ id, name, price, cat, pri, bought, boughtAt }],
  media:    [{ id, type:'book'|'movie'|'music', title, creator, year, status, rating, review, cover, date }],
  money:    [],   // 已移除模块，仅保留数据
  schedule: []    // 已移除模块，仅保留数据
}
```

`normalize()` 负责全字段类型清洗与兜底，**新增字段必须同步加进 normalize 的映射**，否则存取一圈会丢。

> ⚠️ **normalize 的铁律（踩过坑）**：函数里必须区分两个对象 —— `def`（`BLANK()` 返回值，只读的默认值基准）与 `base`（`clone(def)` 后的实际返回值）。所有 `num(x, 默认值, …)` 的兜底参数一律引用 **`def.meta.X`**，绝不能引用 `base.meta.X`。
>
> 原因：`Object.assign(out.meta, raw.meta)` 会把用户数据写进 `out`（也就是 `base`），一旦兜底值取自 `base`，它就等于用户那笔非法值。曾因此出现「`proteinPerKg:'x'` 清洗后仍是 `'x'`」。`fatPerKg:0` 当时看似正确纯属巧合（`0` 被下限截断成默认值 `0.4`）。

## 代码架构

单文件内按注释分区（`/* ===== N. 分区名 ===== */`）：

| 分区 | 内容 |
|---|---|
| 0. 基础工具 | `$` `$$` `esc` `uid` `clamp` 日期工具、`ICON` 图标表 |
| 1. 数据层 | `BLANK()` 默认结构、`normalize()` 清洗、`load()` / `save()` |
| 2-3. 提示与表单 | `banner()` `toast()` `openForm()` 声明式表单构建器 |
| 4. 统计计算 | 各类指标函数 |
| 5. 图表 | `ringSVG` `donutSVG` `lineChartSVG` `sparklineSVG` `overallHeatHTML` |
| 6-7. 演示数据 / 导入导出 | `seedDemo` `exportJSON` `importJSON` |
| 8. 导航路由 | `VIEWS_META` `render()` |
| 9-16. 各视图 | `VIEWS.index` `VIEWS.habit` … |
| 17. 交互动作 | `ACTIONS` 对象，`data-act` 事件委托 |
| 18. 事件绑定与启动 | `init()` |

**约定**：
- 视图函数返回 HTML 字符串，不操作 DOM
- 所有交互走 `document` 上的 `data-act` 委托，不逐个绑定
- `<select>` / `<input>` 走 `change` / `input` 监听，不走 click
- 表单统一用 `openForm({fields})`，字段类型：text / number / select / textarea / stars / color

## 训练记录（健身模块核心）

`DB.workouts` 是本项目里唯一「复杂」的数据结构，几条规则：

**容量（volume）** = `sets × reps × weight`，单位为 kg。有氧类 `sets/reps/weight` 为 0，容量计 0。

**运动消耗 `burnOf(date)`** 三级回退，避免重复计算：
1. 记录里显式填了 `kcal` → 直接用
2. 没填但有时长 → `时长(分钟) × MET`（有氧 9.2 / 力量 6.4）
3. 什么都没有 → 0

**热量缺口公式**：
```
缺口 = TDEE(基础消耗) + 运动消耗 − 摄入
```
> 注意早期版本只有 `TDEE − 摄入`，漏了运动项。改公式时 `intakeStats().sumDef` 要一起改，它负责近 7 日累计。

**`weekRange(ref)` 必须传基准日**：无参时默认今天。传参才能取「任意一周」的起止（`workoutStreakWeeks` / `volumeByWeek` 依赖此项）。曾经该函数忽略参数恒返回本周，导致连续周数虚报 104。

**`isRestDay(plan)`** 用**前缀匹配**判断休息日，`text.indexOf('休息') === 0`。默认计划里是「休息 / 拉伸」，用全等匹配会把它误判成训练日。

**训练日连续周数 `workoutStreakWeeks()`**：本周还没练时宽容跳过（周初正常），但**只宽容本周**，往前的任一空周即终止。用 `DB.workouts` 里的**最小日期**做下界，不能用 `workouts[0]`（它只是当前顺序的首条）。

## 减脂方案（营养计算）

**BMR · Mifflin-St Jeor**
```
男性：BMR = 10×kg + 6.25×cm − 5×age − 5
女性：BMR = 10×kg + 6.25×cm − 5×age − 161
```
缺体重 / 身高 / 年龄任一 → 返回 `null`（不返回 `NaN`）。`cutPlan()` 在 `bmr === null` 或无体重时整体返回 `null`，视图据此显示引导文案。

**TDEE** = `BMR × 活动系数`，系数取自 `ACTIVITY_LV`（1.2 / 1.375 / 1.55 / 1.725 / 1.9）。

**宏量分配 `macroSplit(kcal, kg)`** —— 蛋白与脂肪按体重定量，碳水补足剩余热量：
```
protein = proteinPerKg × kg      (默认 2.0 g/kg → 4 kcal/g)
fat     = fatPerKg     × kg      (默认 0.9 g/kg → 9 kcal/g)
carb    = max(0, (kcal − protein×4 − fat×9) / 4)
```
若 `kcal` 低于蛋白 + 脂肪的基本需求，返回对象带 `neg: true`，视图显示警告。

**碳水循环**：训练日 `target + 150` kcal（全给碳水），休息日 `target − 150`，周均不变。可由 `meta.carbCycle` 关闭。

**预计速度**：`缺口 × 7 / 7700` kg/周（7700 kcal ≈ 1 kg 脂肪）。

**训练模板 `FAT_TEMPLATES`**：内置「减脂 5 天分化」「减脂 3 天全身」，每个动作带 `r`（相对重量占比）。`instantiateTpl()` 用 `lastWeightOf(name)` 取该动作最近一次实际重量 × `r`，四舍五入到 0.5 kg；无历史记录则留 `0` 由用户填。套用后存入 `DB.templates`，可「记为今日训练」或「写入本周计划」。

## 习惯评分算法

参照 [Loop Habit Tracker](https://github.com/iSoron/uhabits)：

```
score = Σ w(i)·v(i) / Σ w(i)      w(i) = e^(−0.1·i)
```

- 30 天窗口，λ = 0.1，半衰期 ≈ 6.93 天
- `v(i)` 单日完成度：勾选类 0/1，计数与数值类取 `min(值/目标, 1)`
- `scoreWindow()` 对新习惯收窄窗口（最少 7 天），避免刚建就被压低
- 关键性质：**同样完成 10 天，集中在近期 vs 集中在早期，评分差 ≥20 分，而等权完成度完全相同**

## 视觉约定

**字号层级**（改字号请遵守，否则会退化成"一片小字"）：

| 层级 | 桌面 | 手机 |
|---|---|---|
| 环形数字 | 44 | 38 |
| 问候语 / 页面标题 | 32 / 28 | 26 / 23 |
| KPI 数字 | 26 | 27 |
| **日期行（Lead）** | **24** | **25** |
| 区块标题 | 16 | 17 |
| 列表主文字 | 15 | 15 |
| 日期 / 说明（Meta） | 14 | 15 |
| 标签 | 13 | 13.5 |
| 图例 | 12.5 | 13 |

- **全站最小字号 12.5px**，不得出现更小
- 重要信息（日期等）必须 **≥ 正文（14px）**，不能压在正文之下
- 移动端覆盖写在 `@media(max-width:900px)` 里，**不要写进 560px 块**

配色：暖纸质底 `#F6F3EE`，赭红 `#A8452A` / 墨绿 `#4E6B57` / 靛蓝 `#3E5C76`，衬线体数字（Georgia）。

## 怎么测

项目用 Node `vm` + 极简 DOM 桩跑内联脚本，无需浏览器：

```bash
node -e "
const fs=require('fs'),vm=require('vm');
const src=fs.readFileSync('生活工作台.html','utf8');
const js=src.slice(src.indexOf('<script>')+8, src.lastIndexOf('</scr'+'ipt>'));
// 构造 document / window / localStorage 桩，然后 vm.runInNewContext(js + 测试代码, 沙箱)
"
```

测试要点（历史回归覆盖）：
- 6 个视图渲染无 `undefined` / `NaN`，`<div>` 标签配平
- 各统计函数边界（记录不足时返回 `null` 而非 `Infinity`）
- 存储：损坏识别与恢复、导入导出往返、满 20 条备份提醒
- 字号：全站无 `<12px`、层级严格递减、移动端块位置正确
- 空数据下所有函数不抛错
- **训练模块**：容量公式、`burnOf` 三级回退、缺口含运动项、`weekRange(ref)` 传参、`isRestDay` 前缀匹配、`workoutStreakWeeks` 不越界与断档终止
- **减脂模块**：BMR 精确值（男性 70/175/25 → 1663.75；女性 55/165/30 → 1270.25）、TDEE 含活动系数、蛋白/脂肪 g/kg 定量、碳水补足且三大营养素热量合计 ≈ 目标、训练日 − 休息日 = 300 kcal、`mealPlan` 合计 ≈ 目标且 `0`/`null` 返回空数组、缺参数返回 `null`、极端缺口标 `neg`、模板 `r` 换算带上次重量、模板经 normalize 往返不丢字段
- **normalize 污染回归**：`proteinPerKg:'x'` → `2`（而非 `'x'`），且**连续两次调用默认值不漂移**（原 bug 的直接症状）

> 写测试时注意：断言里要**同时打出具测量值**（如 `'计划 N 天 (' + v + ')'`），否则失败时看不出是代码错还是期望值错——本次有两个失败项就是期望值写错而非代码错。

## 已知遗留

1. `生活工作台-完整版.html` 是**早期 UI**，后几轮的改进（信息密度、习惯评分、字号层级、训练记录、减脂方案）**没有回移**。如果要，需要把它当独立文件同步
2. 无截图。README 里可以补
3. 手机上用 `file://` 打开时 localStorage 可能不持久，建议「添加到桌面」或后续做成 WebView 套壳 APK
4. `workouts.note` 字段已在数据结构与 normalize 里预留，但表单尚未开放输入
5. 减脂方案的宏量百分比在餐次示例里用的是固定 28/45/27 比例，与 `cutPlan()` 按 g/kg 算出的实际比例可能略有出入（示例只用于给餐次结构参考）

## 下一步可以做什么

- 把完整版 UI 同步到最新
- 补 README 截图
- 打包成 Android APK（参考 [self-life](https://github.com/Donk567-god/self-life) 的 WebView 套壳做法）
- 从豆瓣/IMDb 导入书影音数据（参考 [Yamtrack](https://github.com/FuzzyGrim/Yamtrack)）
- 训练模块可继续扩展：休息计时器、组间记录、自定义模板编辑（当前只能套用内置模板）
- 饮食：接入食物库做「按克数记账」，替代当前按餐次估算
