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
- **The rules live in the system prompt** (`--append-system-prompt-file`), so they survive
  compaction; the task is the first message.
- **Never `--strict-mcp-config` on a session.** It drops the claude.ai connectors too, not only
  local servers (0 Slack tools with it, 27 without). The screen uses it on purpose.

## The wake screen

- Only people in `wakers` can wake the agent. Solo (only an owner and the bot so far), the owner's
  messages wake it directly. Once anyone else has written, every message is screened, the owner's
  too, since they may be answering a teammate.
- An explicit @mention of the bot always wakes it; a message that @mentions another person and not
  the bot never does.
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
  Slack ends a stream 10 minutes after it starts, so a new one starts after 9.
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
  its own, so nobody needs their own account. In a private channel the password is posted alone;
  in a public one only the address is, and `/preview-login` shows the login privately to people on
  the wake list. Controls shows link, user and password at the top.
- The preview runs the same start script as a developer would (so `.env` loads). A start script
  that puts its servers in sessions of their own is not stopped by closing its terminal, so
  `preview_stop` also kills what listens on the preview's two ports.
- An extension that calls the preview's API over `http://127.0.0.1` must send a Secure cookie by
  hand: a Python cookie jar never sends one back over http.
- Project-specific Live-tab parts are extensions outside the repo (2026-11-02): the first one, a
  chart that knew an app's private API, was in the public code for 20 minutes.
- One preview at a time per `ui_origin`: Funnel serves it at the root; `/s` stays the listener's.

## Driving the Slack app for tests

- `set-value` on the composer fills it, but pressing Return does not send; click Send.
- The first click on a background Slack window is often lost; bring it to the front first. Slack
  does not scroll unless it is frontmost.
