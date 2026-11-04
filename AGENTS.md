# AGENTS.md

Rules for agents changing this repo.

- One script, `slack-agent`, run with `uv run --script` (its dependencies are declared inline).
  Keep it one file.
- No company, person or project data in the repo: no Slack IDs, tokens, hostnames, repo names or
  people's names. They belong in a bot's config (`config.example.json` shows the keys; README has
  the table). A new setting gets a key in `DEFAULTS`, the example config and the README table.
- The agent decides; code shows it facts. Fix a wrong agent decision with better facts or a
  clearer general line in `SYSTEM`, never with a rule in code or a line for one person.
- Record what you learn about Slack or the listener in `NOTES.md`, in the section it belongs to.
- Check a change: `python3 -c "import ast; ast.parse(open('slack-agent').read())"`, then
  `SLACK_AGENT_CONFIG=<a config> ./slack-agent --help`. For the listener, restart it (README,
  Running it) and watch its log while you try the change in Slack.
- Commit to `main` and push it to GitHub without asking, once the checks above pass and the diff
  holds no private data (the repo is public). Never leave a commit only in a local clone.
- A running listener keeps its old code until it is restarted. Kill it by PID, in a command of
  its own.
