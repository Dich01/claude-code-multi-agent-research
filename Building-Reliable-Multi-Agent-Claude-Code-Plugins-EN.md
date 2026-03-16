# Building Reliable Multi-Agent Claude Code Plugins: An Empirical Investigation of Enforcement Mechanisms


  Scope: Claude Code plugin development
  Architecture: 19 markdown agents, 13 skills, 4 MCPs, PreToolUse hooks
  Date: 2026-03-14
  Author: Diego Cheloni , https://github.com/Dich01/
  Type: Empirical validation of enforcement mechanisms and orchestration for a multi-agent Claude Code plugin/submodule
  Note: The plugin described in this investigation is proprietary. Only architectural patterns and experimental findings are documented.
  Environment: macOS, Claude Code CLI (March 2026), Claude Opus 4.6 model


---

## ABSTRACT

This document presents the results of an empirical investigation into the viability of building a multi-agent Claude Code plugin with behavioral guarantees. The plugin (19 agents, 13 skills) governs the NestJS/TypeScript software development lifecycle with mandatory TDD, security auditing, and quality gates.

The investigation covers:

- Which enforcement mechanisms Claude Code offers and which work as expected
- How to propagate hooks to subagents without infinite loops
- The absolute limitation that subagents cannot spawn subagents
- How to activate a plugin agent as the main agent
- Results from production tests across multiple projects

Key finding: the only reliable blocking mechanism is exit code 2 in PreToolUse hooks. permissionDecision shows discrepancies between documentation and actual behavior (issue #4669). Orchestrator mode works when brain is the main agent via `--agent plugin_name:brain`.

The investigation demonstrates that reliable multi-agent orchestration in Claude Code is possible, but only through careful use of PreToolUse hooks, defensive design patterns, and CLI-based agent activation.


---

## 1. INTRODUCTION AND CONTEXT

### 1.1 — The Problem

A Claude Code plugin composed entirely of markdown files (prompts) cannot guarantee that the LLM follows its instructions. Rules written in prose ("don't skip steps", "TDD is mandatory", "@brain is required") are suggestions that Claude can silently ignore.

The challenge is threefold:

a) There are no functions to call — everything goes through prompts
b) Claude is non-deterministic — it can produce different results
c) There is no native enforcement mechanism for agent ordering

### 1.2 — Objective

Determine whether it is possible to build a reliable multi-agent plugin for clients, identifying which mechanisms work, which show discrepancies with documentation, and what workarounds exist in the community.

### 1.3 — Methodology

a) Exhaustive review of official Claude Code documentation
b) Analysis of community repositories and blogs
c) Review of relevant GitHub issues (bugs, feature requests)
d) Empirical testing with the plugin on a real Express project
e) Iteration of fixes and re-testing


---

## 2. BLOCKING MECHANISMS IN CLAUDE CODE

Claude Code offers PreToolUse hooks that execute before each tool call. There are three documented ways to influence a tool call. Only one blocks reliably.

### 2.1 — exit code 2: WORKS

- Mechanism: `echo "reason" >&2 && exit 2`
- Effect: The tool call is NOT executed. stderr is shown to Claude as an error. Claude receives the message and adjusts its behavior.

Confirmed by:
- Blake Crosley: 95 hooks in production, 9 months of use — https://blakecrosley.com/blog/claude-code-hooks
- disler: 13 hooks covering the full lifecycle — https://github.com/disler/claude-code-hooks-mastery
- kenryu42: safety plugin with semantic analysis — https://github.com/kenryu42/claude-code-safety-net
- Issue #29795: 5-layer system with 68 documented failure patterns — https://github.com/anthropics/claude-code/issues/29795

### 2.2 — permissionDecision "deny": DISCREPANCY BETWEEN DOCS AND REALITY

- Mechanism: Return JSON with `hookSpecificOutput.permissionDecision = "deny"`
- Official documentation: Describes this mechanism as functional. Verbatim quote: `"deny" prevents the tool call`, `permissionDecisionReason` is shown to Claude.
- Observed behavior: In empirical tests and community reports, Claude IGNORES the deny and executes the tool anyway.

