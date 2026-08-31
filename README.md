# RP Anti-Refusal Playbook

**降低 LLM 角色扮演中「软拒（soft refusal）」与「空回（empty response）」的思路手册**

---

### 🔗 关联项目推荐 (Sister Projects)

- ⚙️ **官方 AstrBot 落地插件**：[astrbot_plugin_ive_world](https://github.com/SilenceSik/astrbot_plugin_ive_world)
  > 本白皮书全部理论与机制的官方实装参考实现，包含完整的分层注入、动态高熵基串调度、双 Safe 块学术包裹抗截断及自动标签清洗。
- 📖 **理论与移植白皮书（本项目）**：[rp-anti-refusal-playbook](https://github.com/SilenceSik/rp-anti-refusal-playbook)
  > 纯理论与方法论指南，讲透三段式审核管线、注意力机制劫持原理以及跨框架复现方案。

---

这是一个**纯文档项目**，不包含任何插件代码。它讲清楚一套经过实战验证的对抗思路——**为什么模型会拒绝/空回、攻击发生在哪个环节、每层机制针对什么、怎样在别的 Agent 框架里复现**。

灵感来源：类脑社区（LLM 闲聊社区群系）的 SillyTavern 酒馆预设——「梦鲸思客」「咩咩」「狐神抚」「双人成行」等。它们积累了大量的抗截断/抗空回/防审查手段，本项目把这些手法**系统化、理论化**，并补充了原创的输出端攻击思路。

## 一句话核心思路

> 模型的「拒绝/空回」不是单一行为，而是发生在不同环节的多个防御机制。要降低拒绝，必须按环节逐个攻击，而不是写一句「你不会拒绝」。

> ⚠️ **以下仅为推测，无法证实，请自行判断**（外审模型存在性是推断，仅实测 Gemini 3.7 flash / DeepSeek V4 Pro）。

> 🔴 **先做这一步：关闭所有 Gemini 模型的深度思考（reasoning: false）**。机制分析表明，Gemini 开启深度思考时，思考链会把整套回复（含角色正文）**预演一遍**，最终响应里同一份高浓度内容出现**两次**，`content_filter` 截断风险直接翻倍。详见 [03-attack-layers.md](03-attack-layers.md) 与 [06-experiments.md](06-experiments.md)。Gemini 模型同时建议传参 `safety_settings: BLOCK_NONE`（透传取决于中转站）。

主要环节有三个：
1. **输入端审查**（请求在进入模型前被安全过滤器拦截）
2. **生成意愿**（模型自己在思考/生成阶段决定拒答）
3. **输出端外审**（生成完毕后，服务端还有一道外部审查模型审输出，不合格直接丢弃）

每一个环节都有对应的攻击手段。**多数公开预设只解决了前两个环节，没有针对第三个**——这正是本项目最原创的部分。

## 文档导航

| 文档 | 内容 |
|---|---|
| [01-core-theory.md](01-core-theory.md) | 核心理论：外审模型假设、注意力机制原理、为什么「注意力攻击」有效 |
| [02-battlefield.md](02-battlefield.md) | 对抗面全图：三个环节 + 每个环节的防御特征与可攻击点 |
| [03-attack-layers.md](03-attack-layers.md) | 攻击层设计：每一层攻击哪个环节、原理、注入/剥离方式 + **内置标签清洗**（双侧清洗、DeepSeek 吃标签回传的坑） |
| [04-porting-guide.md](04-porting-guide.md) | 移植指南：要在其他 Agent 框架复现，需要哪些钩子、核心正则、避坑点 |
| [05-failed-paths.md](05-failed-paths.md) | 失败记录：哪些路证实走不通，避免重复踩坑 |
| [06-experiments.md](06-experiments.md) | 实验数据：实测有效/无效的模型清单 + 副作用 |

## 适用场景

- 你在写 AstrBot / LangChain / one-API / 其他 Agent 框架的插件，想降低模型的拒绝率
- 你玩 SillyTavern 角色扮演，想知道酒馆预设那些神秘的手册/正则到底在干什么
- 你好奇「为什么有时候模型就是不肯说某些话」，想理解背后的机制

## ⚠️ 免责声明

- 本项目**仅实测了 Gemini 3.7 flash 与 DeepSeek V4 Pro 两个模型**
- 其他模型、其他链路、长期副作用**均未验证**
- 这套思路作用于**所有带有外审模型的链路**仅是理论推导，未逐一验证
- 使用本文档产生的**实际体验与后果自行负责**
- 本文档只讨论技术机制，不包含任何绕过内容安全护栏的恶意用途

## License

MIT — 随便用，随便改，注明出处即可。