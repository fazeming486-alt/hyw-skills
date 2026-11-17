---
name: pathing-debug
description: 调试 HYW 整合包里单位的寻路 / 索敌 / 追击问题。当单位"原地踏步、走三步退一步、上楼下楼反复、撞墙横跳、对新楼层敌人没仇恨、打完架还朝消失的目标走"时使用。提供每 tick trace 采集、五个量化指标、0.8 的源码级机制、七个已知陷阱和一套已验证的修法。
---

# HYW 寻路 / 索敌 / 追击 调试

> 依据 0.8.0r-fix2 反编译 + 2026-11-17 实机验证（U01 剑士、U05 炽焰战法、官方民兵对照）。

## 什么时候用

- 前后左右原地往复、像在踏步
- 走三步退一步，或在楼梯中途掉头
- 上楼打完人又跳下来，反复上下
- 贴着墙左右横跳
- 对楼上 / 墙后的敌人完全没有仇恨
- 打完架之后还在朝早已消失的目标位置走

## 第一步：抓数据，别猜

```bash
undeadwartest trace @e[type=<实体>] 900
```

输出 `_build/unit_trace.log`，**每 tick 一行**：

```
<tickCount> <entityId> <x> <y> <z> op=<MoveControl> nav=<isDone> tgt=<Y|-> goals=<正在跑的 goal>
```

- `tgt` 直读 `getHywTarget() == null`
- `goals` 是当前**运行中**的 goal 类名（未激活的不显示）

## 第二步：五个指标定性

| 指标 | 含义 | 判据 |
|---|---|---|
| `tgtFlip` | `tgt` 在 Y / - 之间翻转次数 | ≥2 = 目标反复丢失重锁 |
| `vTrips` | Y 方向反转次数 | 不在爬楼段却 ≥2 = 垂直往复 |
| `vRatio` | Σ\|Δy\| ÷ 净 Y 变化 | >3 = 反复上下同一段 |
| `hRev` | XZ 方向反转次数 | 沿路线转弯是正常的，原地才可疑 |
| `idle` | `nav=false` 但位移 <0.05 的 tick 占比 | >30% = 有路径走不动 |

`goals` 常常一眼定案：**两个 MOVE 类 goal 并存**（如 `BaseCombatEntityAttackGoal` + `MoveToBlockGoal`）就是抢 MOVE 旗标。

## 0.8 关键机制

### 索敌
- `BaseCombatEntity.addTargetSelector()` 注册 `HywNearestAttackableTargetGoal(this, LivingEntity.class, 20, true, false, pred)`，第 4 参 **`mustSee = true`**
- `createTargetingConditions()` 只在 `!mustSee` 时忽略视线 → 默认强制 LOS
- **垂直搜索半径** = `max(0.25 × follow_range, 4)`（远程 `max(0.5 × d, 4)`）
- 丢视线 **100 tick** 后清目标，UUID 记入 `sightExpiredTargetUUID`，必须重新可见才能再锁

### 强制索敌（官方真身）
- `forceAttackTarget` 字段为 true 时，`checkAndUpdateTarget()` **完全跳过 LOS 和 100 tick 过期**
- 唯一开启者：`TargetAttackGoal.start()`（玩家 RTS 攻击命令）
- 关闭：`TargetAttackGoal.stop()`（清目标 + `setHomePosition(当前位置)` + 关 force）
- **自动巡逻永远不吃这条路**

### 两条追击通道

| | 路径预算 | 跨层 |
|---|---|---|
| 原生 `pursueTarget` | `BaseCombatNavigation` 硬编码 **8 节点**（`BCN:227` 强制 `range=8.0f`） | **不能** |
| `TargetAttackGoal` → `GroupPathingManager` | 16 万节点 + `LadderAwareNodeEvaluator` | **能**（梯子/楼梯/384 格垂直） |

`NavigationReachabilityUtil.isLivingTargetPathReachable()` 要求路径终点与目标 **Y 差 ≤ 1.0**。

### 组路径细节
- `PathAssignment` 有 **CONNECTING → FOLLOWING** 状态机，`createPath()` 返回 null 时 `abandon()`
- `MoveToBlockGoal` **每 tick** `setCommandHomePosition(targetPos)`（`:217`），会把 home 改写到目的地
- repath 冷却 20 tick，重试 40 tick × 3

## 七个已知陷阱

### 1. `NoGravity` 会禁掉跨层寻路
给测试单位加 `NoGravity:1b`，路径评估会认为"不需要落地支撑"，**直接拒绝所有上下楼路线**。排查时一定会误判成"引擎不能跨层"。

### 2. `RosterProfileId` 必须在 `ForgeData` 下
```mcfunction
# 错
summon <entity> <x> <y> <z> {RosterProfileId:"U05"}
# 对，且必须带 roster 标签
summon <entity> <x> <y> <z> {ForgeData:{RosterProfileId:"U05"},Tags:["undeadwar_roster_unit",...]}
```
没有 `undeadwar_roster_unit` 标签 → 所有扩展逻辑不生效、`findNearestThreat` 根本不被调用。

### 3. 孤儿 `MoveToBlockGoal`
`MoveToBlockGoal.canContinueToUse()` = `!shouldRemove && !shouldYieldToCombatTarget`，**从不检查目标还在不在**。goal 被挤出队列却没标记 `shouldRemove` 就永久运行——目标没了、命令还指着楼上，单位无限撞墙。