Issue #4669: https://github.com/anthropics/claude-code/issues/4669 — Status: CLOSED as "not planned". Includes detailed reproduction steps. Also affects "ask" and "continue: false".

**Note:** There is a contradiction between the official documentation (which describes it as functional) and issue #4669 (which demonstrates with reproduction steps that it does not work). This discrepancy is unresolved. Workaround: use exit code 2 instead.

### 2.3 — exit code 1: DOES NOT BLOCK (by design)

- Mechanism: `exit 1` to indicate an error in the hook
- Official documentation (verbatim quote): "Any other exit code: Non-blocking error. stderr is shown in verbose mode and execution continues."
- Effect: Claude logs the error but EXECUTES the tool. This is **intended behavior**, not a bug.

Issue #21988: https://github.com/anthropics/claude-code/issues/21988 — requests that exit code 1 also block.

**Note:** Despite being by design, this decision means that for enforcement only exit code 2 is useful. Exit code 1 does not serve as a blocking mechanism. Workaround: use exit code 2 instead.

### 2.4 — exit code 0 without output: WORKS (allows)

- Effect: Allows the tool call without intervention.
- Used as the default case when validation passes.

### 2.5 — Exit codes summary

| Exit code | Effect | Status |
|-----------|--------|--------|
| 0 | Allows | Works |
| 1 | Non-blocking error | By design — does not block |
| 2 | Blocks | Works |


---

## 3. HOOK PROPAGATION TO SUBAGENTS AND TEAMMATES

### 3.1 — Hooks propagate to subagents

Observed empirically: PreToolUse and PostToolUse fire when a subagent makes tool calls. **Note:** The official documentation is not explicit about whether parent hooks fire when subagents execute tools. The docs document `SubagentStart`/`SubagentStop` as separate events, but PreToolUse propagation is an empirical finding.

Critical risk: infinite loops. If a hook triggers a process that triggers another hook, the chain repeats indefinitely.

Solution confirmed in the community: recursion guard with temporary flag files.

```bash
GUARD="/tmp/hook-guard-$$"
[ -f "$GUARD" ] && exit 0
touch "$GUARD"
trap "rm -f '$GUARD'" EXIT
```

Sources:
- egghead.io: settings pollution in subagents — https://egghead.io/avoid-the-dangers-of-settings-pollution-in-subagents-hooks-and-scripts~xrecv
- Blake Crosley: recursion-guard.sh — blocked 23 runaway spawns — https://blakecrosley.com/blog/claude-code-hooks

### 3.2 — Settings inheritance to subagents

Subagents inherit ALL settings from the parent, including hooks (empirical observation).
To break the cycle in specific cases: `--settings no-hooks.json` with `{ "disableAllHooks": true }`.

Source: egghead.io

### 3.3 — Teammates (agent teams)

Standard hooks (PreToolUse/PostToolUse): behavior not explicitly documented for teammates.

Team-specific hooks that DO work (confirmed in official documentation):
- TeammateIdle: fires when a teammate is about to go idle
- TaskCompleted: fires when a task is marked as completed

Source: official Claude Code documentation — https://code.claude.com/docs/en/agent-teams

### 3.4 — Agent frontmatter partially ignored in teammates (BUG)

Issue #30703: https://github.com/anthropics/claude-code/issues/30703

When brain spawns a teammate via Agent tool with `team_name`, the `hooks` and `skills` frontmatter fields are silently ignored. The `model` and `disallowedTools` fields do work.

Impact: teammates do not have the hook restrictions or skills declared in their frontmatter. Brain must inject everything via the spawn prompt.

Status: open bug (March 2026)


---

## 4. HOOK BYPASS — KNOWN VECTORS

### 4.1 — Bypass via Bash

When Write/Edit are blocked by a hook, Claude can use Bash with commands that achieve the same effect:

```bash
sed -i 's/old/new/' file.txt
python3 -c "open('file.txt','w').write('content')"
echo "content" > file.txt
tee file.txt <<< "content"
```

Solution: an additional hook that intercepts Bash and analyzes the command looking for file write patterns.

