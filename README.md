# slack-agent

A Slack bot that lets you give a coding agent work from Slack. You DM it or tag it; it makes a
fresh git workspace for the task, starts Claude Code there on your own Claude subscription, and
the agent answers, shows its diff and opens pull requests in Slack. Teammates in the
conversation can talk to the agent too; risky steps wait for the owner's click.

It is one Python script (`slack-agent`) plus one JSON config per bot. The code holds nothing
about a company or a person: who owns the bot, which repo it works on and how to run a preview
of the app all come from the config, so one checkout serves bots for several companies.

## How it works

```
Slack ──Socket Mode──▶ listener (slack-agent listen, one per config)
  DM / @mention           │  makes an Orca workspace (git worktree, own branch)
  code channel msgs       │  starts Claude Code in a terminal there, task as first message
  buttons, /commands      │  types later Slack messages into that terminal as "[slack] name: …"
                          ▼
                     Claude Code session ──▶ slack-agent say | ask | step | finish | …  ──▶ Slack
                          │                   (helpers; the agent never writes in the terminal)
                          └─ PreToolUse hook: slack-agent gate (risky shell commands wait for a click)
```

- **Listener.** `slack-agent listen` keeps a Socket Mode connection to Slack. A DM or a tag from
  an owner starts a task: it reacts 👀, makes an Orca workspace (a child worktree of the
  listener's own workspace, on its own branch from `origin/main`) and starts the agent in a
  terminal there. The task text is the agent's first message; the Slack rules go in its system
  prompt, so they survive compaction. A tag with no words means "take this thread": the task is
  the thread's first message.
- **Where it answers.** The agent decides. A one-message answer stays in the thread. Anything that
  changes files, runs tests or pushes moves to a Slack *code channel* (`slack-agent
  open-channel`), which has a Code tab with the diff, a context bar and per-channel slash commands
  (`/done /diff /pr /plan /progress /preview /preview-login`).
- **Talking to it.** Later messages in the thread or code channel reach the agent as one line,
  `[slack] <name>: <text>`, typed into its terminal. Only people on the wake list can wake it. In
  a conversation with more than the owner, each message goes through a short screen (Claude
  Haiku with Claude Code's own prompt stripped, about 590 tokens and 1 s a call). It answers
  `agent`, `maybe`, `other`, `unsafe: …` or `unrelated: …`. `other` is kept as context for the
  next woken message; `maybe` is forwarded marked "(maybe not to you)" and the agent takes it or
  asks; unsafe or off-project messages from non-owners are held with Pass it on / Drop buttons
  only an owner can use.
- **Replies.** The agent posts only through the helpers (`say`, `ask` with buttons, `step` for
  live thinking steps, `finish` with a summary). Nobody reads its terminal. The claude.ai Slack
  connector, if the agent has it, is for reading only: it posts as the owner. With
  `"slack_connector": false` the session has no connector and reads Slack with `slack-agent read`,
  as the bot, in the channels the bot works in (below, [Slack access](#slack-access)).
- **Approvals.** A Claude Code PreToolUse hook (`slack-agent gate`) stops risky shell commands
  (a plain force push, push to main, deleting a remote branch, merging or closing a pull request
  with `gh pr` or `gh api`, `gh release`, `gh repo delete`, `rm -r`/`rm -f` outside the task's
  workspace and the temp directory, `git reset --hard`, `git clean -f`, `git checkout --`) at
  Approve/Deny buttons; only an owner's click counts. The message names what is risky, shows the
  agent's reason (the Bash tool's description) and the whole command. "Approve all like this in
  this task" also lets later steps of the same kinds through in that task. Pushing branches other
  than main (with `--force-with-lease` too), opening pull requests, deleting local branches and
  deleting files in the workspace need no approval. The gate splits the command into simple
  commands first, so text in a heredoc or a quoted argument does not count.
- **Watcher.** Every 10 s the listener compares each code channel's workspace diff with its Code
  tab and updates it, and asks GitHub for the branch's open PR. This does not depend on the agent,
  so another agent CLI gets it too. Only the approval gate is Claude-specific.
- **Preview (optional).** `/preview` runs the task workspace's app on free ports with its own
  database in a PostgreSQL container, publishes it with Tailscale Funnel at `ui_origin`, and
  shows the address and a one-off login in the Controls tab (posts them when `test_ui` is off). The
  agent starts one itself for any change people can run or see. It stops, and the database is dropped, when the task
  ends.
- **Links.** `slack-agent link` puts any other link people open (a deployment the agent started,
  a page it hosts) in the Controls tab, with an optional note such as the login.
- **Finish.** `/done`, the Finish button or the agent's `finish` posts a summary with feedback
  buttons and archives the code channel. The workspace is removed when its branch has nothing
  unpushed; otherwise the bot says where it kept it.

- **Backends.** By default each task is an Orca workspace with Claude Code in an Orca terminal.
  With `"backend": "t3"` it is a [T3 Code](https://t3.codes) thread in its own worktree instead
  (below, [On T3 Code](#on-t3-code)); the rest is the same.

[NOTES.md](NOTES.md) has the design decisions, what Slack allows and what went wrong on the way.

## What you need

- macOS or Linux. On macOS the tokens live in the Keychain (the `security` CLI); on Linux in a
  private file (step 4). On Linux, Orca's CLI is `orca-ide` (`orca` there is often GNOME's screen
  reader); the script calls `orca-ide` when it exists.
- [Orca](https://github.com/stablyai/orca) with its `orca` CLI on `PATH`: workspaces and terminals.
  Or, with `"backend": "t3"`, a running T3 Code server and a `t3_tool` command (below).
- [Claude Code](https://claude.com/claude-code), signed in to the subscription the bot should use.
- [uv](https://docs.astral.sh/uv/) (the script declares its one dependency, `slack_sdk`, inline).
- `git` and `gh` (signed in) for the repo the bot works on.
- For previews only: Docker (or OrbStack) with a PostgreSQL container, and Tailscale with Funnel.
- A Slack workspace where you may install your own app.

## Install

1. **Get the code.** Clone it wherever you keep code; below, `<checkout>` is the folder it makes.
   ```sh
   git clone https://github.com/pawel-kotowski/slack-agent.git
   ```
2. **Make the Slack app.** At api.slack.com/apps → Create New App → From a manifest, paste
   [`manifest.example.json`](manifest.example.json) with your bot's name, descriptions and
   suggested prompts. The bot's handle is fixed when the app is created (renaming later changes
   only the display name), so pick the name first. Install it to the workspace.
   Check: the bot shows up under Apps in Slack.
3. **Make the tokens.** Basic Information → App-Level Tokens → Generate, scope
   `connections:write` (an `xapp-` token). OAuth & Permissions → Bot User OAuth Token (`xoxb-`).
4. **Store them in the Keychain** under one service name (`slack-agent` by default; use another
   per bot when you run several):
   ```sh
   security add-generic-password -U -s slack-agent -a bot -w 'xoxb-…'
   security add-generic-password -U -s slack-agent -a app -w 'xapp-…'
   ```
   Optional, for `slack-agent rotate` and manifest changes by API: the app configuration tokens as
   `config-access` and `config-refresh` (api.slack.com/apps → Your App Configuration Tokens).
   Check: `security find-generic-password -s slack-agent -a bot -w` prints the token.

   On Linux, put them in `~/.config/slack-agent/<service>.json` instead, readable by you only:
   ```sh
   install -D -m 600 /dev/null ~/.config/slack-agent/slack-agent.json
   echo '{"bot": "xoxb-…", "app": "xapp-…"}' > ~/.config/slack-agent/slack-agent.json
   ```
5. **Make the listener's workspace.** In Orca, add the repo and create a workspace for the
   listener, named for example `slack-agent`; below, `<listener workspace>` is its folder. Task
   workspaces are made as its children.
6. **Write the config.** Copy [`config.example.json`](config.example.json) to
   `~/.config/slack-agent/config.json` (or anywhere, and point `SLACK_AGENT_CONFIG` at it) and
   fill it in; the keys are in the table below. Slack user IDs are in each profile → ⋮ → Copy
   member ID.
   Check: `<checkout>/slack-agent --help` prints the help without "no config".
7. **Previews (optional).** Start a PostgreSQL container and store its password:
   ```sh
   docker run -d --name slack-agent-pg -e POSTGRES_PASSWORD=… -p 127.0.0.1:55432:5432 postgres:16
   security add-generic-password -U -s slack-agent -a postgres -w '…'
   ```
   Turn on Tailscale Funnel for the machine and set `ui_origin` to its address and port. Fill the
   `preview` block with the environment, setup and start command of your app.
   Leave `preview` as `null` to switch previews off (the slash commands go too).
8. **Start the listener** in a terminal of the listener's workspace:
   ```sh
   orca terminal create --worktree "path:<listener workspace>" \
     --title "slack-agent listener" --command '<checkout>/slack-agent listen'
   ```
   Check: the log (`<state_dir>/listener.log`) ends with `listening as U…`.
9. **Try it.** DM the bot "what does this repo do? no changes". It reacts 👀, answers in the
   thread and closes it. Then ask for a small change: it should open a code channel, show the diff
   in the Code tab, and stop at Approve/Deny if it tries something risky.

## Config

One JSON file per bot. `~` is expanded in paths and in the two command lists.

| Key | What it is |
|---|---|
| `bot_name` | The bot's name, as the agent and the screen call it. |
| `owners` | `{slack_user_id: name}`. Owners start tasks, finish them, approve risky steps and release held messages. The first owner is the one the agent works for. |
| `wakers` | `{slack_user_id: name}`. People whose messages may wake a running session (owners too). Everyone else's messages are context only. |
| `repo`, `repo_label`, `repo_url` | The repo tasks work on: local checkout, `org/name`, web address. |
| `project` | One line on what the project is. The screen holds requests that are not about it. |
| `project_notes` | Extra instructions for the agent about this project (setup, what not to start without asking). `{owner}` is replaced. |
| `workspace` | The listener's Orca workspace; task workspaces are its children. On T3, the listener's own checkout (never removed). |
| `agent_command` | How to start Claude Code, as a list: `["claude"]`, or a wrapper that picks an account. The screen uses it too. |
| `claude_args` | Arguments for every session. Default: `--dangerously-skip-permissions --disallowedTools=AskUserQuestion` (the session is unattended; questions go through `slack-agent ask`). |
| `usage_check` | Optional command; exit code 1 means the subscription is nearly spent, and no new task starts. |
| `claude_projects` | Claude Code's `projects` folder for that account (the Live tab reads session transcripts there). |
| `keychain_service` | Keychain service of the tokens. Default `slack-agent`. |
| `state_dir` | Sessions, log, lock. Default `~/.local/state/slack-agent/<config file name>`. |
| `preview` | `null`, or the preview block: `pg_container`, `pg_port`, `user`, `env` (variables for the app; `{user} {password} {db} {pg_port} {pg_password} {host} {branch} {api} {ui}` are filled in), `setup` (runs once per workspace), `setup_marker`, `command` (starts the app on `{api}` and `{ui}`). |
| `ui_origin` | The public address previews are served at (Tailscale Funnel). |
| `test_ui`, `ui_port` | Test feature: a Live HTML tab (the session's state, from Claude's transcripts) and a Controls tab per code channel, served by the listener on `ui_port` behind `ui_origin/s`. |
| `backend` | `orca` (default): a task is an Orca workspace and a terminal. `t3`: a T3 Code thread in its own worktree (below). |
| `t3_tool` | T3 only. The command that calls one T3 MCP tool: `<cmd> <tool> '<json args>'` prints the result as JSON, and on an error exits 1 with the error on stderr. Default `["t3-tool"]`. |
| `t3_project` | T3 only. The T3 project id of `repo`; when `null`, the project whose folder is `repo`. |
| `t3_instance`, `t3_model` | T3 only. The provider instance (a Claude instance in T3's settings, which picks the subscription) and model of every task thread. |
| `cleanup_command` | T3 only. Optional shell command run in a task's workspace before it is removed: the repo's own cleanup (stopping its containers, deleting test accounts). |
| `slack_connector` | `true` (default): task sessions keep the claude.ai Slack connector of the account, which reads everything the owner can read. `false`: sessions start with a deny rule on it (`mcp__claude_ai_Slack`) and read Slack with `slack-agent read`, as the bot. The screen never has it. |
| `channels` | Channel IDs the bot works in, or `null` (default) for every channel it is a member of. A tag elsewhere gets a short "not on my list" only the tagger sees, and `slack-agent read` refuses other channels. DMs and the code channels of the bot's own tasks are always allowed. |
| `extensions` | Python files outside this repo with project-specific additions (below). Any other key in the config is theirs to read. |

## Slack access

What the bot can read is its scopes plus the channels it is a member of; what a task's session
can read also depends on `slack_connector`.

| Scope | What the bot does with it | Without it |
|---|---|---|
| `channels:history`, `groups:history` | Gets every message in its channels (`message.channels`, `message.groups`), not only tags; reads the last messages for the screen, what it missed since it last woke, the first message of a thread it is tagged in, and `slack-agent read`. A bot token reads only channels it is a member of. | Only tags reach it: every reply in a task has to tag the bot, the screen has no context, and `read` fails. |
| `channels:read`, `groups:read` | `conversations.info`: whether a channel is a code channel Slack's own UI opened (and the request it came from), and whether a preview's channel is private. `channels:read` also lets it list the name and topic of every public channel, member or not. | Code channels the bot opened itself still work (they are in its state). A code channel opened from Slack's own UI is not picked up, and a preview's login is treated as in a public channel. |

The narrowest setup for one project: a private channel, only the `groups:*` pair of each (no
`channels:*`, so no public channel can be read or listed), that channel in `channels`, and
`"slack_connector": false`. Code channels are private (`is_private`, checked 2026-11-04), so the
`groups:*` pair covers them too. ("Groups" is Slack's old API name for private channels.)

## Extensions

Code that knows one project (its API, its data) does not go in this repo. Put it in a Python file
next to your config and list it in `extensions`. The listener loads each file and sets its `agent`
to the slack-agent module, so it can use `agent.CONFIG`, `agent.read_json`, `agent.update_state`,
`agent.publish_test_ui` and the rest. A file may define any of:

| Name | Called |
|---|---|
| `live_html(channel, token) -> str` | when the Live tab is built; the HTML is added at its end |
| `live_data(channel, name) -> str \| None` | for `ui_origin/s/<token>/<name>.js`; return a script (JSONP), or `None` if the name is not yours |
| `preview_started(web, channel)` | after a preview starts |
| `COMMANDS = {name: fn(web, channel, args)}` | `slack-agent <name> <channel> <args…>` |

Tell the agent about an extension's commands in `project_notes`.

## Several bots

A bot is a Slack app plus a config. For another company or project, make its app in that Slack
workspace, store its tokens under its own `keychain_service`, write its config, and start a second
listener with `SLACK_AGENT_CONFIG=<that config> slack-agent listen`. Each config gets its own state
folder and lock, so the listeners do not see each other. Two bots that serve previews need
different `ui_origin` ports and PostgreSQL containers.

## Running it

- **Always on (Linux):** run the listener as a systemd user service with `Restart=always`
  (`Environment=SLACK_AGENT_CONFIG=…`, `ExecStart=<path>/slack-agent listen`, `WantedBy=default.target`)
  and `loginctl enable-linger` so it starts at boot without a login. The lock keeps it to one.
- **Log:** `<state_dir>/listener.log`. Every tag is logged as `mention` when it arrives, so a tag
  that does nothing shows there. `wake` / `skip` lines say what the screen decided and why.
- **Restart:** list the listener's PIDs with `ps -axo pid,command | grep "[s]lack-agent listen"`
  and kill them in a command of its own (a longer command line that names the same words matches
  your own shell), then start it again as in step 8. Sessions survive a restart: the listener
  reads `sessions.json` and the agents keep running in their terminals.
- **After sleep:** the listener checks its Slack connection every 30 s and restarts itself after
  90 s without one. Slack sends missed events again after the reconnect, minutes late.
- **Stuck?** `kill -USR1 <listener python pid>` writes every thread's stack to
  `<state_dir>/stacks.txt`.
- **Tokens:** `slack-agent rotate` renews the app configuration token (it lasts 12 hours). The bot
  and app tokens do not expire.
- **Bot photo:** it is the app icon, set at api.slack.com/apps → your app → Basic Information.
  There is no API for it.

## On T3 Code

With `"backend": "t3"`, the listener runs each task as a T3 Code thread, through T3's MCP tools
(`t3_tool` calls them), instead of Orca:

- **A task** is a thread in a new worktree from the newest `origin/main` (branch named after the
  task), on `t3_instance` and `t3_model`, at full access. The listener makes it idle, writes the
  agent's files into the worktree, then sends the task as the first message.
- **The agent's setup** comes from two files in its worktree, because T3 gives the agent no
  per-thread command line or environment: `.claude/settings.local.json` (the `slack-agent gate`
  hook, `SLACK_AGENT_CONV`, `SLACK_AGENT_CONFIG`, and a deny rule for `AskUserQuestion`) and
  `CLAUDE.local.md` (the Slack rules). Both are added to the repo's `.git/info/exclude`.
  `claude_args` and `agent_command` do not apply to task threads (the screen still uses
  `agent_command`); set `claude_projects` to the instance's home's `projects` folder for the Live tab.
- **Messages** go in with `t3_thread_send` (`auto`: starts an idle thread or steers a running
  turn); Compact sends `/compact` after the running turn; Slack's stop button interrupts the turn.
  A message to a thread that is gone or archived starts a new thread in the same worktree that
  carries it. When a turn fails, the listener says so in Slack (checked every 30 s).
- **Previews** run as a transient `systemd-run --user` unit on Linux and as a detached process
  elsewhere, and stop with the task.
- **Cleanup** archives the thread, runs `cleanup_command`, and removes the worktree and its branch,
  when nothing is unpushed (as with Orca).
- Add the repo to T3 first (`t3 project add <repo>`). The listener does not need Orca.
