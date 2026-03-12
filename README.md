# N8n-on-android-
How to get n8n on android, full setup, real and working, tested by myself. I can't guarantee that in the future will keep working or that I will update it. 
If it works thanks me here 
https://www.facebook.com/share/182Tgqr2QZ/

# 🤖 Run n8n on Android with Termux — Free, No VPS, No Root

Host **n8n** for free on any Android phone using Termux. No cloud server, no monthly fees, no root required. Works as a permanent home server accessible from any device on your network.

> Built and tested on a real Android phone running 24/7. Repurpose your old phone as a Raspberry Pi alternative.

---

## ✨ Features

- **PostgreSQL** database (not SQLite — production ready)
- **PM2** process manager with auto-restart and log rotation
- **Auto-start** every time Termux opens
- **Access from any device** on your WiFi network
- **Dynamic IP detection** — works at home, office, anywhere
- **Webhooks & AI agents work** from other devices (not just localhost)
- **Battery optimized** — tuned PostgreSQL, capped Node.js memory, minimal cron wakeups
- **Timezone aware** — scheduled triggers run at correct local time
- **Auto-cleanup** — executions pruned every 2 hours, WAL logs capped
- **Backup script** included — one command saves your entire database
- **One command update** — workflows and data always untouched

---

## 📋 Requirements