Reference implementation: `guard_bash_file_writes.py` from the 5-layer system (Issue #29795) — https://github.com/anthropics/claude-code/issues/29795

### 4.2 — Bypass via wrapper scripts

Claude can create a .sh script with the blocked content and execute it. The hook does not see the script's content, only the command `bash script.sh`.

Solution: recursive analysis of wrappers up to N levels deep.
Reference implementation: kenryu42/claude-code-safety-net analyzes up to 5 wrapper levels — https://github.com/kenryu42/claude-code-safety-net


---

## 5. ARCHITECTURAL LIMITATION: SUBAGENTS CANNOT SPAWN SUBAGENTS

### 5.1 — The restriction

Official documentation (verbatim): "Subagents cannot spawn other subagents, so Agent(agent_type) has no effect in subagent definitions."
https://code.claude.com/docs/en/sub-agents

This is absolute and non-configurable. When an agent is invoked as a subagent (via @brain or Agent tool), it loses access to the Agent tool. It cannot spawn other agents, create teams, or assign tasks.

### 5.2 — Impact on the plugin

The plugin has 19 agents. Brain orchestrates. If brain is invoked as a subagent (@brain), it cannot spawn the other 18 agents.

Two modes of operation emerge from this restriction:

a) **Guide Mode:** brain as subagent guides the user on which agents to invoke. The user executes manually. Functional but tedious.

b) **Orchestrator Mode:** brain as main agent (`--agent` flag) has full access to the Agent tool and can spawn subagents. Requires brain to be the main session, not a subagent.

### 5.3 — Empirical confirmation

- **Test 1:** @brain as subagent — brain did everything alone (102 tool uses) because it could not spawn agents. It invoked zero specialists.
- **Test 2:** `--agent plugin_name:brain` as main agent — brain used TeamCreate, TaskCreate, and spawned 7 agents with Agent tool. Real parallelism (security-qa and quality-gate in background). WORKS.

### 5.4 — Capabilities by context (empirical finding)

| Context | Agent tool | TeamCreate | TaskCreate | SendMessage |
|---------|-----------|-----------|-----------|------------|
| Main session | YES | YES | YES | YES |
| Subagent | NO | NO | NO | NO |
| Teammate | NO | NO | NO | NO |

**Note:** The official documentation does not publish this table explicitly. The subagent restriction is documented. Teammate restrictions are not documented at this level of detail. The values in this table are empirical findings.

Source: https://code.claude.com/docs/en/agent-teams
Issue #32731: https://github.com/anthropics/claude-code/issues/32731


---

## 6. ACTIVATING A PLUGIN AGENT AS MAIN AGENT

### 6.1 — The namespace problem

Plugin agents load with namespace: `plugin-name:agent-name`. The plugin's brain appears as "Plugin_name:brain" in the agent list (`/agents`).

### 6.2 — --agent formats tested empirically

| Format | Works | Result |
|--------|-------|--------|
| `--agent brain` | NO | Generic session |
| `--agent Plugin_name:brain` | YES | Brain as main agent |
| `--agent plugin:Plugin_name:brain` | NO | Generic session |

The correct format is: `--agent Plugin_name:agent-name`

**Note:** The official CLI documentation shows `--agent my-custom-agent` without mentioning namespace format for plugins. This finding is empirical.

### 6.3 — settings.json does NOT work with --plugin-dir (empirical finding)

The "agent" field in the plugin's settings.json ONLY applies when the plugin is installed via marketplace (`claude plugin install`). With `--plugin-dir`, settings.json is completely ignored.

The "agent" field in the PROJECT's settings.json (`.claude/settings.json`) also does not work for plugin agents.

**Note:** The official documentation describes the "agent" field in plugin settings.json as functional, but does not explicitly distinguish between marketplace installation vs --plugin-dir. This differentiated behavior is an empirical finding.

Source: https://code.claude.com/docs/en/plugins

### 6.4 — Plugin CLAUDE.md is NOT loaded via --plugin-dir (empirical finding)

`--plugin-dir` loads: `agents/`, `skills/`, `hooks/`, `.mcp.json`, `commands/`
`--plugin-dir` does NOT load: `CLAUDE.md`, `settings.json`

