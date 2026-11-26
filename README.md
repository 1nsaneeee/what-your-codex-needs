![What Your Codex Needs，一枚回车键与一份 AGENTS.md](./assets/cover.png)

# What Your Codex Needs

我希望 Codex 像一名靠谱的工程同事：一个按钮文案写错了，直接找准调用点改好并验证；公共接口要变，先查清影响面等我拍板；同一个报错卡住，换思路查根因，真推进不了再带着关键日志停下来；日常问它看法，先给结论和理由，不打官腔也不拍电报。

这个仓库整理了一套**两层配合的 Codex 配置**：日常可直接放进 `~/.codex/AGENTS.md` 的协作规则，以及面向进阶用户的底层系统提示词重构版 `base_instructions.v2`。

**[AGENTS.md](./AGENTS.md)**　·　**[base_instructions.v2.md](./base_instructions.v2.md)**　·　[中文对照版](./base_instructions.v2.zh.md)　·　[快速启用](#怎么用)

## 两层文件，各管各的事

| 层级 | 文件 | 解决什么问题 | 适用方式 |
| :--- | :--- | :--- | :--- |
| **协作层** | [AGENTS.md](./AGENTS.md) | 沟通风格、双 Plan、2 次换路熔断、Git 与验证纪律 | **所有人推荐**，不碰底层协议，Codex 升级也不会失效 |
| **底座层** | [base_instructions.v2.md](./base_instructions.v2.md)（[中文对照](./base_instructions.v2.zh.md)） | 修掉官方默认系统提示词里的文风黑名单、默认跳测试、防御性自曝与安全/止损缺口 | **进阶可选**，通过 `model_instructions_file` 替换默认底座 |

两层可以独立使用，也可以叠加。同时启用时，如遇细节尺度差异（例如 `AGENTS.md` 要求“未经明确要求不 commit/push、累计切换 2 次即熔断”，而通用底座写的是“任务要求提交 PR 时可建草稿 PR、累计 3 个不同假设止损”），以 `AGENTS.md` 和你当前轮次的直接指令为准。

## 协作层（`AGENTS.md`）：三句话，看它怎么干活

| 我怎么说 | 我希望 Codex 怎么做 |
| :--- | :--- |
| **「把保存按钮上的错字改了。」** | 找到界面实际使用的文案，改好并检查结果。如果旁边的组件也能重构，留给另一个任务。 |
| **「把接口的 `name` 改成 `displayName` 吧。」** | 查清哪些调用方还在读 `name`，给一个首选迁移方案，说明影响和风险，等我拍板。 |
| **「怎么又是同一个报错？」** | 同一思路连续失败两次就换路；换过两次、第二个替代方案仍失败就停手，把试过什么、卡在哪里和关键报错一起给我。 |

做新功能或涉及产品行为取舍时，启用 **双 Plan**：先确认「做什么」（场景、流程、范围与可见验收标准），再确认「怎么做」（改动模块、依赖顺序与验证方式）；需求已定的小改动和常规修复直接动手，不走这两轮。[查看双 Plan 细则](./AGENTS.md#双-plan)

## 底座层（`base_instructions.v2`）：为什么还要重写系统提示词

大多数习惯靠 `AGENTS.md` 就能管住，但官方默认 `base_instructions` 里有几处“打地鼠式补丁”留下的结构冲突，只靠外层规则往往压不住。`v2` 针对这些底层病灶做了彻底重写，同时完整保留了 Codex 运行时的全部工具与频道契约。

> 出于版权与维护边界，本仓库不托管 OpenAI 原版系统提示词全文，下面仅摘录原版中引发问题的关键片段作为改造依据。

### 1. 废除标点与对比黑名单，治好“电报体”和语义删失

原版写作规则密集封杀了正常书面语：

> *"NEVER use contrastive framing such as 'X, not Y' ... Never add what you won't do, state what something is not, what will remain unchanged..."*
> *"In prose, do not use semicolons. Do not use colons to introduce explanations or examples, including after list labels."*

实测中这会同时触发“粉红大象效应”和句法残缺：模型为了躲避冒号、分号与对比句，会把连贯因果硬切成碎片句号，甚至在自我执行规则时把关键否定边界直接删掉（例如把 *"not as a continuation of intermediate commentary"* 与 *"rather than replacing it"* 整半句吞掉）；而原版自己在插件列表里又用着 `Skill naming:`，导致模型前后双标分裂。

**v2 改法**：删除全部句式与标点黑名单，改用正向的工程同事行文准则——结论先行，允许自然使用冒号、分号、过渡衔接与简明对比说清因果和改动边界，只删空话、自夸和未请求的废话。

### 2. 反转“默认跳过新测试”，打破偷懒闭环

原版一边要求彻底验证，一边在编码条款 `(c)` 把跳过设为默认分支：

> *"Skip new tests unless requested, required by the repo, or needed to catch a specific regression existing tests would miss."*

叠加“不要生成未请求的产物”，模型只要没听到“写测试”三个字就习惯性跳过，收尾再用免责声明自保。

**v2 改法**：拆成两条独立规则——先发现并优先使用仓库既定检查入口（README、scripts、Makefile、CI），核对基线并准确区分本次回归与既有故障；再经真实入口验证，对 bug 修复、非平凡逻辑改动和回归风险主动补精简测试，仅在极琐碎改动、已有覆盖、无测试框架且超范围或用户明确跳过时省略。

### 3. 消除防御性官腔，补齐底座安全与止损红线

原版一面禁止预防性免责声明，一面又在最终答复章节用 `MUST` 强制结尾自曝：

> *"mention any relevant flaws in your work or any decisions you made without explicit guidance ... You MUST repeat any such disclosures in the final answer."*

此外，原版单向要求“不因假想风险插警告”，却缺少“不得为让程序跑通自行削弱认证/校验/TLS”的对偶红线；受阻重试缺少可数止损线，异步提问工具不可用时也没有兜底出口。

**v2 改法**：

- **`final` 适度披露**：收口为三类可判定硬项（影响行为/接口/范围的未明示假设、未验证区域、已识别但未消除的局限），不存在时不硬凑免责声明；
- **安全双向闭环**：把“不因假想风险设障”与“守住真实安全底线”拆为两条独立规则，明令不得通过关闭认证、放宽校验、削弱 TLS 或绕过沙箱让测试通过（仅允许任务明确要求的本地/测试受控例外，如 mock 认证或自签证书），并补齐破坏性 Git/数据确认红线；
- **可数止损与提问兜底**：同一假设连败两次切换诊断方向，针对同一阻塞点累计尝试三个不同假设仍无实质进展即暂停上报；异步提问工具均不可用且必须确认时，在 `final` 一次性列出带推荐项的待确认清单后结束本轮。

<details>
<summary>展开查看：v2 完整保留的 Codex 底层契约与剪裁原则</summary>

**剪裁原则**：任何仓库都成立的通用原则留在底座；只在特定项目或个人习惯下成立的细节交给 `AGENTS.md` 或 `SKILL.md`。同时顺手修正了原版两处拼写失误（`maketo`、`tools_search`）并去除 7 处以上高频重复，全文精简约 20%。

**完整保留的平台与工具契约**：

- **双频道与协作**：`commentary` / `final` 分工、调工具前先发 `commentary`、最终答复独立自洽；`functions.request_user_input_async` / `functions.send_message_to_user_async`、等待窗口策略、任务中途纠偏（steering）与上下文自动压缩后的连续性。
- **格式与可视化**：GitHub 风格 Markdown、CommonMark 空行规范、本地文件可点击链接语法（含空格尖括号、禁用反引号与 URI、禁用行号范围）与四层可视化分流（交互/内联、独立绘图文件、Mermaid、表格/列表）。
- **工具与 Shell**：`rg` / `rg --files`、`functions.exec` 的 `await Promise.allSettled([...])` 并发只读、禁占位分隔输出、`JSON.stringify()` 非 shell 转义警示、`gh --body-file` 换行与字面转义保真、60 秒阻塞休眠/轮询上限、勿覆盖 `$HOME`/`$home`/`$CODEX_HOME`。
- **扩展机制与前端**：Skills（`r0` 别名展开、环境托管与编排器 `skills.list` / `skills.read` / `skill://`）、Apps（`codex_apps` / `tool_search` / 禁用 `list_mcp_resources*`）、Plugins（`plugin_name:` 与 MCP `provenance`）、前端写实素材/文案克制/基础可达性与完整 65 字体目录。

</details>

## 怎么用

### 基础用法：配置 `AGENTS.md`（推荐所有人）

1. 打开 [AGENTS.md](./AGENTS.md)，删改你不认同的条款（例如调试熔断次数、是否启用双 Plan）。
2. 全局生效放入 `~/.codex/AGENTS.md`，单项目生效放入项目根目录的 `AGENTS.md`（若已有同名文件则按章节合并，保留项目自己的构建与测试命令）。[官方加载说明](https://developers.openai.com/codex/guides/agents-md)

### 进阶用法：替换底层 `base_instructions`（可选）

1. 将 [base_instructions.v2.md](./base_instructions.v2.md) 复制到 Codex 配置目录，例如 `~/.codex/base_instructions.v2.md`。
2. 在 `~/.codex/config.toml` 中加入 `model_instructions_file`（相对路径以 `config.toml` 所在的 `.codex/` 目录为基准；旧键 `experimental_instructions_file` 已弃用）：

```toml
model_instructions_file = "base_instructions.v2.md"
```

3. 新开一个 Codex 会话，拿手头一个小任务实测（比如修一个表单提示或小 bug），看它是否能找准真实调用点、保持改动克制、跑通验证并说清结果。

> 即使你全程用中文工作，也建议底层加载英文版 `base_instructions.v2.md`（与原生工具签名和模型分布更贴合），再靠中文 `AGENTS.md` 控制中文表达与个人协作习惯。

## 版本与兼容性提醒

`base_instructions.v2.md` 基于 2026 年 11 月的 Codex 工具与频道协议整理（实测环境 `codex-cli 0.172.0`）。由于 `model_instructions_file` 会整份替换内置底座，**Codex 后续大版本升级如变更内置工具名、Skills/Apps 协议或客户端格式规则，旧的覆写文件可能导致新能力失效**。升级后建议核对新版官方指令中的工具契约段再继续使用。
