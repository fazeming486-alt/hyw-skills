---
name: wind-mage
description: 亡灵战争整合包「风系法师」类型兵种专项（现役 U13 亡灵军中阶风系法师）。纯法杖 ANCHOR 站桩法师，覆盖风暴之书与五法术池、按法术射程过滤与位移专用窗口、风甲四件套与神化词缀、法力上限叠加实测、图标与部署验收。当用户要做/改风系法师、风法杖、风甲或风系法术池时使用；双战斗融合法师用 battlemage，通用建兵用 new-hyw-unit。
---

# 风系法师专项（U13）

## 0. 定位与边界

- U13 = **亡灵军中阶风系法师**，类型 = **站桩法师 / ANCHOR**（中庸全面，**不是固定不动**；固定不动属于炮台法师）。
- 定位 = **强击退 + 控场 + 位移敏捷**，不是输出位。风刃只是填充。
- 基础移速 0.36（与 U05 相同，非装备叠加后）；敏捷靠法术位移，不靠提高基础属性。
- **纯法杖施法者**：不要套用 `battlemage` 的 BC 近战 + 铁魔法双战斗融合路线。
- 等级铁律见 `new-hyw-unit` §0：固定 1，禁止 HYW 等级机制。

## 1. 装备（全 epic + 神化词缀）

| 槽位 | 物品 | 词缀 | 备注 |
|---|---|---|---|
| mainhand | `wind_spellbooks:wind_staff` | `apotheosis:telepathic`（念动） | 无 `attack_range`，不会被 HYW 压到 3 格；念动无槽位限制，确定生效 |
| head | `wind_spellbooks:aeromancer_helmet` | `apotheosis:armor/attribute/ironforged` | 护甲 +；2026-11-17 用户要求补上头盔 |
| chest | `wind_spellbooks:aeromancer_chestplate` | `apotheosis:armor/dmg_reduction/blockading` | 物理减伤 |
| legs | `wind_spellbooks:aeromancer_leggings` | `apotheosis:armor/attribute/stalwart` | 击退抗性 +0.15~0.2，契合强击退 |
| feet | `wind_spellbooks:aeromancer_boots` | `apotheosis:armor/attribute/windswept` | 风袭，移速 ×1.15~1.35 |

- 品质一律 `rarity:"apotheosis:epic"`（紫），不是 mythic。
- **每件护甲必须带 `HywIntrinsicItem:1b`**：否则 HYW `normalizeManagedEquipment` 把护甲移出槽位，Apotheosis 属性词缀完全不生效（`design/法师实体性能瓶颈定位_2026-11-17.md` 实证 U05 修复前后 speed 0.36→0.486、armor 0→24）。
- 附魔统一 protection4 / unbreaking3 / mending1。
- 避开 `blessed/max_health`（策略层禁止）、`damage_bonus`/`ammo_capacity`（已知解析失败）。

## 2. 法术书与法术池

- 法术书 = `wind_spellbooks:wind_spell_book`（英文 **Book of Tempests**，中文"风暴之书"为意译；jar 只有 en_us，无官方中文）。**不是** `irons_spellbooks:netherite_spell_book`（远古法典，v1 用过已替换）。
- 容器 NBT：`maxSpells:8, spellWheel:1b, mustEquip:1b, improved:0b`。
- 5 个法术（等级写在 NBT 的 `level` 字段）：

| id | L | CD | 蓝耗 | 效果 | 中文名 |
|---|---:|---:|---:|---|---|
| `wind_spellbooks:wind_blade` | 3 | 1s | 15→33 | 瞬发穿刺风刃，穿透多目标 | 风刃* |
| `wind_spellbooks:almighty_push` | 1 | 15s | 40→85 | 360° 击退 + 撞墙伤害 | **神罗天征**（用户口径，勿写"万能推"） |
| `irons_spellbooks:gust` | 3 | 12s | 30→75 | 吟唱 0.8s，前方扇形击退 | 呼啸之风（官方） |
| `wind_spellbooks:wind_jump` | 1 | 5s | 30 | 升空位移 + 自带 5 格击退 | 风跃* |
| `wind_spellbooks:iron_slash` | 2 | 8s | 30→90 | 吟唱 1s，前向冲刺 10+2L 格 | 铁斩* |

\* 非官方意译，wind_spellbooks 无 zh_cn。

**禁止入池**：`wind_spellbooks:tornado`（《交接手册》记载进服 1.5–2.5 分钟自发施放炸服，多次复现）、`tailwind`、`aeropic`、`invisibility`、`throw`、`fujin`、`feather_rush`。