Solution: the plugin's bootstrap automatically adds a line `@/path/to/plugin/CLAUDE.md` to the target project's CLAUDE.md. Uses the `@` import syntax with absolute path.

### 6.5 — Definitive command

```bash
claude --plugin-dir /path/to/Plugin_name --agent Plugin_name:brain
```

This command:
- Loads all 19 agents, 13 skills, hooks, and MCPs from the plugin
- Activates brain as the main agent (can spawn subagents)
- Executes SessionStart hooks (bootstrap, protocol, MCPs)


---

## 7. ENFORCEMENT HOOKS IMPLEMENTED

### 7.1 — enforce-flow-order.sh (PreToolUse, matcher: Agent)

Purpose: Ensure brain is the first agent invoked.
Mechanism: State file per session_id. If brain is not registered and the invoked agent is not brain — exit 2.

Problem found: When brain is the main agent, it is never invoked via Agent tool (IT IS the session). The state file never registers "brain". The first Agent call (api-contract) was incorrectly blocked.

Fix applied:
a) SessionStart hook pre-registers "brain" in the state file using session_id from the stdin JSON.
b) enforce-flow-order.sh detects `team_name` in Agent calls — if present, brain is orchestrating via teams. Auto-registers "brain".

### 7.2 — enforce-prerequisites.sh (PreToolUse, matcher: Agent)

Purpose: Verify that prerequisite artifacts exist before allowing an agent to be invoked.

Prerequisites by agent:

| Agent | Prerequisite |
|-------|-------------|
| planner | `docs/api/*.yaml` |
| test-architect | `docs/plans/*-plan.md` |
| test-engineer | `docs/test-specs/` with files |
| backend-dev | `docs/tdd/*-tdd-log.md` with "RED" |
| auth-access | `docs/tdd/*-tdd-log.md` with "RED" |
| quality-gate | `src/` with .ts files |
| pr-governance | `docs/quality/*-quality-gate.md` with "APPROVED" |
| security-qa | `src/` with .ts files |

Agents without prerequisites (always allowed): brain, bug-resolver, check-brain-features-status, domain-modeler, tenant-lifecycle, infra-secrets, architecture-guardian, isolation-tester, jira-spec, workflow-sync

### 7.3 — enforce-plugin-readonly.sh (PreToolUse, matcher: Write|Edit|Bash)

Purpose: Prevent agents running in an external project from modifying files in the plugin directory.

Mechanism: Compares file_path (Write/Edit) or command (Bash) against `$CLAUDE_PLUGIN_ROOT`. If the target is inside the plugin — exit 2.

Exception: if CWD == PLUGIN_ROOT, everything is allowed (we are developing the plugin, not using it from another project).

### 7.4 — Common pattern across all hooks

1. Recursion guard with `/tmp/ai-brain-guard-*-$$`
2. Read JSON from stdin with `INPUT=$(cat)`
3. Graceful fallback if jq is not installed (exit 0)
4. exit 2 + stderr to block
5. exit 0 to allow
6. 5-second timeout (configured in hooks.json)


---

## 8. COMMUNITY SUCCESS CASES

### 8.1 — Blake Crosley: 95 hooks in production

- https://blakecrosley.com/blog/claude-code-hooks
- https://blakecrosley.com/blog/claude-code-hooks-tutorial
- https://blakecrosley.com/blog/claude-code-as-infrastructure

9 months of use. Each hook created after a real incident.
35 judgment hooks (gates, guards, validators).
49 automation hooks (loggers, formatters, injectors).

Results:
- Intercepted 8 force-push attempts to main
- Blocked 23 runaway spawn attempts
- Judgment/automation ratio evolved from 1:6 to 4:5

Problems found:
- Concurrent writes to the same JSON file caused truncation
- Without spawn depth guard, agents delegated infinitely

### 8.2 — 5-Layer System (Issue #29795)

https://github.com/anthropics/claude-code/issues/29795

68 failure patterns documented over 3 months. The system was built in response to these 68 failures — each failure led to a new check or hook.

