<div align="center">

# 🪨 StoneNodes VPS Manager

**A Discord bot that turns your LXC host into a full self-service VPS business.**

Create • Start • Stop • Reinstall • SSH • Monitor — all from Discord, with a
custom **Green & Stone** theme.

![Python](https://img.shields.io/badge/python-3.10%2B-3CB878?style=for-the-badge&logo=python&logoColor=white)
![discord.py](https://img.shields.io/badge/discord.py-2.0%2B-4F7965?style=for-the-badge&logo=discord&logoColor=white)
![LXD](https://img.shields.io/badge/LXD%2FLXC-required-3F4B3D?style=for-the-badge)
![License](https://img.shields.io/badge/status-self--hosted-B98D45?style=for-the-badge)

</div>

---

## ✨ Features

- 🖥️ **Full VPS lifecycle** — create, start, stop, reinstall, resize, clone, snapshot, and delete LXC containers straight from Discord.
- 🎛️ **One-tap control panel** — `+manage <vps-id>` opens a specific VPS's buttons instantly (no dropdown hunting).
- ⏳ **Timed VPS rentals** — create a VPS with a duration in days; it auto-suspends when time's up, and admins can `+extend` it.
- ⛏️ **Automatic crypto-mining / abuse detection** — sustained high CPU+RAM+Disk usage auto-suspends the VPS and DMs both the owner and the admin.
- 📋 **`+all-vm`** — one compact list of every VPS on the bot: owner, container ID, and RAM.
- 🔌 **Port forwarding** built on native LXD proxy devices — no manual iptables.
- 👑 **Admin tools** — quotas, suspensions, whitelisting, multi-node support, resource monitoring.
- 🤝 **VPS sharing** — owners can grant/revoke access to their VPS for teammates.
- 🎨 **Green & Stone theme** — consistent color palette across every embed, status color, and auto-created Discord role.
- 🔐 **Config via `.env`** — no secrets baked into the source code.

---

## 📋 Table of Contents

1. [Requirements](#1-requirements)
2. [Install LXD/LXC](#2-install-lxdlxc-on-the-server)
3. [Get the bot's files](#3-get-the-bots-files-onto-the-server)
4. [Create the Discord bot](#4-create-the-discord-bot)
5. [Configure the bot](#5-configure-the-bot)
6. [Run the bot](#6-run-the-bot)
7. [Commands](#7-commands)
8. [LXC/LXD Troubleshooting](#8-lxclxd-troubleshooting)
9. [What changed in this version](#9-what-changed-in-this-version)
10. [Security notes](#10-security-notes)

---

## 1. Requirements

| Requirement | Notes |
|---|---|
| OS | Ubuntu 22.04 or 24.04 LTS (recommended) |
| Access | Root or sudo on the server |
| Python | 3.10 or newer |
| Discord Bot | Token + application ([Section 4](#4-create-the-discord-bot)) |
| Virtualization | Server must support **nested/privileged LXC containers** |

> ⚠️ **Heads up:** OpenVZ VPS plans and most budget "container" VPS plans
> block LXC-in-LXC nesting at the host level. Use a proper KVM/dedicated
> server for the host running this bot.

---

## 2. Install LXD/LXC on the server

Most "the bot doesn't work" reports are actually LXD problems — get this
working on its own *before* touching the bot.

```bash
# Remove any old apt-based lxd/lxc (deprecated, causes conflicts)
sudo apt remove --purge lxd lxd-client -y 2>/dev/null

# Install snapd, then LXD via snap (the officially supported channel)
sudo apt update
sudo apt install snapd -y
sudo snap install lxd

# Initialize LXD with sane defaults
sudo lxd init --auto
```

Add your user to the `lxd` group so it can talk to LXD without `sudo`:

```bash
sudo usermod -aG lxd $USER
newgrp lxd   # or just log out and back in / reboot
```

✅ **Sanity check** — confirm LXD actually works before moving on:

```bash
lxc list
lxc launch ubuntu:22.04 test-container
lxc list
lxc delete test-container --force
```

If that all runs cleanly, you're ready for the bot. If not, jump to
[Section 8](#8-lxclxd-troubleshooting) first.

---

## 3. Get the bot's files onto the server

```bash
cd ~
git clone https://github.com/atifqmi-max/paid-lxc-v7.git stonenodes-bot
cd stonenodes-bot
```

Drop in the rebranded `bot.py`, `requirements.txt`, and `.env.example`
provided with this README (see [Section 9](#9-what-changed-in-this-version)
for what's different).

Set up a virtual environment:

```bash
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

---

## 4. Create the Discord bot

1. Go to the [Discord Developer Portal](https://discord.com/developers/applications) → **New Application**.
2. Name it (e.g. `StoneNodes`) → go to the **Bot** tab → **Add Bot**.
3. Under **Privileged Gateway Intents**, turn on:
   - `SERVER MEMBERS INTENT`
   - `MESSAGE CONTENT INTENT`
4. Click **Reset Token** and copy it — you'll paste it into `.env` next.
   🔒 **Never** put this token in the bot's source code or share it publicly.
5. Go to **OAuth2 → URL Generator**, select scope `bot`, and permissions:
   `Send Messages`, `Embed Links`, `Read Message History`, `Manage Roles`.
   Open the generated URL to invite the bot to your server.

To grab the IDs the bot needs: enable **Developer Mode** in
**Discord Settings → Advanced**, then right-click your username →
**Copy User ID** for `MAIN_ADMIN_ID`.

---

## 5. Configure the bot

```bash
cp .env.example .env
nano .env
```

Minimum required:

```ini
DISCORD_TOKEN=your-real-bot-token-here
MAIN_ADMIN_ID=your-discord-user-id
YOUR_SERVER_IP=your-server-public-ip
DEFAULT_STORAGE_POOL=default
```

Branding (already set, change if you like):

```ini
BOT_NAME=StoneNodes
PREFIX=+
BOT_LOGO_URL=https://your-image-host.com/your-logo.png
```

Double-check your real storage pool name — it isn't always `default`:

```bash
lxc storage list
```

---

## 6. Run the bot

Test in the foreground first so errors are visible immediately:

```bash
source venv/bin/activate
python3 bot.py
```

Seeing `Logged in as StoneNodes#1234`? You're live — try `+help` in Discord.
Stop with `Ctrl+C`, then set it up as a permanent systemd service:

```bash
sudo nano /etc/systemd/system/stonenodes-bot.service
```

```ini
[Unit]
Description=StoneNodes VPS Manager Discord Bot
After=network.target snap.lxd.daemon.service

[Service]
Type=simple
User=YOUR_LINUX_USERNAME
WorkingDirectory=/home/YOUR_LINUX_USERNAME/stonenodes-bot
ExecStart=/home/YOUR_LINUX_USERNAME/stonenodes-bot/venv/bin/python3 bot.py
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now stonenodes-bot
sudo systemctl status stonenodes-bot
journalctl -u stonenodes-bot -f   # live logs
```

---

## 7. Commands

Default prefix is `+` (change via `PREFIX` in `.env`).

| Command | Description |
|---|---|
| `+help` | Interactive help menu |
| `+myvps` | List your own VPS |
| `+manage` | Browse and control all your VPS |
| `+manage <vps-id>` | Jump straight into one VPS's control panel (Start / Stop / Reinstall / SSH / Stats) |
| `+create <ram> <cpu> <disk> @user [duration_days]` | *(Admin)* create a new VPS for a user, optionally with a rental duration |
| `+extend <container> <days>` | *(Admin)* add days to a VPS's duration — unsuspends it if it was suspended |
| `+all-vm` | *(Admin)* compact one-line-per-VPS list: owner, container ID, RAM |
| `+suspend-vps <container> [reason]` | *(Admin)* suspend a VPS |
| `+unsuspend-vps <container>` | *(Admin)* lift a suspension |
| `+ports` | Manage your port forwards |
| `+node list` | *(Admin)* list LXD nodes |

> 💡 When a VPS is created, the owner gets a DM with their **VPS ID** and the
> exact `+manage <id>` command to use — the number matches what's shown there.

---

### ⏳ Timed VPS rentals & extending

Add a number of days to `+create` to make a VPS expire automatically:

```
+create 4 2 40 @someuser 30
```

This creates a 4GB/2-core/40GB VPS that **auto-suspends after 30 days**. The
owner gets a DM when it's created (showing the expiry date) and another DM
the moment it auto-suspends. To give someone more time — and automatically
unsuspend + restart their VPS if it already expired:

```
+extend stonenodes-vps-123456789-1 15
```

Leave `duration_days` off `+create` entirely for a VPS with no expiry.

### ⛏️ Automatic crypto-mining / resource-abuse detection

Every few minutes, the bot checks all running (non-whitelisted) VPS. If a
VPS's **CPU, RAM, and Disk usage are all simultaneously very high** for
several checks in a row — the classic signature of a crypto-miner running
flat out — the bot automatically:

1. Stops the container and marks it suspended.
2. DMs the owner explaining exactly why (with the measured CPU/RAM/Disk %).
3. DMs the configured `MAIN_ADMIN_ID` so an admin can review it.

A normal one-off spike (compiling something, a game server update, etc.)
won't trigger this — it only fires on **sustained** high usage across **all
three** metrics at once. An admin can reverse a false positive with
`+unsuspend-vps <container>`, or `+whitelist-vps <container> add` to exempt
that VPS from monitoring entirely.

---

## 8. LXC/LXD Troubleshooting

<details>
<summary><b>🔴 "Failed to run: ... apparmor" or containers won't start</b></summary>

Containers made by this bot are **privileged + nesting-enabled** (so users
can run Docker inside their VPS). Some hardened kernels/older AppArmor
versions block this.

```bash
sudo apt install apparmor apparmor-utils -y
sudo systemctl restart snap.lxd.daemon
```

Still stuck? Check `dmesg | tail -50` right after a failed `lxc start` —
AppArmor denials show the exact rule that failed.
</details>

<details>
<summary><b>🔴 "No storage pool found" / storage errors</b></summary>

```bash
lxc storage list
```

If it's empty:

```bash
sudo lxc storage create default dir
```

(Use `zfs` or `btrfs` instead of `dir` if your filesystem supports it —
faster container creation and snapshots.)
</details>

<details>
<summary><b>🔴 "Permission denied" talking to the LXD socket</b></summary>

Your Linux user isn't in the `lxd` group yet:

```bash
sudo usermod -aG lxd $USER
# fully log out/in, or reboot
```
</details>

<details>
<summary><b>🔴 Containers start but have no internet access</b></summary>

```bash
lxc network list
lxc network show lxdbr0
```

If `lxdbr0` is missing/broken:

```bash
sudo lxd init --auto
```

Make sure IP forwarding is on:

```bash
echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```
</details>

<details>
<summary><b>🔴 Port forwarding (`+ports`) doesn't work from outside</b></summary>

This bot uses LXD's native `proxy` device — usually the real culprit is the
**host firewall**, not LXD:

```bash
sudo ufw allow 20000:40000/tcp
sudo ufw allow 20000:40000/udp
```

Adjust the range to whatever you're actually using, and check your cloud
provider's security-group/firewall panel too — many block ports at the
network level as well as the OS level.
</details>

<details>
<summary><b>🔴 LXD is completely broken — start fresh</b></summary>

```bash
sudo snap remove lxd --purge
sudo apt remove --purge lxd lxd-client -y
sudo snap install lxd
sudo lxd init --auto
```

⚠️ This deletes all existing containers — only do this if nothing important
is running.
</details>

<details>
<summary><b>🔴 Bot can't find the "lxc" command</b></summary>

```bash
which lxc
```

If the bot runs as a different user than the one you tested with, make sure
*that* user is also in the `lxd` group.
</details>

---

## 9. What changed in this version

- 🪨 Rebranded to **StoneNodes** with a `+` prefix — both configurable in `.env`.
- 🎨 New **Green & Stone** theme across every embed, VPS status color, the
  help menu, and the auto-created Discord role.
- 🎯 Added direct VPS management — `+manage <vps-id>` opens that VPS's panel
  immediately, and the creation DM now tells owners their ID and the exact
  command to use.
- ⏳ Added **timed VPS rentals** — `+create` takes an optional
  `duration_days`, auto-suspending the VPS when it runs out — and a new
  `+extend <container> <days>` admin command to add time back (and
  auto-unsuspend).
- ⛏️ Added **automatic crypto-mining/abuse detection** — a background check
  auto-suspends any VPS with sustained high CPU + RAM + Disk usage all at
  once, DMs the owner with the reason, and DMs the admin for review.
- 📋 Added **`+all-vm`** — a compact, working, one-line-per-VPS list of every
  VPS on the bot (owner, container ID, RAM). Also fixed the existing
  `+list-all` command, which was silently building its per-VPS breakdown but
  never actually sending it.
- 🧹 Removed leftover branding from earlier resellers of this codebase
  ("Zycron", "Powered by Your Bot") that was still hardcoded in a few embeds.
- 🔐 **Security fix** — the original file had a real Discord bot token, admin
  user ID, and role ID hardcoded as defaults directly in the source. These
  are now read only from `.env`, and the bot refuses to start without a
  token. **If you were using the original file, regenerate your bot token in
  the Developer Portal — the old one was exposed in plain text.**

---

## 10. Security notes

> 🔐 **Never commit your `.env` file.** It contains your live bot token.
> Add `.env` to your `.gitignore` before pushing this repo anywhere.

> ⚠️ **Privileged containers.** Every VPS this bot creates is privileged and
> nesting-enabled (`security.privileged=true`, `security.nesting=true`) so
> customers can run Docker. Privileged containers have weaker isolation from
> the host than unprivileged ones — a user who breaks out of their container
> could potentially affect the host or other containers. Keep the host OS
> patched, don't run anything sensitive directly on the host, and consider
> unprivileged containers if your customers don't actually need
> Docker-in-Docker.

<div align="center">

Made with 🪨 + 🌿 for **StoneNodes**

</div>
