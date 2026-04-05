# AI-Infra-Skills 剩余文件创建任务

## 背景

这是 ai-infra-skills repo 的文件创建任务。需要创建 11 个新文件 + 更新 3 个已有文件。所有文件都在 `/home/ubuntu/scipts/verl-grounding/ai-infra-skills/` 下。

**源文件位置**: `/home/ubuntu/scipts/repos/` 下的 12 个 clone 的 repo

## 通用规则

- 去除 repo 特定的文件路径，保留通用模式
- 用 `<!-- source: repo -->` 标注内容来源
- SKILL.md 文件使用 YAML frontmatter
- 保持实用性和可操作性
- 子文件（非 SKILL.md）不要 YAML frontmatter

## 任务 1: 更新 05-profiler/SKILL.md 的 frontmatter

文件: `.claude/skills/05-profiler/SKILL.md`

只改 frontmatter，把：
```yaml
---
name: profiler-ncu
description: "Performance profiling, benchmarking, and NCU analysis for AI infrastructure"
---
```
改为：
```yaml
---
name: profiler
description: Performance profiling, benchmarking, and NCU analysis for AI infrastructure
model: claude-opus-4-6
tools: [Bash, Read, Grep, Glob]
---
```

## 任务 2: 更新 agents/fsdp-engine-expert.md

文件: `.claude/agents/fsdp-engine-expert.md`

读取当前文件，然后泛化：
- 去掉所有 AReaL 特定引用（areal/engine/fsdp_engine.py 等路径）
- tools 中去掉 `Task`，保留 `[Read, Grep, Glob]`
- 保留通用的 FSDP2 使用指导、配置模式、并行策略、troubleshooting
- 参考源: `/home/ubuntu/scipts/repos/AReaL/.claude/agents/fsdp-engine-expert.md`

## 任务 3: 更新 agents/megatron-engine-expert.md

文件: `.claude/agents/megatron-engine-expert.md`

同样泛化，去掉 AReaL 特定引用，tools 去掉 `Task`。
- 参考源: `/home/ubuntu/scipts/repos/AReaL/.claude/agents/megatron-engine-expert.md`

## 任务 4: 创建 agents/algorithm-expert.md

文件: `.claude/agents/algorithm-expert.md`

```yaml
---
name: algorithm-expert
description: RL algorithm expert for LLM training. Use for GRPO, PPO, DAPO, reward shaping, advantage normalization, or training loss computation.
tools: [Read, Grep, Glob]
model: opus
---
```

泛化的 RL 算法专家：
- PPO-family 算法表（PPO, GRPO, Dr.GRPO, GSPO, DAPO, RLOO, SAPO）及关键特征
- Workflow 模式（单轮、多轮、视觉）
- Reward function 设计（签名、常见 rewards）
- Loss computation（PPO clipped objective 公式）
- Advantage normalization levels（batch, group, None）
- Debugging：advantage stats, importance ratio, clipping rate
- 常见问题表
- 参考源: `/home/ubuntu/scipts/repos/AReaL/.claude/agents/algorithm-expert.md`

## 任务 5: 创建 agents/launcher-scheduler.md

文件: `.claude/agents/launcher-scheduler.md`

```yaml
---
name: launcher-scheduler
description: Cluster launching and resource scheduling expert (Slurm/Ray/Kubernetes). Use for deployment, GPU allocation, or multi-node coordination issues.
tools: [Read, Grep, Glob]
model: sonnet
---
```

泛化的集群启动/调度专家：
- Launcher vs Scheduler 职责
- 配置：cluster spec, scheduler config
- 环境变量传播链
- 诊断流程（symptom → cause → first checks 表）
- 最佳实践
- 参考源: `/home/ubuntu/scipts/repos/AReaL/.claude/agents/launcher-scheduler-expert.md`

## 任务 6: 创建 agents/code-verifier.md

文件: `.claude/agents/code-verifier.md`

```yaml
---
name: code-verifier
description: Code verification agent. Use PROACTIVELY after code changes to run formatting, linting, and tests before commit.
tools: [Read, Grep, Glob, Bash]
model: haiku
---
```

泛化的代码验证 agent：
- 识别变更文件（git status/diff）
- 运行格式化和 linting（pre-commit, ruff, clang-format, mdformat）
- 分类运行测试（按 GPU 需求分类）
- 自动修复行为
- 结果报告格式
- 参考源: `/home/ubuntu/scipts/repos/AReaL/.claude/agents/code-verifier.md`

## 任务 7: 创建 agents/model-debug-agent.md

文件: `.claude/agents/model-debug-agent.md`

```yaml
---
name: model-debug-agent
description: Debug model onboarding and inference pipeline failures. Use when a model fails to load, convert, or run through an inference/deployment pipeline.
tools: [Read, Grep, Glob, Bash, Edit, Write]
model: sonnet
---
```