## 3. 射程过滤与位移专用窗口

- 射程表在 `CasterSpellRules.MAX_RANGE`：wind_blade 48 / almighty_push 8 / gust 8 / wind_jump 8 / iron_slash 24。Java 侧拿不到 ISB 法术自身射程（`SpellPoolEntry` 无 range 字段），只能靠这张硬编码表。
- 法术选择是 HYW 均匀随机，无权重；扩展靠"收窄候选池 / 返回单元素列表"控制。
- U13 专属门（`CasterSpellRules.filter()`，按 `"U13".equals(profile.id())` 守卫）：
  - 距离 **< 4 格** → 只放 `wind_jump`（升空脱困，自带 5 格击退）；
  - 距离 **18–24 格** → 只放 `iron_slash`（唯一前冲补位）；
  - 两个位移术从普通随机池 `removeIf` 剔除，否则站桩时会随机前冲再走回，重演前后振荡。
- **不改** `approachStopRange()` / `getAttackReach()` / `allowsRangedCombatMovement()=false`；原生 kite 对施法者关闭，改回 true 会振荡。

## 4. 数值与法力实测

- U13 当前：level 1 / hp 70 / hpBonus 0 / mana 400 / standoff 14 / resume 17 / meleeNavigation false / ANCHOR。
- 真实血量实测 = 70（不是按 `(hp-hpBonus)/5×4` 推算的 56，以实机 `/attribute get` 为准）。
- **法力必须实测**：`RosterLoadoutService` 写 `max_mana base set 400` + `mana_regen base 0` + `magic_data.mana=400`；风甲材质类带 `MAX_MANA "Max Mana" ADDITION 150.0`，四件风甲各 +150 → 2026-11-17 实测最终 **1000**（400 + 4×150）。法术书通过 `mage.setArmySpellbook()` 安装，不走 curios 槽，其自身 MAX_MANA 加成未观察到额外增量。
- 实测命令：
  ```
  undeadwartest spawn U13 Dev
  attribute @e[tag=undeadwar_role_u13,limit=1] irons_spellbooks:max_mana get
  attribute @e[tag=undeadwar_role_u13,limit=1] irons_spellbooks:max_mana base get
  data get entity @e[tag=undeadwar_role_u13,limit=1] irons_spellbooks:magic_data
  ```

## 5. 图标规范（U05 同款）

- 32×32：背景 `(18,24,32,255)`；**2px 绿色边框 `(72,116,64)`**；半透明白框 `255,255,255,60`；白色头像框 14×14（x=9..22 / y=6..19）；底部色条 y=23..27 **绿色**（与 U05/U01 一致，用户 2026-11-17 拍板）。
- 头像从皮肤标准脸部 x=8..15 / y=8..15 NEAREST 放大。`/Downloads/原神_法尔伽.png` 其实是标准布局，这一区有眼睛；曾误裁 x=16..23/y=0..7 得到色块。
- U01 图标坑：旧版头像是"带下界合金头盔"版本，用户要求**纯僵尸头**；从 vanilla `minecraft:textures/entity/zombie/zombie.png` 标准脸 x=8..15/y=8..15 取。
- 图标做完后**先给用户目视确认**再编译部署，避免双端重启返工。

## 6. 构建、部署与验收

- 源码模块 `mods_src/undeadwar_hywcombat`；`JAVA_HOME=<java-runtime-beta>` + `gradlew.bat build -q`；`profileSelfCheck` 必须 PASS（新增兵种要同步把 `ProfileSelfCheck` 的 roster 数量断言 +1）。
- jar 覆盖到 `_build/hywcombat_test/mods/` 与 `_build/hywcombat_client/mods/`，双端重启（25566 / RCON 25576），客户端直连 127.0.0.1:25566。
- 测试召唤用 `undeadwartest spawn U13 Dev`；**`hywspawn u13` 无效**（HYW 原生命令不认新 id）；玩家实际用卷轴 `undeadwar_hywcombat:roster_u13_scroll`。
- 验收清单：RosterProfileId=U13、level=1、hp=70、mana base 400 / 总值 1000、move base 0.36 / 总值 0.486、armor=23、四件风甲 epic + HywIntrinsicItem、主手 wind_staff + telepathic、法术书 wind_spell_book；贴身只放 wind_jump、18–24 格只放 iron_slash、无前后振荡。
- 回归：U01/U05 等级=1、U05 move base 0.36 / poise 0.35、U01 move 0.35 均不变。
