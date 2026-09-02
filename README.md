# PAID_LXC_v7 Discord Bot

A Discord bot that lets admins create and manage LXC/LXD containers ("VPS") for
users directly from Discord — creation, resizing, suspension, snapshots, port
forwarding, multi-node support, and an admin panel.

> ⚠️ **Security notice:** the copy of `PAID_LXC_v7_bot.py` you have contains a
> **hardcoded Discord bot token** as a fallback default (line 19). Treat that
> token as compromised — go to the
> [Discord Developer Portal](https://discord.com/developers/applications),
> open your bot's **Bot** tab, and click **Reset Token** immediately. Only put
> the *new* token in your `.env` file, and never push `.env` to GitHub.

## Requirements

- **Python 3.10+**
- **Linux host with LXD/LXC installed and initialized** (the bot shells out
  to the `lxc` command via `subprocess`, so it must be runnable on the same
  machine as the bot, with permission to manage containers — typically means
  running the bot as root or a user in the `lxd` group)
- A Discord bot application + token (with **Server Members** and **Message
  Content** privileged intents enabled in the Developer Portal, since the bot
  uses `commands.Bot` with prefix commands)

## 1. Get the code

```bash
git clone https://github.com/atifqmi-max/paid-lxc-v7.git
cd paid-lxc-v7
```

(Or just place `PAID_LXC_v7_bot.py` in its own folder.)

## 2. Install Python dependencies

```bash
python3 -m venv venv
source venv/bin/activate       # Windows: venv\Scripts\activate
pip install -r requirements.txt
```

`requirements.txt`:
```
discord.py>=2.3.2
requests>=2.31.0
```

Everything else the bot imports (`sqlite3`, `subprocess`, `shlex`, `json`,
`threading`, etc.) is part of the Python standard library — nothing extra to
install for those.

## 3. Install LXD/LXC on the host

```bash
sudo snap install lxd
sudo lxd init          # accept defaults, or configure storage pool/network
```

Whatever storage pool name you set during `lxd init` should match
`DEFAULT_STORAGE_POOL` in your `.env`.

## 4. Configure environment variables

Copy the provided `.env` template into your project folder and fill in real
values (see the file for what each variable does):

- `DISCORD_TOKEN` – your **new** bot token (see security notice above)
- `MAIN_ADMIN_ID` / `VPS_USER_ROLE_ID` – right-click your Discord profile /
  the role → **Copy ID** (Developer Mode must be enabled in Discord settings)
- `YOUR_SERVER_IP` – the IP users will connect to their containers on

The bot reads these with `os.getenv(...)`, so you need to actually load them
into the process environment. Two easy options:

**Option A — export before running:**
```bash
export $(grep -v '^#' .env | xargs)
python3 PAID_LXC_v7_bot.py
```

**Option B — use a process manager that loads `.env` automatically**, e.g.
`pm2`, `systemd` with `EnvironmentFile=`, or Docker's `--env-file .env`.

## 5. Run the bot

```bash
python3 PAID_LXC_v7_bot.py
```

On first run it creates `vps.db` (SQLite) and `bot.log` in the working
directory.

## Notes

- The bot must run with permission to execute `lxc` commands (root, or a user
  in the `lxd` group) since it manages containers via `subprocess`.
- Because this bot can create containers and execute arbitrary commands
  inside them via `.exec`, restrict `MAIN_ADMIN_ID` / admin list carefully —
  anyone with admin access effectively has root on your host's containers.
- Keep `.env` and `vps.db` out of version control (add them to `.gitignore`).