泛化的模型调试 agent（NOT AutoDeploy 特定）：
- 工作流: 确定 GPU 需求 → 检查可用性 → 运行推理流水线 → 分析错误 → 迭代
- 模型注册表模式
- 常见陷阱: 权重名不匹配、模块层级差异
- 迭代策略: 减少隐藏层加速调试、切换 sharding/world_size
- 参考源: `/home/ubuntu/scipts/repos/TensorRT-LLM/.claude/agents/ad-debug-agent.md`

## 任务 8: 创建 06-code-review/SKILL.md

文件: `.claude/skills/06-code-review/SKILL.md`

需要先 `mkdir -p .claude/skills/06-code-review`

```yaml
---
name: code-review
description: PR workflow, commit conventions, and CI failure triage for AI infrastructure projects
model: claude-opus-4-6
tools: [Bash, Read, Grep, Glob]
---
```

概述链接 pr-workflow.md 和 ci-failure-triage.md。跨领域原则：分布式代码难以全面测试，检查所有并行组合，验证数值正确性。

## 任务 9: 创建 06-code-review/pr-workflow.md

文件: `.claude/skills/06-code-review/pr-workflow.md`

`<!-- source: AReaL -->`

泛化的 PR 工作流：
1. Conventional Commits（类型、格式、规则、示例）
2. PR Review 工作流（4 阶段：分析、任务规划、执行、评分）
3. Create PR 工作流（验证、rebase、squash、创建）
4. PR 描述模板
- 参考源:
  - `/home/ubuntu/scipts/repos/AReaL/.claude/commands/review-pr.md`
  - `/home/ubuntu/scipts/repos/AReaL/.claude/commands/create-pr.md`
  - `/home/ubuntu/scipts/repos/AReaL/.claude/commands/gen-commit-msg.md`

## 任务 10: 创建 06-code-review/ci-failure-triage.md

文件: `.claude/skills/06-code-review/ci-failure-triage.md`

`<!-- source: TensorRT-LLM, SGLang, torchtitan -->`

泛化的 CI 故障调查：
1. CI 故障检索（从 GitHub Actions/Jenkins/GitLab 获取失败详情）
2. Bisect CI 回归（6 阶段方法论）
3. Pipeline 故障分析（按根因分桶）
4. 关键模式诊断表
- 参考源:
  - `/home/ubuntu/scipts/repos/TensorRT-LLM/.claude/skills/ad-pipeline-failure-pr/SKILL.md`
  - `/home/ubuntu/scipts/repos/TensorRT-LLM/.claude/skills/ci-failure-retrieval/SKILL.md`
  - `/home/ubuntu/scipts/repos/sglang/.claude/skills/sglang-bisect-ci-regression/SKILL.md`
  - `/home/ubuntu/scipts/repos/torchtitan/.claude/skills/torch_bisect/SKILL.md`

## 任务 11: 创建 commands/commit.md

文件: `.claude/commands/commit.md`

需要先 `mkdir -p .claude/commands`

```yaml
---
name: commit
description: Generate intelligent commit messages based on staged changes using Conventional Commits format.
allowed-tools: [Bash, Read, Grep]
---
```

- Usage: `/commit [--amend] [--scope <scope>]`
- 工作流: 分析 staged changes → 分类 → 确定 scope → 生成消息 → 确认 → commit
- 参考源: `/home/ubuntu/scipts/repos/AReaL/.claude/commands/gen-commit-msg.md`

## 任务 12: 创建 commands/review-pr.md

文件: `.claude/commands/review-pr.md`

```yaml
---
name: review-pr
description: Intelligent PR code review with dynamic domain-based agent allocation.
allowed-tools: [Read, Grep, Glob, Bash]
---
```

- Usage: `/review-pr [PR_NUMBER] [--quick]`
- 4 阶段工作流
- 参考源: `/home/ubuntu/scipts/repos/AReaL/.claude/commands/review-pr.md`

## 任务 13: 创建 commands/create-pr.md

文件: `.claude/commands/create-pr.md`

```yaml
---
name: create-pr
description: Rebase, squash, and create a PR on GitHub with intelligent title and description.
allowed-tools: [Bash, Read, Grep]
---
```

- Usage: `/create-pr [--draft] [--base <branch>]`
- 包含 fork 工作流支持
- 参考源: `/home/ubuntu/scipts/repos/AReaL/.claude/commands/create-pr.md`

## 任务 14: 创建 commands/bisect.md

文件: `.claude/commands/bisect.md`

```yaml
---
name: bisect
description: Git bisect to find the commit that introduced a regression.
allowed-tools: [Bash, Read, Grep]
---
```

- 4 阶段: 收集信息 → 设置 → bisect 循环 → 报告
- 参考源: `/home/ubuntu/scipts/repos/torchtitan/.claude/skills/torch_bisect/SKILL.md`

## 执行顺序建议

1. 先创建目录：`mkdir -p .claude/skills/06-code-review .claude/commands`
2. 更新 3 个已有文件（任务 1-3）
3. 创建 4 个 agent 文件（任务 4-7）
4. 创建 3 个 skill 文件（任务 8-10）
5. 创建 4 个 command 文件（任务 11-14）

每个文件的内容要参考对应的源文件，读取后泛化。