- Any Android phone (old phones work great)
- [Termux](https://f-droid.org/en/packages/com.termux/) app from F-Droid *(not Play Store version)*
- ~3GB free storage
- WiFi connection

---

## ⚡ Install

**Step 1** — Open Termux and run:
```bash
nano ~/install.sh
```

**Step 2** — Paste the entire script below, then press `Ctrl+X` → `Y` → `Enter`

**Step 3** — Run:
```bash
chmod +x ~/install.sh && bash ~/install.sh
```

**Step 4** — Wait ~15 minutes. When done, close Termux, reopen it, wait 25 seconds, then open:
```
http://localhost:5678
```

---

## 📜 The Script

```bash
#!/bin/bash

echo "================================================"
echo "   n8n Termux Installer - Ultimate Setup"
echo "================================================"
echo ""

# Step 1 - Update Termux
echo "[1/16] Updating Termux..."
pkg update -y && pkg upgrade -y

# Step 2 - Install Node.js
echo "[2/16] Installing Node.js..."
pkg install nodejs -y

# Step 3 - Install Python and build tools (temporary, removed at end)
echo "[3/16] Installing build tools (temporary)..."
pkg install python make clang -y

# Step 4 - Install PostgreSQL
echo "[4/16] Installing PostgreSQL..."
pkg install postgresql -y

# Step 5 - Install n8n (skip native compilation)
echo "[5/16] Installing n8n (5-10 mins, please wait)..."
npm install -g n8n --ignore-scripts

# Step 6 - Install PM2
echo "[6/16] Installing PM2..."
npm install -g pm2

# Step 7 - Setup PM2 log rotation
echo "[7/16] Setting up PM2 log rotation..."
pm2 install pm2-logrotate
pm2 set pm2-logrotate:max_size 10M
pm2 set pm2-logrotate:retain 3
pm2 set pm2-logrotate:compress true

# Step 8 - Fix distutils (needed for Python 3.13+)
echo "[8/16] Fixing Python distutils..."
PYLIB=/data/data/com.termux/files/usr/lib/python3.13
mkdir -p $PYLIB/distutils
echo "" > $PYLIB/distutils/__init__.py
cat > $PYLIB/distutils/version.py << 'EOF'
class StrictVersion:
    def __init__(self, s):
        self.vstring = s
    def __str__(self):
        return self.vstring
    def __lt__(self, other):
        return self.vstring < str(other)
    def __le__(self, other):
        return self.vstring <= str(other)
    def __eq__(self, other):
        return self.vstring == str(other)
    def __ge__(self, other):
        return self.vstring >= str(other)
    def __gt__(self, other):
        return self.vstring > str(other)
EOF

# Step 9 - Create sqlite3 stub
echo "[9/16] Creating sqlite3 stub..."
SQLITE_DIR=/data/data/com.termux/files/usr/lib/node_modules/n8n/node_modules/sqlite3
mkdir -p $SQLITE_DIR/build/Release
touch $SQLITE_DIR/build/Release/node_sqlite3.node
cat > $SQLITE_DIR/lib/sqlite3.js << 'EOF'
// Stub - not needed when using PostgreSQL
class Database {
  constructor() { throw new Error('Use PostgreSQL instead'); }
}
module.exports = { Database, verbose: () => module.exports };
EOF

# Step 10 - Initialize PostgreSQL and create database
echo "[10/16] Setting up PostgreSQL database..."
initdb ~/pgdata
pg_ctl -D ~/pgdata start
sleep 6
createdb n8n
echo "Database created!"

# Optimize PostgreSQL for low power, low memory and controlled storage
cat >> ~/pgdata/postgresql.conf << 'PGEOF'

# Battery and memory optimization
shared_buffers = 32MB
work_mem = 4MB
maintenance_work_mem = 16MB
max_connections = 10
checkpoint_completion_target = 0.9
wal_buffers = 4MB
bgwriter_lru_maxpages = 5
bgwriter_delay = 500ms
effective_cache_size = 64MB
synchronous_commit = off

# WAL logs - capped so they never grow forever
wal_level = minimal
max_wal_senders = 0
max_wal_size = 80MB
min_wal_size = 32MB
wal_keep_size = 0
archive_mode = off

# Auto-vacuum tuning to prevent database bloat
autovacuum = on
autovacuum_naptime = 5min
autovacuum_vacuum_scale_factor = 0.05
autovacuum_analyze_scale_factor = 0.02
autovacuum_vacuum_cost_delay = 20ms
PGEOF

# Restart PostgreSQL to apply all settings
pg_ctl -D ~/pgdata restart
sleep 4

# Step 11 - Create custom nodes directory (survives n8n updates)
echo "[11/16] Setting up directories..."
mkdir -p ~/.n8n/custom
mkdir -p ~/.n8n/nodes

# Step 12 - Save all settings permanently
echo "[12/16] Saving all settings..."
grep -v 'DB_TYPE\|DB_POSTGRESDB\|DB_SQLITE\|EXECUTIONS\|N8N_\|NODE_OPTIONS\|TZ\|GENERIC' ~/.profile > $TMPDIR/profile_clean 2>/dev/null && mv $TMPDIR/profile_clean ~/.profile || true

# Database
echo 'export DB_TYPE=postgresdb' >> ~/.profile
echo 'export DB_POSTGRESDB_DATABASE=n8n' >> ~/.profile
echo 'export DB_POSTGRESDB_HOST=localhost' >> ~/.profile
echo 'export DB_POSTGRESDB_PORT=5432' >> ~/.profile
echo 'export DB_POSTGRESDB_USER=$(whoami)' >> ~/.profile
echo 'export DB_POSTGRESDB_PASSWORD=' >> ~/.profile

# Timezone - change this to your timezone if needed
# Full list: https://en.wikipedia.org/wiki/List_of_tz_database_time_zones
echo 'export TZ=Europe/Bucharest' >> ~/.profile
echo 'export GENERIC_TIMEZONE=Europe/Bucharest' >> ~/.profile

# Network - accessible from any device on same network
echo 'export N8N_SECURE_COOKIE=false' >> ~/.profile
echo 'export N8N_HOST=0.0.0.0' >> ~/.profile
echo 'export N8N_PORT=5678' >> ~/.profile
echo 'export N8N_PROTOCOL=http' >> ~/.profile

# Executions - 24 hour retention, saves everything for debugging
echo 'export EXECUTIONS_DATA_PRUNE=true' >> ~/.profile
echo 'export EXECUTIONS_DATA_MAX_AGE=24' >> ~/.profile
echo 'export EXECUTIONS_DATA_SAVE_ON_ERROR=all' >> ~/.profile
echo 'export EXECUTIONS_DATA_SAVE_ON_SUCCESS=all' >> ~/.profile
echo 'export EXECUTIONS_DATA_SAVE_MANUAL_EXECUTIONS=true' >> ~/.profile
echo 'export EXECUTIONS_PROCESS=main' >> ~/.profile

# Battery and memory
echo 'export NODE_OPTIONS="--max-old-space-size=256"' >> ~/.profile

# Directories
echo 'export N8N_USER_FOLDER=/data/data/com.termux/files/home/.n8n' >> ~/.profile

# Disable telemetry (saves CPU and battery)
echo 'export N8N_DIAGNOSTICS_ENABLED=false' >> ~/.profile
echo 'export N8N_VERSION_NOTIFICATIONS_ENABLED=false' >> ~/.profile
echo 'export N8N_HIRING_BANNER_ENABLED=false' >> ~/.profile

# Step 13 - Setup cron jobs
echo "[13/16] Setting up cron jobs..."
pkg install cronie -y 2>/dev/null

cat > ~/cleanup-executions.sh << 'EOF'
#!/bin/bash
source ~/.profile
psql -d n8n -c "DELETE FROM execution_entity WHERE \"startedAt\" < NOW() - INTERVAL '24 hours';" 2>/dev/null
psql -d n8n -c "DELETE FROM execution_data WHERE \"executionId\" NOT IN (SELECT id FROM execution_entity);" 2>/dev/null
EOF
chmod +x ~/cleanup-executions.sh

# Every 2 hours - less wakeups = better battery
(crontab -l 2>/dev/null | grep -v 'cleanup-executions'; echo "0 */2 * * * bash ~/cleanup-executions.sh") | crontab -

# Step 14 - Create backup script
echo "[14/16] Creating backup script..."
cat > ~/backup-n8n.sh << 'EOF'
#!/bin/bash
source ~/.profile
BACKUP_DIR=~/n8n-backups
mkdir -p $BACKUP_DIR
FILENAME="n8n-backup-$(date +%Y%m%d-%H%M%S).sql"
pg_dump n8n > $BACKUP_DIR/$FILENAME
echo "Backup saved to: $BACKUP_DIR/$FILENAME"
# Keep only last 5 backups
ls -t $BACKUP_DIR/*.sql | tail -n +6 | xargs rm -f 2>/dev/null
echo "Done! Backups kept: $(ls $BACKUP_DIR/*.sql | wc -l)"
EOF
chmod +x ~/backup-n8n.sh

# Step 15 - Create autostart script
echo "[15/16] Creating autostart script..."
cat > ~/start-n8n.sh << 'EOF'
#!/bin/bash

# Load all settings FIRST
source ~/.profile

# Database
export DB_TYPE=postgresdb
export DB_POSTGRESDB_DATABASE=n8n
export DB_POSTGRESDB_HOST=localhost
export DB_POSTGRESDB_PORT=5432
export DB_POSTGRESDB_USER=$(whoami)
export DB_POSTGRESDB_PASSWORD=

# Timezone
export TZ=Europe/Bucharest
export GENERIC_TIMEZONE=Europe/Bucharest

# Network
export N8N_SECURE_COOKIE=false
export N8N_HOST=0.0.0.0
export N8N_PORT=5678
export N8N_PROTOCOL=http

# Executions
export EXECUTIONS_DATA_PRUNE=true
export EXECUTIONS_DATA_MAX_AGE=24
export EXECUTIONS_DATA_SAVE_ON_ERROR=all
export EXECUTIONS_DATA_SAVE_ON_SUCCESS=all
export EXECUTIONS_DATA_SAVE_MANUAL_EXECUTIONS=true
export EXECUTIONS_PROCESS=main

# Battery
export NODE_OPTIONS="--max-old-space-size=256"

# Directories
export N8N_USER_FOLDER=/data/data/com.termux/files/home/.n8n

# Disable telemetry
export N8N_DIAGNOSTICS_ENABLED=false
export N8N_VERSION_NOTIFICATIONS_ENABLED=false
export N8N_HIRING_BANNER_ENABLED=false

# Start PostgreSQL
pg_ctl -D ~/pgdata start 2>/dev/null
sleep 15

# Start cron
crond 2>/dev/null

# Auto-detect current IP on any network
LOCAL_IP=$(ifconfig 2>/dev/null | grep 'inet ' | grep -v '127.0.0.1' | awk '{print $2}' | head -1)

# Replace localhost with real IP for everything
export N8N_HOST=$LOCAL_IP
export N8N_EDITOR_BASE_URL=http://$LOCAL_IP:5678
export WEBHOOK_URL=http://$LOCAL_IP:5678
export VUE_APP_URL_BASE_API=http://$LOCAL_IP:5678

# Create directories
mkdir -p ~/.n8n/custom
mkdir -p ~/.n8n/nodes

# Start n8n with PM2
if pm2 list | grep -q "n8n"; then
    pm2 restart n8n
else
    pm2 start n8n --name "n8n" \
        --max-memory-restart 300M \
        --node-args="--max-old-space-size=256" \
        -- start
    pm2 save
fi

echo ""
echo "================================================"
echo " n8n is running!"
echo " On this phone:  http://localhost:5678"
echo " Other devices:  http://$LOCAL_IP:5678"
echo " Webhooks:       http://$LOCAL_IP:5678/webhook/..."
echo " Timezone:       Europe/Bucharest"
echo ""
echo " Shortcuts:"
echo "   n8nstatus    - check if running"
echo "   n8nlogs      - live logs"
echo "   n8nrestart   - restart n8n"
echo "   n8nbackup    - backup database"
echo "   n8nupdate    - update n8n"
echo "   myip         - show current IP"
echo "================================================"
echo ""
EOF
chmod +x ~/start-n8n.sh

# Add autostart and aliases to .bashrc
grep -v 'start-n8n\|Auto-start n8n\|alias n8n\|alias myip\|alias backup' ~/.bashrc > $TMPDIR/bashrc_clean 2>/dev/null && mv $TMPDIR/bashrc_clean ~/.bashrc || true
echo '' >> ~/.bashrc
echo '# Auto-start n8n' >> ~/.bashrc
echo 'bash ~/start-n8n.sh' >> ~/.bashrc
echo '' >> ~/.bashrc
echo '# Shortcuts' >> ~/.bashrc
echo "alias n8nlogs='pm2 logs n8n'" >> ~/.bashrc
echo "alias n8nrestart='pm2 restart n8n'" >> ~/.bashrc
echo "alias n8nstatus='pm2 status'" >> ~/.bashrc
echo "alias myip='ifconfig | grep inet | grep -v 127.0.0.1'" >> ~/.bashrc
echo "alias n8nbackup='bash ~/backup-n8n.sh'" >> ~/.bashrc
echo "alias n8nupdate='npm install -g n8n@latest --ignore-scripts && pm2 restart n8n'" >> ~/.bashrc

# Step 16 - Clean up ALL unnecessary files
echo "[16/16] Cleaning up to save disk space..."
npm cache clean --force
rm -rf /data/data/com.termux/files/usr/include
rm -rf /data/data/com.termux/files/home/.cache/node-gyp
npm uninstall -g sqlite3 2>/dev/null
pkg uninstall clang llvm libllvm lld ndk-sysroot make python python-pip python-ensurepip-wheels libcompiler-rt -y 2>/dev/null
pkg autoremove -y 2>/dev/null

echo ""
echo "================================================"
echo "   Installation Complete!"
echo "================================================"
echo ""
echo "DAILY USE:"
echo "1. Open Termux"
echo "2. Wait 25 seconds"
echo "3. On this phone:   http://localhost:5678"
echo "4. Other devices:   http://YOUR-IP:5678"
echo "   (IP shown automatically every time)"
echo ""
echo "SHORTCUTS:"
echo "  n8nstatus    - is n8n alive?"
echo "  n8nlogs      - see live logs"
echo "  n8nrestart   - restart n8n"
echo "  n8nbackup    - backup database"
echo "  n8nupdate    - update to latest n8n"
echo "  myip         - show current IP"
echo ""
echo "BACKUP:"
echo "  Run n8nbackup anytime"
echo "  Saved to ~/n8n-backups/"
echo "  Last 5 backups kept automatically"
echo ""
echo "TO UPDATE n8n:"
echo "  n8nupdate"
echo "  (workflows and data untouched)"
echo ""
echo "BATTERY TIPS (do these manually once):"
echo "  - Termux notification: tap Acquire Wakelock"
echo "  - Settings > Battery > Termux: Unrestricted"
echo "  - Settings > WiFi > Keep on during sleep: Always"
echo "  - Turn off Bluetooth, GPS, NFC"
echo "  - Developer options: all animations to 0"
echo "  - Set WiFi to 2.4GHz only"
echo ""
echo "If you ever need build tools back:"
echo "  pkg install python make clang"
echo "================================================"
```

---

## 🕹️ Daily Shortcuts

| Command | What it does |
|---|---|
| `n8nstatus` | Check if n8n is running |
| `n8nlogs` | See live logs |
| `n8nrestart` | Restart n8n |
| `n8nbackup` | Backup database |
| `n8nupdate` | Update to latest n8n |
| `myip` | Show your current network IP |

---

## 🌍 Changing Your Timezone

Find your timezone from the [full list](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones) and replace `Europe/Bucharest` in the script with yours before running. Example: `America/New_York`, `Europe/London`, `Asia/Tokyo`.

---

## 🔋 Battery Tips (do once after install)

- Termux notification → tap **Acquire Wakelock**
- Settings → Battery → Termux → **Unrestricted**
- Settings → WiFi → Keep on during sleep → **Always**
- Turn off Bluetooth, GPS, NFC
- Developer options → all animations to **0**
- Set WiFi to **2.4GHz only**

For true CPU wakelock (prevents throttling with screen off):
```bash
# Install Termux:API from Play Store first, then:
pkg install termux-api
termux-wake-lock
```

---

## 💾 Backup & Restore

**Backup:**
```bash
n8nbackup
# Saved to ~/n8n-backups/ — last 5 kept automatically
```

**Restore:**
```bash
pg_ctl -D ~/pgdata start
psql n8n < ~/n8n-backups/YOUR-BACKUP-FILE.sql
```

---

## 🔄 Updating n8n

```bash
n8nupdate
```
Your workflows, credentials and data are never touched.

---

## 🌐 Accessing From Other Devices

Every time Termux opens it automatically detects your current IP and displays:
```
Other devices: http://192.168.x.x:5678
```
Use that URL from any phone, laptop or tablet on the same WiFi. Works at home, office or a friend's place — no manual configuration needed.

---

## ❓ Troubleshooting

**n8n not loading after opening Termux?**
Wait 25 seconds — PostgreSQL needs time to start. If still not working:
```bash
pg_ctl -D ~/pgdata start
sleep 5
pm2 restart n8n
```

**Webhooks showing 0.0.0.0 instead of real IP?**
```bash
pm2 restart n8n
```
The IP is detected fresh on every restart.

**Check if everything is running:**
```bash
n8nstatus
pg_ctl -D ~/pgdata status
```

---

## ⚙️ What's Optimized

- **PostgreSQL** — shared buffers reduced, WAL logs capped at 80MB, auto-vacuum enabled, max 10 connections
- **Node.js** — heap capped at 256MB
- **PM2** — restarts if memory exceeds 300MB, log rotation at 10MB
- **Cron** — runs every 2 hours instead of every hour
- **Telemetry** — fully disabled
- **Executions** — pruned after 24 hours automatically

---

## 📱 Why Use a Phone as a Server?

- **Free** — repurpose a phone you already own
- **Low power** — 3-8W idle, similar to a Raspberry Pi
- **Built-in UPS** — battery keeps it running during power cuts
- **Built-in WiFi** — no dongle needed
- **More RAM** than most Pis
- **No e-waste** — give old hardware a second life

---

*Tested on Android with Termux. Contributions welcome.*

