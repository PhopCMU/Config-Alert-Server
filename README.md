# 1. ติดตั้ง Package

```bash
sudo apt update

sudo apt install -y \
curl \
jq \
sysstat \
util-linux
```

---

# 2. สร้าง Folder

```bash
sudo mkdir -p /opt/server-monitor/{state,logs}
```

---

# 3. config.conf

```bash
sudo nano /opt/server-monitor/config.conf
```

```bash
BOT_TOKEN="YOUR_BOT_TOKEN"

CHAT_ID="YOUR_CHAT_ID"

# ==========================================
# CPU
# ==========================================

CPU_LIMIT=90

CPU_CORE_LIMIT=95

CPU_RECOVERY=80

CPU_COOLDOWN=300

# ==========================================
# RAM
# ==========================================

RAM_LIMIT=90

RAM_RECOVERY=80

RAM_COOLDOWN=300

# ==========================================
# DISK
# ==========================================

DISK_LIMIT=90

DISK_RECOVERY=80

DISK_COOLDOWN=1800

# ==========================================
# DOCKER
# ==========================================

DOCKER_COOLDOWN=300
```

---

# 4. monitor.sh

```bash
sudo nano /opt/server-monitor/monitor.sh
```

```bash
#!/bin/bash

# =========================================================
# SERVER MONITOR - PRODUCTION STABLE
# Ubuntu 22 / Ubuntu 24
# Docker / Bun / Node.js / PostgreSQL
# =========================================================

export LC_ALL=C

# =========================================================
# BASE
# =========================================================

BASE_DIR="/opt/server-monitor"

CONFIG_FILE="${BASE_DIR}/config.conf"

STATE_DIR="${BASE_DIR}/state"

LOG_DIR="${BASE_DIR}/logs"

LOG_FILE="${LOG_DIR}/monitor.log"

LOCK_FILE="/tmp/server-monitor.lock"

# =========================================================
# CHECK CONFIG
# =========================================================

if [ ! -f "$CONFIG_FILE" ]; then
    echo "config.conf not found"
    exit 1
fi

source "$CONFIG_FILE"

# =========================================================
# CREATE DIR
# =========================================================

mkdir -p "$STATE_DIR"
mkdir -p "$LOG_DIR"

# =========================================================
# LOCK
# =========================================================

exec 200>"$LOCK_FILE"

flock -n 200 || exit 1

# =========================================================
# SERVER INFO
# =========================================================

HOSTNAME=$(hostname)

SERVER_IP=$(hostname -I | awk '{print $1}')

TIME=$(date "+%Y-%m-%d %H:%M:%S")

NOW=$(date +%s)

# =========================================================
# LOG
# =========================================================

log() {

echo "[$TIME] $1" >> "$LOG_FILE"

}

# =========================================================
# TELEGRAM
# =========================================================

send_telegram() {

MESSAGE="$1"

curl -s \
--max-time 10 \
--retry 3 \
-X POST \
"https://api.telegram.org/bot${BOT_TOKEN}/sendMessage" \
-d chat_id="${CHAT_ID}" \
-d text="${MESSAGE}" \
> /dev/null

}

# =========================================================
# SAFE INTEGER
# =========================================================

safe_int() {

VALUE="$1"

if [[ "$VALUE" =~ ^[0-9]+$ ]]; then
    echo "$VALUE"
else
    echo "0"
fi

}

# =========================================================
# MPSTAT
# =========================================================

MPSTAT_OUTPUT=$(mpstat -P ALL 1 1)

# =========================================================
# CPU TOTAL
# =========================================================

CPU_USAGE=$(echo "$MPSTAT_OUTPUT" | awk '

/Average:/ && $2 == "all" {

printf "%.0f", 100 - $NF

}

')

CPU_USAGE=$(safe_int "$CPU_USAGE")

# =========================================================
# CPU PER CORE
# =========================================================

CORE_ALERTS=""

while read -r cpu idle
do

    if [[ "$cpu" == "all" ]] || \
       [[ "$cpu" == "" ]]; then
        continue
    fi

    if ! [[ "$idle" =~ ^[0-9]+(\.[0-9]+)?$ ]]; then
        continue
    fi

    usage=$(awk -v idle="$idle" '
    BEGIN {
        printf "%.0f", 100 - idle
    }')

    usage=$(safe_int "$usage")

    if [ "$usage" -ge "$CPU_CORE_LIMIT" ]; then

        CORE_ALERTS="${CORE_ALERTS}
🔥 Core ${cpu}: ${usage}%"

    fi

done < <(

echo "$MPSTAT_OUTPUT" | awk '

/Average:/ {

print $2, $NF

}

'

)

# =========================================================
# RAM
# =========================================================

RAM_USAGE=$(free | awk '/Mem:/ {
print int($3/$2 * 100)
}')

RAM_USAGE=$(safe_int "$RAM_USAGE")

# =========================================================
# DISK
# =========================================================

DISK_USAGE=$(df / | awk 'END {
gsub("%","",$5)
print $5
}')

DISK_USAGE=$(safe_int "$DISK_USAGE")

# =========================================================
# LOAD
# =========================================================

LOAD_AVG=$(uptime | awk -F'load average:' '{print $2}')

UPTIME=$(uptime -p)

# =========================================================
# TOP PROCESS
# =========================================================

TOP_CPU=$(ps -eo pid,psr,comm,%cpu \
--sort=-%cpu | head -2 | tail -1)

TOP_RAM=$(ps -eo pid,comm,%mem \
--sort=-%mem | head -2 | tail -1)

# =========================================================
# CHECK METRIC
# =========================================================

check_metric() {

TYPE="$1"

VALUE="$2"

LIMIT="$3"

RECOVERY="$4"

COOLDOWN="$5"

EMOJI="$6"

STATE_FILE="${STATE_DIR}/${TYPE}.state"

START_FILE="${STATE_DIR}/${TYPE}.start"

LAST_ALERT_FILE="${STATE_DIR}/${TYPE}.last"

# =====================================================
# ALERT
# =====================================================

if [ "$VALUE" -ge "$LIMIT" ]; then

    LAST_ALERT=0

    if [ -f "$LAST_ALERT_FILE" ]; then
        LAST_ALERT=$(cat "$LAST_ALERT_FILE")
    fi

    DIFF=$((NOW - LAST_ALERT))

    if [ "$DIFF" -lt "$COOLDOWN" ] && \
       [ -f "$STATE_FILE" ]; then
        return
    fi

    if [ ! -f "$STATE_FILE" ]; then

        touch "$STATE_FILE"

        echo "$NOW" > "$START_FILE"

    fi

    echo "$NOW" > "$LAST_ALERT_FILE"

    send_telegram "${EMOJI} ${TYPE} ALERT

🖥 Server: ${HOSTNAME}

🌐 IP: ${SERVER_IP}

🕒 Time: ${TIME}

📈 ${TYPE}: ${VALUE}%

⚠️ Threshold: ${LIMIT}%

${CORE_ALERTS}

📊 Load Avg:${LOAD_AVG}

⏳ Uptime:${UPTIME}

🔥 TOP CPU:
${TOP_CPU}

🧠 TOP RAM:
${TOP_RAM}"

    log "${TYPE} ALERT ${VALUE}%"

# =====================================================
# RECOVERY
# =====================================================

elif [ "$VALUE" -le "$RECOVERY" ]; then

    if [ -f "$STATE_FILE" ]; then

        START=$(cat "$START_FILE")

        DURATION=$((NOW - START))

        MINUTES=$((DURATION / 60))

        SECONDS=$((DURATION % 60))

        rm -f "$STATE_FILE"

        rm -f "$START_FILE"

        rm -f "$LAST_ALERT_FILE"

        send_telegram "✅ ${TYPE} RECOVERED

🖥 Server: ${HOSTNAME}

🌐 IP: ${SERVER_IP}

🕒 Time: ${TIME}

📉 Current ${TYPE}: ${VALUE}%

⏱ Incident Duration:
${MINUTES}m ${SECONDS}s"

        log "${TYPE} RECOVERED ${VALUE}%"

    fi

fi

}

# =========================================================
# DOCKER ENGINE
# =========================================================

DOCKER_STATE_FILE="${STATE_DIR}/docker-engine.state"

DOCKER_LAST_FILE="${STATE_DIR}/docker-engine.last"

DOCKER_STATUS=$(systemctl is-active docker 2>/dev/null)

if [ "$DOCKER_STATUS" != "active" ]; then

    LAST_ALERT=0

    if [ -f "$DOCKER_LAST_FILE" ]; then
        LAST_ALERT=$(cat "$DOCKER_LAST_FILE")
    fi

    DIFF=$((NOW - LAST_ALERT))

    if [ "$DIFF" -ge "$DOCKER_COOLDOWN" ]; then

        echo "$NOW" > "$DOCKER_LAST_FILE"

        if [ ! -f "$DOCKER_STATE_FILE" ]; then
            echo "$NOW" > "$DOCKER_STATE_FILE"
        fi

        send_telegram "🚨 DOCKER ENGINE DOWN

🖥 Server: ${HOSTNAME}

🌐 IP: ${SERVER_IP}

🕒 Time: ${TIME}

❌ Docker daemon is not running"

        log "DOCKER ENGINE DOWN"

    fi

else

    if [ -f "$DOCKER_STATE_FILE" ]; then

        START=$(cat "$DOCKER_STATE_FILE")

        DURATION=$((NOW - START))

        MINUTES=$((DURATION / 60))

        SECONDS=$((DURATION % 60))

        rm -f "$DOCKER_STATE_FILE"

        rm -f "$DOCKER_LAST_FILE"

        send_telegram "✅ DOCKER ENGINE RECOVERED

🖥 Server: ${HOSTNAME}

🌐 IP: ${SERVER_IP}

🕒 Time: ${TIME}

✔️ Docker daemon back online

⏱ Downtime:
${MINUTES}m ${SECONDS}s"

        log "DOCKER ENGINE RECOVERED"

    fi

fi

# =========================================================
# DOCKER CONTAINER
# =========================================================

docker ps -a \
--format "{{.Names}}|{{.Status}}" | \
while IFS="|" read -r NAME STATUS
do

STATE_FILE="${STATE_DIR}/container-${NAME}.state"

LAST_FILE="${STATE_DIR}/container-${NAME}.last"

# =====================================================
# UNHEALTHY / DOWN
# =====================================================

if [[ "$STATUS" == *"unhealthy"* ]] || \
   [[ "$STATUS" == *"Exited"* ]] || \
   [[ "$STATUS" == *"Restarting"* ]]; then

    LAST_ALERT=0

    if [ -f "$LAST_FILE" ]; then
        LAST_ALERT=$(cat "$LAST_FILE")
    fi

    DIFF=$((NOW - LAST_ALERT))

    if [ "$DIFF" -ge "$DOCKER_COOLDOWN" ]; then

        echo "$NOW" > "$LAST_FILE"

        if [ ! -f "$STATE_FILE" ]; then
            echo "$NOW" > "$STATE_FILE"
        fi

        send_telegram "🚨 CONTAINER ALERT

🖥 Server: ${HOSTNAME}

🌐 IP: ${SERVER_IP}

🕒 Time: ${TIME}

🐳 Container: ${NAME}

📉 Status:
${STATUS}"

        log "CONTAINER ALERT ${NAME}"

    fi

# =====================================================
# RECOVERY
# =====================================================

else

    if [ -f "$STATE_FILE" ]; then

        START=$(cat "$STATE_FILE")

        DURATION=$((NOW - START))

        MINUTES=$((DURATION / 60))

        SECONDS=$((DURATION % 60))

        rm -f "$STATE_FILE"

        rm -f "$LAST_FILE"

        send_telegram "✅ CONTAINER RECOVERED

🖥 Server: ${HOSTNAME}

🌐 IP: ${SERVER_IP}

🕒 Time: ${TIME}

🐳 Container: ${NAME}

📈 Status:
${STATUS}

⏱ Downtime:
${MINUTES}m ${SECONDS}s"

        log "CONTAINER RECOVERED ${NAME}"

    fi

fi

done

# =========================================================
# CHECK SYSTEM
# =========================================================

check_metric \
"CPU" \
"$CPU_USAGE" \
"$CPU_LIMIT" \
"$CPU_RECOVERY" \
"$CPU_COOLDOWN" \
"🔥"

check_metric \
"RAM" \
"$RAM_USAGE" \
"$RAM_LIMIT" \
"$RAM_RECOVERY" \
"$RAM_COOLDOWN" \
"⚠️"

check_metric \
"DISK" \
"$DISK_USAGE" \
"$DISK_LIMIT" \
"$DISK_RECOVERY" \
"$DISK_COOLDOWN" \
"💽"

# =========================================================
# CLEAN OLD LOG
# =========================================================

find "$LOG_DIR" -type f -mtime +7 -delete
```

---

# 5. Permission

```bash
sudo chmod +x /opt/server-monitor/monitor.sh
```

---

# 6. systemd service

```bash
sudo nano /etc/systemd/system/server-monitor.service
```

```ini
[Unit]
Description=Server Monitor

[Service]
Type=oneshot
ExecStart=/opt/server-monitor/monitor.sh
```

---

# 7. systemd timer

```bash
sudo nano /etc/systemd/system/server-monitor.timer
```

```ini
[Unit]
Description=Run Server Monitor Every Minute

[Timer]
OnBootSec=1min
OnUnitActiveSec=1min
Unit=server-monitor.service

[Install]
WantedBy=timers.target
```

---

# 8. Enable

```bash
sudo systemctl daemon-reload

sudo systemctl enable --now server-monitor.timer
```

---

# 9. Test

```bash
sudo /opt/server-monitor/monitor.sh
```

---

# 10. ตรวจสอบ Timer

```bash
systemctl list-timers | grep server-monitor
```

---

# 11. ดู Log

```bash
cat /opt/server-monitor/logs/monitor.log
```
