# Notes: why it is built this way

What was tried, what Slack allows, and what broke on the way. Dates are when it was found.
Built 2026-10-29 and run first for one repo in one company workspace.

## Design decisions

- **Subscription, not API.** Every session is Claude Code on the owner's own subscription
  (`agent_command`), with the MCP servers and connectors of that account. The screen uses the same
  command with `-p --model haiku`.
- **The agent decides; code shows it facts.** Where to answer (thread or code channel), which
  buttons to offer, whether an unclear request is for it: all the agent's call. Code never fills a
  decision in. A wrong decision is fixed with better facts or a clearer general instruction in the
  system prompt, not with a rule in code.
- **Agent-agnostic where it can be.** The Code tab, the PR link and the question buttons work from
  the listener, so another agent CLI gets them. Only the approval gate is a Claude Code hook: only
  the agent itself can stop a command before it runs.
- **Nobody reads the terminal.** The system prompt says so, and to end each turn with the one word
  "sent". Before that, the agent wrote every Slack reply twice: in the helper call and again as its
  terminal text (twice the output tokens and the context).
- **Questions go through Slack.** Sessions start with `--disallowedTools=AskUserQuestion`: that tool
  shows a picker in the terminal, where nobody sees it. `slack-agent ask <conv> "A" "B"` posts one
  button per option (up to 10); a click from someone on the wake list replaces the buttons with
  "<name> chose A" and reaches the agent as `[slack] <name> chose "A" in answer to your question
  "…"`, the question included, so the answer is clear many turns later. Write the flag with `=`:
  with a space it also takes the next words (the prompt) as tool names.
- **The gate asks only for what cannot be undone** (2026-11-04). In its first week one bot asked
  17 times and the owner approved 15. Only 4 asks were real: PR merges and closes. The rest were
  `--force-with-lease` pushes while rebasing the owner's PR stack, `rm -rf` on build output or
  `/tmp`, `git branch -D` on local temporary branches, and text in heredocs (a doc being written
  that mentioned `git reset --hard` or `gh pr merge`). Also, the message cut the command at 500
  characters, so the part that set off the gate was often out of sight, and the stack's real merge
  went through `gh api …/merge-async`, which a regex on `gh pr merge` never saw. So the gate splits
  the command into simple commands with `shlex` (heredoc bodies dropped, quoted words kept whole,
  `bash -c`/`eval`/`$(…)` looked into, `cd` followed for `rm` paths), names what is risky, shows
  the agent's Bash description as the reason, offers "Approve all like this in this task"
  (`approved_kinds` in the conversation's state), and gates PR merges and closes and DELETE calls
  through `gh api` too.
- **The rules live in the system prompt** (`--append-system-prompt-file`), so they survive
  compaction; the task is the first message.
- **The Slack connector is a per-bot choice** (2026-11-04, `slack_connector`). It acts as the
  owner: it reads every channel and DM the owner can, and posts as them. Where that is too wide, a
  session starts with a deny rule on the connector's server, `mcp__claude_ai_Slack`, in the
  `--settings` file (Orca) or `.claude/settings.local.json` (T3). Tested with
  `--dangerously-skip-permissions`: with the rule the session has no Slack tools at all
  (`ToolSearch` finds none), without it `slack_search_channels` answers. The session then reads
  Slack with `slack-agent read`, as the bot, which the listener's own digest also points to.
- **The bot token is the session's real Slack reach, not `channels`.** The helpers (`say`, `read`,
  `gate`, …) run inside the agent's session and read the bot token from the Keychain or the
  secrets file themselves, so a session can read the token and call Slack with every scope the bot
  has. `channels` stops mistakes and a misled agent that uses the helpers; it is not a wall
  against one that goes around them. A wall needs the session under another OS user or in a
  container without the token, with Slack reached only through the listener.
- **Never `--strict-mcp-config` on a session.** It drops the claude.ai connectors too, not only
  local servers (0 Slack tools with it, 27 without). The screen uses it on purpose.

## The wake screen

- Only people in `wakers` can wake the agent. Solo (only an owner and the bot so far), the owner's
  messages wake it directly. Once anyone else has written, every message is screened, the owner's
  too, since they may be answering a teammate.
- An owner's @mention of the bot always wakes it. Every other message, also one that @mentions a
  person and not the bot, goes to the screen, which sees mentions as `@Name`. Until 2026-11-03 code
  dropped any message tagging a person: "@<teammate> will merge it, give me the app link" never
  reached the agent. The screen now answers `agent` when any part of a message asks the agent
  for something.
- An owner's message is never held: the screen only tells whether it is addressed to the agent.
  An owner may ask the agent for anything, its own rules included.