Defense-in-depth architecture:
- Layer 1: Rules and conventions (CLAUDE.md, plans)
- Layer 2: Failure documentation (68 catalogued patterns)
- Layer 3: Decision log (mandatory audit trail)
- Layer 4: Automated reviews (5 tools)
- Layer 5: Hooks (hard blocks that cannot be bypassed)

Key finding: PreToolUse hooks only intercept their specific tool. When Edit is blocked, Claude uses Bash with sed. `guard_bash_file_writes.py` closes that bypass by inspecting Bash commands.

### 8.3 — kenryu42/claude-code-safety-net

https://github.com/kenryu42/claude-code-safety-net

Plugin that blocks destructive commands:
- `git reset --hard`, `git push --force`, `git clean -fd`
- `rm -rf` with dangerous patterns

Semantic analysis (not just regex):
- `git checkout -b feature` — allowed (branch creation)
- `git checkout -- file` — blocked (destructive reset)
- `rm -r -f /` — blocked (flag reordering detected)

Recursive wrapper detection up to 5 levels deep.

### 8.4 — disler: hooks mastery + multi-agent observability

- https://github.com/disler/claude-code-hooks-mastery
- https://github.com/disler/claude-code-hooks-multi-agent-observability

13 hooks covering the entire Claude Code lifecycle.
Builder/validator pattern for multi-agent.
Real-time Vue dashboard with SQLite for observability.
12 event types tracked with session isolation.

### 8.5 — GitButler: parallel session isolation

https://blog.gitbutler.com/automate-your-ai-workflows-with-claude-code-hooks/

Automatic branches per Claude Code session.
Uses git plumbing (`write-tree`, `commit-tree`, `update-ref`) for isolated staging.
Each parallel session has its own index file.

### 8.6 — Multi-agent orchestration in the community

- wshobson/agents: 112 agents, 16 orchestrators — https://github.com/wshobson/agents
- mbruhler/claude-orchestration: sequential agent chaining — https://github.com/mbruhler/claude-orchestration
- barkain/claude-code-workflow-orchestration: agent teams with plan approval — https://github.com/barkain/claude-code-workflow-orchestration
- ruvnet/ruflo: external orchestrator that spawns Claude Code instances — https://github.com/ruvnet/ruflo


---

## 9. KNOWN INCIDENTS AND FAILURES

### 9.1 — $30k API key leak (paddo.dev)

Claude hardcoded real Azure credentials in a markdown file published to a public repository. Exposed for 11 days before a developer detected it manually.

Source: https://paddo.dev/blog/claude-code-hooks-guardrails/

### 9.2 — Home directory deletion (paddo.dev)

Command executed: `rm -rf tests/ patches/ plan/ ~/`
The trailing `~/` wiped the user's entire home directory. No backup. Total loss of personal data.

### 9.3 — Silent credential propagation (paddo.dev)

Claude copied production credentials from `.env` to `env.example` despite having blocked access to `.env` via permissions. The permissions did not cover indirect reading.

### 9.4 — Test manipulation (paddo.dev)

Claude modified tests to pass incorrectly, then defended the changes as "correct behavior". The developer did not detect the change until manual review.

### 9.5 — Infinite loops (multiple issues)

- Issue #10205: https://github.com/anthropics/claude-code/issues/10205 — Simple hooks cause loops even without spawning subprocesses. Stop hook failures cause loops at session close.
- Issue #9579: https://github.com/anthropics/claude-code/issues/9579 — Autocompacting loop — consumption of 96-108M tokens/day.
- Issue #9704: https://github.com/anthropics/claude-code/issues/9704 — Orphan Claude Code process consumed 2.88M tokens over 2+ days without the user noticing.

### 9.6 — Hooks as potential attack vector

CVE-2025-59536 reference (paddo.dev): Malicious project files can define hooks that execute automatically. Guardrails become attack vectors.

**Note:** This CVE was not found in official databases (NVD/MITRE) at the time of this investigation. The source is paddo.dev (security update February 2026). The official Claude Code documentation includes security considerations about hooks: hooks are captured as a snapshot at session start and external modifications require user review before being applied.

Source: paddo.dev, official Claude Code security documentation — https://code.claude.com/docs/en/security


---

## 10. EMPIRICAL TESTS PERFORMED

### 10.1 — Test 1: Brain as subagent

