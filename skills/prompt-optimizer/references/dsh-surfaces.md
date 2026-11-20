# DSH 里的提示词载体

改 DSH 的提示词，先确认改的是哪一层。改错层，改得再对也不生效。

## 一、系统提示怎么装配

`systemPrompt` 服务在每次模型调用前装配提示词，输入有四类：

| 输入 | 来源 | 特点 |
|---|---|---|
| sections | 各插件注册的有序段落 | 按 order 排序；scoped section 覆盖同名 global section；`complete` 段落会替换整个提示词 |
| runtime contexts | 各插件注册的动态上下文 | 每个 step 重新求值，适合放"当前这一步的事实" |
| tool schemas | 工具注册表 | 名字、描述、参数一起进提示词；描述改动直接影响首轮 token |
| variables | 插值变量 | `[a-z][a-z0-9_]*`，被段落引用 |

装配后还有一道 presentation waterfall，它的文本是权威的。这一点意味着：**你写在某个插件 section 里的字，不一定原样到达模型**。判断某段文字是否生效，不能只看配置文件，要看装配结果或实际行为。

推论：同一句约束不要在两处注册（例如 AGENTS.md 里一次、preset 里一次）。两处都在，权重不会翻倍，只会让"到底哪条管用"变得无法归因。

## 二、四类载体

### AGENTS.md

- 全局 `~/.dsh/AGENTS.md`；项目内同名文件按项目路径覆盖，更具体的优先。
- 每个会话都会注入，随每轮对话计 token。写进去的每一条都在持续付费。
- 适合：跨项目的长期约定、身份与语言、安全边界、决策沉淀的位置。
- 不适合：单次任务的做法（放 skill）、某个模型的具体调优（放 preset）。

### agent preset

- 是 YAML 声明，由 bundle patch 携带；**不能直接编辑某个文件改一个 preset**。
- 内置 preset（`standard`、`ptc`、`minimal`、`cordis`）的声明在 Web bundle 的 `presets/<id>.patch.yml`。
- 改内置 preset 用覆盖：Loader 行 id 写 `preset-<id>`，`config` 必须完整重述（覆盖会替换整个 config，不是合并）。
- 人格用 `@deepseek-ai/dsh-persona` 的 `config.prefix`；工具、sections、策略由挂载的插件贡献。
- 安装/覆盖走 `plugin_manager` 的 `install_bundle`，目标是一个含 `package.json` 与 `cordis.patch.yml` 的目录；这一步需要 Full access 或授权。
- 验证：`list_bundles` 看已装 bundle，`list_plugins` 看 `preset-<id>` 的激活状态。
- **已存在的会话和孩子会话保留启动时的插件版本**；改完 preset 必须在新建会话里验证行为，旧会话的对比没有意义。

### 工具 description

- 改的是工具定义里的 `description` 和参数描述，不是某个 prompt 文件。
- 遵循 `dsh-doc` 的 `references/tool-design.md`：
  - 描述行为而不是实现；不写内部机制、结果里的标记、执行细节。
  - 参数规则（默认值、范围、配对、何时设置）写在该参数上，不写在工具描述里。
  - 每条事实只说一次，不与其他工具重复。
  - 删掉模型能从调用结果学到的约束（文件不存在读失败，不需要写在描述里）。
- 改动后测首轮 prompt token 的前后差。工具描述膨胀是最隐蔽的成本来源，因为它不随对话增长，但每次调用都付。

### SKILL.md

- 位置 `~/.dsh/skills/<name>/SKILL.md`，frontmatter 必须有 `name` 与 `description`，`name` 用 kebab-case。
- `description` 只写**何时使用**（触发条件与症状），绝不写流程摘要。写了摘要，模型会照着摘要执行而不读正文 —— 一条被压缩过的流程比没有流程更糟。
- frontmatter 必须是合法 YAML：`description` 里出现「冒号 + 空格」（例如 `Triggers: 提示词优化`）会让整份 frontmatter 解析失败，该 skill 被**静默跳过** —— 目录里没有它，模型侧看到的是「这个 skill 不存在」，没有任何报错。装完先用任一 YAML 解析器验一遍，再确认它出现在技能目录里。
- 正文放方法、判据、正反例；重型参考拆到 `references/`，从正文链接过去。
- 装之前查重名与职责重叠，两个 skill 争同一个触发词，实际效果是两个都不生效。

## 三、放在哪一层

| 你想表达的 | 放这里 | 不要放 |
|---|---|---|
| 跨项目的长期约束、语言、安全边界 | `AGENTS.md` | preset（只对部分会话生效） |
| 某一类 agent 的人格、工具集、默认流程 | agent preset | AGENTS.md（会污染所有会话） |
| 工具怎么调、参数怎么填 | 工具定义 | AGENTS.md（模型看不到调用时的近因上下文） |
| 一类任务的做法、判据、清单 | `SKILL.md` 正文 | `SKILL.md` 的 description |
| 这一次请求的数据、用户原话 | user message | system prompt（同时是注入面的来源） |

数据与指令分离不只是整洁问题：把用户输入拼进 system，等于让输入获得指令权限。

## 四、验证

1. **改完开新会话。** preset 与插件改动对已有会话不生效，旧会话里出现的改善或恶化都不算证据。
2. **对比首轮 token。** 从 provider 返回的 usage 或会话用量取，不要只数字符。
3. **行为对比用 subagent 采样。** 同一任务重复派发 5 次以上，看输出分布，不看单次。协议见 [verification.md](verification.md)。
4. **确认装配结果。** 涉及 sections 或插件时，先确认你写的内容确实进入了装配，再谈措辞优化。装配不进去，一切改写都是零。

## 五、风险

- 覆盖内置 preset 时忘了重述其他字段，会静默丢掉整份 config 里的工具。
- 已存会话保留旧插件版本，容易得出"改了没用"的错误结论。
- `AGENTS.md` 每轮计费，把一次性任务的做法写进去，会长期抬高每次调用的成本。
- skill 的 description 写成流程摘要，会让模型跳读正文，且很难在事后发现 —— 表现是"这个 skill 好像没起作用"。
