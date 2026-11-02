# A T3 Code backend for the listener

The owner is moving his agent setup from Orca to [T3 Code](https://t3.codes) and wants this bot
to work there too (*"also slack agent should work"*). Today every task runs through Orca: a
worktree per task (`orca worktree create`), Claude started in an Orca terminal
(`orca terminal create --command claude …`), Slack messages typed into that terminal
(`terminal send --text … --enter`, `/compact`, Esc to interrupt), preview servers in more
terminals, and `worktree rm` / `terminal close` on cleanup.

## What T3 offers instead

T3 runs agents as **threads**, each in a project folder or its own git worktree, on a T3
server. A script reaches them through T3's MCP tools; a helper on the owner's machines,
`~/.agents/bin/t3-tool` (outside this repo: read it, do not copy it in), calls any tool:

```bash
t3-tool tools                                            # every tool and its parameters
t3-tool t3_thread_launch '{"projectId":…,"workspaceStrategy":{"type":"worktree","branch":…,"baseRef":"main","startFromOrigin":true},"modelSelection":{"instanceId":…,"model":…},"runtimeMode":"full-access","message":…}'
t3-tool t3_thread_send '{"threadId":…,"mode":"auto","message":…}'   # auto: start, steer a running turn, or queue
t3-tool t3_thread_interrupt '{"threadId":…}'
t3-tool t3_thread_read '{"threadId":…,"limit":20}'
t3-tool t3_thread_list '{"allProjects":true,"statuses":["running","waiting"]}'
t3-tool t3_thread_organize '{"threadId":…,"action":"archive"}'
```

All were tested against a real server on 2026-11-02. Launching without `message` creates the
worktree and an idle thread. There is no MCP tool to open a raw terminal or delete a thread.

## The task

Add a backend setting (`"backend": "orca"` default, or `"t3"`) so the same listener runs its
tasks as T3 threads, with the Orca path unchanged:

- A task: a T3 thread in its own worktree from the newest `origin/main`, on the configured
  provider instance and model, full access. The thread's agent must get what the Orca path gives
  Claude: the system prompt, the task, the `PreToolUse` Bash gate (`slack-agent gate`) and
  `SLACK_AGENT_CONV`. Find out how T3's Claude provider loads settings and instructions (a
  worktree's `.claude/settings.local.json` and `CLAUDE.local.md` are likely; verify, do not
  assume) and pick the way that really reaches the agent.
- Messages: `t3_thread_send`; `/compact` as a message; the interrupt control as
  `t3_thread_interrupt`; "the Claude tab is gone" becomes "the thread is gone or failed".
- Preview servers: started without Orca terminals (for example a transient
  `systemd-run --user` unit on Linux, a detached process elsewhere) and stopped on cleanup.
- Cleanup: the repo's own cleanup, archive the thread, remove the worktree and branch.
- `watch_work` and anything else that reads Orca state reads T3 state on this backend.
- New keys go in `DEFAULTS`, `config.example.json` and the README table (AGENTS.md rule); a
  NOTES.md section records what you learned about T3.

## Test it

Against the T3 server on this machine (the desktop app; `t3-tool envs` shows it), without
Slack: a throwaway script outside the repo that drives the listener's own task functions with a
stub Slack client, in a scratch git repo you add as a T3 project (`t3 project add <dir>`):
start a task, forward a message mid-turn, compact, interrupt, start and stop a preview, close.
Check from T3 (`t3_thread_read`) that the agent saw the system prompt and that the gate ran.
Afterwards archive the threads, remove the scratch project and repo. Use the Claude instance
`claude-personal` and model `claude-haiku-5-5` for the tests.

## How to finish

- Work in this worktree; scope is this repo only, one file (`slack-agent`) plus its docs. Do not
  touch `~/.agents` or any running listener; the coordinator switches a live bot after review.
- Checks: AGENTS.md's (`ast.parse`, `--help` with a config) and your test script's run.
- Commit everything as ONE Conventional Commit on your branch; never push; never amend a
  reported commit. Your final reply names the SHA, what the test run showed, and anything that
  does not reach parity with the Orca backend and why.
