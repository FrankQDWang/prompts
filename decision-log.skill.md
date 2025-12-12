下面给你一套从 0 到 1 的完整方案：包含 ADR 模板、Codex Skill 的创建、以及自动在每次 git commit 时触发（由 skill 自己判断是否“重大变更”，是才生成/更新文档）。

目标：每次提交都跑一次，但不重大就 NO-OP；重大就自动生成 docs/adr/ 下的新 ADR、更新 docs/decisions.md 索引、必要时更新 CHANGELOG.md，并把这些文件自动 git add 进本次提交。

⸻

1) 在仓库里先把文档骨架建好（含 ADR 模板）

在你的 repo 根目录执行：

```
mkdir -p docs/adr
cat > docs/adr/ADR_TEMPLATE.md <<'EOF'
# ADR-____: <标题>

Date: YYYY-MM-DD
Status: Proposed | Accepted | Superseded | Deprecated

## Context（背景/问题）
- 现象/痛点：
- 业务目标（可验证的指标）：
- 约束（时间/人力/兼容/合规/性能/成本）：

## Decision（决策）
- 我们决定：
- 不做/不选的选项（以及原因）：

## Alternatives（备选方案）
1) 方案A：优点 / 缺点 / 风险 / 成本
2) 方案B：优点 / 缺点 / 风险 / 成本

## Implementation（实施要点）
- 关键改动点（链接到关键文件/模块/PR/commit）：
- 发布策略（灰度/feature flag/回滚条件）：
- 观测指标（上线后看什么）：

## Consequences（影响/代价）
- 正面收益：
- 负面代价（复杂度/维护成本/锁定/性能/可靠性）：

## Outcome（结果回填，上线后补）
Review Date: YYYY-MM-DD
- 指标结果（目标 vs 实际）：
- 意外副作用/事故：
- 下一步（继续/修正/回滚/写 Superseded ADR）：
EOF

cat > docs/decisions.md <<'EOF'
# Decisions Index

> One-line summaries of major decisions, linking to ADRs.

EOF

cat > CHANGELOG.md <<'EOF'
# Changelog

All notable changes will be documented in this file.
(Format inspired by Keep a Changelog.)

## [Unreleased]
### Added
### Changed
### Fixed
EOF
```

ADR 的核心结构（title/status/context/decision/consequences）是业界常用的 Nygard 风格；模板站点也给了这种结构的标准形式。 ￼
CHANGELOG.md 的“只记录 notable changes”思路来自 Keep a Changelog。 ￼

⸻

2) 创建你的 Codex Skill（第一次搭建）

Codex Skills 的约定是：在 ~/.codex/skills/**/SKILL.md 放一个带 YAML frontmatter 的文件（至少 name、description），启动时会先加载名字与简介，需要时再读正文。 ￼

执行：

```
mkdir -p ~/.codex/skills/decision-log
cat > ~/.codex/skills/decision-log/SKILL.md <<'EOF'
---
name: decision-log
description: On each commit, decide if staged changes are a major change; if yes, create ADR + update decisions index + update changelog, then stage doc files.
---

# decision-log

## Major change rubric (classify staged diff)
Treat as MAJOR if any applies:
- API/contract/schema changes or migrations
- architecture boundary / dependency swap
- business rule / metric definition changes
- perf/cost/reliability tradeoffs
- anything likely to be questioned later ("why did we do this?")

## Output rules
If NOT major:
- Print exactly: NO-ADR
- Do not modify files.

If major:
1) Create a new ADR copied from docs/adr/ADR_TEMPLATE.md, fill it, and save as:
   docs/adr/NNNN-<kebab-title>.md
   - NNNN is next available number (4 digits).
2) Append one line to docs/decisions.md:
   - YYYY-MM-DD | <short decision summary> | [ADR NNNN](docs/adr/NNNN-<kebab-title>.md)
3) If user-facing changes exist, update CHANGELOG.md under [Unreleased] with concise bullets.
4) Stage doc outputs:
   git add docs/adr docs/decisions.md CHANGELOG.md

## Guardrails
- Never modify non-doc code files.
- Keep ADR minimal; focus on Decision + Alternatives + Consequences + Outcome placeholder.
EOF
```

