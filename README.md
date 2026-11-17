# HYW 兵种 Skills 集群

亡灵战争整合包（UndeadWar × Hundred Years War 0.8）的兵种工作 skills。
每个子目录是一个独立 WorkBuddy skill（含标准 `SKILL.md` frontmatter），**开箱即用**：
把本仓库的任一级子目录复制到 `~/.workbuddy/skills/`（用户级）或项目 `.workbuddy/skills/` 即可加载。

## Skills

| Skill | 用途 |
|---|---|
| [`new-hyw-unit`](new-hyw-unit/SKILL.md) | **通用建兵流程**：名册 JSON、召唤卷轴、皮肤/图标、AI 服务、构建部署、实机验收、伤害记录。新建或修复任意兵种先读这个。 |
| [`battlemage`](battlemage/SKILL.md) | **战斗法师专项**（现役 U05 炽焰战斗法师）：BC近战+铁魔法双战斗融合、四层开火门控、移速平方换算、击退抗性官方分档、寻路根因链、调参验收流程。 |
| [`pathing-debug`](pathing-debug/SKILL.md) | **寻路/索敌/追击调试**：每 tick trace 采集、五个量化指标、两条追击通道（8 节点 vs 组路径）、七个已知陷阱（NoGravity 禁跨层、孤儿 goal、纯距离攻击判定、手搓组路径退化、Y 变化判据失效…）、跨层索敌与 follow_range 的真实关系。 |

## 使用约定

- 通用流程看 `new-hyw-unit`；战斗法师类型的独有坑看 `battlemage`；任何单位出现"踏步/往复/撞墙/没仇恨"先看 `pathing-debug`。
- 调数值务必走"编译 → 双端部署 → 实机读属性 → 报告实测值"，不要以"编译成功"当验收。
- 关键坐标：隔离服 25566 / RCON 25576；服务端 `_build/hywcombat_test`，客户端 `_build/hywcombat_client`。
- 移速换算铁律：玩家属性线性、生物属性平方（`Mob.setSpeed` 把值写进输入向量），不可横向比较 base。

## 来源

从项目 `亡灵战争_整合包` 的实战经验沉淀，主要依据 0.8.0r-fix2 反编译与 2026-11 实机验证。
