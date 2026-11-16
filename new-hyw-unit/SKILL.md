---
name: new-hyw-unit
description: Add or repair a Hundred Years War army unit in the UndeadWar modpack, including roster data, summon tokens, icons, skins, AI services, build and deploy, in-game verification, and damage notes. Use when the user asks to create a unit, add a summon token, fix a unit's skin or behaviour, or change unit stats. Do not use for spell-table research alone.
---

# New HYW Unit Workflow

Follow this order. Do not report "done" until the verify step passes in-game.

## 0. Before writing code

- Re-read `design/亡灵军_兵种统计.md` and `design/人类联邦军_兵种统计.md`. They are the only source of truth for existing units.
- Write down: 代号, 名称, 档位 (低/中/高), 大类 (纯中世纪 / 魔法 / 反载), 实体 id, 生命, 装备, 技能, 标签.
- **Casters: pick a HYW native Iron's-adapted entity first** — `hundred_years_war:iron_mage` (human, `IronMageEntity`) or `hundred_years_war:hyw_skeleton_mage` / `hyw_wither_skeleton_mage` (undead). They already have a spell pool reader, ranged-only AI, and creation-mode-safe targeting. Do NOT bolt a caster onto a melee unit (`hyw_zombie` / `warrior`) with a custom Java target service.
- **HYW 的单位面板种族按实体 id 路径前缀判定**（`UnitDescriptionHelper.inferRaceId`）：`hyw_skeleton*` / `hyw_zombie*` / `hyw_wither_skeleton*` → 亡灵；其它（含 `iron_mage`）→ 雇佣兵/人类。需要「亡灵 + 自定义皮肤」时，注册一个路径以 `hyw_skeleton` 开头的自建实体（rtsbridge 里继承 `HywSkeletonMageEntity`），再套 `IronMage` 人形渲染器。
- 自建实体不在 HYW 的皮肤白名单里，`sanitizeSkinSelection()` 会把 `custom:` 皮肤清空；在子类里覆盖 `sanitizeSkinSelection()` 与 `clearCustomSkin()`。
- **单位面板的「mod 名」= 实体注册命名空间。** 想让面板显示「百年战争」，实体必须注册在 `hundred_years_war` 命名空间（U04 用 `ModEntities.HYW_ENTITIES` 的第二个 `DeferredRegister`），只改 lang 无效。

## 1. Assets

1. Skin PNG must be 64x64. Put it in all three places:
   - `modpack/config/hundredyearswar/textures/undeadwar/<name>.png`
   - `_build/server/config/hundredyearswar/textures/undeadwar/<name>.png`
   - `C:/Users/Administrator/Desktop/.minecraft/versions/1.20.1-forge-47.4.28/config/hundredyearswar/textures/undeadwar/<name>.png`
2. **Legacy icon (optional)**：只有还需要兵种卡片/旧令牌的地方才要
   `roster/<id>_token.png` + 两个模型 JSON。新兵种的召唤物品用原生卷轴，不需要再画图标。
3. **2026-11-02 起召唤物品是原生卷轴，不再用兵种图标当物品贴图**：卷轴直接引用
   `hundred_years_war:item/scroll_*` 或 rtsbridge 的卷轴模型。旧的 `roster/<id>_token.png` 只作为
   历史素材保留，新兵种不需要再画召唤令图标（除非另做 UI 卡片）。

## 2. Register the summon scroll (startup script, needs both-side restart)

- **2026-11-02 起不再注册 `undeadwar:*_spawn_token`。** 召唤物品直接用百年战争原生卷轴：
  步兵用 `hundred_years_war:scroll_tacz_mercenary`，盾兵 `scroll_shieldman`，破盾手 `scroll_warrior`，
  亡灵 `scroll_zombie`，自建法师用 `hundred_years_war:ice_mage_scroll`（`ModScrollItem`）。
- **2026-11-03 起自建实体的卷轴也注册到 `hundred_years_war` 命名空间**（`ModItems.HYW_ITEMS`，单独的 `DeferredRegister`），否则物品 tooltip 的模组名会显示 `undeadwar_rtsbridge`。对应的 item model 必须放在 `modpack/kubejs/assets/hundred_years_war/models/item/`（KJS 资源包），不要往 HYW 的只读 jar 里塞。
- 每种兵种在 `10_army_tokens.js` 里生成一个带 NBT 的卷轴堆：
  `{undeadwar_role:"<ID>", display:{Name:'{"text":"… · 卷轴"}'}}`，放进创造栏。
- `20_army_summon_tokens.js` 必须同时挂 **`ItemEvents.rightClicked` + `BlockEvents.rightClicked`**：
  右键空气走 ItemEvents（按物品 id 过滤），右键方块走 BlockEvents。**`BlockEvents.rightClicked`
  的第一个参数是方块 id 过滤器，不能传物品 id**；要么不传过滤器，在回调里检查
  `event.item.id`。只挂 ItemEvents 会让对着地面用卷轴漏拦截，HYW 会原生召出默认兵
  （没配装/没枪/没技能）。spawn 我们的配装后 `event.cancel()`。
- **一次右键会同时触发 `BlockEvents` 和 `ItemEvents`**（实测同一 tick 两条 `role scroll used`），
  同一次点击会召出两只。用「玩家 UUID + 标记」做 10 tick 去重，**重复进来的那条也必须
  `event.cancel()`**——它若不取消，事件会落到 `BaseScrollItem` 再原生召一只，仍然表现为“放两名”。
- **读 NBT 必须用 `event.item.nbt`，不要用 `getTag()`**。KubeJS 的 ItemStack 包装在运行时把
  `getTag()` 解析成 null（我们实测日志 `tag=null`），`item.nbt` 才是 KubeJS 的 NBT 属性。
- 未标记且没有 `EntityData` 的原生卷轴会在回调里被 **cancel + 提示**，避免再召出随机默认兵；
  带 `EntityData` 的回收卷轴必须放行给 HYW 原生还原。
- **卷轴图标保留自己的美术**：给带标记的卷轴堆加 `CustomModelData:<cmd>`，在资源包里覆盖
  `hundred_years_war:scroll_*` 的 item model，用 `overrides` 把 cmd 映射到旧的
  `undeadwar:item/roster/<id>_token`。cmd=0（原生/回收卷轴）仍显示 HYW 卷轴贴图。
