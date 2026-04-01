# Self-Evolution System

## Overview

Self-Evolution is a framework that enables an AI Agent's Skill system to **automatically learn from execution experience and improve over time**. Inspired by the [OpenSpace](https://github.com/HKUDS/OpenSpace) Self-Evolution architecture.

Core capabilities:

- **Automatic Trace Recording** — Captures the agent's reasoning process, tool calls, and results for every task
- **Intelligent Quality Analysis** — Uses LLM to evaluate task success rate and resource efficiency
- **Three Evolution Modes** — FIX, DERIVED, and CAPTURED for continuous skill optimization
- **Full Version Traceability** — Maintains a DAG (Directed Acyclic Graph) of skill lineage for audit and rollback

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

**Workflow:**

1. User sends a request → Agent executes the task
2. Execution Recorder captures the full trace (steps, tool calls, token usage, etc.)
3. After completion, Execution Analyzer uses LLM to evaluate trace quality
4. If optimization is needed, Skill Evolver automatically performs evolution
5. Evolved skill versions are stored in Skill Store for future use

---

## 2. Core Concepts

### 2.1 Execution Trace

After each task, the system automatically generates an execution trace:

```typescript
interface ExecutionTrace {
  id: string;
  task: string;              // Original user request
  steps: TraceStep[];        // Sequence of execution steps
  skillsUsed: string[];      // Skills that were used
  tokenUsage: {
    prompt: number;
    completion: number;
    total: number;
  };
  duration: number;          // Execution time in ms
  success: boolean;          // Whether the task succeeded
  createdAt: Date;
}

interface TraceStep {
  type: 'thinking' | 'tool_call' | 'tool_result' | 'skill_inject';
  content: string;
  timestamp: Date;
  metadata?: Record<string, unknown>;
}
```

### 2.2 Execution Analysis

The analyzer evaluates each trace and produces a structured result:

```typescript
interface ExecutionAnalysis {
  successScore: number;       // 0.0 - 1.0, task completion quality
  efficiencyScore: number;    // 0.0 - 1.0, resource utilization efficiency
  issues: Issue[];            // Problems found
  patterns: Pattern[];        // Reusable patterns
  suggestions: EvolutionSuggestion[];  // Evolution recommendations
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

The system supports three skill evolution modes for different scenarios:

| Mode | Scenario | Description |
|------|----------|-------------|
| **FIX** | A skill has defects | Patch the existing skill and bump its version |
| **DERIVED** | A skill needs specialization | Create a specialized child skill from a parent |
| **CAPTURED** | A new reusable workflow is discovered | Extract the workflow as a brand-new skill |

```typescript
interface EvolutionSuggestion {
  type: 'FIX' | 'DERIVED' | 'CAPTURED';
  skillName: string;
  reason?: string;
  priority: number;          // 1-5, higher is more urgent
}

interface EvolutionResult {
  type: string;
  parentSkillId?: string;    // FIX and DERIVED have a parent node
  skillId: string;
  skillVersion: string;
  changes: string;
  createdAt: Date;
}
```

---

## 3. Implementation

### 3.1 Skill Evolver

The Skill Evolver is the core component that performs evolution based on analysis suggestions:

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

  /** FIX: Patch defects in an existing skill */
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

  /** DERIVED: Create a specialized child from a parent skill */
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

  /** CAPTURED: Extract a new skill from a successful execution */
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

The Evolution Engine orchestrates the entire flow and serves as the main entry point:

```typescript
interface EvolutionConfig {
  enabled: boolean;
  analysisThreshold: number;     // Success score below this triggers analysis (default: 0.7)
  evolutionThreshold: number;    // Efficiency score below this triggers evolution (default: 0.5)
  minTracesForEvolution: number; // Minimum traces before first evolution (default: 5)
  autoEvolve: boolean;           // Auto-execute or require approval (default: true)
  maxEvolutionPerRun: number;    // Max evolutions per cycle (default: 3)
}

class EvolutionEngine {
  private config: EvolutionConfig;
  private analyzer: ExecutionAnalyzer;
  private evolver: SkillEvolver;
  private store: SkillStore;

  constructor(
    config: EvolutionConfig,
    analyzer: ExecutionAnalyzer,
    evolver: SkillEvolver,
    store: SkillStore,
  ) {
    this.config = config;
    this.analyzer = analyzer;
    this.evolver = evolver;
    this.store = store;
  }

  /** Main entry point after task execution */
  async processExecution(ctx: Context, trace: ExecutionTrace): Promise<void> {
    if (!this.config.enabled) return;

    // 1. Analyze the execution trace
    const analysis = await this.analyzer.analyze(ctx, trace);

    // 2. Record skill judgments
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

    // 3. Check if evolution is needed
    if (!this.shouldEvolve(analysis)) return;

    // 4. Execute evolution
    if (this.config.autoEvolve) {
      await this.executeEvolution(ctx, analysis, trace);
    }
  }

  private shouldEvolve(analysis: ExecutionAnalysis): boolean {
    if (analysis.efficiencyScore < this.config.evolutionThreshold) return true;
    return analysis.suggestions.some(s => s.priority >= 4);
  }

  private async executeEvolution(
    ctx: Context,
    analysis: ExecutionAnalysis,
    trace: ExecutionTrace,
  ): Promise<void> {
    let evolved = 0;
    for (const suggestion of analysis.suggestions) {
      if (evolved >= this.config.maxEvolutionPerRun) break;

      const result = await this.evolver.evolve(ctx, suggestion, trace);
      await this.store.updateSkillIndex(result);
      evolved++;
    }
  }

  /** Get evolution statistics */
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

```
src/
├── evolution/
│   ├── engine.ts          # Evolution Engine - main entry point
│   ├── analyzer.ts        # Execution Analyzer - LLM-based trace evaluation
│   ├── evolver.ts         # Skill Evolver - FIX / DERIVED / CAPTURED
│   ├── store.ts           # Skill Store - persistence layer
│   └── types.ts           # Type definitions
├── recorder/
│   ├── recorder.ts        # Execution Recorder - captures traces
│   └── trace.ts           # Trace types & utilities
└── skill/
    ├── loader.ts          # Skill Loader - load & match skills
    ├── registry.ts        # Skill Registry - registration center
    └── injector.ts        # Skill Injector - inject into agent context

skills/
├── _index.json            # Skill index
├── code-review/
│   └── SKILL.md
└── ...
```

---

## 5. Evolution Triggers

Evolution fires automatically when any of the following conditions is met:

| Trigger | Condition | Description |
|---------|-----------|-------------|
| Tool degradation | Same tool error 3 times in a row | Tool may need a fix |
| Low efficiency | EfficiencyScore < 0.5 | Wasted resources during execution |
| Success rate drop | < 70% success in last 10 executions | Skill quality degrading |
| Pattern discovery | High-frequency repeated workflow | Good candidate for a new skill |

---

## 6. Key Metrics

| Metric | Description | Target |
|--------|-------------|--------|
| Skill Hit Rate | Percentage of tasks matched to a skill | > 60% |
| Skill Success Rate | Success rate of tasks using skills | > 80% |
| Evolution Quality | Quality improvement after evolution | +15% |
| Token Savings | Token savings from using skills | > 30% |
| Time to Evolution | From issue detection to evolution completion | < 5min |

---

## References

- [OpenSpace](https://github.com/HKUDS/OpenSpace) — Self-Evolution architecture reference
