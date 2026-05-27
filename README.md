ด้านล่างคือ Version Production Stable ล่าสุด
แก้ปัญหา:

* Ubuntu 24
* mpstat format
* CPU Per Core
* Docker unhealthy
* Recovery Alert
* Spam Alert
* Telegram timeout
* Invalid integer
* Locale issue
* systemd compatibility

เรียบร้อยแล้ว

รองรับ:

* Docker
* Docker Compose
* Bun
* Node.js
* NestJS
* ElysiaJS
* PostgreSQL
* Redis
* nginx
* n8n

---

# 1. Install Dependencies

```bash
sudo apt update

sudo apt install -y \
curl \
jq \
sysstat
```

---

# 2. Create Directory

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

CPU_LIMIT=90
CPU_CORE_LIMIT=95
CPU_RECOVERY=80

RAM_LIMIT=90
RAM_RECOVERY=80

DISK_LIMIT=90
DISK_RECOVERY=80
```

---

# 4. monitor.sh (old)

```bash
#!/bin/bash

# =========================================================
# SERVER MONITOR - PRODUCTION STABLE
# Ubuntu 22 / Ubuntu 24
# Docker / Bun / Node.js / NestJS / PostgreSQL
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
# รองรับ 4 / 8 / 16 / 32 CORE
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

    # skip invalid
    if [[ "$cpu" == "all" ]] || \
       [[ "$cpu" == "" ]]; then
        continue
    fi

    # validate idle
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

EMOJI="$5"

STATE_FILE="${STATE_DIR}/${TYPE}.state"

START_FILE="${STATE_DIR}/${TYPE}.start"

# =====================================================
# ALERT
# =====================================================

if [ "$VALUE" -ge "$LIMIT" ]; then

    if [ ! -f "$STATE_FILE" ]; then

        touch "$STATE_FILE"

        echo "$NOW" > "$START_FILE"

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

    fi

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

DOCKER_STATUS=$(systemctl is-active docker 2>/dev/null)

if [ "$DOCKER_STATUS" != "active" ]; then

    if [ ! -f "$DOCKER_STATE_FILE" ]; then

        echo "$NOW" > "$DOCKER_STATE_FILE"

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

FILE="${STATE_DIR}/container-${NAME}.state"

# =====================================================
# UNHEALTHY
# =====================================================

if [[ "$STATUS" == *"unhealthy"* ]]; then

    if [ ! -f "$FILE" ]; then

        echo "$NOW" > "$FILE"

        send_telegram "⚠️ CONTAINER UNHEALTHY

🖥 Server: ${HOSTNAME}

🌐 IP: ${SERVER_IP}

🕒 Time: ${TIME}

🐳 Container: ${NAME}

📉 Status:
${STATUS}"

        log "CONTAINER UNHEALTHY ${NAME}"

    fi

# =====================================================
# DOWN
# =====================================================

elif [[ "$STATUS" == *"Exited"* ]] || \
     [[ "$STATUS" == *"Restarting"* ]]; then

    if [ ! -f "$FILE" ]; then

        echo "$NOW" > "$FILE"

        send_telegram "🚨 CONTAINER DOWN

🖥 Server: ${HOSTNAME}

🌐 IP: ${SERVER_IP}

🕒 Time: ${TIME}

🐳 Container: ${NAME}

📉 Status:
${STATUS}"

        log "CONTAINER DOWN ${NAME}"

    fi

# =====================================================
# RECOVERY
# =====================================================

else

    if [ -f "$FILE" ]; then

        START=$(cat "$FILE")

        DURATION=$((NOW - START))

        MINUTES=$((DURATION / 60))

        SECONDS=$((DURATION % 60))

        rm -f "$FILE"

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
"🔥"

check_metric \
"RAM" \
"$RAM_USAGE" \
"$RAM_LIMIT" \
"$RAM_RECOVERY" \
"⚠️"

check_metric \
"DISK" \
"$DISK_USAGE" \
"$DISK_LIMIT" \
"$DISK_RECOVERY" \
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

# ตรวจสอบ Timer

```bash
systemctl list-timers | grep server-monitor
```

---

# ดู Log

```bash
cat /opt/server-monitor/logs/monitor.log
```

---

# ตัวอย่าง Alert

```text
🔥 CPU ALERT

🖥 Server: api-prod-01

🌐 IP: 10.0.0.15

📈 CPU: 96%

🔥 Core 2: 99%
🔥 Core 5: 97%

🔥 TOP CPU:
991 2 bun 98.1
```

---

# ตัวอย่าง Recovery

```text
✅ CPU RECOVERED

🖥 Server: api-prod-01

📉 Current CPU: 22%

⏱ Incident Duration:
12m 14s
```


