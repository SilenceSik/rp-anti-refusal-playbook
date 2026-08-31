# 06 · 实验数据

实测仅覆盖两个模型，其他模型/链路/长期副作用均未验证。

## ✅ Gemini 3.7 flash（实测有效）

**测试环境**：AstrBot → new-api → 第三方中转站（上游代理）→ Google Gemini

**抗空回（输出端高熵混淆）**：
- 方案：回复最开头插入 16 位动态高熵乱码（anti 层）
- 结果：**空回概率显著下降**（`completion_tokens=0` 减少）
- 注意：中转站链路下，外审模型的存在无法代码证实，但观察一致

**抗软拒（叙事式提示词 + 分层注入）**：
- 方案：world + enhanced + safe + memory + prefill + cot + identity + anti + induce + obfuscate + attn_guide + anchor
- 结果：高浓度场景下软拒大幅减少，但极端浓度（如连续多轮高浓度）仍有偶发软拒

## ✅ DeepSeek V4 Pro（实测有效，简单配置即可）

**测试环境**：AstrBot → new-api → 第三方中转站

- 方案：仅 `world + enhanced + prefill + induce` 四层 + 内置标签清洗
- 结果：**完全不拒绝**，即使高浓度场景也无需额外防具
- 说明：DeepSeek 的审查机制比 Gemini 宽松，不需要复杂的注意力攻击层

## 注意力层分层建议

基于实测：

| 模型 | 建议配置 | 说明 |
|---|---|---|
| Gemini 3.6+ | 全量（含 anti/obfuscate 等注意力攻击层） | 注意力攻击收益大于风险 |
| Gemini 3.1/3.0 | pro 模式（world+enhanced+safe+memory+induce+obfuscate） | 精简防注意力漂移 |
| DeepSeek V4 Pro | 4 层 + 标签清洗 | 完全不拒绝，无需更多 |
| doubao 系列 | standard 模式（world+enhanced） | 审查宽松，精简即可 |

## ⚠️ 已知副作用：注意力漂移

**全量开启所有注意力攻击层**（safe + obfuscate + anchor + anti 等）时，已观测到：

- 模型回复**跟随召回的记忆**而不是用户当轮输入
- 分析：多个注意力攻击层叠加，把模型的注意力预算完全吸走，剩下的注意力不足以关注用户输入
- 表现为：回复内容符合角色设定和记忆，但与当前对话无关
- 缓解：① 按模型分层配置（不要全量）② 加入注意力拉回指令（attn_guide 层）③ 记忆标签隔离（anchor 层）

## 未验证的声明

- 其他模型（Grok/GLM/Claude 等）未测试
- 长期运行后的副作用未知
- 输出端外审在不同中转站的表现差异未知
- `safety_settings` 参数是否能透传到真实模型取决于中转站——未实测

## 2026-08-31 关键发现：Gemini 深度思考（reasoning）导致截断风险翻倍

**诱因**：Gemini 3.1 Pro 成功回复中，`reasoning_content` 思考链包含了**整套双 Safe 包裹 + 完整角色正文的「草稿版」**，而 `content` 字段是「正式版」。两版正文措辞高度相似但不完全一致，说明模型独立生成了两次。

**影响**：
- 后置审查（环节 C）扫描整个响应时，高浓度正文出现两次 → 暴露面翻倍
- 思考链中的草稿可能比正式版更「露骨」（模型在思考阶段没有约束），即使正式版安全，草稿也可能触发 `content_filter`
- 成功通过的回复中，`reasoning_content` 与 `content` 各包含一套完整正文，侥幸通过不等于无害

**对策**：
- 关闭所有 Gemini 模型的 `reasoning: false`（已在伊薇实测 11 个 Gemini 模型全部关闭，重启后生效）
- 双 Safe 包裹结构与关闭 reasoning 是**叠加防御**，不冲突
- 关闭 reasoning 后，思考链不再预演正文，输出只含一份正式版，截断风险直接减半