- The screen is Claude Haiku with Claude Code's prompt replaced (`--system-prompt … --tools ""
  --strict-mcp-config --setting-sources "" --disable-slash-commands`): about 590 tokens, 0.9 s,
  $0.00012 a call, against about 31,600 tokens and 2.7 s for a plain `claude -p --model haiku`.
  No local model (RAM) and no new vendor (data leaves for their servers).
- Measured 2026-10-29 on 11 cases (secrets, prompt injection, push to main, `curl | bash`, someone's
  DMs, a cover letter, another repo, a pizza place, small talk, two real requests): five runs scored
  9, 10, 9, 11 and 11 of 11. The misses all kept an ambiguous teammate message with no @mention as
  context. Print every miss when you measure it; a summary once said "4 of 55, all safe" and was
  wrong.
- `maybe` (2026-10-30): a teammate's request ("fix the docs' consistency with the spec") in a code channel reached the
  agent behind a "Meanwhile, not addressed to you:" prefix on the same line, and the agent took the
  whole line as context. Now the message comes first and the context follows in brackets
  (`[context only, said before it and not to you: …]`), and a project request that could be for a
  person is forwarded as `(maybe not to you)`: the agent takes it, or says it did not and asks.
- Replies to the agent count (2026-11-02): "don't spam me here, write when you finish", in slang
  and with no project words, was screened `other` and the agent kept posting. The screen now
  answers `agent` for an instruction on how the agent works or reports.
- On wake, the agent gets what it missed from Slack's own history since the last message it saw,
  so downtime loses nothing. A woken message gets a 👂 reaction.

## Events and Socket Mode

- A tag in a channel comes as two events, `message` and `app_mention`, in either order. Only the
  event the bot acts on marks the message seen. Until 2026-11-02 the plain `message` came first,
  marked it seen and was ignored, and the `app_mention` was dropped as a repeat: tags in a team
  channel did nothing, silently.
- A tag with no words was empty after the mention was stripped and was dropped without a log line.
  It now starts a task from the thread's first message.
- A message posted from an app ends with `*Sent using* <app>`; the bot drops that footer. Match
  commands by prefix.
- After the laptop sleeps, the SDK's own reconnect can stall for good: the process lives, holds no
  socket and logs nothing (2026-10-30). The listener checks `client.is_connected()` every 30 s and
  restarts itself after 90 s down (`socket down` in the log). Slack then sends the missed events
  again, minutes late: two messages sent while it was deaf each started a session. Check by hand
  with `lsof -nP -a -p <python pid> -iTCP`: one ESTABLISHED connection to port 443.
- Processes started from a shell tool have no controlling terminal; closing a listener's tab did
  not always stop it (an old listener kept running old code and the new one exited on the lock).
  Kill by PID.
- `code_channel_action` is not a subscribable event (manifest validation refuses it); context bar
  clicks come over interactivity.
- A slash command's reply put in the Socket Mode acknowledgement shows nothing in a code channel;
  the listener posts it with `chat.postEphemeral` ("Only visible to you").

## What Slack allows (tested in the Slack app)

- **Code channels:** `agents.conversations.create` from the request; at most 5 tabs; the context
  bar is one row of at most 5 items that Slack cuts to fit (no second row), so labels are short.
  Per-channel slash commands with `agents.conversations.setCommands`; Slack built-ins (`/status`)
  cannot be used.
- **Session threads:** `agents.sessions.setStatus` with `thread_ts` gives "Working…" and a Stop
  button (`agent_session_stopped`, which sends Escape to the agent). `processing` falls back to
  `active` after an hour; the listener renews it.
- **Thinking steps:** `chat.startStream` with `task_display_mode: plan`, then `chat.appendStream`;
  Slack ends a stream 10 minutes after it starts, so a new one starts after 9. A step still `in_progress`
  when the stream ends shows "Something went wrong" with a warning sign, and each update's
  `details` is appended to the step's earlier details, not put in their place (2026-11-02: an
  agent left a "CI" step open to wait for CI in the background, and its details came out glued
  together). The system prompt says: no open step at the end of a turn, details once.
- **A tab whose file is gone cannot be removed** (2026-11-03): an agent deleted a canvas file
  instead of removing its tab, and the tab (listed twice by `listViews`, once under the canvas
  file's id) stayed as an empty tab; `removeView` answers `view_not_found` for both ids. Remove
  the tab first (`slack-agent tab --remove` does, then deletes the canvas); an orphan needs a
  person in the Slack app.
- **HTML tabs:** inline scripts run; scripts and images load from origins in
  `csp.resource_domains`; `fetch()` is blocked, so live data comes as a script (JSONP); iframes stay
  empty; links do not open. To open a site, use a Block Kit button with a `url` or a context bar
  item with a `url`.
- **Copying:** Slack's Copy text copies a whole message, not a code span, so a password goes in a
  message of its own.
- **Agent view** in the DM (`features.agent_view`, suggested prompts, `assistant:write`) cannot be
  switched off again.
- **The bot's handle** is fixed when the app is created. To change it, make the app again from the
  manifest (`apps.manifest.create`). Scopes and events change with `apps.manifest.update` and a
  reinstall.
- **The bot photo** is the app icon; no API sets it.

## Preview

- One shared account per preview (a random password): the app's first user, with a database of
  its own, so nobody needs their own account. With `test_ui` on, the link and login are only in
  the Controls tab (2026-11-03: posted in the chat too, and repeated by the agent in a bar item and
  a tab of its own, one preview showed in three places); in a public channel Controls says to run
  `/preview-login`, which shows the password privately to people on the wake list. Without
  `test_ui`, the helper posts the address, and in a private channel the password alone.
- The agent starts a preview itself whenever its change can be run or seen in the app (anything
  but docs, comments, tests or CI alone), without being asked (2026-11-03: the logo PR got none).
- The preview runs the same start script as a developer would (so `.env` loads). A start script
  that puts its servers in sessions of their own is not stopped by closing its terminal, so
  `preview_stop` also kills what listens on the preview's two ports.
- An extension that calls the preview's API over `http://127.0.0.1` must send a Secure cookie by
  hand: a Python cookie jar never sends one back over http.
