---
name: battlemage
description: 亡灵战争整合包「战斗法师」类型兵种（现役 U05 炽焰战斗法师）的调参、双战斗融合、门控链、移速/击退抗性换算与实机验收工作流。Use when the user asks to tune, debug, create, or verify a battlemage-type caster (BC melee + Iron's Spells dual combat), including movement speed, knockback resistance, spell range gating, target acquisition, or pathfinding. Do not use for pure-staff casters (use the general new-hyw-unit skill) or non-caster melee units.
---

# 战斗法师（Battlemage）类型 · 专项 skill

> 集群定位：本 skill 是 `new-hyw-unit`（通用建兵流程）的**专项补充**。
> 通用流程（名册 JSON、卷轴、皮肤、构建部署、卷轴回收）先看 `new-hyw-unit`，本文只讲战斗法师特有的坑。
> 适用实体：现役 `undeadwar_hywcombat:roster_mage` 的 `U05` profile（`RosterMageEntity extends AbstractCasterUnit`）。

## 0. 一句话定义

战斗法师 = **HYW 原生 IronMage 施法链路 + 独立近战 goal（`MageMeleeStrikeGoal`）+ 强制贴脸导航**。
HYW 的 `IronMageEntity` 是纯 `RangedAttackMob`，永不挥剑；本类型靠"施法 + 独立近战"并行实现双战斗。

---

## 1. 现役配置基线（U05，2026-11-17 调参后）

| 项 | 值 | 位置 |
|---|---:|---|
| 实体 | `undeadwar_hywcombat:roster_mage`（与 U03/U04/U06/U07/U08/U09/U12 共享） | roster.json |
| 血 / 法力 | 70 / 300（+四件套 4×125 → 实测 800） | roster.json |
| **移速 base** | **0.36**（=玩家疾跑；生物平方机制，见 §4） | `RosterLoadoutService.movementSpeed()` case U05 |
| **击退抗性** | **0.35**（精英人形档；U05 特判，不改其它兵种） | `RosterLoadoutService.applyDelayed()` poise |
| 跟随距离 | 48（未动） | `casterAttributes(24, 48, ...)` |
| 探敌范围 | 64（`THREAT_SCAN_RANGE`，未动） | `AbstractCasterUnit.java:37` |
| attack_reach | 24（施法门控） | profile |
| 近战 reach | 3.2（`isWithinMeleeReach`） | `AbstractCasterUnit` |
| standoff / resume | 2.5 / 3.2 | roster.json |
| 主手 | `spartanweaponry:netherite_longsword`（锋利VI/火附II/横扫III，史诗紫+炼狱/雷击） | roster.json |
| 护甲 | `irons_spellbooks:pyromancer_*` 四件（spell1.20/fire1.40 → pm6.72） | roster.json |
| 法术池 | flaming_strike L4 / catch_flame L6 / combustion_jet L4 / fire_arrow L5 / oakskin L7 | roster.json |

**神化品质（2026-11-17 已改）**：高阶单位装备必须是神化橙（`apotheosis:mythic`）。
- 亡灵阵营的 `RosterTier.UNDEAD` 是 `affixRarity=null`，**不走自动 tier 策略**（`RosterAffixPolicy.apply` 开头就 `!hasHumanPolicy()` 返回）；即使是亡灵加 `tier:"SPECIAL"` 也无效，因为 `hasHumanPolicy()` 要求 `!isUndead()`。
- 因此高阶亡灵的神化词缀必须**在 roster.json 的装备 NBT 里写死**：`affix_data:{rarity:"apotheosis:mythic",affixes:{"<affixId>":1.0f}}`。
- U05 现值：武器 `netherite_longsword` = mythic + `sword/attribute/infernal` + `sword/special/thunderstruck`；四件 pyromancer 护甲 = mythic + 分别 ironforged / blockading / stalwart / windswept。
- 词缀 id 模板在 `RosterAffixPolicy.Slot` 的 fallback 里；照 U06/U10 高阶亡灵的写法。血量类词缀（blessed/max_health）被禁用，别用。

---

## 2. 双战斗融合实现（不可重造轮子）

- **近战 goal**：`MageMeleeStrikeGoal`，优先级 2、间隔 20 tick（1 秒/刀）、有效 3.2 格、不占 MOVE/LOOK、独立冷却不碰 `attackCoolDown`。`RosterMageEntity` 构造即 `addGoal(2, new MageMeleeStrikeGoal(this))`。
  - 平A 原 10 tick → DPS 240+ 过强 → 改 20 tick，**别改回去**。
  - 施法中不挥刀：`!isIronSpellCasting()` 才 strike。
- **施法链路**：`castWhenReady()` → `tryStartExternalCompatAttack()` → HYW `IronMageEntity` 外部兼容分支。**不要**用 `MeleeCapableRangedAttacker` + 切 `isUsingRanged=false`——那会在 24 格外挥空刀耗光冷却（历史踩坑）。
- **法袍/法杖不能与剑共存**：`StaffItem` 属性只挂主手，剑已占主手 → 不加法杖，法术书走饰品槽（spellbook accessory）。

---

## 3. 四层门控链（"认领"≠"开火"，最容易搞错）

从外到内，逐层过才开火：

1. **认领/追击**：扩展扫描 `findNearestThreat()` 每 5 tick、半径 64（`THREAT_SCAN_RANGE`）；原生 goal 用 `follow_range=48`。**只管认领与追击，不管开火**。
2. **施法射程**：`castWhenReady()` 里 `distanceToSqr > attack_reach²(24)` 直接 return。
3. **真实视线**：`canFireAtTarget()` 覆写为 `hasLineOfSight`（HYW 原生恒 true）；失败挂 10 tick 短 CD。
4. **近战形态门控**：`usesMeleeNavigation=true` 时，`tryStartExternalCompatAttack` 要求 ≤3.2 格，否则返回 true 且挂 5 tick CD——**连 24 格内的火球都不放，必须贴脸**。
5. **每法术射程**（第 5 层）：`CasterSpellRules.MAX_RANGE` 在候选池阶段剔除超距法术（flaming_strike=4、catch_flame=6、combustion_jet=10；fire_arrow 不在表按 32；oakskin 自增益不限距）。

**例子**：30 格敌人 → 64 内被扫描 → 认领追击 → 24 内仍不开火（近战门控）→ 贴到 3.2 格才施法。

**64 格与仇恨**：64 格内敌人**不会自动产生仇恨值**，只是进扫描候选池参与威胁评分（基础 −距离×0.2、当前目标+5、远程单位+3、正在攻击自己/友军+5）。**受击锁敌才是无条件真仇恨**：`hurt()` 里攻击者在 64 格内且 `isValidTarget` → 立刻 `setHywTarget` + `setAttackCoolDown(0)`。

---

## 4. 移速换算（最容易搞错，已踩坑）

**铁律：玩家与生物的 `movement_speed` 不可横向比较。**

- 玩家输入幅度恒 **1.0**（线性）；`LocalPlayer` 键盘输入 → 加速度 = 属性 × 常数。
- 生物：`Mob.setSpeed()`（`Mob.java:581-583`）把属性值**同时写进输入向量 zza** → 实际加速度 ∝ **属性²**。
- 换算：**实际速度比 = 生物属性² ÷ 玩家属性**。

| 目标观感 | 生物属性 base |
|---|---:|
| 玩家步行（0.10） | 0.316 |
| 玩家疾跑（0.13） | **0.36**（现役 U05） |
| 玩家 3 倍步行 | 0.548 |

**额外拖慢**：HYW `manageRangedUsingItemSpeedModifier`（`BaseCombatEntity.java:4610`）在 `isUsingItem()` 时挂 **-0.5 MULTIPLY_TOTAL**（速度砍半）——法师施法起手/拉弓时会触发，这是"调了移速感觉没快多少"的常见原因；`manageOverlapSpeedModifier`（:4627）拥挤时再 -0.5。全局倍率配置 `hyw_global_modifier.json5` 当前 1.0。

**wiki 公式**：`0.21168a(1−0.91f)f³` 是**分数** `0.21168a ÷ [(1−0.91f)·f³]`（网页文本抓取拍平了），a=属性、f=方块阻力。它描述玩家；生物公式再乘一个 a（平方）。

---

## 5. 击退抗性官方分档（不要拍脑袋设 1.0）

HYW 原生**没有任何可移动人形兵种是 1.0**。1.0 只给器械/Boss。

| 值 | 用途 |
|---:|---|
| 1.0 | 14 种攻城器械、Pavise、招募旗、战船、唯一 DesertRaiderBoss |
| 0.9 / 0.85 / 0.8 | 巨人 / 兽王 / 亡灵领主 |
| 0.6 | 棕熊（唯一） |
| 0.35 | 精英刺客 / **战斗法师（本类型现值）** |
| 0.2 | 正规重装近战（盾兵/长矛/僵尸/凋零骷髅/凋零法师） |
| 0.0 | 绝大多数（含全部骑士、弓手、法师） |

- 盾兵靠举盾临时 `max(0.2, 0.8)`，基础仍 0.2——**官方靠状态加成抗击退，不把基础写满**。
- 战斗法师不用盾，填 0.35 就是实打实 0.35；若担心被推飞，用状态/技能临时加成。
- **坑**：`RosterLoadoutService` 曾有错误注释"HYW 原生近战都设 1.0"（Warrior 实际 0.0、HywZombie 0.2），已修正，别再复活。

---

## 6. 寻路：根因链（11-05 → 11-14）

**现状**：不是"RTS 寻路过去式"这么简单——旧 `undeadwar_rtsbridge`（`PyromancerEntity`/`MageApproachGoal`）冻结；现役走 `AbstractCasterUnit.tickNativeMoveAttack` + `RosterNavigation`/`RtsPursuitGoal`（压 HYW priority-3 `MoveToBlockGoal` 走 `GroupPathingManager` 组路径）。

根因（详见项目 `design/HYW_0.8_寻路与索敌诊断.md`）：
1. 0.8 `BaseCombatNavigation` 硬编码 **8 格**搜索半径 → 原生 `pursueTarget` 只 2–4 节点截断路径；跨层靠组路径（16 万节点）。
2. **所有原地往复总根因**：`GoalSelectorSafeIterationMixin` 修 CME 时丢掉 vanilla `Map<Flag,WrappedGoal>` 旗标仲裁 → 多 MOVE goal 互拉；已重建 phase1/2/3。
3. **"改了没生效"总根因**：早期测错服（25575 是 0.7.1 基线，hywcombat 没装）；正确是 25566/25576。
4. 原生随机横移 `strafeAroundTarget` + 法师双控制器无迟滞 → `shouldSuppressWanderBehavior` 恒真 + 1.5 格迟滞修正。

**原则**：不自己写 pathfinder；复用 HYW 原生 goal + 少量自定义 goal；避免和 `BaseCombatEntityAttackGoal` 抢 MOVE。

---

## 7. 调参与实机验收流程（必做，别只改代码）

改 `RosterLoadoutService.java` 后：

1. **停服**（若服务端在跑）：`python _build/stop_hywcombat_test.py`。
2. **编译**：`cd mods_src/undeadwar_hywcombat && ./gradlew build --offline --no-daemon "-Dorg.gradle.jvmargs=-Xmx1500m"`（有 `profileSelfCheck` 断言）。
3. **部署双端**：把 `build/libs/undeadwar_hywcombat-0.1.0.jar` 复制到
   `_build/hywcombat_test/mods/` 与 `_build/hywcombat_client/mods/`，`md5sum` 三处比对一致。
4. **回档**（如需测手感）：`python _build/t0_restore.py`（现役 + 第二档同步回 T0）。
5. **起服**：Bash `run_in_background=true` 跑 `python _build/run_hywcombat_test.py`，等日志 `RCON running on`；再起 `python _build/t0_watchdog.py`。
6. **起客户端**：Bash `run_in_background=true` 跑 `python _build/launch_hywcombat_client.py --username Dev --quickplay-server 127.0.0.1:25566`，等日志 `Dev joined the game`。
7. **实机读属性**（关键，别跳过）：
   ```
   undeadwartest spawn U05 hostile
   execute as @e[type=undeadwar_hywcombat:roster_mage,limit=1] run attribute @s minecraft:generic.movement_speed get
   execute as @e[type=undeadwar_hywcombat:roster_mage,limit=1] run attribute @s minecraft:generic.knockback_resistance get
   execute as @e[type=undeadwar_hywcombat:roster_mage,limit=1] run attribute @s minecraft:generic.follow_range get
   undeadwartest clear
   ```
   期望：0.36 / 0.35 / 48.0。
8. **报告**：把实测值和"改了什么"列清楚，别以"编译成功"当验收。

---

## 8. 常见误判

- **"移速值改了但看起来没快"** → 先查 `manageRangedUsingItemSpeedModifier` 是否施法砍半（§4），再查是不是期望过高（0.30→0.36 实际只 +11%）。
- **" resistances 不像个兵"** → 1.0 是器械/Boss 档；人形精英最多 0.35（§5）。
- **"贴脸了也不放法术"** → 查 §3 四层门控，尤其近战形态 3.2 格门 + `CasterSpellRules.MAX_RANGE`。
- **"隔层就丢目标"** → `canTrackTargetWithoutLineOfSight` 应让 64 格内保留；开火仍需真实 LOS。
- **"改动没生效"** → 先确认测的服是 25566、jar MD5 三处一致、客户端也重启了。

---

## 9. 相关文件索引

- 配置：`data/undeadwar_hywcombat/roster/roster.json`（U05 条目）
- 源码：`roster/RosterLoadoutService.java`、`roster/CasterArchetypes.java`（`case "U05" -> BATTLE`）、`roster/CasterSpellRules.java`、`entity/RosterMageEntity.java`、`entity/AbstractCasterUnit.java`、`ai/MageMeleeStrikeGoal.java`、`ai/RosterNavigation.java`、`ai/RtsPursuitGoal.java`
- 项目文档：`design/亡灵兵种_U05炽焰战斗法师_全参数普查_2026-11-16.md`、`design/HYW原生兵种数值全表_2026-11-16.md`、`design/四核心数值_中文类比解释_2026-11-16.md`、`design/四追问_移速门控与仇恨_2026-11-16.md`、`design/HYW_0.8_寻路与索敌诊断.md`
