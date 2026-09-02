# PAID LXC v7 — Discord VPS Manager Bot

Discord bot jo LXC/LXD containers ko "VPS" ki tarah manage karta hai —
create, resize, suspend, snapshot, port-forward, multi-node support, aur
poora admin panel — sab kuch Discord commands se.

> ⚠️ **Pehle ye kar lo:** is file ke andar (`PAID_LXC_v7_bot.py` line 19) ek
> **hardcoded Discord token** default value ke tor par mila tha. Discord
> Developer Portal → Bot tab → **Reset Token** kar ke naya token le lo, aur
> sirf wo naya token `.env` mein daalo. `.env` ko kabhi GitHub pe push mat
> karna.

---

## 📋 Requirements

- Ubuntu/Debian VPS ya server (root access)
- Python 3.10+
- LXD/LXC installed & initialized
- Discord Bot Token (Server Members Intent + Message Content Intent ON)

---

## 📦 Installation

```bash
# 1) Repo clone karo
git clone https://github.com/atifqmi-max/paid-lxc-v7.git
cd paid-lxc-v7

# 2) Python virtual environment banao
python3 -m venv venv
source venv/bin/activate

# 3) Dependencies install karo
pip install -r requirements.txt
```

**requirements.txt:**
```
discord.py>=2.3.2
requests>=2.31.0
```
(Baaki sab — `sqlite3`, `subprocess`, `json`, `threading` waghera — Python ke
sath already aata hai, alag se install nahi karna.)

---

## 🖥️ LXD/LXC Setup

```bash
sudo snap install lxd
sudo lxd init
```

`lxd init` mein jo storage pool name doge, wahi `.env` mein
`DEFAULT_STORAGE_POOL` mein daalna hai.

---

## ⚙️ Configuration (.env)

Root folder mein `.env` file banao (already di gayi hai) aur values fill
karo:

```env
DISCORD_TOKEN=your_new_bot_token_here
BOT_NAME=Night Cloud Free - Nodes 1
PREFIX=.
YOUR_SERVER_IP=your.server.ip
MAIN_ADMIN_ID=your_discord_user_id
VPS_USER_ROLE_ID=your_role_id
DEFAULT_STORAGE_POOL=default
BOT_VERSION=1.0
BOT_DEVELOPER=Admin
```

ID's nikalne ke liye Discord Settings → Advanced → **Developer Mode** ON
karo, phir apne profile / role pe right-click → **Copy ID**.

Bot `os.getenv()` se ye values read karta hai, isliye `.env` ko process mein
load karna padega — is ke liye neeche systemd method use karo (wo automatic
load kar deta hai `EnvironmentFile` se), ya manually:

```bash
export $(grep -v '^#' .env | xargs)
```

---

## ▶️ Manual Run (testing ke liye)

```bash
source venv/bin/activate
export $(grep -v '^#' .env | xargs)
python3 lxc-bot-v7.py
```

Terminal band karte hi bot bhi band ho jayega — 24/7 ke liye neeche wala
systemd method use karo.

---

## 🔁 24/7 Run — systemd Service (Recommended)

Ye wahi tareeqa hai jo pehle use hota tha. `nohup` / `screen` / `pm2` waghera
ke bajaye **systemd** zyada reliable hai — server reboot hone pe bhi bot
khud start ho jata hai, aur crash hone pe khud restart hota hai.

### 1) Service file banao

```bash
sudo nano /etc/systemd/system/paidlxc.service
```

Ye paste karo (paths apne actual clone location ke hisaab se badlo):

```ini
[Unit]
Description=PAID LXC v7 Discord Bot
After=network.target

[Service]
Type=simple
User=root
WorkingDirectory=/root/paid-lxc-v7
EnvironmentFile=/root/paid-lxc-v7/.env
ExecStart=/root/paid-lxc-v7/venv/bin/python3 /root/paid-lxc-v7/PAID_LXC_v7_bot.py
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

> `WorkingDirectory`, `EnvironmentFile`, aur `ExecStart` ke paths apne actual
> folder ke mutabiq set karo (`pwd` chala kar current path check kar sakte
> ho).

### 2) Service enable + start karo

```bash
sudo systemctl daemon-reload
sudo systemctl enable paidlxc.service
sudo systemctl start paidlxc.service
```

### 3) Status/logs check karo

```bash
sudo systemctl status paidlxc.service      # running hai ya nahi
sudo journalctl -u paidlxc.service -f      # live logs
```

### 4) Common commands

```bash
sudo systemctl restart paidlxc.service     # bot restart
sudo systemctl stop paidlxc.service        # bot stop
sudo systemctl disable paidlxc.service     # auto-start band karna
```

**Agar `systemctl status` mein "failed" ya "activating (auto-restart)" dikhe:**
- `sudo journalctl -u paidlxc.service -n 50 --no-pager` se exact error dekho
- Zyada tar wajah: galat path in `ExecStart`/`WorkingDirectory`, ya `.env`
  mein missing/galat `DISCORD_TOKEN`
- `venv/bin/python3` ka path double-check karo: `ls venv/bin/python3`

---

## 🔄 Bot Update Karna

Jab bhi repo mein naya code aaye:

```bash
cd paid-lxc-v7
git pull
sudo systemctl restart paidlxc.service
```

Sirf jo file change hui ho wahi update karni ho to `git pull` khud detect
kar leta hai — sirf changed files overwrite hoti hain, baaki (jaise `.env`,
`vps.db`, `bot.log`) untouched rehti hain (bashart ke wo `.gitignore` mein
hon).

---

## ⚠️ Security Notes

- `.env` aur `vps.db` ko kabhi git commit ya public repo mein mat daalna
- Sirf trusted logon ko `MAIN_ADMIN_ID` / admin list mein rakho — `.exec`
  command se admin ke paas container ke andar root-level access hai
- Compromised token turant Developer Portal se reset karo
