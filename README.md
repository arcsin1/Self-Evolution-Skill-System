# Self-Evolution Skill System

**[中文文档](./self_evolution_cn.md)** | **[English](./self_evolution_en.md)**

> A framework that enables AI Agent's Skill system to automatically learn from execution experience and improve over time. Inspired by [OpenSpace](https://github.com/HKUDS/OpenSpace).


最近在看了社区实现，也很有意思的一个事情：让 AI Agent 的技能系统**自己进化**。

不是手动调 prompt，不是人工优化 workflow，而是 Agent 跑完任务后，自己复盘、自己发现问题、自己生成新版本的 Skill。

## 它是怎么工作的？

简单说就一个循环：

```
执行任务 → 记录轨迹 → 分析质量 → 自动进化
```

每次 Agent 完成任务，系统会自动记录整个过程（推理步骤、工具调用、Token 消耗），然后用 LLM 分析这次执行有没有问题。如果有，就自动进化。

## 三种进化模式

这个我觉得最有意思：

| 模式 | 干嘛的 |
|------|--------|
| FIX | Skill 有 bug？自己修，自动升版本 |
| DERIVED | 需要针对特定场景？从父 Skill 派生一个专门的 |
| CAPTURED | 发现新的高效工作流？直接提取成新 Skill |

也就是说，你写了一个通用的 code-review Skill，跑着跑着它自己发现"React 项目的 review 可以单独拎出来"，就会自动派生一个 `code-review-react` 出来。

而且所有进化关系用 DAG 维护，随时可以回滚。

## 为什么我觉得这件事值得做？

现在大家用 AI Agent，基本上是：

- 写 Skill → 跑一下 → 效果不好 → 人工改 → 再跑
- 经验分散在每次对话里，无法积累
- 同样的坑踩很多次

这套系统想解决的就是：**让执行经验变成可积累的资产**。

每次执行都是一次训练数据，Agent 用得越多，Skill 越强。

## 技术栈

纯 TypeScript 设计实现，核心就几个模块：

- **Execution Recorder** — 记录轨迹
- **Execution Analyzer** — LLM 分析质量
- **Skill Evolver** — 执行进化（FIX/DERIVED/CAPTURED）
- **Evolution Engine** — 编排整个流程




#AI #Agent #SelfEvolution #LLM #人工智能 #提示工程 #SkillSystem
