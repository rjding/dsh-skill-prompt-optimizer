# dsh-prompt-optimizer

A [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness) skill that turns a rough one-liner into a prompt a model can actually follow — delivered as **one copyable code block**, nothing else.

一句话进，一个代码块出。不写诊断、不写样本、不写记账，除非你要。

## What it does

- **只交成品**：默认输出只有一个代码块，里面是要用的提示词，块外不写解释。
- **语言跟随输入**：你写中文，就给中文提示词；图像模型只吃英文时，它先问，不自己决定。
- **缺信息先问**：风格、用途、目标模型没说清时，弹一次选择框（2–4 个可点选项），选完再给成品。
- **能删就删**：身份、已配置的事实、别处已有的规则，它不会重写一遍。

## Install

```sh
dsh plugin add dsh-prompt-optimizer
```

装完在技能目录里会出现 `prompt-optimizer`。新会话直接说「提示词优化一下这段 …」，或显式加载该技能。

## Usage

```
帮我优化这句话「你得给我写个好点的周报，别太啰嗦，也别太干」
```

输出：

```
```text
（可直接使用的提示词）
```
```

需要它讲理由时再说一句「为什么这么改」，它才会展开诊断。

## What it will not do

- 不会替你在「写实 / 卡通 / 3D」之间自己挑一个。
- 不会在你用中文提问时，擅自给你一段英文提示词。
- 不会为了显得严谨，编造评测指标或提升幅度。

## Uninstall

```sh
dsh plugin remove dsh-prompt-optimizer
```

## License

MIT
