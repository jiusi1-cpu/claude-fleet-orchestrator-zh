---
name: "Claude舰队调度器"
description: "项目级、默认 DryRun 的 Claude worker 调度 skill，带任务包校验、路径门禁和可审计 watcher 流程。"
---

# Claude Fleet Orchestrator

## Skill Name

`Claude舰队调度器`

## Scope

This is a project-local skill, not a global skill.

- keep it inside the current project under `.superpowers/skills/claude-fleet-orchestrator/`
- do not install it into `~/.agents/skills`
- do not run it outside a project that has `.project-fingerprint.json`

## Core Flow

1. Codex reads project context and project fingerprint.
2. Codex prepares one JSON task packet per worker.
3. Codex calls `scripts/run_claude_worker.ps1`.
4. The worker script performs project, path, and task-packet validation.
5. Default mode is DryRun, which writes only planning artifacts.
6. Codex uses `scripts/watch_claude_fleet.ps1` to inspect run status.
7. Codex reads `output.json` only after terminal states appear.

## Default DryRun

This skill defaults to DryRun.

- it does not start Claude by default
- it does not consume tokens by default
- it writes `prompt.txt`, `dryrun_prompt.txt`, `dryrun_plan.json`, `state.json`, and `output.json`
- it marks state as `dryrun_completed`

## Execute Mode

Execute mode is high risk and must be opt-in.

- real execution requires explicit `-Execute`
- real execution still goes through all guardrails
- it is blocked on `main` and `master` unless `-AllowMainWorktree` is explicitly set
- it must not use `--dangerously-skip-permissions`
- bypass permissions requires explicit `-AllowBypassPermissions`

Do not enter Execute mode until DryRun validation passes and a human confirms the isolated branch or worktree plan.

## Codex Role

Codex is the only orchestrator and reviewer.

Codex must:

- generate small task packets
- validate task scope before dispatch
- launch the worker script
- launch the watcher
- review `output.json`
- review changed files and violations
- decide whether any next step is allowed

## Claude Worker Role

Claude workers are bounded executors, not coordinators.

Workers must:

- handle exactly one task packet
- remain inside `allowed_files`
- avoid all `forbidden_files`
- never merge, rebase, or push
- never use API keys
- never start long-running background processes
- return structured JSON only

## Task Packet JSON Format

Primary format:

```json
{
  "task_id": "",
  "task_name": "",
  "role": "",
  "worktree_name": "",
  "branch_name": "",
  "objective": "",
  "read_first": [],
  "allowed_files": [],
  "forbidden_files": [],
  "commands_to_run": [],
  "acceptance_criteria": [],
  "output_requirements": [],
  "stop_conditions": [],
  "max_minutes": 30,
  "allow_network": false,
  "allow_api_keys": false,
  "allow_long_running_process": false,
  "allow_file_write": false
}
```

Markdown templates are human-only guidance. JSON is the machine-validated source of truth.

## state.json Format

```json
{
  "run_id": "",
  "worker_name": "",
  "task_packet": "",
  "status": "pending|running|dryrun_completed|completed|failed|stuck|invalid|blocked",
  "mode": "dryrun|execute",
  "started_at": "",
  "updated_at": "",
  "exit_code": null,
  "error": ""
}
```

## output.json Format

```json
{
  "run_id": "",
  "worker_name": "",
  "status": "",
  "mode": "dryrun|execute",
  "summary": "",
  "changed_files": [],
  "allowed_files": [],
  "forbidden_files": [],
  "violations": [],
  "tests_run": [],
  "test_result": "",
  "risks": [],
  "diff_summary": "",
  "commit_message": "",
  "raw_output_path": "",
  "prompt_path": "",
  "stdout_path": "",
  "stderr_path": "",
  "created_at": "",
  "updated_at": ""
}
```

## Watcher Mechanism

The watcher is read-only.

- it never starts workers
- it never modifies business code
- it never merges anything
- it can run once with `-Once`
- it can validate `output.json` with `-ValidateOutput`
- it can flag `completed_but_invalid_output`
- it can flag `stale_running`

## Security Gates

Before any run:

- verify `.project-fingerprint.json`
- verify required project directories
- verify `RunId` and `WorkerName`
- verify run paths stay under `agents/runs`
- reject duplicate `RunId` unless `-ForceNewRun`
- validate JSON task packet
- reject network, API keys, and long-running process flags

Before any Execute run:

- verify git repository
- verify current branch is not `main` or `master`
- require isolated `worktree_name` and `branch_name`
- record pre and post git snapshots
- validate changed files against `allowed_files` and `forbidden_files`

## Forbidden Actions

- do not start Claude by default
- do not merge `main`
- do not run on `main` unless a human explicitly overrides
- do not bypass file boundaries
- do not write outside `agents/runs` for run artifacts
- do not treat incomplete output as `completed`
- do not copy this skill into global skill directories

## Multica Import Notes

Import only the project-local orchestration files and validate them first.

- run `scripts/validate_claude_fleet.ps1`
- run `scripts/test_claude_fleet_dryrun.ps1`
- first import must remain DryRun-only
- first imported task must be `demo-readonly-audit.json`

## When Real Claude Testing Is Allowed

Only after:

- project fingerprint passes
- DryRun validation passes
- watcher validation passes
- human review approves an isolated git branch or worktree

## When Business-Code Testing Is Forbidden

Do not enter business-code testing when any of these are true:

- DryRun validation has not passed
- the project is on `main` or `master`
- task packet has broad or missing file boundaries
- network or API-key permissions are requested
- output validation is incomplete