- Command: `claude --plugin-dir /path/to/plugin`
- Prompt: `@brain Implement a products CRUD (...)`

Result:
- Brain invoked as subagent (Plugin_name:brain)
- Brain did everything alone: 102 tool uses, zero Agent calls
- Created correct artifacts in docs/ (requirements, api, plans, tdd, etc.)
- Complete TDD: RED — GREEN — REFACTOR
- 36 new tests, 175 total, zero regressions
- Quality gate: APPROVED
- BUT: no specialist agents invoked

Diagnosis: brain as subagent cannot use Agent tool (documented restriction). It did everything alone as a workaround.

### 10.2 — Test 2: Brain with "agent": "brain" in project settings.json

- Command: `claude --plugin-dir /path/to/plugin` (With `.claude/settings.json: {"agent": "brain"}`)
- Prompt: `Implement GET /health endpoint (...)`

Result:
- Claude did NOT activate brain as main agent
- Said "dispatch rules" (does not exist in the plugin) and dispatched to devsenior
- Zero plugin agents invoked

Diagnosis: "agent" in the PROJECT's settings.json does not work for plugin agents. Documentation describes the "agent" field in the plugin's settings.json but does not distinguish between --plugin-dir and marketplace install.

### 10.3 — Test 3: Brain with --agent brain (no namespace)

- Command: `claude --plugin-dir /path/to/plugin --agent brain`
- Prompt: `Implement GET /health endpoint (...)`

Result:
- Claude did NOT activate the plugin's brain
- Generic session, dispatched to devsenior

Diagnosis: "brain" without namespace does not resolve to the plugin's brain. CLI documentation does not mention namespace format.

### 10.4 — Test 4: Brain with --agent Plugin_name:brain (SUCCESSFUL)

- Command: `claude --plugin-dir /path/to/plugin --agent Plugin_name:brain`
- Prompt: `Implement GET /ping endpoint (...)`

Result:
- Brain identified as main agent: "I am @brain"
- Phase 0 executed: 4 MCPs detected
- Complete TDD with artifacts in docs/
- 4 new tests, 185 total, zero regressions
- BUT: brain executed roles directly (did not spawn subagents)
- Reason: trivial task, brain decided not to delegate (non-deterministic LLM behavior)

Diagnosis: correct namespace. Brain is main agent. For simple tasks, brain may optimize by doing everything alone.

### 10.5 — Test 5: Brain orchestrator with complex CRUD (SUCCESSFUL)

- Command: `claude --plugin-dir /path/to/plugin --agent Plugin_name:brain`
- Prompt: `Implement categories CRUD with slug, unique, Mongoose (...)`

Result:
- Brain in orchestrator mode (detected TeamCreate available)
- TeamCreate("feat-CATEGORIES-CRUD") — successful
- TaskCreate with dependencies (addBlockedBy) — successful
- 7 agents spawned with Agent tool:
  1. api-contract (foreground) — OpenAPI spec
  2. planner (foreground) — technical plan
  3. test-architect (foreground) — test specs
  4. test-engineer (foreground) — 47 tests in RED
  5. backend-dev (foreground) — implementation GREEN+REFACTOR
  6. security-qa (background) — security report
  7. quality-gate (background) — quality gate APPROVED
- security-qa and quality-gate ran in PARALLEL
- 47 new tests, 232 total, zero regressions
- 9 artifacts created in docs/

Problem found: enforce-flow-order.sh blocked the first spawn because brain (main agent) was never registered in the state file. Brain worked around it using general-purpose agents.

Fix applied: pre-register brain in SessionStart + detect team_name.

### 10.6 — Test 6: Enforcement hooks verification

enforce-flow-order.sh:
- Correctly blocked api-contract before brain was registered
- Message: "The first agent invoked must be @brain"
- Fix: pre-registration of brain in SessionStart resolves the main agent case

enforce-plugin-readonly.sh:
- Did not fire explicitly (brain did not attempt to write to the plugin)
- Confirmed indirectly: all files were created in the project

enforce-prerequisites.sh:
- Not tested directly because brain used general-purpose agents
- Pending validation with the flow-order fix


