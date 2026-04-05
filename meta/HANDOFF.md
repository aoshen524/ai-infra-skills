# AI Infra Skills Repo — 任务交接文档

> Status update (2026-04-05): this handoff reflects the earlier bootstrap phase and is
> now historical context. The missing-file checklist here has been completed. For the
> current 12-repo coverage state and placement audit, see
> [COVERAGE_AUDIT.md](/home/ubuntu/scipts/verl-grounding/ai-infra-skills/COVERAGE_AUDIT.md).

## 项目目标

创建一个独立的 Claude Code skills 集合 repo，整合 12 个明星开源推理/训练/RL/算子框架的 professional skills 和 behavioral constraints。

## 目标框架

| 领域 | 项目 |
|------|------|
| 推理 | vLLM, SGLang, Ollama, TensorRT-LLM |
| 训练 | Megatron-LM, PyTorch/torchtitan |
| RL | veRL, AReaL, SLIME |
| 算子 | FlashAttention, FlashInfer |
| 通信 | NCCL |

## 源码位置

- 所有 12 个 repo 已 clone 到 `/home/ubuntu/scipts/repos/`（`--depth 1`）
- 当前 repo（verl fork）在 `/home/ubuntu/scipts/verl-grounding/`
- skills repo 在 `/home/ubuntu/scipts/verl-grounding/ai-infra-skills/`

## 已完成的文件

```
ai-infra-skills/
├── .claude/
│   ├── rules/                          ✅ 全部完成
│   │   ├── api-config.md
│   │   ├── code-style.md
│   │   ├── distributed.md
│   │   ├── models.md
│   │   └── testing.md
│   │
│   └── skills/
│       ├── 01-server/                  ✅ 全部完成
│       │   ├── SKILL.md
│       │   └── gpu-resource.md
│       │
│       ├── 02-env-source-log/          ⚠️ 部分完成
│       │   ├── SKILL.md               ✅
│       │   ├── build-and-test.md      ✅
│       │   ├── ci-workflow.md         ✅
│       │   └── debug-cuda-crash.md    ❌ 需要创建
│       │
│       ├── 03-design/                  ✅ 全部完成
│       │   ├── SKILL.md
│       │   ├── model-onboard.md
│       │   ├── add-kernel.md
│       │   ├── add-workflow.md
│       │   └── engine-expert.md
│       │
│       ├── 04-review-plan/             ⚠️ 部分完成
│       │   ├── SKILL.md               ✅
│       │   ├── review-domains.md      ✅
│       │   └── review-templates.md    ❌ 需要创建
│       │
│       ├── 05-profiler/                ❌ 全部需要创建
│       │   ├── SKILL.md
│       │   ├── benchmark-kernel.md
│       │   └── performance-tuning.md
│       │
│       └── 06-code-review/             ❌ 全部需要创建
│           ├── SKILL.md
│           ├── pr-workflow.md
│           └── ci-failure-triage.md
│
├── .claude/agents/                     ❌ 全部需要创建
│   ├── fsdp-engine-expert.md
│   ├── megatron-engine-expert.md
│   ├── algorithm-expert.md
│   ├── launcher-scheduler.md
│   ├── code-verifier.md
│   └── ad-debug-agent.md
│
├── .claude/commands/                   ❌ 全部需要创建
│   ├── commit.md
│   ├── review-pr.md
│   ├── create-pr.md
│   └── bisect.md
│
├── knowledge/                          ❌ 全部需要创建
│   ├── kernels/
│   │   ├── debug-2cta-hang.md         (有旧内容需覆盖)
│   │   ├── tma-sync-hazard.md
│   │   ├── sass-mma-analysis.md
│   │   └── jit-architecture.md
│   ├── distributed/
│   │   ├── debug-distributed.md
│   │   └── nccl-patterns.md
│   └── ci-cd/
│       └── sglang-ci-system.md
│
└── CLAUDE.md                           ❌ 需要创建
```

## 剩余任务及源文件映射

### 1. debug-cuda-crash.md
**源文件**:
- `/home/ubuntu/scipts/repos/flashinfer/.claude/skills/debug-cuda-crash/SKILL.md`
- `/home/ubuntu/scipts/repos/sglang/.claude/skills/debug-cuda-crash/SKILL.md`
**内容**: 综合 API logging levels (FlashInfer 0/1/3/5, SGLang 0/1/3/5/10), 常见 CUDA 错误, compute-sanitizer, 多进程调试, kernel printf

### 2. review-templates.md
**源文件**: `/home/ubuntu/scipts/repos/AReaL/.claude/data/review-pr-templates.md`
**内容**: 泛化 review 模板，保留领域 checklist