- **回收（蹲下右键）拿到的是 HYW 原生卷轴**：`BaseCombatEntity#interact` 用 `getScrollType()` 新建
  `ItemStack` + `EntityData`。TACZ 兵种拿到 `scroll_tacz_mercenary`，显示名/图标都是原生的
  （“人类联邦军”+小人），再放置走 `restoreEntityDataFromScroll` 还原旧血量 → 看起来是残血。
  修法：在 `PlayerEvents.inventoryChanged` + 每 5/10 tick 扫背包，把“只有 `EntityData`、
  没有 `undeadwar_role`”的卷轴替换成带标记 + `CustomModelData` + 兵种名的兵种卷轴
  （`20_army_summon_tokens.js` / `56_rts_tokens.js`）。之后放置走我们的召唤路径，按设计值满血生成。
- 自建实体必须在 `getScrollType()` 里返回同款卷轴（U04 → `ModItems.ICE_MAGE_SCROLL`，
  driver/artillery → `ModItems.DRIVER_SCROLL` / `ARTILLERY_SCROLL`）。
- Startup scripts register items → **server and client must both restart** after sync, otherwise the Forge handshake fails.

## 3. Add the role (server script, reloadable)

- `modpack/kubejs/server_scripts/20_army_summon_tokens.js`, array `UW_HYW_ROLES`:
  - `id`, `role`, `name`, `faction`, `entity`, `skin: 'custom:undeadwar/<name>.png'`, `level`, `hp`, `hpBonus`
  - `tags` for faction + special behaviour
  - `gun` / `melee` / `sidearm` / `armor` (armor is a list of `item replace` strings; plain item ids are allowed when there are no enchants/affixes)