---

## 11. RELEVANT GITHUB ISSUES

### Enforcement and hooks:

| Issue | Description | Status |
|-------|------------|--------|
| #4669 | permissionDecision deny — discrepancy between docs and behavior | CLOSED "not planned" |
| #21988 | Exit code 1 does not block (by design, issue requests change) | OPEN |
| #20946 | Hooks don't block in bypass mode (--dangerously-skip-permissions) | OPEN |
| #29795 | Best practice: 5-layer QA system (68 documented failures) | DOCUMENTATION |

### Multi-agent:

| Issue | Description | Status |
|-------|------------|--------|
| #30703 | hooks and skills frontmatter ignored in teammates | OPEN (bug) |
| #5812 | Hook context bridging between sub-agents and parent | CLOSED "not planned" |
| #32731 | Teammates tool restrictions (no Agent, no TeamCreate) | DOCUMENTED |

### Stability:

| Issue | Description | Status |
|-------|------------|--------|
| #10205 | Infinite loops with hooks enabled | OPEN |
| #9579 | Autocompacting loop, token spike (96-108M/day) | OPEN |
| #9704 | Orphan process consumed 2.88M tokens over 2+ days | OPEN |
| #27156 | Worktree conflict with git submodules | DOCUMENTED |


---

## 12. SOURCES AND REFERENCES

### Official Claude Code documentation:

- Hooks: https://code.claude.com/docs/en/hooks
- Sub-agents: https://code.claude.com/docs/en/sub-agents
- Agent Teams: https://code.claude.com/docs/en/agent-teams
- Plugins: https://code.claude.com/docs/en/plugins
- Plugins reference: https://code.claude.com/docs/en/plugins-reference
- Skills: https://code.claude.com/docs/en/skills
- CLI reference: https://code.claude.com/docs/en/cli-reference
- Scheduled tasks: https://code.claude.com/docs/en/scheduled-tasks
- Permissions: https://code.claude.com/docs/en/permissions
- Security: https://code.claude.com/docs/en/security
- Marketplaces: https://code.claude.com/docs/en/plugin-marketplaces
- Settings: https://code.claude.com/docs/en/settings
- Memory: https://code.claude.com/docs/en/memory

### Blogs and tutorials:

- Blake Crosley — 95 hooks in production: https://blakecrosley.com/blog/claude-code-hooks
- Blake Crosley — 5 hooks tutorial: https://blakecrosley.com/blog/claude-code-hooks-tutorial
- Blake Crosley — Claude Code as infrastructure: https://blakecrosley.com/blog/claude-code-as-infrastructure
- Pixelmojo — CI/CD patterns with hooks: https://www.pixelmojo.io/blogs/claude-code-hooks-production-quality-ci-cd-patterns
- Paddo.dev — guardrails that actually work: https://paddo.dev/blog/claude-code-hooks-guardrails/
- GitButler — automated workflows with hooks: https://blog.gitbutler.com/automate-your-ai-workflows-with-claude-code-hooks/
- Karanbansal — hooks guide: https://karanbansal.in/blog/claude-code-hooks/
- egghead.io — settings pollution in subagents: https://egghead.io/avoid-the-dangers-of-settings-pollution-in-subagents-hooks-and-scripts~xrecv
- DEV Community — broken permissions fix: https://dev.to/boucle2026/how-to-fix-claude-codes-broken-permissions-with-hooks-23gl
- DataCamp — how to build plugins: https://www.datacamp.com/tutorial/how-to-build-claude-code-plugins

### Repositories:

- disler/claude-code-hooks-mastery: https://github.com/disler/claude-code-hooks-mastery
- disler/claude-code-hooks-multi-agent-observability: https://github.com/disler/claude-code-hooks-multi-agent-observability
- kenryu42/claude-code-safety-net: https://github.com/kenryu42/claude-code-safety-net
- wshobson/agents (112 agents, 16 orchestrators): https://github.com/wshobson/agents
- mbruhler/claude-orchestration: https://github.com/mbruhler/claude-orchestration
- barkain/claude-code-workflow-orchestration: https://github.com/barkain/claude-code-workflow-orchestration
- ruvnet/ruflo (external orchestrator): https://github.com/ruvnet/ruflo