### 3. 05-profiler/ (3 files)
**源文件**:
- `/home/ubuntu/scipts/repos/flashinfer/.claude/skills/benchmark-kernel/SKILL.md`
- `/home/ubuntu/scipts/repos/sglang/.claude/skills/sglang-auto-benchmark/SKILL.md`
- `/home/ubuntu/scipts/repos/sglang/.claude/skills/generate-profile/SKILL.md`
- `/home/ubuntu/scipts/repos/flash-attention/AI/SM90_BLOCK_SIZE_TUNING.md`
- `/home/ubuntu/scipts/repos/flash-attention/AI/SASS_MMA_ANALYSIS.md`
**内容**: SKILL.md 概述, benchmark-kernel.md (CUPTI/Events/bench_gpu_time), performance-tuning.md (block size/SASS/MMA)

### 4. 06-code-review/ (3 files)
**源文件**:
- `/home/ubuntu/scipts/repos/AReaL/.claude/commands/review-pr.md`
- `/home/ubuntu/scipts/repos/AReaL/.claude/commands/create-pr.md`
- `/home/ubuntu/scipts/repos/AReaL/.claude/commands/gen-commit-msg.md`
- `/home/ubuntu/scipts/repos/TensorRT-LLM/.claude/skills/ad-pipeline-failure-pr/SKILL.md`
- `/home/ubuntu/scipts/repos/TensorRT-LLM/.claude/skills/ci-failure-retrieval/SKILL.md`
- `/home/ubuntu/scipts/repos/sglang/.claude/skills/sglang-bisect-ci-regression/SKILL.md`
**内容**: SKILL.md 概述, pr-workflow.md (PR/commit 规范), ci-failure-triage.md (CI 调查/bisect)

### 5. agents/ (6 files)
**源文件**:
- `/home/ubuntu/scipts/repos/AReaL/.claude/agents/fsdp-engine-expert.md`
- `/home/ubuntu/scipts/repos/AReaL/.claude/agents/megatron-engine-expert.md`
- `/home/ubuntu/scipts/repos/AReaL/.claude/agents/algorithm-expert.md`
- `/home/ubuntu/scipts/repos/AReaL/.claude/agents/launcher-scheduler-expert.md`
- `/home/ubuntu/scipts/repos/AReaL/.claude/agents/code-verifier.md`
- `/home/ubuntu/scipts/repos/TensorRT-LLM/.claude/agents/ad-debug-agent.md`
**内容**: 泛化为通用 AI 基础设施专家角色，去除 repo 特定引用

### 6. commands/ (4 files)
**源文件**:
- `/home/ubuntu/scipts/repos/AReaL/.claude/commands/gen-commit-msg.md`
- `/home/ubuntu/scipts/repos/AReaL/.claude/commands/review-pr.md`
- `/home/ubuntu/scipts/repos/AReaL/.claude/commands/create-pr.md`
- `/home/ubuntu/scipts/repos/torchtitan/.claude/skills/torch_bisect/SKILL.md`
**内容**: commit (Conventional Commits), review-pr, create-pr, bisect

### 7. knowledge/ (7 files)
**源文件**:
- `/home/ubuntu/scipts/repos/flash-attention/AI/DEBUG_2CTA.md`
- `/home/ubuntu/scipts/repos/flash-attention/AI/RACECHECK_TMA_HAZARD.md`
- `/home/ubuntu/scipts/repos/flash-attention/AI/SASS_MMA_ANALYSIS.md`
- `/home/ubuntu/scipts/repos/flash-attention/AI/SM90_BLOCK_SIZE_TUNING.md`
- `/home/ubuntu/scipts/repos/AReaL/.claude/skills/debug-distributed/SKILL.md`
- `/home/ubuntu/scipts/repos/nccl/CONTRIBUTING.md`
**内容**: 内核调试知识, 分布式训练调试, NCCL 模式, SGLang CI 系统概述, FlashInfer JIT 架构

### 8. CLAUDE.md (顶层)
**内容**: 项目概述, 目标框架, 目录结构, 6 skill 分类说明, 使用方式, 来源标注

## 通用规则

所有文件遵循:
- 去除 repo 特定的文件路径，保留通用模式
- 用 `<!-- source: repo -->` 标注内容来源
- SKILL.md 文件使用 YAML frontmatter
- 保持实用性和可操作性

## 权限注意事项

**重要**: 新会话启动后，先执行 `! claude config set dangerousMode true` 或在弹出 Write 权限提示时按 **Shift+Tab** 选择 "Yes, allow all edits during this session"，否则每次写文件都会弹确认。

全局 settings 在 `/home/ubuntu/.claude/settings.json`，项目 settings 在 `/home/ubuntu/scipts/verl-grounding/.claude/settings.json`，两者都已配置 Write allow 但子 Agent 可能不完全继承。