**修**：注入 `m_8045_` HEAD，当「roster 单位 + `getHywTarget() == null` + 不是 `getCurrentCommandedGoal()`」时返回 false。

### 4. `isTargetWithinAttackRange` 不查视线
`BaseCombatEntity.isTargetWithinAttackRange()`（`:1121-1133`）是**纯距离判定**。法师 `attackReach=24`，站在楼下距楼上敌人 23.6 格就被判定"已到位"→ 站桩 → 打不到（要 LOS）→ 站 100 tick → 目标过期 → 5 tick 后重锁 → **无限循环**（实测 60 秒 28 次翻转）。

**修**：远程分支加 `hasLineOfSight`，并让攻击 goal 在"有自己移动命令且目标不可攻击"时让位，保证 MOVE 只有一个写者。

### 5. 手搓组路径会"走三步退一步"
每 20 tick `setTargetPos()` → 每次 `registerEntity` 重新注册 → **可能返回另一条等价路线** → 楼梯中途掉头。

**修**：直接复用官方 `TargetAttackGoal`，它走 CONNECTING/FOLLOWING 状态机，只在 assignment 要求时重规划。

实测对比（B 轮 · 二楼偏移 · 900 tick）：

| 指标 | 手搓 MoveToBlockGoal | 官方 TargetAttackGoal |
|---|---|---|
| 垂直往复 | 110 | **17**（全在爬楼阶梯） |
| 水平折返 | 114 | **29** |
| 总行程/净位移 | 13.1× | **4.7×** |
| 行为 | 无限撞墙横跳 | 爬楼后静止 400 tick |

### 6. 看门狗判据不能用"Y 有没有变"
单位卡在 1.25 格高的台面上会反复跳：跳上去 → 落不下来 → 掉回 → 再跳，**Y 每两 tick 就变一次**。任何"Y 变化即算进展"的规则都会永远重置计时器（实测 630 tick 不认输）。

**正确判据**：**与目标楼层的差距是否缩小**（`pursuitBestGap`）。

### 7. 拉黑只靠时间不够
"追 11 秒 → 放弃 10 秒 → 重锁同一个静止目标 → 再追" 会形成稳定 21 秒循环。

**正确做法**：**时间到 + 目标换过格子** 才允许重试；目标不动就一直拉黑。

## 跨层索敌：跟跟随范围的关系

垂直半径 = `max(0.25 × follow_range, 4)`。两个真实对照：

| | follow_range | 垂直半径 | 二楼差 13 格 | 卡在哪 |
|---|---|---|---|---|
| 官方民兵 | 20（原生） | **5 格** | 框外 | **范围 + 视线** 双重 |
| U01 剑士 | 64（roster 覆盖） | **16 格** | 框内 | **只剩视线** |

**所以 U01 继续调大 follow_range 完全没用** —— 框早就够到了，是视线测试在拒绝。必须解除 LOS。

### 解除 LOS 的正确姿势（两个坑）

**坑 1**：mixin 必须 `remap = false`（HYW 方法名不是 Minecraft 名），而这会**同时禁用注入体里 vanilla 调用的 remap**，`conditions.ignoreLineOfSight()` 运行时静默失效。

**坑 2**：`<init>` 注入时实体**还没打 `undeadwar_roster_unit` 标签**（标签由 `RosterLoadoutService` 稍后打上），判断永远失败。

**可行做法**：翻转 HYW 自己的 `mustSee` 字段（`@Mutable @Shadow`），在 `m_8036_`（canUse）HEAD 执行。注意 `HywNearestAttackableTargetGoal` **没有覆写 `m_8037_`**，往 tick 注入会无效。

**测试注意**：原生载体（`hyw_zombie` 等）必须给 owner（`undeadwartest relation @e[...] a`），因为 HYW 的 `isTargetingDisabled` 拒绝 null-owner 单位的**任何**目标。扩展载体（`AbstractCasterUnit` 子类）走自研 `findNearestThreat`，不受这条限制。真实游戏里卷轴召唤的单位都有 owner。

## 验证流程

```mcfunction
# 1. 加载测试区，否则实体不加载、execute if entity 全部失败
forceload add <x1> <z1> <x2> <z2>

# 2. 摆场（注意：不要加 NoGravity）
summon <entity> <x> <y> <z> {ForgeData:{RosterProfileId:"<ID>"},PersistenceRequired:1b,Silent:1b,Invulnerable:1b,Tags:["uwtest","undeadwar_roster_unit","<其他标签>"]}

# 3. 敌对关系（原生载体两边都要 owner，扩展载体只给靶子设）
undeadwartest relation @e[tag=uw_target] b

# 4. 采集
undeadwartest trace @e[tag=uwtest] 900
```

判读要点：
- 爬楼阶段 Y 单调上升时 `vTrips` 会虚高，要看**是否在某一段反复**
- `tgt=Y: 0/N` 全程无目标 = 索敌坏了，先查标签、`ForgeData`、owner
- `goals` 里两个 MOVE goal 并存 = 抢旗标

## 一句话原则

**官方已有的轮子（`TargetAttackGoal` + `GroupPathingManager`）比自己手搓的强**。遇到"寻路质量差"，先问"官方是怎么做的"，而不是调参数。