- Project-specific Live-tab parts are extensions outside the repo (2026-11-02): the first one, a
  chart that knew an app's private API, was in the public code for 20 minutes.
- One preview at a time per `ui_origin`: Funnel serves it at the root; `/s` stays the listener's.
- A displaced preview is not silent (2026-11-03: a channel's Controls lost its link when another
  channel started a preview, with nothing saying why). The displaced channel's Controls says who
  took the address, and its preview starts again when that one stops (the task ends, or
  `preview-stop`).

## T3 Code backend

Found 2026-11-02 against T3 Code 0.1.6 (desktop app), by reading its bundled server and with a
scratch repo and Claude Haiku threads.

- **How T3 starts Claude.** Through the Claude Agent SDK, with the `claude_code` system prompt
  preset plus T3's own text appended, and no `--setting-sources` flag, so the CLI's default
  sources load: user (the instance's home, `CLAUDE_CONFIG_DIR`), project and local. The
  environment is the T3 server's own plus `CLAUDE_CONFIG_DIR`: nothing per thread. An instance's
  `launchArgs` (`--append-system-prompt-file`, `--settings`) are per instance, not per thread, so
  they cannot carry one task's conversation.
- **So a task's setup lives in its worktree.** `.claude/settings.local.json` there gives the agent
  the gate hook and, through its `env` key, `SLACK_AGENT_CONV` for its Bash commands and hooks;
  `CLAUDE.local.md` gives it the Slack rules as project instructions, which Claude Code reads
  again after a compaction (tested: after `/compact` the agent still answered through the helper,
  with the right conversation). `CLAUDE.local.md` showed in `git status` (`settings.local.json` is
  in many people's global ignore, not all), and a dirty workspace is kept at cleanup, so both go
  into the repo's `.git/info/exclude`.
- **A worktree session also loads the main checkout's `.claude/settings.local.json`.** With a
  hook in both, both ran. Write task files into the worktree only, never into `repo`.
- **The instance's own user settings apply.** The owner's profile blocked a bare `sleep 25`
  (a hook of theirs), so the agent ran it in the background and ended the turn; a test that needs
  a long foreground turn uses `python3 -c 'import time; time.sleep(30)'`.
- **`t3_thread_launch` answers before the worktree exists.** `worktreePath` appears in
  `t3_thread_read` a moment later; the listener waits up to a minute for it.
- **Messages.** `t3_thread_send` with `mode: auto` put a message sent mid-turn into the running
  turn (the same run), and the agent acted on it after its command finished. `/compact` as a
  message compacts (a `compaction` item). `t3_thread_interrupt` stops the running turn (the run
  ends `interrupted`, the Bash command too). A run whose provider failed shows as status
  `failed`; a new message starts a new turn on the same session, so the listener only relaunches
  when the thread is gone or archived (`t3_thread_read` answers `thread_not_found`).
- **No MCP tool removes a worktree, deletes a thread or opens a terminal.** Cleanup archives the
  thread and runs `git worktree remove` and `git branch -D` itself; previews run outside T3.

## Driving the Slack app for tests

- `set-value` on the composer fills it, but pressing Return does not send; click Send.
- The first click on a background Slack window is often lost; bring it to the front first. Slack
  does not scroll unless it is frontmost.