⸻

3) 配置 Git：把 hooks 放进仓库并启用

Git hook 机制：pre-commit 会在真正创建 commit 之前运行；返回非 0 会中止提交，也可以用 --no-verify 跳过。 ￼
hooks 默认在 $GIT_DIR/hooks，但可以用 core.hooksPath 指向 repo 内目录，便于版本控制。 ￼

在 repo 根目录执行：

```
mkdir -p .githooks
git config --local core.hooksPath .githooks
chmod -R +x .githooks
```

⸻

4) 写 pre-commit：每次提交自动跑 codex skill（自动判断 + 自动生成）

新建 .githooks/pre-commit：

```
cat > .githooks/pre-commit <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

# 1) 无 staged 变更就退出
if git diff --cached --quiet; then
  exit 0
fi

# 2) codex 不存在就放行（避免卡住你提交）
if ! command -v codex >/dev/null 2>&1; then
  echo "[decision-log] codex not found; skipping"
  exit 0
fi

# 3) 触发 codex exec（非交互），让 skill 自己判断是否 major
# --full-auto 是 CLI 的无人值守快捷方式（减少人工批准阻塞） [oai_citation:5‡OpenAI开发者](https://developers.openai.com/codex/cli/reference/)
# --enable skills 强制打开 skills feature flag（如果你的版本默认没开） [oai_citation:6‡OpenAI开发者](https://developers.openai.com/codex/cli/reference/)
prompt=$(cat <<'PROMPT'
Use the "decision-log" skill.

Read ONLY the staged diff: `git diff --cached`.
Apply the skill rubric to decide MAJOR vs NOT major.

If NOT major: print exactly "NO-ADR" and do nothing.

If MAJOR:
- Create docs/adr/NNNN-<kebab-title>.md based on docs/adr/ADR_TEMPLATE.md and fill it.
- Update docs/decisions.md and CHANGELOG.md (only if user-facing).
- Stage doc outputs: git add docs/adr docs/decisions.md CHANGELOG.md
Constraints:
- Do not modify non-doc code files.
PROMPT
)

# 这里不强依赖输出解析：skill 会自己 git add；我们再兜底 add 一次
codex exec --cd . --full-auto --enable skills "$prompt" || true
git add docs/adr docs/decisions.md CHANGELOG.md 2>/dev/null || true

exit 0
EOF

chmod +x .githooks/pre-commit
```

说明：pre-commit 脚本必须是可执行文件，否则 Git 会忽略。 ￼

⸻

5) 你问“pre-commit 还需要改吗？”——需要，而且我已经按“从零模板化”改好了

相对你之前设想，我这里做了三点关键调整：
	1.	明确只读 staged diff：用 git diff --cached，确保“提交的内容是什么”与“决策记录依据”一致。
	2.	把模板路径写死：让 skill 先复制 docs/adr/ADR_TEMPLATE.md 再填内容，避免生成格式漂移。
	3.	强约束只改 docs：防止 Codex 在 hook 里顺手动你代码（这是自动化里最常见的坑）。

⸻

6) 使用方式（你现在就能用）

照常开发、照常：

```
git add -A
git commit -m "xxx"
```

	•	如果不重大：什么都不会发生（或只打印 NO-ADR）。
	•	如果重大：本次 commit 会自动包含 ADR / decisions / changelog 的更新。
	•	紧急情况下跳过：git commit --no-verify -m "xxx"（Git 官方说明 pre-commit 可这样 bypass）。 ￼

⸻

如果你愿意，我还能把“重大变更判定”做得更稳一点：例如让 skill 根据变更触及的路径（db/migrations/、api/、proto/、auth/）或关键字（schema、migration、breaking）加权判断，减少“误判导致写太多 ADR”。这不需要你补充信息，我也能直接给你一个更强的 rubric 版本。
