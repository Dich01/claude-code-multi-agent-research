# Technical Report: Building Reliable Multi-Agent Claude Code Plugins

## Overview
This report documents an empirical investigation into the enforcement mechanisms and orchestration patterns required to build a 19-agent autonomous software development lifecycle (SDLC) plugin within the **Claude Code** environment. 

The research identifies critical discrepancies between official documentation and CLI behavior, specifically regarding tool-blocking hooks and agent hierarchy limitations.

## Key Research Findings

* **Reliable Enforcement**: The only deterministic way to block a tool call is using `exit code 2` in `PreToolUse` hooks.
* **Documentation Discrepancies**: Findings show `permissionDecision "deny"` is frequently ignored by the LLM, contradicting official documentation.
* **Orchestration Constraints**: Subagents are strictly prohibited from spawning other subagents. 
* **Orchestrator Mode**: To enable a "Brain" orchestrator, the plugin must be initialized as the main session agent using the mandatory namespace format: `--agent Plugin_name:brain`.
* **Silent Failures**: The `--plugin-dir` flag fails to load `settings.json` and `CLAUDE.md`, necessitating custom bootstrap logic.

## Technical Architecture
The investigated system employs a **hub-and-spoke** model with a centralized orchestrator managing 19 specialized markdown agents and 13 skills.

### Defense-in-Depth Enforcement
* **Flow Order**: Ensures the Orchestrator initiates all sessions and registers the session state.
* **Prerequisite Gates**: Blocks specialist agents until required artifacts (OpenAPI specs, TDD logs) are detected.
* **Read-Only Protection**: Prevents agents from modifying the plugin's core logic during execution via directory path comparison.
* **Recursion Guards**: Implements temporary flag files (`/tmp/hook-guard-$$`) to prevent infinite loops during hook propagation.

## Empirical Validation: Express to NestJS Migration
The patterns described in this report were validated through a complete framework migration of a multi-tenant production project:
* **Agents Spawned**: 11 unique agents coordinated by the central Brain.
* **Paralellism**: Confirmed 3-way parallel execution (Architecture-Guardian, Test-Architect, Isolation-Tester).
* **Reliability**: 422 total tests with 93% coverage and zero regressions upon completion.

## Reproduction Command
To activate the orchestrator mode identified in this research:

```bash
claude --plugin-dir /path/to/Plugin_name --agent Plugin_name:brain
