# Self-Evolution System

## Overview

Self-Evolution 是一套让 Agent 的 Skill（技能）系统**自动从执行经验中学习和进化**的框架。灵感来源于 [OpenSpace](https://github.com/HKUDS/OpenSpace) 的 Self-Evolution 架构。

核心能力：

- **自动记录执行轨迹** — 捕获 Agent 每次任务的推理过程、工具调用和结果
- **智能分析执行质量** — 基于 LLM 评估任务成功率和资源效率
- **三种进化模式** — FIX（修复）、DERIVED（派生）、CAPTURED（捕获），持续优化 Skill
- **完整的版本追溯** — 维护 Skill 进化的 DAG（有向无环图）关系，可审计、可回滚

---

## 1. Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                    Self-Evolution Cycle                      │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│   User Request ──▶ Agent Loop ──▶ Execution Recorder         │
│        │                │                  │                  │
│        ▼                ▼                  ▼                  │
│   Skill Injection ◀── Skill Registry ◀── Execution Analyzer  │
│                              │                               │
│                              ▼                               │
│                        Skill Evolver                         │
│                              │                               │
│                              ▼                               │
│                        Skill Store                           │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

**工作流程：**

1. 用户发起请求 → Agent 执行任务
2. Execution Recorder 全程记录执行轨迹（步骤、工具调用、Token 消耗等）
3. 任务完成后，Execution Analyzer 基于 LLM 分析轨迹质量
4. 如果检测到优化空间，Skill Evolver 自动执行进化
5. 进化后的新版本 Skill 存入 Skill Store，供后续任务使用

---

## 2. Core Concepts

### 2.1 Execution Trace（执行轨迹）

每次 Agent 完成任务后，系统自动生成一条执行轨迹，包含：

```typescript
interface ExecutionTrace {
  id: string;
  task: string;              // 用户原始请求
  steps: TraceStep[];        // 执行步骤序列
  skillsUsed: string[];      // 使用了哪些 Skill
  tokenUsage: {
    prompt: number;
    completion: number;
    total: number;
  };
  duration: number;          // 执行耗时(ms)
  success: boolean;          // 任务是否成功
  createdAt: Date;
}

interface TraceStep {
  type: 'thinking' | 'tool_call' | 'tool_result' | 'skill_inject';
  content: string;
  timestamp: Date;
  metadata?: Record<string, unknown>;
}
```

### 2.2 Execution Analysis（执行分析）

分析器对每条轨迹进行评估，输出结构化结果：

```typescript
interface ExecutionAnalysis {
  successScore: number;       // 0.0 - 1.0，任务完成质量
  efficiencyScore: number;    // 0.0 - 1.0，资源利用效率
  issues: Issue[];            // 发现的问题
  patterns: Pattern[];        // 可复用的模式
  suggestions: EvolutionSuggestion[];  // 进化建议
}

interface Issue {
  type: string;               // 'tool_error' | 'redundant_step' | 'missing_context' ...
  description: string;
  severity: 'low' | 'medium' | 'high';
}

interface Pattern {
  type: string;
  description: string;
  steps: string[];
  reusable: boolean;
}
```

### 2.3 Three Evolution Modes

系统支持三种 Skill 进化模式，覆盖不同场景：

| 模式 | 场景 | 说明 |
|------|------|------|
| **FIX** | Skill 存在缺陷 | 在原 Skill 基础上修复，生成新版本 |
| **DERIVED** | 需要针对特定场景优化 | 基于父 Skill 派生出专门化的子 Skill |
| **CAPTURED** | 发现新的可复用工作流 | 从成功执行中提取，创建全新 Skill |

```typescript
interface EvolutionSuggestion {
  type: 'FIX' | 'DERIVED' | 'CAPTURED';
  skillName: string;
  reason?: string;
  priority: number;          // 1-5，越高越紧急
}

interface EvolutionResult {
  type: string;
  parentSkillId?: string;    // FIX 和 DERIVED 有父节点
  skillId: string;
  skillVersion: string;
  changes: string;
  createdAt: Date;
}
```

---

## 3. Implementation

### 3.1 Skill Evolver

Skill 进化器是核心组件，负责根据分析建议执行具体的进化操作：

```typescript
class SkillEvolver {
  constructor(
    private llmClient: LLMClient,
    private store: SkillStore,
    private registry: SkillRegistry,
    private skillDir: string,
  ) {}

  async evolve(
    ctx: Context,
    suggestion: EvolutionSuggestion,
    trace: ExecutionTrace,
  ): Promise<EvolutionResult> {
    switch (suggestion.type) {
      case 'FIX':
        return this.fixSkill(ctx, suggestion, trace);
      case 'DERIVED':
        return this.deriveSkill(ctx, suggestion, trace);
      case 'CAPTURED':
        return this.captureSkill(ctx, suggestion, trace);
      default:
        throw new Error(`Unknown evolution type: ${suggestion.type}`);
    }
  }

  /** FIX: 修复现有 Skill 的缺陷 */
  private async fixSkill(
    ctx: Context,
    suggestion: EvolutionSuggestion,
    trace: ExecutionTrace,
  ): Promise<EvolutionResult> {
    const existing = await this.registry.loadSkill(suggestion.skillName);
    const patch = await this.generateFixPatch(ctx, existing, suggestion, trace);
    const newVersion = incrementVersion(existing.version);
    const newContent = applyPatch(existing.content, patch);

    const newSkillId = await this.saveSkillVersion(
      ctx, suggestion.skillName, newVersion, newContent, 'FIX', existing.id,
    );

    return {
      type: 'FIX',
      parentSkillId: existing.id,
      skillId: newSkillId,
      skillVersion: newVersion,
      changes: patch.toString(),
      createdAt: new Date(),
    };
  }

  /** DERIVED: 基于父 Skill 派生专门化版本 */
  private async deriveSkill(
    ctx: Context,
    suggestion: EvolutionSuggestion,
    trace: ExecutionTrace,
  ): Promise<EvolutionResult> {
    const parent = await this.registry.loadSkill(suggestion.skillName);
    const derivedContent = await this.generateDerivedContent(ctx, parent, suggestion, trace);
    const newName = `${suggestion.skillName}-${generateSuffix()}`;

    const newSkillId = await this.saveSkillVersion(
      ctx, newName, '1.0.0', derivedContent, 'DERIVED', parent.id,
    );

    return {
      type: 'DERIVED',
      parentSkillId: parent.id,
      skillId: newSkillId,
      skillVersion: '1.0.0',
      changes: `Derived for specific context: ${suggestion.reason}`,
      createdAt: new Date(),
    };
  }

  /** CAPTURED: 从成功执行中捕获新 Skill */
  private async captureSkill(
    ctx: Context,
    suggestion: EvolutionSuggestion,
    trace: ExecutionTrace,
  ): Promise<EvolutionResult> {
    const skillContent = await this.generateCapturedContent(ctx, suggestion, trace);
    const skillName = generateSkillName(suggestion, trace);

    const newSkillId = await this.saveSkillVersion(
      ctx, skillName, '1.0.0', skillContent, 'CAPTURED',
    );

    return {
      type: 'CAPTURED',
      skillId: newSkillId,
      skillVersion: '1.0.0',
      changes: 'Captured from successful execution',
      createdAt: new Date(),
    };
  }
}
```

### 3.2 Evolution Engine

进化引擎协调整个流程，是系统的主入口：

```typescript
interface EvolutionConfig {
  enabled: boolean;
  analysisThreshold: number;     // 触发分析的成功率阈值 (default: 0.7)
  evolutionThreshold: number;    // 触发进化的效率阈值 (default: 0.5)
  minTracesForEvolution: number; // 最少积累多少轨迹才触发进化 (default: 5)
  autoEvolve: boolean;           // 是否自动执行进化 (default: true)
  maxEvolutionPerRun: number;    // 单次执行最多进化几个 Skill (default: 3)
}

class EvolutionEngine {
  private config: EvolutionConfig;
  private analyzer: ExecutionAnalyzer;
  private evolver: SkillEvolver;
  private store: SkillStore;

  constructor(config: EvolutionConfig, analyzer: ExecutionAnalyzer, evolver: SkillEvolver, store: SkillStore) {
    this.config = config;
    this.analyzer = analyzer;
    this.evolver = evolver;
    this.store = store;
  }

  /** 任务执行完成后的主入口 */
  async processExecution(ctx: Context, trace: ExecutionTrace): Promise<void> {
    if (!this.config.enabled) return;

    // 1. 分析执行轨迹
    const analysis = await this.analyzer.analyze(ctx, trace);

    // 2. 记录 Skill 评判
    for (const skillName of trace.skillsUsed) {
      await this.store.saveJudgment({
        skillId: skillName,
        traceId: trace.id,
        success: analysis.successScore > 0.7,
        score: analysis.successScore,
        reasoning: `Efficiency: ${analysis.efficiencyScore.toFixed(2)}`,
        judgedAt: new Date(),
      });
    }

    // 3. 判断是否需要进化
    if (!this.shouldEvolve(analysis)) return;

    // 4. 执行进化
    if (this.config.autoEvolve) {
      await this.executeEvolution(ctx, analysis, trace);
    }
  }

  private shouldEvolve(analysis: ExecutionAnalysis): boolean {
    if (analysis.efficiencyScore < this.config.evolutionThreshold) return true;
    return analysis.suggestions.some(s => s.priority >= 4);
  }

  private async executeEvolution(ctx: Context, analysis: ExecutionAnalysis, trace: ExecutionTrace): Promise<void> {
    let evolved = 0;
    for (const suggestion of analysis.suggestions) {
      if (evolved >= this.config.maxEvolutionPerRun) break;

      const result = await this.evolver.evolve(ctx, suggestion, trace);
      await this.store.updateSkillIndex(result);
      evolved++;
    }
  }

  /** 获取进化统计 */
  async getStats(ctx: Context): Promise<EvolutionStats> {
    const skills = await this.store.listSkills(ctx);
    const byType = new Map<string, number>();
    for (const s of skills) {
      byType.set(s.evolutionType, (byType.get(s.evolutionType) ?? 0) + 1);
    }
    return { totalSkills: skills.length, byType: Object.fromEntries(byType) };
  }
}
```

---

## 4. Project Structure

 > 伪代码结构设计
```
src/
├── evolution/
│   ├── engine.ts          # Evolution Engine - 进化引擎主入口
│   ├── analyzer.ts        # Execution Analyzer - 执行分析器
│   ├── evolver.ts         # Skill Evolver - Skill 进化器
│   ├── store.ts           # Skill Store - 持久化存储层
│   └── types.ts           # 类型定义
├── recorder/
│   ├── recorder.ts        # Execution Recorder - 执行记录器
│   └── trace.ts           # Trace 类型与工具函数
└── skill/
    ├── loader.ts          # Skill Loader - 加载与匹配
    ├── registry.ts        # Skill Registry - 注册中心
    └── injector.ts        # Skill Injector - 注入到 Agent 上下文

skills/
├── _index.json            # Skill 索引
├── code-review/
│   └── SKILL.md
└── ...
```

---

## 5. Evolution Triggers

进化在以下条件满足时自动触发：

| 触发条件 | 阈值 | 说明 |
|----------|------|------|
| 工具降级检测 | 连续 3 次相同工具错误 | 工具可能需要修复 |
| 效率过低 | EfficiencyScore < 0.5 | 执行过程存在浪费 |
| 成功率下降 | 过去 10 次成功率 < 70% | Skill 质量退化 |
| 模式发现 | 高频重复工作流 | 适合捕获为新 Skill |

---

## 6. Key Metrics

| Metric | Description | Target |
|--------|-------------|--------|
| Skill Hit Rate | 匹配到 Skill 的任务比例 | > 60% |
| Skill Success Rate | 使用 Skill 的任务成功率 | > 80% |
| Evolution Quality | 进化后 Skill 的质量提升 | +15% |
| Token Savings | 使用 Skill 节省的 Token | > 30% |
| Time to Evolution | 从问题发现到进化完成 | < 5min |

---

## References

- [OpenSpace](https://github.com/HKUDS/OpenSpace) — Self-Evolution 架构参考
