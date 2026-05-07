# Personal setup notes — OpenClaw on Azure Ubuntu VM + Telegram

End-to-end notes from a fresh Azure Ubuntu VM to a working Telegram bot backed by GitHub Copilot.

> These are personal install notes, not official docs. Official channel reference lives in [docs/channels/telegram.md](docs/channels/telegram.md). For the install matrix see [docs/install/](docs/install).

## 0. What "connecting" Copilot means

OpenClaw does **not** wrap or shell out to the GitHub Copilot CLI (`copilot` / `gh copilot`). The [extensions/github-copilot/](extensions/github-copilot) plugin runs its own OAuth device-code flow against `https://github.com/login/device` and calls Copilot's API directly. Token is stored under `~/.openclaw/`.

So a "local Copilot CLI" and an "OpenClaw on the VM" are two independent clients. They share only the GitHub account / Copilot subscription. You authenticate **per machine** inside OpenClaw.

## 1. VM toolchain

Node 22+ is required (per [AGENTS.md](AGENTS.md)).

```bash
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt-get install -y nodejs git build-essential
sudo npm i -g pnpm
node -v   # >= 22
```

`pnpm` is the package manager OpenClaw uses (monorepo with hundreds of workspace packages — `npm`/`yarn` won't cut it).

## 2. Clone + bootstrap

```bash
git clone https://github.com/openclaw/openclaw.git
cd openclaw
pnpm install
pnpm build
```

## 3. Make the CLI runnable

Easiest path — always invoke via the workspace:

```bash
pnpm openclaw --help
```

If you want a global `openclaw` command:

```bash
pnpm setup            # creates PNPM_HOME and adds it to PATH
source ~/.bashrc
pnpm link --global
openclaw --help
```

### Issue we hit

```
ERR_PNPM_NO_GLOBAL_BIN_DIR  Unable to find the global bin directory
```

Fix: run `pnpm setup`, then `source ~/.bashrc` so `PNPM_HOME` is in `PATH`. After that `pnpm link --global` works.

If `openclaw` is still not found after linking:

```bash
echo $PNPM_HOME
echo $PATH | tr ':' '\n' | grep -i pnpm
# if missing:
echo 'export PNPM_HOME="$HOME/.local/share/pnpm"' >> ~/.bashrc
echo 'export PATH="$PNPM_HOME:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

## 4. Log in to GitHub Copilot (model provider)

### Issue we hit

```
pnpm openclaw login github-copilot
# Unknown command: openclaw login. No built-in command or plugin CLI metadata owns "login".
```

There is no top-level `login` command. Provider auth lives under `models auth login`:

```bash
pnpm openclaw models auth login --provider github-copilot
```

This prints a code, e.g.:

```
URL:  https://github.com/login/device
Code: EBB6-5E91
```

Open the URL on any device, paste the code, approve. On success:

```
Updated ~/.openclaw/openclaw.json
Auth profile: github-copilot:github (github-copilot/token)
Default model available: github-copilot/claude-opus-4.7 (use --set-default to apply)
```

Pin it as default if you like:

```bash
pnpm openclaw models auth login --provider github-copilot --set-default
# or
pnpm openclaw models default set github-copilot/claude-opus-4.7
```

Useful related commands:

```bash
pnpm openclaw models list
pnpm openclaw models auth --help
```

## 5. Telegram bot

### 5a. Create the bot
On Telegram, message **@BotFather** → `/newbot` → save the token.

> If you ever paste/leak the token: `@BotFather` → `/revoke` → choose the bot → use the new token. Treat tokens like passwords.

### 5b. Get your numeric Telegram user ID
Message `@userinfobot` (or `@RawDataBot`) and copy the numeric `id`. Required for `allowFrom`.

### 5c. Configure on the VM

OpenClaw config lives at `~/.openclaw/openclaw.json` (or `config.json5`). It was created in step 4 by the login command. Edit it:

```bash
nano ~/.openclaw/openclaw.json
```

Add a `channels.telegram` block. Example using env-var token (recommended):

```json5
{
  channels: {
    telegram: {
      enabled: true,
      dmPolicy: "allowlist",
      allowFrom: ["<your-numeric-telegram-id>"],
      groups: { "*": { requireMention: true } },
    },
  },
}
```

```bash
echo 'export TELEGRAM_BOT_TOKEN="<new-bot-token>"' >> ~/.profile
source ~/.profile
chmod 600 ~/.openclaw/openclaw.json
```

Or put `botToken` directly in the config (still `chmod 600`):

```json5
channels: {
  telegram: {
    enabled: true,
    botToken: "<new-bot-token>",
    dmPolicy: "allowlist",
    allowFrom: ["<your-numeric-telegram-id>"],
  },
}
```

DM policies: `pairing` (default), `allowlist`, `open` (needs `allowFrom: ["*"]`), `disabled`. See [docs/channels/telegram.md](docs/channels/telegram.md) for full options.

### 5d. Sanity check + start gateway

```bash
pnpm openclaw doctor
pnpm openclaw doctor --fix     # if it flags safe repairs
pnpm openclaw gateway          # foreground; long polling, no inbound port needed
```

For a persistent service:

```bash
pnpm openclaw gateway restart --deep
pnpm openclaw gateway status --deep
```

Long polling means **no Azure NSG rule** is required — the VM only needs outbound HTTPS to `api.telegram.org` and `api.github.com`/`api.githubcopilot.com`.

### 5e. Test

DM your bot from your phone (e.g. `@Robollms_bot`). Because your numeric ID is in `allowFrom`, it will respond. Without it, use pairing instead:

```bash
pnpm openclaw pairing list telegram
pnpm openclaw pairing approve telegram <CODE>
```

Pairing codes expire after 1 hour.

### 5f. Groups
1. Add the bot to the group.
2. In `@BotFather`: `/setprivacy` → off (or make the bot a group admin) so it can see all messages.
3. Update `channels.telegram.groups` and `groupPolicy` to match your access model.

## 6. Quick reference

| Thing | Path / command |
|---|---|
| Config | `~/.openclaw/openclaw.json` |
| Auth profiles | `~/.openclaw/agents/<agentId>/agent/auth-profiles.json` |
| Channel creds | `~/.openclaw/credentials/` |
| CLI help | `pnpm openclaw --help` |
| Provider login | `pnpm openclaw models auth login --provider <id>` |
| Diagnose | `pnpm openclaw doctor [--fix]` |
| Run gateway | `pnpm openclaw gateway` |
| Telegram docs | [docs/channels/telegram.md](docs/channels/telegram.md) |
| Copilot plugin | [extensions/github-copilot/](extensions/github-copilot) |
| Telegram plugin | [extensions/telegram/](extensions/telegram) |

## 7. Issues encountered & fixes recap

| Symptom | Cause | Fix |
|---|---|---|
| `ERR_PNPM_NO_GLOBAL_BIN_DIR` on `pnpm link --global` | `PNPM_HOME` not initialized | `pnpm setup && source ~/.bashrc` |
| `openclaw: command not found` after link | `PNPM_HOME` missing from `PATH` | export `PNPM_HOME` and prepend it to `PATH` in `~/.bashrc` |
| `Unknown command: openclaw login` | No top-level `login`; provider auth is nested | Use `pnpm openclaw models auth login --provider github-copilot` |
| Bot token leaked in chat | Pasted live token in plaintext | `@BotFather` → `/revoke` → use new token; never commit tokens |