- **Caster roles add `spellbook: { item, tag }`** (Iron's spell container NBT). The summon command appends
  `HywIntrinsicSpellbookAccessory:{id:"<item>",Count:1b,tag:<tag>}` and `uwHywApplyRoleLoadout` skips `setEquipment(0)` for these roles.
  - HYW reads the book from the entity's **spellbook accessory slot**, not the off-hand: `IronMageEntity.collectSpellPool()` → `appendEquippedSpells("spellbook", getSpellbookAccessoryStack())`.
  - `setEquipment(0)` must be skipped: HYW's default `equipment.json` spawns a random spell book (fireball / chain lightning / starfall) into the same slot and merges into your pool.
  - Off-hand books still contribute (OFFHAND is scanned), but the accessory slot is the canonical/visible one.
- `hpBonus: 30` only if the unit's helmet has `apotheosis:armor/attribute/blessed`; otherwise `hpBonus: 0`.
- `/hywspawn <id>` and the token path both work automatically once the role exists.

## 4. Behaviour

- **Native mage AI handles everything**: random spell pick from the ready pool, no melee goal (`IronMageEntity` is a pure `RangedAttackMob` and not `MeleeCapableRangedAttacker`). Native attack reach is 32; for a mid-range caster, override `HywAttributes.ATTACK_REACH` in a custom attribute builder (U04 uses 30).
- **Always override `getAttackReach()` if the unit holds a spell staff.** `BaseCombatEntity#getWeaponAttackReachOverride` reads the mainhand's `attack_range` / `entity_reach` / `attack_reach` attribute; `irons_spellbooks:ice_staff` carries one and silently pins U04 to **3 blocks**. U04 returns `getAttributeValue(HywAttributes.ATTACK_REACH)` directly.
- **Casters stand still while casting**: `IronMageEntity` calls `navigation.stop()` every tick and never cancels on damage/proximity. Do NOT add a long backpedal goal (U04's `IceMageKiteGoal` caused 退—进—退 and was deleted). For native-style movement, add a priority-1 goal using `MoveControl#strafe` (survives `navigation.stop`), move at 0.75 speed while casting, and navigate toward the target when LOS is lost; U04 ships `IceMageNativeGoal`. Casting itself stays on HYW's `handleAttackLogic`, so the spell pool/cooldown/equipment remain intact.
- **Ranged attack latency**: HYW's `BaseCombatEntity` waits `getAttackDamageTickDelay()` (IronMage = 10 ticks) before calling `performRangedAttack`, and `HywNearestAttackableTargetGoal` only scans every 40 ticks when idle (+ random phase). For a caster, override `getAttackDamageTickDelay`/`getBaseAttackAnimationTime` to 2, scan every 5 ticks, and retarget the attacker on `hurt`; otherwise the first spell can be 2–4.5 s late. U04 implements all three in `IceMageEntity`.
- **KubeJS has two copies**: edit `modpack/kubejs/...` but the dev server reads `_build/server/kubejs/...`. Always run `python _build/sync_kubejs.py` and rebuild/restart; otherwise newly spawned units keep the old spellbook/role data. This was the real cause of "10 mages cast nothing" on 2026-11-02.
- Do not write a custom targeting service. HYW `BaseCombatEntity.isValidTarget` already ignores creative players (`creative_target_ignored`), treats null-owner units as hostile to players/owned units, and only targets `Enemy` types for mobs.
- Only if you need a non-Iron's behaviour, add a Java service under `gameplay/`, and remember every tick must set a real target, not just clear one.
- Keep per-unit state by UUID; use a plain counter for intervals (KubeJS `event.server.tickCount` does not exist).
- `MageSpellBridge` is now diagnostics only (`/uwspell pool` prints the accessory slot, casting state, last spell, cooldowns).

## 4.5 卷轴回收 / 放置（2026-11-02）

HYW 的回收链路：玩家**蹲下右键**一个受控单位 → `BaseCombatEntity.m_6071_` 调
`getScrollType()` 拿卷轴物品 → `BaseScrollItem.saveEntityDataToItem(stack, saveWithoutId())`；
放回时 `BaseScrollItem` 用**卷轴自己的 EntityType supplier** 新建实体，再 `entity.load(EntityData)`。
因此有三个硬约束：

1. **自建实体必须返回自己的卷轴。** `IceMageEntity.getScrollType()` 返回
   `ModItems.ICE_MAGE_SCROLL`（`ModScrollItem` 绑定 `ModEntities.ICE_MAGE`）。如果沿用 HYW 的
   `hyw_skeleton_mage` 卷轴，放回来的会是原版 `HywSkeletonMageEntity`，丢失 `getAttackReach()` /
   强制起手 / 原生走位。
2. **`load()` 会触发 `sanitizeSkinSelection()`。** 自建实体（driver / artillery / ice mage）不在
   `EntitySkinCatalog` 白名单里，`isValidSkinSelection` 返回 false → `CustomSkinId` 被清空 →
   放下来“没有皮肤”。每个自建实体都要覆盖 `sanitizeSkinSelection()` 与 `clearCustomSkin()`。
3. **普通 HYW 类型（A16 等）也会丢皮肤。** 它们的皮肤是 KJS 直接 `setCustomSkinId` 写进去的，
   卷轴恢复的 `load()` 同样会校验并清空。`20_army_summon_tokens.js` 的 `uwRepairScrollReleased`
   每 5 tick 扫一次：名字能匹配兵种且 `customSkinId` 为空 → 补回 `role.skin` + `hideArmor`。

2026-11-02 起召唤物品就是 **HYW 原生卷轴本体**（`BaseScrollItem`），不是换了贴图的令牌。
创造栏给的是带 `undeadwar_role` 标记的原生卷轴；回收得到的是同 item id、带 `EntityData` 的卷轴。
旧版 `undeadwar:*_spawn_token` 已停用，旧存档中的令牌会被移除。

## 4.6 HYW 原生弓手（2026-11-02 U02 落地）

- 载体：`hundred_years_war:hyw_skeleton_archer`（`HywSkeletonArcherEntity extends DualFormArcherEntity`）。面板 mod=百年战争、种族=亡灵；原生 `follow_range 100`、`ATTACK_REACH 100`、`RANGED_ATTACK_DAMAGE 2.5`、抛物线弹道 + 目标预判。
- 装备：远程武器不是普通主手槽，而是 `WEAPON_RANGED` 同步数据。KJS 里：
  `unit.setRangedWeapon(Item.of(bowId, bowNbt))` → `data merge entity <uuid> {CombatFormMode:1,AttackMode:0}`（RANGED）→ `unit.setMeleeWeapon(EMPTY)`。
  `CombatFormMode` 存在实体 NBT，`data merge 1` 会走 `setCombatFormMode(RANGED)` → `setUsingRanged(true)` 并把弓放进主手；只调 `setUsingRanged` 不落 NBT，重载后会回 AUTO。
- **箭矢写死原版**：`performRangedAttackInternal` 用 `new ItemStack(Items.ARROW)`，无法让它发射 Spartan 下界合金箭实体。伤害 = `HywAttributes.RANGED_ATTACK_DAMAGE × 3` + 力量附魔（`等级×0.5+0.5`）；平射（FLAT）再 ×0.33；击退/点燃读 Punch/Flame。要拉伤害就改 `hundred_years_war:ranged_attack_damage` base（KJS `attribute` 命令）。
- 只远程 = RANGED + 空近战武器；保留近战 = 保留默认铁剑/自定义近战并设 `CombatFormMode:0`。
- 卷轴：原生 `hundred_years_war:scroll_skeleton_archer`；U02 用 `CustomModelData:116` 映射到自定义 32×32 图标。

## 4.7 自定义皮肤 + 贴脸法术近战（2026-11-02 U03 落地）

- **原生骷髅法师不能用 `custom:` 皮肤**：`HywSkeletonMageRenderer#getTextureLocation` 写死 `textures/entity/skeleton/skeleton.png`。要上用户皮肤，必须自建实体 + `AbstractIronMageRenderer` 子类（模板见 rtsbridge `NecromancerEntity / NecromancerModel / NecromancerRenderer`）：
  - `getTextureLocation` → `EntitySkinTextureManager.resolveTexture(entity.getCustomSkinId(), entity)`，失败回退 jar 内 `textures/entity/<name>.png`；
  - `ModClient` 注册新的 `ModelLayerLocation` + `HywHumanoidModel::createBodyLayer`；
  - 回收卷轴必须是绑定自建实体的 `ModScrollItem`，否则放回来是原版骷髅法师。
- **贴脸法术近战**：HYW 的 `getAttackReach()` **同时是施法门控和吟唱取消判定**——压到 3 格会让所有目标法术必须贴脸、目标一走远就取消吟唱。正确做法：`ATTACK_REACH = 24`（正常施法距离），贴脸另加走位 goal（`NecromancerApproachGoal`：>3.2 格追击、≤2.8 格站桩）。`getAttackReach()` 仍要重写直接读 `HywAttributes.ATTACK_REACH`，防血杖的 `attack_range` 覆盖；`tick/hurt` 复制 U04 的 5 tick 扫描 + 强制起手 + 受击立刻还手。不挥主手武器，近战手段全是法术。
- **彼岸之花 NPC 无画面/无伤害**：`apprenticecodex:higanbana` 是 `CastType.INSTANT` 的召唤武器法术，`AbstractSummonWeaponSpell#onCast` 只对 CONTINUOUS 或玩家 `castInstantWeapon` 走造刀；NPC 走 `onCast` 时不会调 `onCastNoWeapon`。修法：在 HYW 起手后（`tryStartExternalCompatAttack` 的 super 之后）反射读 `IronMageEntity.activeSpellCast`，若 spellId 是 higanbana，反射调 `onCastNoWeapon` + `onInitialCastWithWeapon`（后者设 damage/4 连斩/首发待机）。反射获取法术实例：`SpellRegistry.REGISTRY` → `Supplier.get()` → `IForgeRegistry.getValue(ResourceLocation)`。
  - **距离门控**：刀只覆盖 3.4 格；目标 >4 格时不要造刀，反射调 `IronMageEntity.cancelCurrentCast("...")` 取消这次施法，并通过 `MagicData.getPlayerCooldowns().removeCooldown(spellId)` 清掉误加的冷却，`setAttackCoolDown(10)` 让它重选别的法术。否则会对着远程目标空放。
- **法力（2026-11-03 起为有限值）**：U03/U04 走 `ManaCasterSupport`——基础 `mana: 300` + 装备加成（U04 冰霜术士 2 件 + 冰杖 = 550）；每 tick 回蓝，战斗中 `1%/s`、脱战 `3%/s`（停火 5 秒后）；蓝 <20% 时拒放单次消耗 >50 的法术；蓝不够则 `cancelCast` + `clearCooldown` + 10 tick 重选。KJS 不再每 5 tick 顶满，也不再设 `mana_regen 0`。
- **猩红教徒 GeckoLib 护甲没有 `cultist_layer_1/2.png`**：`hideArmor: true`，渲染器在渲染期间临时清空 HEAD/CHEST/LEGS/FEET（U04 同款），护甲属性仍生效。
- **驱役亡灵**：`getSummonCount(level) = level + 2`；`getRecastCount() = 2` 但 recast 走 `PlayerRecasts`，**只对玩家**；NPC 只放首轮随从。Lv2 = 4 只。
- **法术书**：`irons_spellbooks:necronomicon_spell_book` + `spell_container{maxSpells:8,...}`；池子里的法术等级按 `level` 字段生效。
- **法师挥刀通用 goal（2026-11-03）**：`MageMeleeStrikeGoal` 由 U06 的 `AssassinStrikeGoal` 泛化而来，U05/U06 共用；优先级 2、3.2 格、20 tick、`!isIronSpellCasting()` 才挥刀，不占 HYW `attackCoolDown`。U05 注册后主手长剑才真正参与平 A（否则剑只加近战法术伤害）。
- **起手射程门控（2026-11-03）**：`AbstractUndeadMageEntity.SPELL_CAST_RANGE` 按法术给有效射程（引火 3 / 炽焰斩击 4 / 彼岸 4 / 地狱 8.8 / 燃烧喷流 10 / 爆裂炽焰箭 30 / 震荡波 9 / 暗影斩 12 / 血步 24 / 连锁 25 / 雪球 25 / 冰锥·射线·血斩·闪电球 30）。起手后 `canStartSpell` 判定，超距则 `cancelCast + setCooldown(spell,20)（不 clearCooldown）+ setAttackCoolDown(40)`，**不扣蓝**；自身 buff/召唤类不在表内 = 不限距离。只判起手，**不判吟唱途中**（途中离场仍交给 HYW 原生 ATTACK_REACH / `shouldAIStopCasting`）。
  - U04 是独立类（不继承基类），在 `IceMageEntity#tryStartExternalCompatAttack` 里调 `isSpellCastTargetInRange`。
  - **防空放刷屏（2026-11-03 二修）**：射程门控是在 HYW 起手之后才取消，起手动画/音效已经播出去 → 远距离会反复“隔空挥刀 + 音效”。照暗影刺客的低蓝预判思路补两层：① `castStartRange()` 起手前距离阈值（U05 10 / U03 15 / 默认 24），超出直接不进入施法、交给走位 goal 靠近；② 取消后 `setAttackCoolDown(40)`，并给被拒法术 `ManaCasterSupport.setCooldown(spell, 20)`（不立刻清冷却），避免下一轮又随机选中同一个短手法术。
- **法术等级按稀有度门槛选（2026-11-03 规则）**：先查《铁魔法_法术总表》的「稀有度门槛」，取**最低的史诗等级**，不要凭感觉给低等级。U05 池：炽焰斩击 Lv4（史4）/ 引火 Lv6（史6）/ 燃烧喷流 Lv4（史4）/ 爆裂炽焰箭 Lv5（史5）/ 橡肤 Lv7（史7）。
- **U05 炽焰战斗法师（2026-11-03 改版，已升高阶）**：等级 70、70 血、四件炽焰术士（法力 800、spell 1.20 / fire 1.40 → pm 6.72、护甲 20、保护 IV×4）；近战 goal；5 法术池；爆裂炽焰箭 Lv5（100.8 直伤 + 50.4 爆炸）、橡肤 Lv7 给 75% 减伤（约 255s / CD 90s）。地狱浮现（`raise_hell`）与热能斩击（`thermal_slice`，召唤武器 NPC 适配差）已取消。
- **召唤武器类法术（thermal_slice / higanbana 等）在 HYW NPC 上是空放**：都继承 `AbstractSummonWeaponSpell`，NPC 走 `onCast` 时不会调 `onCastNoWeapon`。**选池时优先避开**；确实要用就照 `NecromancerEntity#spawnHiganbanaWeapon` 反射补 `onCastNoWeapon + onInitialCastWithWeapon`（U05 曾用 thermal_slice 落地过，2026-11-03 已换成 fire_arrow）。

## 4.8 人类阵营命名与编制（2026-11-03）

- 人类阵营统一称 **MDS**（原「联邦军 / A 型」前缀全部退场）。MDS 在设计上是**援军**：以通用部队和特种部队为主，也编有精英；精英一般是各国前来支援主战场的部队（例：奥德利斯精英机枪手）。
- 命名规则：
  - 通用 / 特种：`MDS <职能>` —— MDS 侦察手、MDS 突破手、MDS 医务兵、MDS 狙击手、MDS 盾兵……
  - 指挥官保留专名：`MDS 指挥官 赤霄`
  - **E01 例外**：`东大陆联邦精英制式步兵`，不挂 MDS 前缀
  - 各国援军精英：`<国名>精英<兵种>`，如 `奥德利斯精英机枪手`；仍属人类阵营，不是独立阵营
- 剧情基线：东联邦大陆海战，东大陆（东联邦）士兵数量更多；MDS 是外来援军，精英比例更高。
- 实现：`faction` 不变；只改 `role.name` / `role.role` / 卷轴显示名 / 创造栏 / lang。**不要改 `undeadwar_role_*` 标签**——那是逻辑 id，改名会影响技能脚本与存档。

## 4.9 TACZ 机枪长连射（2026-11-03）

- HYW 原生 TACZ 兵种默认是**短连射**：`BaseTaczCombatEntity` 每次随机 `2–6` 发（`getMinBurstShots/getMaxBurstShots`），射完随机停顿 `5–30 tick`（`getMinBurstPauseTicks/getMaxBurstPauseTicks`）；`sampleBurstShots()/sampleBurstPauseTicks()` 是 private，不能直接改字段。
- 机枪手要「火力倾泻」，自建实体继承 `TaczMercenaryEntity`（→ `AbstractTaczHumanoidEntity` → `BaseTaczCombatEntity`），覆盖四个 protected 工厂方法：
  ```java
  @Override protected int getMinBurstShots() { return 40; }
  @Override protected int getMaxBurstShots() { return 75; }
  @Override protected int getMinBurstPauseTicks() { return 1; }
  @Override protected int getMaxBurstPauseTicks() { return 3; }
  ```
  发数按弹匣容量设（75 发 → 一次压制打空弹匣）；停顿压到 1–3 tick，只留枪械自身冷却与换弹。
- 也可以完全跳过 burst：覆盖 `tryStartExternalCompatAttack`，直接调 `TaczIntegration.performNpcGunAttack`，只按 `result.cooldownTicks()` 设冷却，不累计 `burstShotsRemaining`。
- `getUnitSpreadMultiplier()` 默认 15（NPC 散布倍率）；机枪可保持，或降到 10–12（压制允许散布）。
- **泛型坑**：`TaczMercenaryRenderer` / `TaczMercenaryModel` 的泛型固定为 `TaczMercenaryEntity`，直接写 `HywHumanoidRenderer<MgGunnerEntity, TaczMercenaryModel>` 会编译失败（`M` 必须是 `HywHumanoidModel<T>`）。正确做法：新增 `MgGunnerModel extends HandgonneManModel<MgGunnerEntity>`（覆盖 `shouldPreserveBasePose`），再写 `MgGunnerRenderer extends HywHumanoidRenderer<MgGunnerEntity, MgGunnerModel>`；模型层注册仍复用 `TaczMercenaryModel.LAYER_LOCATION`，不需要新 layer。
- 实体注册到 `hundred_years_war` 命名空间；人类实体 id **不要**带 `hyw_skeleton` 前缀。
- **自建 TACZ 实体必须覆盖 `getScrollType()` 返回自己的 `ModScrollItem`**，否则回收/放置会还原成原生 `TaczMercenaryEntity`，长连射与配装全部丢失。
- **MG3 的真实 id 是 `daffas_arsenal:migraine3`**（`daffas` 枪包的 lang 显示名 "MG3"；数据在 `data/daffas_arsenal/data/guns/migraine3_data.json`，100 发、900 RPM、单发 12）。不要写 `tacz:mg3`——那个 id 不存在。
- **E02 实测（2026-11-03）**：`/hywspawn e02` → 70 血、自定义皮肤、MG3 `GunCurrentAmmoCount 100`、四件 epic；对 2000 血 NoAI 僵尸 **3 秒打 30 发、196.4 伤害**，确认长连射生效。裸 `/summon` 没有 KJS 配装时主手会走 fallback（`glock_17`），这是父类 `createFallbackTaczMainhand()` 的默认行为，不是 MG3 失效。

## 4.10 卷轴图标规格（2026-11-03）

- **画布 32×32 PNG（RGBA）**，风格以突破手 `roster/a02_token.png` / 侦察手 `a18_token.png` 为模板。
- 固定结构（逐像素布局，不要自由发挥）：
  - 外层 **2px 兵种类别色边框**（整圈 32×32 的 x/y 0–1 与 30–31）；
  - 内部背景 `(18,24,32,255)`；
  - 头像白框：1px 半透明白 `(255,255,255,60)`，矩形 **x=8..23 / y=5..20**；
  - 头像区 **14×14**（x=9..22, y=6..19）：取皮肤头部正面 `(8,8)-(16,16)` 8×8，用 `Image.NEAREST` 放大到 14×14；
  - 底部**只有一条色条**：y=23..27，x=9..22（y=27 收边为 x=8..23）。**不要再画头像框内的大色块**——驾驶员/炮兵现有图标不符合这条规范，不能当模板。
- 兵种类别色（必须统一）：
  - 突击 `#C83737`、侦察 `#6EB9F0`、医疗 `#7ED98C`、指挥 `#EBC23C`、工程 `#E18C32`
- 生成方式：拿 `a02_token.png` 当底，把红色 `(200,55,55)` 全部替换成目标类别色，再把头像贴到 `(9,6)`；参考脚本 `_build/_gen_e02_token.py`。
- 模型：`assets/undeadwar/models/item/roster/<id>_token.json`（`layer0: undeadwar:item/roster/<id>_token`）；卷轴模型放在 `assets/hundred_years_war/models/item/<scroll>.json`，用 `custom_model_data` → 该图标模型。
- 图标要同时写三处：`modpack/kubejs/assets/...`、`_build/server/kubejs/assets/...`、客户端 `.minecraft/.../kubejs/assets/...`；客户端按 **F3+T** 重载资源包即可看到，不用重启。

## 4.11 玩家默认血量 200 与血量 HUD（2026-11-03）

- **玩家血量 200 HP**：`kubejs/server_scripts/30_player_health.js` 在 `PlayerEvents.loggedIn` / `respawned` 把 `minecraft:generic.max_health` 基础值钉到 200，并 `setHealth(200)`。改完必须 `sync_kubejs.py` + 重启（脚本是登录事件，`reload` 不够）。
- **血量显示用 Simple Health Bar（客户端 only）**：`SimpleHealthBar-1.0.2-forge-1.20.1.jar`（Modrinth，`server_side: unsupported`，只需放客户端 `mods/`）。它把原版心形替换成一条血条，支持自定义最大生命/吸收/毒/凋零/冰冻颜色，200 血不会画 100 颗心。
  - **不要再用 CustomHealthHUD**（2026-11-03 试装后回退）：它按 `maxHealth/2` 逐颗画心，200 血 = 100 颗心排成一长排 650px+，视觉很差（用户实测截图确认）。
  - 发布整合包时 `SimpleHealthBar-*.jar` 也要进客户端 `mods/`；服务端不需要。
- 测试：`attribute Dev minecraft:generic.max_health get` 应为 200；`data get entity Dev Health` 200。

## 4.12 亡灵军卷轴命名与技能描述（2026-11-03）

- 卷轴显示名来自 `item.hundred_years_war.<scroll_id>` 的 lang（`modpack/kubejs/assets/hundred_years_war/lang/zh_cn.json`）。**必须写「兵种全名（含阶位）· 卷轴」**，不能再用统一的「兵种召唤卷轴」（否则 U04 这类卷轴看起来"什么都没写"）。
  - `scroll_zombie` 亡灵军低阶战士 / `scroll_skeleton_archer` 亡灵军中阶弓箭手 / `necromancer_scroll` 亡灵军中阶亡灵法师 / `ice_mage_scroll` 亡灵军中阶冰霜法师 / `pyromancer_scroll` 亡灵军高阶炽焰战斗法师 / `assassin_scroll` 亡灵军高阶暗影刺客 / `stormcaller_scroll` 亡灵军中阶雷霆术士。
- 卷轴与单位面板的技能描述来自 `unit.hundred_years_war.skill.<entity_id>`（`modpack/kubejs/assets/undeadwar/lang/zh_cn.json`）。**7 个亡灵单位都要有**（`hyw_zombie` / `hyw_skeleton_archer` / `hyw_skeleton_ice_mage` / `hyw_skeleton_necromancer` / `hyw_skeleton_pyromancer` / `hyw_skeleton_assassin` / `hyw_skeleton_stormcaller`），内容必须跟当前装备/法术池一致，改池子时同步改技能描述。

## 4.13 铁魔法 + TACZ 混合兵种（2026-11-04 A22 落地）

- **载体**：继承 `HywSkeletonMageEntity`（复用 `AbstractUndeadMageEntity` 的法术池/法力/门控/support 白名单），但实体 id **不能**带 `hyw_skeleton` 前缀（否则 HYW 面板判亡灵）；注册 `hyw_holy_medic` 并覆写 `getMobType()` 返回 `MobType.UNDEFINED`，人类阵营关系才正常。
- **开枪**：独立 `TaczGunAttackGoal`（优先级 2）直接调 `TaczIntegration.performNpcGunAttack(shooter, target, spreadMultiplier, forceDraw)`，按返回 `AttackResult.cooldownTicks()` 设自己的冷却，**不占 HYW `attackCoolDown`**；首次 `forceDraw=true`。`isIronSpellCasting()` 时不开枪。
- **枪械伤害加成**：在 `LivingHurtEvent` 里按 attacker tag（如 `undeadwar_role_a22`）乘/加；TACZ 每发两次事件，按 bullet UUID + victim UUID 2 tick 去重。+500% = ×6。
- **TACZ 1.1.8 没有 AMMO_MOD 槽**（`AttachmentType` 只有 SCOPE/MUZZLE/STOCK/GRIP/LASER/EXTENDED_MAG/NONE）。FMJ 的 index `type=extended_mag`，所以 `AttachmentEXTENDED_MAG` 只能装 FMJ **或**扩容弹匣，不能同时装。
- **支援法术**：`fortify`/`healing_circle`/`blessing_of_life`/`heal` 都在 HYW support 白名单，`updateIdleSupportTarget` 会自动施放，不经过攻击距离门控；`divine_smite` 是攻击法术，需要给 `SPELL_CAST_RANGE` 加 4 格并在 `castStartRange` 里贴脸。
- **属性注册**：新增实体必须在 `UndeadWarRtsBridge#attributes` 里 `event.put(...)`，否则启动报 `has no attributes`。
- **皮肤**：`custom:undeadwar/undead_holy_medic.png` 放 `config/hundredyearswar/textures/undeadwar/`（客户端 + 服务端）+ jar 内置兜底；图标按 4.10 规范用医疗绿 `#7ED98C`。

## 5. Build and deploy

1. Build in `mods_src/undeadwar_rtsbridge` (that is the live project; `libs/` must contain `HundredYearsWar-*.jar` and `superbwarfare-*.jar` for `compileOnly`):
   ```powershell
   $env:JAVA_HOME="C:\Users\Administrator\AppData\Roaming\.minecraft\runtime\java-runtime-delta"
   .\gradlew.bat build -q
   ```
   Java 17 toolchain is required; the wrapper alone fails auto-detection.
2. Sync KubeJS: `python _build/sync_kubejs.py`.
3. Stop the server, copy `build/libs/undeadwar_rtsbridge-0.1.0.jar` to `_build/server/mods/` and to the client mods folder.
4. Start server: `python _build/run_server.py` (background). Wait for RCON `list` to answer.
5. Start client: `python _build/launch_dev.py --username Dev --quickplay-server 127.0.0.1:25565 --xms 3G --xmx 6G --width 1600 --height 800`.
   The client must be relaunched after the server, otherwise quick-play connects to a dead server.

## 6. Verify (required, in-game)

Use RCON. `@p` only resolves when a player is online; run commands from a player source with `execute as Dev run ...` (also required by `/hywspawn`).

- `execute as Dev run hywspawn u<id>` then `execute at Dev run uwspell pool @e[type=<entity>,limit=1,sort=nearest]`
  - must list every designed spell with slot `spellbook/SPELLBOOK`, plus `饰品槽=<book>`.
- Fight test: summon a durable `Enemy` (e.g. `minecraft:zombie` with `Attributes:[{Name:"minecraft:generic.max_health",Base:400.0}]`) 6 blocks away, then poll `/uwspell pool` — you must see `施法中 <spell>` and cooldown entries.
  - Iron golems are NOT valid targets (not `Enemy`, no owner relation).
  - A mage standing in terrain / fallen down a cliff won't cast; teleport it to open ground first.
- `data get entity <sel> Health` — matches design.
- `data get entity <sel> CustomSkinId` — `custom:undeadwar/<name>.png`; look at it in game.
- Melee check: the mage must never swing; its pool of `BaseCombatEntityAttackGoal` is ranged-only.
- Drop check: kill one, no equipment drops. `SoldierDropService` 只清 `LivingDropsEvent`，
  HYW 的内建装备走 `spawnAtLocation`，要靠 `26_e01_elite.js` 在 `EntityEvents.death` 里先清装备。

## 7. Docs

Update `design/亡灵军_兵种统计.md` (or the human doc):
- 现役总表 row, per-unit profile, 变更记录.
- If numbers matter, add to the damage/DPS section with the measurement source (probe / tooltip / theory).
- Update `交接手册.md` only for traps/architecture changes.

## Traps learned the hard way

- `event.server.tickCount` does not exist in KubeJS → use a module counter.
- Scroll release calls HYW `setEquipment()` and wipes `blessed`; HP silently drops (85→55). Verify HP after any scroll work.
- **HYW 内建装备的 8.5% 掉落绕开 `LivingDropsEvent`**：装备带 `HywIntrinsicItem`，正常死亡掉落会跳过可见槽，
  改由 `dropHywIntrinsicEquipment` 在玩家击杀时每格 8.5% 走 `spawnAtLocation` 直接生成 ItemEntity，
  `SoldierDropService` 看不到。Forge 的 `LivingDeathEvent` 在 `dropAllDeathLoot` **之前**触发
  （已核对补丁后的 `LivingEntity#die` 字节码），所以在 `EntityEvents.death` 里清空可见槽 + 私有的
  `intrinsicEquipment` / `suspendedExternalEquipment` 两张表（`26_e01_elite.js`）。
- Iron's spell config lives in `_build/server/config/irons_spellbooks_spell_config/**/*.json`; `power_multiplier` is the damage knob and needs a server restart.
- **伤害必须以玩家路径实测或反编译公式为准**。`/uwspell mobcast` 只适合验证“起手 / 冷却 / 法术池”，它的伤害结算不完整（higanbana 只结算部分斩击、冰锥/射线投射物不命中、血斩被压缩），不能拿来做数值依据。玩家实测锚点最可靠：猩红斩击 Lv4 在 7.0 倍率下 = 91。
- **Do not put the spell book in the off-hand and call it done**: HYW mages read `HywIntrinsicSpellbookAccessory`, and a random default book otherwise joins the pool.
- **Do not run `setEquipment(0)` on a mage role** — it overwrites the accessory with `equipment.json`'s random book.
- HYW `/hywspawn` needs a player command source; console-only RCON must use `execute as <player> run hywspawn <id>`.
- Iron's spells do not check mana for mob casts (`MagicData.initiateCast` has no mana gate); `max_mana` is cosmetic for NPCs.
- `apprenticecodex:frost_rune` IS castable by the native mage (it finds a ground spot in `checkPreCastConditions`); it is a 20-tick LONG trap, not an instant.
- HYW logs are compiled out (`HywLogger.ENABLE_IRON_MAGE_DEBUG = false`), so `/uwspell pool` is the only runtime observability.
- Iron's armor has no vanilla `_layer_1/_layer_2` textures (GeckoLib-only). Mobs wearing it trigger `Failed to load texture: minecraft:textures/models/armor/<set>_layer_1.png`; give caster roles `hideArmor: true` and let the custom skin be the visual.
- Entity ids are namespaced by the registering mod: rtsbridge entities are `undeadwar_rtsbridge:*`, not `undeadwar:*` (`summon` returns 0 otherwise).
- **Weapon reach can silently override `ATTACK_REACH`**: `BaseCombatEntity#getWeaponAttackReachOverride` treats any mainhand item attribute whose path is `attack_range` / `entity_reach` / `attack_reach` as the weapon's reach. Iron's `ice_staff` carries one, so a U04 with 30 reach only casts inside **3 blocks** — it looks like "attacks melee, ignores gunners/medics". Override `getAttackReach()` in the unit class to read `HywAttributes.ATTACK_REACH` directly, or keep the staff out of the mainhand.
- **`/hyw_ai disabled true` freezes all HYW combat**: `BaseCombatEntity.aiStep` returns before `handleAttackLogic`, so units keep targeting/cooldowns but never attack. TACZ guns still fire (separate AI), which makes it look like only the mage is broken. Check `/uwspell pool`'s `AI=` line; re-enable with `/hyw_ai disabled false`.
- **Iron's `power_multiplier` 实际生效值：2026-11-03 起为 4.0（同日由 5.0 下调，开发服原 7.0）。** 逐法术 `config/irons_spellbooks_spell_config/<namespace>/<spell>.json` **没有参与加载**（启动日志只有 `Loaded 1 global spell config parameters`），真正生效的只有 `global_config.json`；别再看 per-spell 的 1.0 做计算。它同时乘 `getEntityPowerMultiplier`（半径、投射物数量、控制时长、冻结量）。要改必须三处同步：`_build/server/config/...`、`modpack/config/...`、客户端 `.minecraft/.../config/...`，然后重启服务端。改全局会同时影响 U03/U04/U05 与玩家法术。
- HYW's NPC casting **does not consume mana** (`startMobSpellCast` never deducts `MagicData#mana`); a mob's mana pool is cosmetic. Output is gated by cooldown + 20-tick attack interval + cast time only.
- **Mod name in the HYW unit panel is the entity's registry namespace.** A rtsbridge entity always shows our addon, not HYW (`hundred_years_war`). U04 solves this by registering through a second `DeferredRegister.create(ForgeRegistries.ENTITY_TYPES, "hundred_years_war")` in `ModEntities.HYW_ENTITIES`; the entity class stays ours, the id becomes `hundred_years_war:hyw_skeleton_ice_mage`. Changing only lang does not fix the namespace line.
- **Custom entity + custom skin**: `EntitySkinCatalog.getDefaultTextureRelativePath` returns `""` for unknown entity types → the renderer falls back to `hundred_years_war:textures/entity/` (missing) and shows the purple/black checkerboard. Override `getTextureLocation` and call `EntitySkinTextureManager.resolveTexture(entity.getCustomSkinId(), entity)`; always keep a bundled fallback PNG under `assets/<modid>/textures/entity/`.
- **hideArmor does not stop HYW's renderer**: `HywHumanoidRenderer.render` temporarily attaches `HumanoidArmorLayer` regardless. Iron's GeckoLib armor has no vanilla `cryomancer_layer_1.png`, so the armor still renders as purple/black. In the custom renderer, temporarily `setItemSlot(HEAD/CHEST/LEGS/FEET, EMPTY)` around `super.render`, then restore.
- **Want Iron's armor visible? ship vanilla layer textures.** The layer asks for `minecraft:textures/models/armor/cryomancer_layer_1.png` (and `_2` for leggings). We bundle icy-blue recolors of vanilla `iron_layer_1/2.png` under `assets/minecraft/textures/models/armor/` in the rtsbridge jar (also copied into `UndeadWar_zh_cn`); set `hideArmor: false` and the cryomancer helmet/chestplate render as flat armor over the custom skin. They are not the GeckoLib 3D model, but they are visible.
  - 2026-11-03: same trick for `cultist_*` (U03 猩红教徒套) — `cultist_layer_1/2.png`, generator `_build/_gen_cultist_armor_layer.js` (reads vanilla `iron_layer_1/2.png` out of `versions/1.20.1/1.20.1.jar`, maps the 7 grey levels onto a crimson ramp, keeps 64×32 + alpha 230). Copy to three places: rtsbridge resources, `modpack/resourcepacks/UndeadWar_zh_cn/...`, client resourcepack.
- **Codex 召唤武器类法术在 HYW NPC 上是空放**：`artisan_smash` / `thermal_slice` / `thermal_process`（以及 `higanbana`）都继承 `AbstractSummonWeaponSpell`。HYW 起手时 `ensureDefaultAdditionalCastData` 会塞一个空的 `SummonWeaponSpellCastData`，于是 `onServerPreCast`/`onCast` 里的 `!(castData instanceof SummonWeaponSpellCastData)` 判断全部失效 → 不造武器、无伤害。要复用就照 `NecromancerEntity#spawnHiganbanaWeapon` 反射补 `onCastNoWeapon`。U05 的 `thermal_slice` 已按同款落地：`PyromancerEntity#spawnThermalSliceWeapon`（2026-11-03）。
- **recast 类法术对 NPC 只有第一段**：`wall_of_fire` 只在 `onRecastFinished` 生成墙体（NPC 永远是空放）、`flaming_barrage` 只会出 1 颗球。HYW 的 `onCast` 只调一次，没人驱动 `PlayerRecasts`。
- 火焰系数值与 NPC 可用性见 `design/炽焰法术总表.md`；冰霜见 `design/冰霜法术总表.md`。
- **`StaffItem` 的属性只挂在 MAINHAND**（`getDefaultAttributeModifiers(slot)` 只对 MAINHAND 返回 `defaultModifiers`）。**法杖放副手不加任何法术强度**，所以"主手剑 + 副手法杖"的爆发法师是无效配装（U05 踩过）。要么主手拿杖，要么用护甲件补 `spell_power` / `<school>_spell_power`。
- **亡灵法术兵种统一继承 `AbstractUndeadMageEntity`**（2026-11-03 起）：走位 `MageApproachGoal`、法力 `ManaCasterSupport`、5 tick 索敌、受击还手都在基类；子类只写 `createXxxAttributes()`（调 `mageAttributes(reach, followRange)`）、`getScrollType()`，需要时覆盖 `canStartSpell` / `onSpellStarted` / `onCastFinished` 三个钩子（U03 的彼岸之花就是这么挂的）。
- **骷髅音效**：HYW 的 `HywSkeletonMageEntity` 把 `getAmbientSound/getHurtSound/getDeathSound` 写死成 `minecraft:entity.skeleton.*`，自建法师会一直叫。屏蔽点在 `Entity#playSound(SoundEvent,float,float)`（所有音效都走它），用 `MageSoundFilter` 只拦 `entity.skeleton` 前缀，别把整个 `playSound` 清空（会把铁魔法的施法音一起干掉）。
- **`/kubejs reload server_scripts` 救不了 role 表**：它只重跑脚本、重新注册事件，但 `/hywspawn` 是 Brigadier 命令，命令对象在首次注册时就绑定了旧的脚本上下文，重载后仍读旧 `UW_HYW_ROLE_BY_ID`（表现：改了 armor/装备，`hywspawn` 出来的还是旧配装）。**改 `UW_HYW_ROLES` 必须整服重启**（卷轴右键路径会走新事件，但别只测命令就以为改好了）。
- **新增兵种卷轴要同时登记三处**：① `ModItems`（物品注册）② `10_army_tokens.js` 的创造栏条目 ③ `20_army_summon_tokens.js` 里的 **`UW_ROLE_SCROLL_IDS`（右键拦截白名单）** 与 **`UW_ROLE_SCROLL_CMD`（回收卷轴归一化）**。
  漏掉 `UW_ROLE_SCROLL_IDS` 的表现是：右键卷轴**放出来一个没有任何装备和法术池的人** —— 事件没被拦截，落到 HYW `BaseScrollItem` 原生调用 `supplier` 生成裸实体。U05 的 `pyromancer_scroll` 就是这么漏的（2026-11-03 修复）。
- **法力不足时别让 NPC 起手**：HYW 的 `startMobSpellCast` **先**播起手动画/音效，法术的 mana 检查在之后才取消 —— 于是低蓝时会疯狂刷"起手→取消"，看起来像多轮挥刀 + 音效连播但法术永远不落地（U06 实测）。修法：在 `tryStartExternalCompatAttack` 调 `super` **之前**先判法力，不够就直接放弃本次施法（只留平 A）。
- **HYW 的格挡是 360° 的，而且必须 mixin 才修得动**：`applyDefensiveResponse` 不判断方向；更坑的是 HYW 的 `hurt` **先格挡再 `super.hurt`**，所以 `LivingAttackEvent` 永远晚一拍（挡完了才轮到）—— 事件方案实测无效。正确挂点是 **mixin 注入 `BaseCombatEntity#hurt` 的 HEAD**（`BaseCombatEntityGuardMixin`，`MixinConfigs` 写进 jar manifest，`remap=false` + 方法名用 SRG `m_6469_`），在那里临时拆掉格挡（清 `DEFENSIVE_BLOCK_TICKS` 姿态 + `stopUsingItem` 收盾 + `block_chance` 归零，下一 tick 还原）。方向判定用 **`getYRot()`**（与血步的落点同源），不要用 `getLookAngle()`（头朝向会错位）。已落地：`ShieldGuardService`（A21），正面 ±75° 保持强防。
- **法师要挥刀：不要实现 `MeleeCapableRangedAttacker`**。HYW 的攻击循环确实支持它（`isUsingRanged=false` 时走 `doHurtTarget`），但 `ATTACK_REACH` 同时是**施法门控**，24 格会让攻击循环在 24 格外就起手、挥空并耗光冷却，站位 goal 也会打架。可行方案（U06 已落地，U05 复用）：保留法师的施法链路，另加一个**独立挥刀 goal** `MageMeleeStrikeGoal`（优先级 2、不占 MOVE/LOOK）：距离 ≤3.2 且 `!isIronSpellCasting()` 时 `swing(MAIN_HAND,true)` + `doHurtTarget(target)`，用 goal 自己的冷却，不碰 HYW 的 `attackCoolDown` —— 平 A 与法术就能真正交替。类名由旧的 `AssassinStrikeGoal` 泛化而来。
- **`uwHywApplyArmor` 支持稀疏槽位**：`armor: [{slot:'chest', item:'...'}, ...]`，可以只穿某几个护甲位（U05 不戴头盔以露出皮肤）。原来的四元素字符串数组照旧可用。
- **HP formula**: role HP is `base = (hp - hpBonus) / 5` plus a `multiply_total 4.0` modifier with UUID `f081c99d-ff76-4f48-a5a0-3d5d54a0c63f`, then `Health = hp`. A bare `/summon` (no role loadout) shows `base 30 × 5 = 150`, which is not the role HP; always verify with `/hywspawn`.
  - `blessed` 的加成看品质：**rare = +6 基础（+30 最终，hpBonus 30）**；**epic = +8 基础（+40 最终，hpBonus 40）**。U02 用 epic，所以 hpBonus 是 40；写错会得到 90 血。