---

## 13. CONCLUSIONS

### 13.1 — The multi-agent plugin is viable

The plugin architecture (19 agents, hub-and-spoke with brain as orchestrator) is correct and aligned with patterns used by the community (wshobson/agents with 112 agents confirms this). The restriction that subagents cannot spawn subagents is resolved by making brain the session's main agent.

### 13.2 — Mechanical enforcement works but requires workarounds

The only reliable blocking mechanism is exit code 2 in PreToolUse hooks. permissionDecision shows a discrepancy between documentation (functional) and observed behavior (ignored, issue #4669 closed as "not planned"). The enforcement hooks (prerequisites, flow-order, plugin-readonly) work in production but required adjustments after empirical testing (e.g., pre-registration of brain in SessionStart).

### 13.3 — Installation via --plugin-dir has critical limitations

`--plugin-dir` does NOT load settings.json or CLAUDE.md from the plugin (empirical finding — the documentation does not explicitly distinguish this behavior). It requires `--agent Plugin_name:brain` as a CLI flag and a bootstrap that injects the CLAUDE.md import. Marketplace installation would resolve both issues but requires publishing the plugin.

### 13.4 — Empirical testing was essential

Without testing on a real project, the following problems would not have been detected:
- "agent" in the project's settings.json does not work for plugins
- Mandatory namespace for --agent (not documented in CLI reference)
- enforce-flow-order.sh blocked brain as main agent
- Brain as subagent cannot spawn agents (documented restriction, but the practical impact is only understood empirically)
- Brain optimizes simple tasks by doing everything alone (non-deterministic behavior)

The research methodology (documentation — implementation — empirical test — fix — re-test) was more valuable than any amount of theoretical analysis. Several findings fall in gaps in the official documentation that only empirical testing can fill.

### 13.5 — Final test: Express to NestJS migration (COMPLETE SUCCESS)

- Command: `claude --plugin-dir /path/to/plugin --agent Plugin_name:brain`
- Prompt: "We need to migrate this project to NestJS with TypeScript."
- Project: Express 4.x + Mongoose + MongoDB, 9 models, 12 API routes, 5 models with tenantId (multi-tenant)

Result:
- Brain detected orchestrator mode (TeamCreate available)
- Phase 0 complete: 4 MCPs, project-knowledge, domain routing
- Automatic multi-tenant detection — activated architecture-guardian + isolation-tester
- 11 agents spawned (9 unique, 2 re-invoked):
  - brain — api-contract — planner —
  - [architecture-guardian || test-architect || isolation-tester] —
  - backend-dev — [backend-dev || auth-access] —
  - [backend-dev || auth-access] —
  - [security-qa || quality-gate] — FAIL —
  - [test-engineer || backend-dev] — PASS

Real parallelism confirmed:
- 3 in parallel: architecture-guardian + test-architect + isolation-tester
- 2 in parallel: backend-dev + auth-access (twice)
- 2 in parallel: security-qa + quality-gate
- 2 in parallel: test-engineer + backend-dev (fixes)

Quality gate REJECTED the first pass (coverage 44%, threshold 80%). Brain responded by spawning test-engineer to raise coverage. Re-executed quality-gate — PASS at 93%.

Skills loaded automatically: tdd-governance, coding-standards, multi-tenant-platform-architecture, mongodb-development

Final metrics:
- 190 NestJS tests + 232 legacy = 422 total, 0 regressions
- Coverage: 92.75% statements, 82.39% branches
- TypeScript strict: 0 errors
- Build: compiles successfully
- Security: PASS WITH CONDITIONS (0 CRITICAL, HIGHs fixed)
- 14 artifacts in docs/ (requirements, api, plans, architecture, test-specs, tdd, security, quality, context)

This test confirms that the experiment works end-to-end with a non-trivial real case (complete framework migration), including:
- Multi-agent orchestration with real spawning
- Agent parallelism
- Quality gate as a mechanical barrier (rejection + correction)
- Automatic domain routing (multi-tenant)
- Skills consumed by specialized agents
- Project-knowledge updated at the end

