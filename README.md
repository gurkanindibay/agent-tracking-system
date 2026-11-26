# Agent Tracking System

An Android app for keeping an eye on your coding agents (Claude Code, Codex, Antigravity, GitHub Copilot,
DeepSeek) from your phone:

- **Work**: which agent is working on which GitHub issue, the PR it opened and whether that PR's checks
  pass. Each item is marked **active**, **idle** or **stuck**: stuck means failing checks, or no activity
  for longer than the "stuck after" hours you set (default 6).
- **Usage**: tokens used in the last 5 hours, 24 hours and 7 days, how much quota is left and when it
  resets, plus recent sessions, per provider.
- **Start new session**: on an idle or stuck item, start a fresh agent. Claude, Codex and DeepSeek run in
  GitHub Actions. Copilot gets the issue assigned to its coding agent. The new run takes over the issue.

```
app/        Flutter Android app
collector/  Python script that runs on your computer and publishes token usage
relaunch/   GitHub Actions workflow + claim tool to copy into the repository your agents work in
```

## How it knows who works on what

Agents mark an issue they take with two labels, which `relaunch/issue_claim.py` manages:

```
in progress
claim:<harness>/<session-id>     e.g. claim:claude/0193a7f2c4e1
```

The PR that delivers the issue carries the same two labels. The app reads them through the GitHub API,
and also lists open PRs opened by the Copilot coding agent. Have your agents claim issues with:

```bash
python3 scripts/issue_claim.py claim 123 --harness claude --session <id>
python3 scripts/issue_claim.py sync 123 --pr 130      # when the PR is opened
python3 scripts/issue_claim.py release 123 --harness claude --session <id> --pr 130
```

(for example, add these lines to your `AGENTS.md` / `CLAUDE.md`).

## Where the usage numbers come from

| Provider | Source | What you get |
|---|---|---|
| Claude Code | `~/.claude/projects/**/*.jsonl` (collector) | tokens per 5h / 24h / 7d window and per session. Pass `--claude-5h-limit` / `--claude-7d-limit` for "% left" (plan limits are not in the transcripts) |
| Codex | `~/.codex/sessions/**/*.jsonl` (collector) | the plan's real 5h and weekly `used_percent`, reset times, tokens per session |
| Copilot | GitHub billing API (collector, via `gh`) | premium requests this month against `--copilot-allowance` (default 300) |
| DeepSeek | `api.deepseek.com/user/balance` (phone or collector) | balance left (granted + topped up) |
| Antigravity | `~/.config/agent-monitor/antigravity.json` | only what you put there: Antigravity has no public quota API |

Claude and Codex usage lives on the computer the agents run on, so the collector runs there and publishes
a JSON snapshot to a **secret gist** that the app reads. Only project folder names leave your computer,
never full paths or conversation content.

## Setup

1. **Install the app.** Download the APK from the latest
   [release](../../releases) (or the `agent-tracking-apk` artifact of the latest CI run), open it on your
   phone and allow "install unknown apps" when asked.
2. **Run the collector on your computer** (Python 3.10+, [`gh`](https://cli.github.com) logged in with
   the `gist` scope):

   ```bash
   python3 collector/agent_usage_collector.py --create-gist      # prints the gist id once
   python3 collector/agent_usage_collector.py --publish-gist <id> --watch 300 \
       --claude-5h-limit 2000000                                 # optional, see the table
   ```

   Keep it running (cron, a login item, or a terminal tab). Optional: `export DEEPSEEK_API_KEY=...` first.
   `--json` prints the snapshot, `--help` lists every option.
3. **Create a GitHub token** (fine-grained, for the repositories you watch): Issues read/write,
   Pull requests read, Checks read, Actions read/write, Metadata read; account permission Gists read.
4. **In the app's Settings**, enter the token, the repositories (`owner/name`), the gist id, and
   optionally your DeepSeek key. Keys stay in Android's encrypted storage on the phone.
5. **To restart stuck work from the phone**, in each repository your agents work in:
   - copy `relaunch/agent-relaunch.yml` to `.github/workflows/agent-relaunch.yml` and
     `relaunch/issue_claim.py` to `scripts/issue_claim.py`, and commit them to the default branch
     (GitHub only starts workflows from there);
   - add the secrets for the agents you use: `CLAUDE_CODE_OAUTH_TOKEN` (from `claude setup-token`),
     `OPENAI_API_KEY` (Codex), `DEEPSEEK_API_KEY` (runs Claude Code against DeepSeek's
     Anthropic-compatible API);
   - for Copilot, turn on the Copilot coding agent for the repository.

Antigravity runs only in its desktop app, so it can't be restarted from the phone. Pick a cloud agent to
continue its issue instead.

## Development

```bash
cd app && flutter pub get && flutter analyze && flutter test && flutter build apk --release
python3 -m unittest discover -s collector/tests
```

CI (`.github/workflows/ci.yml`) runs the tests and builds the APK on every push. Push a tag like `v0.1.0`
to publish the APK as a release. Money values (the DeepSeek balance, Copilot request counts) are kept as
decimals end to end, never floating point.

## License

MIT
