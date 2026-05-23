# 第十二階段：安全加固與後續擴充(最後階段)
第十二階段的目標不是再新增攻擊功能，而是把你目前的 Raspberry Pi deception lab 變成一個比較安全、可維護、可長期實驗的 MVP。
```
# 目前你已經完成：
[完成] Cowrie SSH honeypot
[完成] Fake Web Admin Panel
[完成] honeycredential / honeyfile
[完成] 集中式 log 收集
[完成] parser / detection rules
[完成] MITRE ATT&CK / Engage mapping
[完成] timeline / report
[完成] 攻擊情境測試

# 第十二階段會處理：
2. UFW 防火牆加固
3. Docker Compose 安全加固
4. 檔案權限整理
5. log 保留與 SD 卡保護
6. 報告與資料備份
7. 建立安全檢查腳本
8. 建立後續擴充規劃
9. 建立 PHASE12_READY.md
```

### Step 12.1：確認目前服務狀態
```
# 先確認目前服務仍然正常。
cd /opt/deception-lab
/opt/deception-lab/scripts/status_lab.sh

# 你應該確認：這三個 port 正常即可。
22/tcp    真實 SSH 管理入口
2222/tcp  Cowrie SSH honeypot
8080/tcp  Fake Web Admin Panel

------------------------------------------------------------------------
# 執行結果：
lss@lss:/opt/deception-lab $ /opt/deception-lab/scripts/status_lab.sh
=== Docker Compose Services ===
NAME                 IMAGE                    COMMAND                  SERVICE    CREATED       STATUS      PORTS
deception-cowrie     cowrie/cowrie:latest     "/cowrie/cowrie-env/…"   cowrie     10 days ago   Up 2 days   0.0.0.0:2222->2222/tcp, [::]:2222->2222/tcp, 2223/tcp
deception-fake-web   deception-lab-fake-web   "gunicorn --bind 0.0…"   fake-web   10 days ago   Up 2 days   0.0.0.0:8080->8080/tcp, [::]:8080->8080/tcp

=== Docker Networks ===
e3ff41c4fcca   deception-lab_deception_net   bridge    local

=== Listening Ports ===
[sudo] password for lss:
tcp   LISTEN 0      4096         0.0.0.0:2222       0.0.0.0:*    users:(("docker-proxy",pid=1652,fd=8))
tcp   LISTEN 0      4096         0.0.0.0:8080       0.0.0.0:*    users:(("docker-proxy",pid=1631,fd=8))
tcp   LISTEN 0      128          0.0.0.0:22         0.0.0.0:*    users:(("sshd",pid=1188,fd=6))
tcp   LISTEN 0      4096            [::]:2222          [::]:*    users:(("docker-proxy",pid=1658,fd=8))
tcp   LISTEN 0      4096            [::]:8080          [::]:*    users:(("docker-proxy",pid=1638,fd=8))
tcp   LISTEN 0      128             [::]:22            [::]:*    users:(("sshd",pid=1188,fd=7))

=== Disk Usage ===
Filesystem      Size  Used Avail Use% Mounted on
/dev/mmcblk0p2   58G  5.2G   50G  10% /

=== Log Directories ===
44K     /opt/deception-lab/data/logs/cowrie
32K     /opt/deception-lab/data/logs/web

=== Collected Logs ===
total 84K
drwxrwxr-x 2 lss lss 4.0K May 15 07:18 .
drwxrwxr-x 9 lss lss 4.0K May 22 03:50 ..
-rw-rw-r-- 1 lss lss  371 May 23 06:55 collection_summary.txt
-rw-rw-r-- 1 lss lss  38K May 23 06:55 cowrie-docker.log
-rw-rw-r-- 1 lss lss  579 May 15 07:18 README.md
-rw-rw-r-- 1 lss lss  777 May 23 06:55 source_manifest.json
-rwxrwxr-x 1 lss lss  18K May 23 06:55 web_access.jsonl
-rwxrwxr-x 1 lss lss 3.9K May 23 06:55 web_auth.jsonl

=== Latest Collection Summary ===
Collection time UTC: 20260522T225525Z

Collected files:

- /opt/deception-lab/data/collected/cowrie-docker.log
  size: 40K
  lines: 301
- /opt/deception-lab/data/collected/web_access.jsonl
  size: 20K
  lines: 58
- /opt/deception-lab/data/collected/web_auth.jsonl
  size: 4.0K
  lines: 10
- /opt/deception-lab/data/collected/source_manifest.json
  size: 4.0K
  lines: 26
```

### Step 12.2：建立安全加固筆記
先建立一份安全原則文件。
```
cat > /opt/deception-lab/SECURITY_NOTES.md <<'EOF'
# Raspberry Pi Deception Lab Security Notes

This lab is designed as a small single-node MVP deception and honeypot environment.

## Safety principles

- Do not expose this lab directly to the Internet during MVP testing.
- Keep this lab in a controlled LAN or isolated VLAN.
- Do not store real credentials in honeyfiles.
- Do not store real SSH private keys in honeyfiles.
- Do not mount the Docker socket into honeypot containers.
- Do not run honeypot containers in privileged mode.
- Do not bridge this lab into production networks.
- Keep Raspberry Pi management SSH on port 22 separate from Cowrie honeypot SSH on port 2222.
- Rotate and archive logs to protect SD card storage.
- Regularly review generated reports before sharing.

## Current exposed services

- 22/tcp: Real Raspberry Pi SSH management
- 2222/tcp: Cowrie SSH honeypot
- 8080/tcp: Fake Web Admin Panel

## Recommended network placement

- Use a dedicated lab subnet if available.
- Use a separate Wi-Fi SSID or VLAN if available.
- Do not place the device in a production server subnet.
- Do not port-forward 2222 or 8080 from the Internet router.
EOF
```
- 確認：
```
cat /opt/deception-lab/SECURITY_NOTES.md
```

### Step 12.3：檢查 UFW 防火牆狀態
- 執行：
```
sudo ufw status verbose

# 你目前應該看到類似：
Status: active
Default: deny incoming, allow outgoing
22/tcp ALLOW
2222/tcp ALLOW
8080/tcp ALLOW

# 這代表目前是：這個狀態是合理的 MVP 設定。
只允許 22、2222、8080 進來
其他 incoming 預設拒絕

------------------------------------------------------------------------
# 執行結果：
lss@lss:/opt/deception-lab $ sudo ufw status verbose
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), deny (routed)
New profiles: skip

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW IN    Anywhere
2222/tcp                   ALLOW IN    Anywhere
8080/tcp                   ALLOW IN    Anywhere
22/tcp (v6)                ALLOW IN    Anywhere (v6)
2222/tcp (v6)              ALLOW IN    Anywhere (v6)
8080/tcp (v6)              ALLOW IN    Anywhere (v6)
```

### Step 12.4：建立更安全的 UFW 備份腳本
- 先建立防火牆規則備份。
```
mkdir -p /opt/deception-lab/backups/ufw
```
```
cat > /opt/deception-lab/scripts/backup_ufw_rules.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

BACKUP_DIR="/opt/deception-lab/backups/ufw"
TS="$(date -u +"%Y%m%dT%H%M%SZ")"

mkdir -p "$BACKUP_DIR"

sudo ufw status verbose > "$BACKUP_DIR/ufw-status-$TS.txt"
sudo cp -a /etc/ufw "$BACKUP_DIR/etc-ufw-$TS"

echo "[+] UFW rules backed up to:"
echo "$BACKUP_DIR/ufw-status-$TS.txt"
echo "$BACKUP_DIR/etc-ufw-$TS"
EOF
```
- 設定權限、執行：
```
chmod +x /opt/deception-lab/scripts/backup_ufw_rules.sh
/opt/deception-lab/scripts/backup_ufw_rules.sh

------------------------------------------------------------------------
# 執行結果：
lss@lss:/opt/deception-lab $ chmod +x /opt/deception-lab/scripts/backup_ufw_rules.sh
/opt/deception-lab/scripts/backup_ufw_rules.sh
[+] UFW rules backed up to:
/opt/deception-lab/backups/ufw/ufw-status-20260522T231217Z.txt
/opt/deception-lab/backups/ufw/etc-ufw-20260522T231217Z
```

### Step 12.5：建立 UFW 安全重設腳本
這個腳本會保留目前 MVP 需要的 port。
```
cat > /opt/deception-lab/scripts/apply_lab_firewall.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

echo "[!] This will reset UFW rules for the deception lab."
echo "[!] Allowed inbound ports will be: 22/tcp, 2222/tcp, 8080/tcp."
read -r -p "Continue? Type YES: " CONFIRM

if [ "$CONFIRM" != "YES" ]; then
  echo "Aborted."
  exit 1
fi

sudo ufw --force reset
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw default deny routed

sudo ufw allow 22/tcp comment 'Real Raspberry Pi SSH management'
sudo ufw allow 2222/tcp comment 'Cowrie SSH honeypot'
sudo ufw allow 8080/tcp comment 'Fake Web Admin Panel'

sudo ufw logging low
sudo ufw --force enable

echo
sudo ufw status verbose
EOF
```
- 設定權限：
```
chmod +x /opt/deception-lab/scripts/apply_lab_firewall.sh

# 先不要急著執行。
只有當你未來防火牆規則亂掉時，再執行：
/opt/deception-lab/scripts/apply_lab_firewall.sh
它會要求你輸入：YES  才會真的重設。
```

### Step 12.6：建立限制內網來源版本的防火牆腳本，可選
如果你想更安全，只允許你的內網 192.168.1.0/24 連到 lab，可以建立這個版本。
```
cat > /opt/deception-lab/scripts/apply_lab_firewall_lan_only.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

LAN_CIDR="192.168.1.0/24"

echo "[!] This will reset UFW rules and only allow lab access from $LAN_CIDR."
echo "[!] Allowed inbound ports from LAN: 22/tcp, 2222/tcp, 8080/tcp."
read -r -p "Continue? Type YES: " CONFIRM

if [ "$CONFIRM" != "YES" ]; then
  echo "Aborted."
  exit 1
fi

sudo ufw --force reset
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw default deny routed

sudo ufw allow from "$LAN_CIDR" to any port 22 proto tcp comment 'Real SSH from LAN only'
sudo ufw allow from "$LAN_CIDR" to any port 2222 proto tcp comment 'Cowrie from LAN only'
sudo ufw allow from "$LAN_CIDR" to any port 8080 proto tcp comment 'Fake Web from LAN only'

sudo ufw logging low
sudo ufw --force enable

echo
sudo ufw status verbose
EOF
```
- 設定權限：
```
chmod +x /opt/deception-lab/scripts/apply_lab_firewall_lan_only.sh

# 建議你目前先不要執行，除非你確定自己的測試來源都在：
192.168.1.0/24
```

### Step 12.7：檢查 Docker Compose 是否使用危險設定
- 執行：
```
grep -nE "privileged|network_mode|/var/run/docker.sock|cap_add" /opt/deception-lab/docker-compose.yml || true

# 理想狀態是沒有輸出。
如果沒有輸出，代表目前沒有看到這些高風險設定：
privileged: true
network_mode: host
/var/run/docker.sock
cap_add
```

### Step 12.8：備份 docker-compose.yml
- 執行：
```
mkdir -p /opt/deception-lab/backups/compose
cp /opt/deception-lab/docker-compose.yml \
  /opt/deception-lab/backups/compose/docker-compose-$(date -u +"%Y%m%dT%H%M%SZ").yml
```
- 確認：
```
ls -lah /opt/deception-lab/backups/compose

------------------------------------------------------------------------
# 執行結果：
lss@lss:/opt/deception-lab $ ls -lah /opt/deception-lab/backups/compose
total 12K
drwxrwxr-x 2 lss lss 4.0K May 23 07:19 .
drwxrwxr-x 4 lss lss 4.0K May 23 07:19 ..
-rw-rw-r-- 1 lss lss  991 May 23 07:19 docker-compose-20260522T231915Z.yml
```

### Step 12.9：加強 docker-compose.yml 的安全選項
你目前服務已經有 security_opt: no-new-privileges:true。
現在我們補上比較保守的 logging 限制，避免 Docker log 無限成長。
- 先備份：
```
cp /opt/deception-lab/docker-compose.yml /opt/deception-lab/docker-compose.phase11.yml
```
- 用 Python 自動加入 Docker logging 設定：
```
python3 - <<'PY'
from pathlib import Path

path = Path("/opt/deception-lab/docker-compose.yml")
text = path.read_text()

logging_block = """    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
"""

services = ["cowrie", "fake-web"]

for service in services:
    marker = f"  {service}:\n"
    if marker not in text:
        raise SystemExit(f"Service {service} not found")

if "max-size: \"10m\"" in text:
    print("[=] Logging limits already present. No change made.")
else:
    # Insert logging block before networks section in each service.
    text = text.replace(
        """    security_opt:
      - no-new-privileges:true

    depends_on:""",
        """    security_opt:
      - no-new-privileges:true
""" + logging_block + """
    depends_on:"""
    )

    text = text.replace(
        """    security_opt:
      - no-new-privileges:true

networks:""",
        """    security_opt:
      - no-new-privileges:true
""" + logging_block + """
networks:"""
    )

    path.write_text(text)
    print("[+] Logging limits added.")
PY
```
- 檢查 Compose：
```
docker compose config

------------------------------------------------------------------------
# 執行結果：
lss@lss:/opt/deception-lab $ docker compose config
name: deception-lab
services:
  cowrie:
    container_name: deception-cowrie
    environment:
      TZ: Asia/Taipei
    image: cowrie/cowrie:latest
    networks:
      deception_net: null
    ports:
      - mode: ingress
        target: 2222
        published: "2222"
        protocol: tcp
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true
    volumes:
      - type: bind
        source: /opt/deception-lab/cowrie/honeyfs
        target: /cowrie/cowrie-git/src/cowrie/data/honeyfs
        read_only: true
        bind: {}
      - type: bind
        source: /opt/deception-lab/cowrie/etc/userdb.txt
        target: /cowrie/cowrie-git/etc/userdb.txt
        read_only: true
        bind: {}
  fake-web:
    build:
      context: /opt/deception-lab/fake-web
      dockerfile: Dockerfile
    container_name: deception-fake-web
    depends_on:
      cowrie:
        condition: service_started
        required: true
    environment:
      HONEYFILE_DIR: /app/honeyfiles
      TZ: Asia/Taipei
      WEB_LOG_DIR: /app/logs
    networks:
      deception_net: null
    ports:
      - mode: ingress
        target: 8080
        published: "8080"
        protocol: tcp
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true
    volumes:
      - type: bind
        source: /opt/deception-lab/data/logs/web
        target: /app/logs
        bind: {}
      - type: bind
        source: /opt/deception-lab/fake-web/honeyfiles
        target: /app/honeyfiles
        read_only: true
        bind: {}
networks:
  deception_net:
    name: deception-lab_deception_net
    driver: bridge
```
- 如果沒有錯誤，重啟服務：
```
docker compose down
docker compose up -d

------------------------------------------------------------------------
# 執行結果：
lss@lss:/opt/deception-lab $ docker compose down
docker compose up -d
[+] down 3/3
 ✔ Container deception-fake-web        Removed                                                                                              0.5s
 ✔ Container deception-cowrie          Removed                                                                                              0.5s
 ✔ Network deception-lab_deception_net Removed                                                                                              0.2s
[+] up 3/3
 ✔ Network deception-lab_deception_net Created                                                                                              0.0s
 ✔ Container deception-cowrie          Started                                                                                              0.6s
 ✔ Container deception-fake-web        Started
```
- 確認：
```
docker compose ps

------------------------------------------------------------------------
# 執行結果：
lss@lss:/opt/deception-lab $ docker compose ps
NAME                 IMAGE                    COMMAND                  SERVICE    CREATED          STATUS          PORTS
deception-cowrie     cowrie/cowrie:latest     "/cowrie/cowrie-env/…"   cowrie     45 seconds ago   Up 44 seconds   0.0.0.0:2222->2222/tcp, [::]:2222->2222/tcp, 2223/tcp
deception-fake-web   deception-lab-fake-web   "gunicorn --bind 0.0…"   fake-web   45 seconds ago   Up 44 seconds   0.0.0.0:8080->8080/tcp, [::]:8080->8080/tcp
```

### Step 12.10：整理檔案權限
- 將專案主要檔案交給 lss 管理：
```
sudo chown -R lss:lss /opt/deception-lab
```
- 設定一般權限：
```
find /opt/deception-lab -type d -exec chmod 775 {} \;
find /opt/deception-lab -type f -exec chmod 664 {} \;
```
- 把 scripts 重新設為可執行：
```
chmod +x /opt/deception-lab/scripts/*.sh
chmod +x /opt/deception-lab/parser/*.py
```
- 確認：
```
ls -lah /opt/deception-lab/scripts

# 你應該看到腳本有 x 權限，例如：-rwxrwxr-x

------------------------------------------------------------------------
# 執行結果：
lss@lss:/opt/deception-lab $ ls -lah /opt/deception-lab/scripts
total 92K
drwxrwxr-x 2 lss lss 4.0K May 23 07:16 .
drwxrwxr-x 9 lss lss 4.0K May 23 07:21 ..
-rwxrwxr-x 1 lss lss  787 May 23 07:16 apply_lab_firewall_lan_only.sh
-rwxrwxr-x 1 lss lss  642 May 23 07:13 apply_lab_firewall.sh
-rwxrwxr-x 1 lss lss  352 May 23 07:11 backup_ufw_rules.sh
-rwxrwxr-x 1 lss lss 1012 May 13 02:48 check_assets.sh
-rwxrwxr-x 1 lss lss  512 May 11 05:20 check_env.sh
-rwxrwxr-x 1 lss lss  759 May 23 06:49 check_phase11_results.sh
-rwxrwxr-x 1 lss lss  519 May 15 07:14 cleanup_old_logs.sh
-rwxrwxr-x 1 lss lss 3.0K May 15 06:52 collect_logs.sh
-rwxrwxr-x 1 lss lss  678 May 22 03:09 generate_report.sh
-rwxrwxr-x 1 lss lss  127 May 11 19:52 logs_lab.sh
-rwxrwxr-x 1 lss lss  203 May 11 19:53 restart_lab.sh
-rwxrwxr-x 1 lss lss  880 May 21 08:12 run_mapping.sh
-rwxrwxr-x 1 lss lss  529 May 20 03:26 run_parser.sh
-rwxrwxr-x 1 lss lss  972 May 15 07:09 show_collected_logs.sh
-rwxrwxr-x 1 lss lss  859 May 20 03:42 show_events.sh
-rwxrwxr-x 1 lss lss 1.1K May 21 08:03 show_mapping.sh
-rwxrwxr-x 1 lss lss  616 May 22 03:20 show_report.sh
-rwxrwxr-x 1 lss lss  181 May 11 19:47 start_lab.sh
-rwxrwxr-x 1 lss lss  637 May 15 07:21 status_lab.sh
-rwxrwxr-x 1 lss lss  151 May 11 19:49 stop_lab.sh
-rwxrwxr-x 1 lss lss 1.3K May 23 06:09 test_web_scenarios.sh
```

### Step 12.11：建立 log retention 檢查
- 你前面已經建立過：
```
/opt/deception-lab/scripts/cleanup_old_logs.sh

------------------------------------------------------------------------
# 執行結果：
lss@lss:/opt/deception-lab $ /opt/deception-lab/scripts/cleanup_old_logs.sh
=== Disk usage before cleanup ===
Filesystem      Size  Used Avail Use% Mounted on
/dev/mmcblk0p2   58G  5.2G   50G  10% /

1.2M    /opt/deception-lab/data

=== Removing archive directories older than 14 days ===

=== Disk usage after cleanup ===
Filesystem      Size  Used Avail Use% Mounted on
/dev/mmcblk0p2   58G  5.2G   50G  10% /

1.2M    /opt/deception-lab/data
```
- 現在確認它存在並可執行：
```
ls -l /opt/deception-lab/scripts/cleanup_old_logs.sh

------------------------------------------------------------------------
# 執行結果：
lss@lss:/opt/deception-lab $ ls -l /opt/deception-lab/scripts/cleanup_old_logs.sh
-rwxrwxr-x 1 lss lss 519 May 15 07:14 /opt/deception-lab/scripts/cleanup_old_logs.sh
```
- 執行一次：
```
/opt/deception-lab/scripts/cleanup_old_logs.sh

------------------------------------------------------------------------
# 執行結果：
lss@lss:/opt/deception-lab $ /opt/deception-lab/scripts/cleanup_old_logs.sh
=== Disk usage before cleanup ===
Filesystem      Size  Used Avail Use% Mounted on
/dev/mmcblk0p2   58G  5.2G   50G  10% /

1.2M    /opt/deception-lab/data

=== Removing archive directories older than 14 days ===

=== Disk usage after cleanup ===
Filesystem      Size  Used Avail Use% Mounted on
/dev/mmcblk0p2   58G  5.2G   50G  10% /

1.2M    /opt/deception-lab/data
```

### Step 12.12：建立每週手動維護腳本
```
這個腳本會做：
1. 顯示系統狀態
2. 清理舊 archive
3. 重新產生報告
4. 顯示磁碟用量
```
```
cat > /opt/deception-lab/scripts/weekly_maintenance.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

echo "=== Deception Lab Weekly Maintenance ==="
date -u

echo
echo "=== Status ==="
/opt/deception-lab/scripts/status_lab.sh

echo
echo "=== Cleanup old logs ==="
/opt/deception-lab/scripts/cleanup_old_logs.sh

echo
echo "=== Generate report ==="
/opt/deception-lab/scripts/generate_report.sh

echo
echo "=== Disk usage ==="
df -h /
du -sh /opt/deception-lab || true
du -sh /opt/deception-lab/data || true
du -sh /opt/deception-lab/reports || true

echo
echo "=== Done ==="
EOF
```
- 設定權限、執行：
```
chmod +x /opt/deception-lab/scripts/weekly_maintenance.sh
/opt/deception-lab/scripts/weekly_maintenance.sh

# 如果執行太久，可以等它跑完；這會重新產生報告。

------------------------------------------------------------------------
# 執行結果：
lss@lss:/opt/deception-lab $ /opt/deception-lab/scripts/weekly_maintenance.sh
=== Deception Lab Weekly Maintenance ===
Fri 22 May 23:31:29 UTC 2026

=== Status ===
=== Docker Compose Services ===
NAME                 IMAGE                    COMMAND                  SERVICE    CREATED         STATUS         PORTS
deception-cowrie     cowrie/cowrie:latest     "/cowrie/cowrie-env/…"   cowrie     7 minutes ago   Up 7 minutes   0.0.0.0:2222->2222/tcp, [::]:2222->2222/tcp, 2223/tcp
deception-fake-web   deception-lab-fake-web   "gunicorn --bind 0.0…"   fake-web   7 minutes ago   Up 7 minutes   0.0.0.0:8080->8080/tcp, [::]:8080->8080/tcp

=== Docker Networks ===
8f795f18a6a1   deception-lab_deception_net   bridge    local

=== Listening Ports ===
tcp   LISTEN 0      4096         0.0.0.0:2222       0.0.0.0:*    users:(("docker-proxy",pid=7376,fd=8))
tcp   LISTEN 0      4096         0.0.0.0:8080       0.0.0.0:*    users:(("docker-proxy",pid=7458,fd=8))
tcp   LISTEN 0      128          0.0.0.0:22         0.0.0.0:*    users:(("sshd",pid=1188,fd=6))
tcp   LISTEN 0      4096            [::]:2222          [::]:*    users:(("docker-proxy",pid=7384,fd=8))
tcp   LISTEN 0      4096            [::]:8080          [::]:*    users:(("docker-proxy",pid=7464,fd=8))
tcp   LISTEN 0      128             [::]:22            [::]:*    users:(("sshd",pid=1188,fd=7))

=== Disk Usage ===
Filesystem      Size  Used Avail Use% Mounted on
/dev/mmcblk0p2   58G  5.2G   50G  10% /

=== Log Directories ===
8.0K    /opt/deception-lab/data/logs/cowrie
32K     /opt/deception-lab/data/logs/web

=== Collected Logs ===
total 48K
drwxrwxr-x 2 lss lss 4.0K May 15 07:18 .
drwxrwxr-x 9 lss lss 4.0K May 22 03:50 ..
-rw-rw-r-- 1 lss lss  371 May 23 07:31 collection_summary.txt
-rw-rw-r-- 1 lss lss 2.0K May 23 07:31 cowrie-docker.log
-rw-rw-r-- 1 lss lss  579 May 15 07:18 README.md
-rw-rw-r-- 1 lss lss  777 May 23 07:31 source_manifest.json
-rw-rw-r-- 1 lss lss  18K May 23 07:31 web_access.jsonl
-rw-rw-r-- 1 lss lss 3.9K May 23 07:31 web_auth.jsonl

=== Latest Collection Summary ===
Collection time UTC: 20260522T233122Z

Collected files:

- /opt/deception-lab/data/collected/cowrie-docker.log
  size: 4.0K
  lines: 15
- /opt/deception-lab/data/collected/web_access.jsonl
  size: 20K
  lines: 58
- /opt/deception-lab/data/collected/web_auth.jsonl
  size: 4.0K
  lines: 10
- /opt/deception-lab/data/collected/source_manifest.json
  size: 4.0K
  lines: 26

=== Cleanup old logs ===
=== Disk usage before cleanup ===
Filesystem      Size  Used Avail Use% Mounted on
/dev/mmcblk0p2   58G  5.2G   50G  10% /

1.1M    /opt/deception-lab/data

=== Removing archive directories older than 14 days ===

=== Disk usage after cleanup ===
Filesystem      Size  Used Avail Use% Mounted on
/dev/mmcblk0p2   58G  5.2G   50G  10% /

1.1M    /opt/deception-lab/data

=== Generate report ===
[+] Refreshing parser and mapping outputs...
[+] Running parser first to refresh detections...
[+] Collecting latest logs...
[+] Collecting logs at UTC time: 20260522T233129Z
[+] Exporting Cowrie Docker logs...
[+] Collecting Fake Web access log...
[+] Collecting Fake Web auth log...
[+] Creating source manifest...
[+] Creating collection summary...
[+] Archiving collected logs...

[+] Collection completed.
[+] Current collected directory:
total 48K
drwxrwxr-x 2 lss lss 4.0K May 15 07:18 .
drwxrwxr-x 9 lss lss 4.0K May 22 03:50 ..
-rw-rw-r-- 1 lss lss  371 May 23 07:31 collection_summary.txt
-rw-rw-r-- 1 lss lss 2.0K May 23 07:31 cowrie-docker.log
-rw-rw-r-- 1 lss lss  579 May 15 07:18 README.md
-rw-rw-r-- 1 lss lss  777 May 23 07:31 source_manifest.json
-rw-rw-r-- 1 lss lss  18K May 23 07:31 web_access.jsonl
-rw-rw-r-- 1 lss lss 3.9K May 23 07:31 web_auth.jsonl

[+] Summary:
Collection time UTC: 20260522T233129Z

Collected files:

- /opt/deception-lab/data/collected/cowrie-docker.log
  size: 4.0K
  lines: 15
- /opt/deception-lab/data/collected/web_access.jsonl
  size: 20K
  lines: 58
- /opt/deception-lab/data/collected/web_auth.jsonl
  size: 4.0K
  lines: 10
- /opt/deception-lab/data/collected/source_manifest.json
  size: 4.0K
  lines: 26

[+] Running event parser...
[+] Parsing completed.
[+] Events written to: /opt/deception-lab/data/events/events.jsonl
[+] Detections written to: /opt/deception-lab/data/events/detections.jsonl
[+] Summary written to: /opt/deception-lab/data/events/events_summary.json

Total events: 68
Total detections: 31

[+] Event output files:
total 72K
drwxrwxr-x 2 lss lss 4.0K May 21 07:53 .
drwxrwxr-x 9 lss lss 4.0K May 22 03:50 ..
-rw-rw-r-- 1 lss lss 2.9K May 23 07:31 attack_coverage.json
-rw-rw-r-- 1 lss lss  20K May 23 07:31 detections.jsonl
-rw-rw-r-- 1 lss lss 2.2K May 23 07:31 engage_coverage.json
-rw-rw-r-- 1 lss lss  24K May 23 07:31 events.jsonl
-rw-rw-r-- 1 lss lss  456 May 23 07:31 events_summary.json
-rw-rw-r-- 1 lss lss 6.0K May 23 07:31 mapping_summary.json

[+] Event summary:
{
  "detections_by_rule": {
    "WEB_HONEYCREDENTIAL_USED": 5,
    "WEB_HONEYFILE_ACCESS": 11,
    "WEB_SCANNER_PROBE": 15
  },
  "detections_by_severity": {
    "high": 16,
    "medium": 15
  },
  "events_by_source": {
    "fake-web": 68
  },
  "events_by_type": {
    "web_honeyfile_access": 11,
    "web_login_attempt": 10,
    "web_request": 47
  },
  "generated_at": "2026-05-22T23:31:29.775520+00:00",
  "total_detections": 31,
  "total_events": 68
}
[+] Running MITRE mapping analyzer...
[+] Mapping analysis completed.
[+] Mapping summary: /opt/deception-lab/data/events/mapping_summary.json
[+] ATT&CK coverage: /opt/deception-lab/data/events/attack_coverage.json
[+] Engage coverage: /opt/deception-lab/data/events/engage_coverage.json

Defined detection rules: 8
Total detections: 31
ATT&CK mapping coverage: 100.0%
Engage mapping coverage: 100.0%

[+] Generating mapping Markdown report...
[+] Mapping markdown report written to: /opt/deception-lab/reports/mapping_report.md

[+] Mapping output files:
-rw-rw-r-- 1 lss lss 2.9K May 23 07:31 attack_coverage.json
-rw-rw-r-- 1 lss lss 2.2K May 23 07:31 engage_coverage.json
-rw-rw-r-- 1 lss lss 6.0K May 23 07:31 mapping_summary.json

[+] Mapping report:
-rw-rw-r-- 1 lss lss 2.0K May 23 07:31 /opt/deception-lab/reports/mapping_report.md

[+] Mapping summary:
{
  "attack_mapping_coverage": {
    "coverage_percent": 100.0,
    "mapped_defined_rules": 8,
    "total_defined_rules": 8,
    "triggered_without_mapping": [],
    "unmapped_defined_rules": []
  },
  "defined_detection_rules": 8,
  "detections_by_rule": {
    "WEB_HONEYCREDENTIAL_USED": 5,
    "WEB_HONEYFILE_ACCESS": 11,
    "WEB_SCANNER_PROBE": 15
  },
  "engage_mapping_coverage": {
    "coverage_percent": 100.0,
    "mapped_defined_rules": 8,
    "total_defined_rules": 8,
    "triggered_without_mapping": [],
    "unmapped_defined_rules": []
  },
  "generated_at": "2026-05-22T23:31:29.853235+00:00",
  "input_detections": "/opt/deception-lab/data/events/detections.jsonl",
  "per_rule": {
    "SSH_HONEYCREDENTIAL_LOGIN": {
      "attack": {
        "rationale": "The attacker used a credential intentionally planted in deception assets. This indicates exposure to unsecured credential material.\n",
        "tactic": "Credential Access",
        "technique": "Unsecured Credentials",
        "technique_id": "T1552"
      },
      "detections": 0,
      "engage": {
        "activity": "Credential Collection",
        "goal": "Elicit",
        "rationale": "A planted SSH credential caused the adversary to reveal credential-use behavior.\n"
      },
      "has_attack_mapping": true,
      "has_engage_mapping": true
    },
    "SSH_HONEYFILE_ACCESS": {
      "attack": {
        "rationale": "Attempting to read honeyfiles such as secrets.txt or backup_config.ini indicates interest in local sensitive data.\n",
        "tactic": "Collection",
        "technique": "Data from Local System",
        "technique_id": "T1005"
      },
      "detections": 0,
      "engage": {
        "activity": "Reveal Adversary Intent",
        "goal": "Elicit",
        "rationale": "Honeyfile access attempts reveal interest in sensitive files, backup data, credentials, and local configuration.\n"
      },
      "has_attack_mapping": true,
      "has_engage_mapping": true
    },
    "SSH_LOGIN_FAILED": {
      "attack": {
        "rationale": "Failed SSH authentication attempts against the honeypot indicate password guessing or brute-force behavior.\n",
        "tactic": "Credential Access",
        "technique": "Brute Force",
        "technique_id": "T1110"
      },
      "detections": 0,
      "engage": {
        "activity": "Expose Decoy Service",
        "goal": "Expose",
        "rationale": "The SSH honeypot exposes a fake login surface for observing authentication attempts.\n"
      },
      "has_attack_mapping": true,
      "has_engage_mapping": true
    },
    "SSH_RECON_COMMAND": {
      "attack": {
        "rationale": "Commands such as whoami, pwd, ls, uname, ip addr, ifconfig, ps, and netstat indicate post-login discovery activity.\n",
        "related_techniques": [
          "T1033 System Owner/User Discovery",
          "T1016 System Network Configuration Discovery",
          "T1057 Process Discovery"
        ],
        "tactic": "Discovery",
        "technique": "System Information Discovery",
        "technique_id": "T1082"
      },
      "detections": 0,
      "engage": {
        "activity": "Collect Adversary Behavior",
        "goal": "Understand",
        "rationale": "Commands entered after login reveal adversary discovery interests and operating style.\n"
      },
      "has_attack_mapping": true,
      "has_engage_mapping": true
    },
    "SSH_TOOL_TRANSFER_COMMAND": {
      "attack": {
        "rationale": "Commands such as wget, curl, ftp, scp, chmod, nc, or bash reverse shell patterns may indicate tool staging or payload transfer.\n",
        "tactic": "Command and Control",
        "technique": "Ingress Tool Transfer",
        "technique_id": "T1105"
      },
      "detections": 0,
      "engage": {
        "activity": "Adversary Direction",
        "goal": "Affect",
        "rationale": "A decoy shell can influence attacker behavior and redirect tool staging activity into a controlled observation environment.\n"
      },
      "has_attack_mapping": true,
      "has_engage_mapping": true
    },
    "WEB_HONEYCREDENTIAL_USED": {
      "attack": {
        "rationale": "Submitting a planted credential to the fake web login panel indicates the adversary discovered and attempted to use unsecured credentials.\n",
        "tactic": "Credential Access",
        "technique": "Unsecured Credentials",
        "technique_id": "T1552"
      },
      "detections": 5,
      "engage": {
        "activity": "Credential Collection",
        "goal": "Elicit",
        "rationale": "Planted web credentials elicit credential reuse behavior against the fake admin panel.\n"
      },
      "has_attack_mapping": true,
      "has_engage_mapping": true
    },
    "WEB_HONEYFILE_ACCESS": {
      "attack": {
        "rationale": "Downloading honeyfiles from the fake web panel indicates attempted collection of local or exposed files.\n",
        "tactic": "Collection",
        "technique": "Data from Local System",
        "technique_id": "T1005"
      },
      "detections": 11,
      "engage": {
        "activity": "Collect Adversary Behavior",
        "goal": "Understand",
        "rationale": "Honeyfile download behavior helps identify the attacker's collection interests.\n"
      },
      "has_attack_mapping": true,
      "has_engage_mapping": true
    },
    "WEB_SCANNER_PROBE": {
      "attack": {
        "rationale": "Requests to paths such as /.env, /wp-admin, /phpmyadmin, or /admin indicate web probing or scanning behavior.\n",
        "tactic": "Reconnaissance",
        "technique": "Active Scanning",
        "technique_id": "T1595"
      },
      "detections": 15,
      "engage": {
        "activity": "Expose Decoy Service",
        "goal": "Expose",
        "rationale": "The fake web interface exposes probeable paths to observe scanner behavior.\n"
      },
      "has_attack_mapping": true,
      "has_engage_mapping": true
    }
  },
  "total_detections": 31,
  "triggered_detection_rules": [
    "WEB_HONEYCREDENTIAL_USED",
    "WEB_HONEYFILE_ACCESS",
    "WEB_SCANNER_PROBE"
  ]
}

[+] Generating timeline and final reports...
/opt/deception-lab/parser/generate_timeline.py:93: DeprecationWarning: datetime.datetime.utcnow() is deprecated and scheduled for removal in a future version. Use timezone-aware objects to represent datetimes in UTC: datetime.datetime.now(datetime.UTC).
  lines.append(f"Generated at: {datetime.utcnow().isoformat()}Z")
[+] Timeline written to: /opt/deception-lab/reports/timeline.md
[+] Markdown report written to: /opt/deception-lab/reports/report.md
[+] JSON report written to: /opt/deception-lab/reports/report.json

[+] Report files:
total 52K
drwxrwxr-x 2 lss lss 4.0K May 22 03:11 .
drwxrwxr-x 9 lss lss 4.0K May 23 07:21 ..
-rw-rw-r-- 1 lss lss 2.0K May 23 07:31 mapping_report.md
-rw-rw-r-- 1 lss lss  12K May 23 07:31 report.json
-rw-rw-r-- 1 lss lss 5.4K May 23 07:31 report.md
-rw-rw-r-- 1 lss lss  18K May 23 07:31 timeline.md

[+] Report generation completed.
Main report: /opt/deception-lab/reports/report.md
Timeline:    /opt/deception-lab/reports/timeline.md
JSON report: /opt/deception-lab/reports/report.json

=== Disk usage ===
Filesystem      Size  Used Avail Use% Mounted on
/dev/mmcblk0p2   58G  5.2G   50G  10% /
17M     /opt/deception-lab
1.2M    /opt/deception-lab/data
48K     /opt/deception-lab/reports

=== Done ===
```

### Step 12.13：建立 SD 卡壽命保護建議文件
```
cat > /opt/deception-lab/SD_CARD_NOTES.md <<'EOF'
# SD Card Notes

This Raspberry Pi deception lab writes logs and reports to the SD card.

Recommendations:

- Use at least a 64 GB high-endurance microSD card.
- Keep Docker logs limited with max-size and max-file.
- Run cleanup_old_logs.sh periodically.
- Avoid exposing the lab to uncontrolled high-volume scanning.
- Back up reports and important configuration files.
- Consider moving /opt/deception-lab/data to USB SSD for longer experiments.

Current log retention script:

- /opt/deception-lab/scripts/cleanup_old_logs.sh

Current report generation script:

- /opt/deception-lab/scripts/generate_report.sh
EOF
```
- 確認：
```
cat /opt/deception-lab/SD_CARD_NOTES.md
```

### Step 12.14：建立完整專案備份腳本
這個腳本會備份設定、scripts、parser、reports，但不備份大型 Docker image。
```
mkdir -p /opt/deception-lab/backups/project
```
```
cat > /opt/deception-lab/scripts/backup_project.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

ROOT="/opt/deception-lab"
BACKUP_DIR="$ROOT/backups/project"
TS="$(date -u +"%Y%m%dT%H%M%SZ")"
OUT="$BACKUP_DIR/deception-lab-backup-$TS.tar.gz"

mkdir -p "$BACKUP_DIR"

tar \
  --exclude="$ROOT/backups" \
  --exclude="$ROOT/parser/venv" \
  --exclude="$ROOT/data/archive" \
  -czf "$OUT" \
  -C /opt deception-lab

echo "[+] Backup created:"
ls -lh "$OUT"
EOF
```
- 設定權限、執行：
```
chmod +x /opt/deception-lab/scripts/backup_project.sh
/opt/deception-lab/scripts/backup_project.sh

------------------------------------------------------------------------
# 執行結果：
lss@lss:/opt/deception-lab $ chmod +x /opt/deception-lab/scripts/backup_project.sh
lss@lss:/opt/deception-lab $ /opt/deception-lab/scripts/backup_project.sh
tar: deception-lab/backups/project/deception-lab-backup-20260522T233607Z.tar.gz: file changed as we read it
```
- 確認：
```
ls -lah /opt/deception-lab/backups/project

------------------------------------------------------------------------
# 執行結果：
lss@lss:/opt/deception-lab $ ls -lah /opt/deception-lab/backups/project
total 8.3M
drwxrwxr-x 2 lss lss 4.0K May 23 07:36 .
drwxrwxr-x 5 lss lss 4.0K May 23 07:34 ..
-rw-rw-r-- 1 lss lss 8.3M May 23 07:36 deception-lab-backup-20260522T233607Z.tar.gz
```

### Step 12.15：建立安全檢查腳本
```
cat > /opt/deception-lab/scripts/security_check.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

ROOT="/opt/deception-lab"

echo "=== Security Check: Services ==="
docker compose -f "$ROOT/docker-compose.yml" ps

echo
echo "=== Security Check: Listening Ports ==="
sudo ss -tulpn | grep -E ':22|:2222|:8080' || true

echo
echo "=== Security Check: UFW ==="
sudo ufw status verbose

echo
echo "=== Security Check: Dangerous Docker Compose Settings ==="
grep -nE "privileged|network_mode|/var/run/docker.sock|cap_add" "$ROOT/docker-compose.yml" || echo "[OK] No obvious dangerous settings found."

echo
echo "=== Security Check: Docker Logging Limits ==="
grep -nA4 "logging:" "$ROOT/docker-compose.yml" || echo "[WARN] No logging limit found."

echo
echo "=== Security Check: Important File Permissions ==="
ls -ld "$ROOT"
ls -ld "$ROOT/data"
ls -ld "$ROOT/scripts"
ls -l "$ROOT/scripts"/*.sh | head

echo
echo "=== Security Check: Disk Usage ==="
df -h /
du -sh "$ROOT" || true
du -sh "$ROOT/data" || true

echo
echo "=== Security Check: Reports ==="
ls -lah "$ROOT/reports" || true

echo
echo "=== Security Check Completed ==="
EOF
```
- 設定權限、執行：
```
chmod +x /opt/deception-lab/scripts/security_check.sh
/opt/deception-lab/scripts/security_check.sh

------------------------------------------------------------------------
# 執行結果：
lss@lss:/opt/deception-lab $ chmod +x /opt/deception-lab/scripts/security_check.sh
lss@lss:/opt/deception-lab $ /opt/deception-lab/scripts/security_check.sh
=== Security Check: Services ===
NAME                 IMAGE                    COMMAND                  SERVICE    CREATED          STATUS          PORTS
deception-cowrie     cowrie/cowrie:latest     "/cowrie/cowrie-env/…"   cowrie     15 minutes ago   Up 15 minutes   0.0.0.0:2222->2222/tcp, [::]:2222->2222/tcp, 2223/tcp
deception-fake-web   deception-lab-fake-web   "gunicorn --bind 0.0…"   fake-web   15 minutes ago   Up 15 minutes   0.0.0.0:8080->8080/tcp, [::]:8080->8080/tcp

=== Security Check: Listening Ports ===
tcp   LISTEN 0      4096         0.0.0.0:2222       0.0.0.0:*    users:(("docker-proxy",pid=7376,fd=8))
tcp   LISTEN 0      4096         0.0.0.0:8080       0.0.0.0:*    users:(("docker-proxy",pid=7458,fd=8))
tcp   LISTEN 0      128          0.0.0.0:22         0.0.0.0:*    users:(("sshd",pid=1188,fd=6))
tcp   LISTEN 0      4096            [::]:2222          [::]:*    users:(("docker-proxy",pid=7384,fd=8))
tcp   LISTEN 0      4096            [::]:8080          [::]:*    users:(("docker-proxy",pid=7464,fd=8))
tcp   LISTEN 0      128             [::]:22            [::]:*    users:(("sshd",pid=1188,fd=7))

=== Security Check: UFW ===
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), deny (routed)
New profiles: skip

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW IN    Anywhere
2222/tcp                   ALLOW IN    Anywhere
8080/tcp                   ALLOW IN    Anywhere
22/tcp (v6)                ALLOW IN    Anywhere (v6)
2222/tcp (v6)              ALLOW IN    Anywhere (v6)
8080/tcp (v6)              ALLOW IN    Anywhere (v6)


=== Security Check: Dangerous Docker Compose Settings ===
[OK] No obvious dangerous settings found.

=== Security Check: Docker Logging Limits ===
[WARN] No logging limit found.

=== Security Check: Important File Permissions ===
drwxrwxr-x 9 lss lss 4096 May 23 07:32 /opt/deception-lab
drwxrwxr-x 9 lss lss 4096 May 22 03:50 /opt/deception-lab/data
drwxrwxr-x 2 lss lss 4096 May 23 07:38 /opt/deception-lab/scripts
-rwxrwxr-x 1 lss lss  787 May 23 07:16 /opt/deception-lab/scripts/apply_lab_firewall_lan_only.sh
-rwxrwxr-x 1 lss lss  642 May 23 07:13 /opt/deception-lab/scripts/apply_lab_firewall.sh
-rwxrwxr-x 1 lss lss  396 May 23 07:35 /opt/deception-lab/scripts/backup_project.sh
-rwxrwxr-x 1 lss lss  352 May 23 07:11 /opt/deception-lab/scripts/backup_ufw_rules.sh
-rwxrwxr-x 1 lss lss 1012 May 13 02:48 /opt/deception-lab/scripts/check_assets.sh
-rwxrwxr-x 1 lss lss  512 May 11 05:20 /opt/deception-lab/scripts/check_env.sh
-rwxrwxr-x 1 lss lss  759 May 23 06:49 /opt/deception-lab/scripts/check_phase11_results.sh
-rwxrwxr-x 1 lss lss  519 May 15 07:14 /opt/deception-lab/scripts/cleanup_old_logs.sh
-rwxrwxr-x 1 lss lss 3019 May 15 06:52 /opt/deception-lab/scripts/collect_logs.sh
-rwxrwxr-x 1 lss lss  678 May 22 03:09 /opt/deception-lab/scripts/generate_report.sh

=== Security Check: Disk Usage ===
Filesystem      Size  Used Avail Use% Mounted on
/dev/mmcblk0p2   58G  5.2G   50G  10% /
26M     /opt/deception-lab
1.2M    /opt/deception-lab/data

=== Security Check: Reports ===
total 52K
drwxrwxr-x 2 lss lss 4.0K May 22 03:11 .
drwxrwxr-x 9 lss lss 4.0K May 23 07:32 ..
-rw-rw-r-- 1 lss lss 2.0K May 23 07:31 mapping_report.md
-rw-rw-r-- 1 lss lss  12K May 23 07:31 report.json
-rw-rw-r-- 1 lss lss 5.4K May 23 07:31 report.md
-rw-rw-r-- 1 lss lss  18K May 23 07:31 timeline.md

=== Security Check Completed ===
```

### Step 12.16：建立後續擴充規劃
- 執行
```
cat > /opt/deception-lab/FUTURE_WORK.md <<'EOF'
# Future Work

This document lists possible future extensions for the Raspberry Pi deception lab.

## 1. Additional honeypot services

Possible services:

- HTTP API honeypot
- FTP honeypot
- SMB decoy share
- MQTT honeypot for IoT scenarios
- Modbus or ICS-like decoy endpoint in a closed lab network

## 2. Better log pipeline

Possible improvements:

- Replace Docker log export with structured Cowrie JSON log
- Add syslog collector
- Add lightweight local SQLite event database
- Export reports to remote storage
- Add log signing or hash chain for evidence integrity

## 3. Better dashboards

Possible improvements:

- Local Streamlit dashboard
- Grafana + Loki
- Lightweight Flask dashboard
- Timeline visualization
- MITRE coverage heatmap

## 4. Better deception content

Possible improvements:

- More realistic fake file trees
- Fake SSH history
- Fake cloud credentials
- Fake API tokens
- Fake configuration backup
- Role-based honeycredentials

## 5. Safer deployment

Possible improvements:

- Move lab to isolated VLAN
- Use USB SSD instead of microSD for logs
- Use LAN-only firewall rules
- Add scheduled backup
- Add health check alerts

## 6. Research extensions

Possible research directions:

- Compare attacker behavior across deception asset placement
- Evaluate honeycredential effectiveness
- Study attacker command sequence patterns
- Generate defensive knowledge from observed behavior
- Map deception actions to MITRE Engage goals
EOF
```
- 確認：
```
cat /opt/deception-lab/FUTURE_WORK.md
```

### Step 12.17：建立最終完成報告索引
- 執行
```
cat > /opt/deception-lab/FINAL_INDEX.md <<'EOF'
# Raspberry Pi Deception Lab MVP Final Index

## Core services

- Cowrie SSH honeypot
  - Port: 2222/tcp

- Fake Web Admin Panel
  - Port: 8080/tcp

- Real Raspberry Pi SSH management
  - Port: 22/tcp

## Main commands

Start lab:

```bash
/opt/deception-lab/scripts/start_lab.sh
/opt/deception-lab/scripts/stop_lab.sh
/opt/deception-lab/scripts/status_lab.sh
/opt/deception-lab/scripts/run_parser.sh
/opt/deception-lab/scripts/run_mapping.sh
/opt/deception-lab/scripts/generate_report.sh
/opt/deception-lab/scripts/security_check.sh
/opt/deception-lab/scripts/backup_project.sh

Main output files
/opt/deception-lab/reports/report.md
/opt/deception-lab/reports/report.json
/opt/deception-lab/reports/timeline.md
/opt/deception-lab/reports/mapping_report.md
Main event files
/opt/deception-lab/data/events/events.jsonl
/opt/deception-lab/data/events/detections.jsonl
/opt/deception-lab/data/events/events_summary.json
/opt/deception-lab/data/events/mapping_summary.json
/opt/deception-lab/data/events/attack_coverage.json
/opt/deception-lab/data/events/engage_coverage.json
Documentation
/opt/deception-lab/README.md
/opt/deception-lab/SECURITY_NOTES.md
/opt/deception-lab/SD_CARD_NOTES.md
/opt/deception-lab/FUTURE_WORK.md
EOF
```

### Step 12.18：更新 README 加入最後指令
```
cat >> /opt/deception-lab/README.md <<'EOF'

## Phase 12 Security and Maintenance

Run security check:

```bash
/opt/deception-lab/scripts/security_check.sh
/opt/deception-lab/scripts/weekly_maintenance.sh
/opt/deception-lab/scripts/backup_project.sh
/opt/deception-lab/SECURITY_NOTES.md
/opt/deception-lab/SD_CARD_NOTES.md
/opt/deception-lab/FUTURE_WORK.md
/opt/deception-lab/FINAL_INDEX.md
EOF
```

### Step 12.19：產生最終報告
```
/opt/deception-lab/scripts/generate_report.sh

------------------------------------------------------------------------
# 執行結果：
lss@lss:/opt/deception-lab $ /opt/deception-lab/scripts/generate_report.sh
[+] Refreshing parser and mapping outputs...
[+] Running parser first to refresh detections...
[+] Collecting latest logs...
[+] Collecting logs at UTC time: 20260523T023435Z
[+] Exporting Cowrie Docker logs...
[+] Collecting Fake Web access log...
[+] Collecting Fake Web auth log...
[+] Creating source manifest...
[+] Creating collection summary...
[+] Archiving collected logs...

[+] Collection completed.
[+] Current collected directory:
total 48K
drwxrwxr-x 2 lss lss 4.0K May 15 07:18 .
drwxrwxr-x 9 lss lss 4.0K May 22 03:50 ..
-rw-rw-r-- 1 lss lss  371 May 23 10:34 collection_summary.txt
-rw-rw-r-- 1 lss lss 2.0K May 23 10:34 cowrie-docker.log
-rw-rw-r-- 1 lss lss  579 May 15 07:18 README.md
-rw-rw-r-- 1 lss lss  777 May 23 10:34 source_manifest.json
-rw-rw-r-- 1 lss lss  18K May 23 10:34 web_access.jsonl
-rw-rw-r-- 1 lss lss 3.9K May 23 10:34 web_auth.jsonl

[+] Summary:
Collection time UTC: 20260523T023435Z

Collected files:

- /opt/deception-lab/data/collected/cowrie-docker.log
  size: 4.0K
  lines: 15
- /opt/deception-lab/data/collected/web_access.jsonl
  size: 20K
  lines: 58
- /opt/deception-lab/data/collected/web_auth.jsonl
  size: 4.0K
  lines: 10
- /opt/deception-lab/data/collected/source_manifest.json
  size: 4.0K
  lines: 26

[+] Running event parser...
[+] Parsing completed.
[+] Events written to: /opt/deception-lab/data/events/events.jsonl
[+] Detections written to: /opt/deception-lab/data/events/detections.jsonl
[+] Summary written to: /opt/deception-lab/data/events/events_summary.json

Total events: 68
Total detections: 31

[+] Event output files:
total 72K
drwxrwxr-x 2 lss lss 4.0K May 21 07:53 .
drwxrwxr-x 9 lss lss 4.0K May 22 03:50 ..
-rw-rw-r-- 1 lss lss 2.9K May 23 07:31 attack_coverage.json
-rw-rw-r-- 1 lss lss  20K May 23 10:34 detections.jsonl
-rw-rw-r-- 1 lss lss 2.2K May 23 07:31 engage_coverage.json
-rw-rw-r-- 1 lss lss  24K May 23 10:34 events.jsonl
-rw-rw-r-- 1 lss lss  456 May 23 10:34 events_summary.json
-rw-rw-r-- 1 lss lss 6.0K May 23 07:31 mapping_summary.json

[+] Event summary:
{
  "detections_by_rule": {
    "WEB_HONEYCREDENTIAL_USED": 5,
    "WEB_HONEYFILE_ACCESS": 11,
    "WEB_SCANNER_PROBE": 15
  },
  "detections_by_severity": {
    "high": 16,
    "medium": 15
  },
  "events_by_source": {
    "fake-web": 68
  },
  "events_by_type": {
    "web_honeyfile_access": 11,
    "web_login_attempt": 10,
    "web_request": 47
  },
  "generated_at": "2026-05-23T02:34:35.747222+00:00",
  "total_detections": 31,
  "total_events": 68
}
[+] Running MITRE mapping analyzer...
[+] Mapping analysis completed.
[+] Mapping summary: /opt/deception-lab/data/events/mapping_summary.json
[+] ATT&CK coverage: /opt/deception-lab/data/events/attack_coverage.json
[+] Engage coverage: /opt/deception-lab/data/events/engage_coverage.json

Defined detection rules: 8
Total detections: 31
ATT&CK mapping coverage: 100.0%
Engage mapping coverage: 100.0%

[+] Generating mapping Markdown report...
[+] Mapping markdown report written to: /opt/deception-lab/reports/mapping_report.md

[+] Mapping output files:
-rw-rw-r-- 1 lss lss 2.9K May 23 10:34 attack_coverage.json
-rw-rw-r-- 1 lss lss 2.2K May 23 10:34 engage_coverage.json
-rw-rw-r-- 1 lss lss 6.0K May 23 10:34 mapping_summary.json

[+] Mapping report:
-rw-rw-r-- 1 lss lss 2.0K May 23 10:34 /opt/deception-lab/reports/mapping_report.md

[+] Mapping summary:
{
  "attack_mapping_coverage": {
    "coverage_percent": 100.0,
    "mapped_defined_rules": 8,
    "total_defined_rules": 8,
    "triggered_without_mapping": [],
    "unmapped_defined_rules": []
  },
  "defined_detection_rules": 8,
  "detections_by_rule": {
    "WEB_HONEYCREDENTIAL_USED": 5,
    "WEB_HONEYFILE_ACCESS": 11,
    "WEB_SCANNER_PROBE": 15
  },
  "engage_mapping_coverage": {
    "coverage_percent": 100.0,
    "mapped_defined_rules": 8,
    "total_defined_rules": 8,
    "triggered_without_mapping": [],
    "unmapped_defined_rules": []
  },
  "generated_at": "2026-05-23T02:34:35.824378+00:00",
  "input_detections": "/opt/deception-lab/data/events/detections.jsonl",
  "per_rule": {
    "SSH_HONEYCREDENTIAL_LOGIN": {
      "attack": {
        "rationale": "The attacker used a credential intentionally planted in deception assets. This indicates exposure to unsecured credential material.\n",
        "tactic": "Credential Access",
        "technique": "Unsecured Credentials",
        "technique_id": "T1552"
      },
      "detections": 0,
      "engage": {
        "activity": "Credential Collection",
        "goal": "Elicit",
        "rationale": "A planted SSH credential caused the adversary to reveal credential-use behavior.\n"
      },
      "has_attack_mapping": true,
      "has_engage_mapping": true
    },
    "SSH_HONEYFILE_ACCESS": {
      "attack": {
        "rationale": "Attempting to read honeyfiles such as secrets.txt or backup_config.ini indicates interest in local sensitive data.\n",
        "tactic": "Collection",
        "technique": "Data from Local System",
        "technique_id": "T1005"
      },
      "detections": 0,
      "engage": {
        "activity": "Reveal Adversary Intent",
        "goal": "Elicit",
        "rationale": "Honeyfile access attempts reveal interest in sensitive files, backup data, credentials, and local configuration.\n"
      },
      "has_attack_mapping": true,
      "has_engage_mapping": true
    },
    "SSH_LOGIN_FAILED": {
      "attack": {
        "rationale": "Failed SSH authentication attempts against the honeypot indicate password guessing or brute-force behavior.\n",
        "tactic": "Credential Access",
        "technique": "Brute Force",
        "technique_id": "T1110"
      },
      "detections": 0,
      "engage": {
        "activity": "Expose Decoy Service",
        "goal": "Expose",
        "rationale": "The SSH honeypot exposes a fake login surface for observing authentication attempts.\n"
      },
      "has_attack_mapping": true,
      "has_engage_mapping": true
    },
    "SSH_RECON_COMMAND": {
      "attack": {
        "rationale": "Commands such as whoami, pwd, ls, uname, ip addr, ifconfig, ps, and netstat indicate post-login discovery activity.\n",
        "related_techniques": [
          "T1033 System Owner/User Discovery",
          "T1016 System Network Configuration Discovery",
          "T1057 Process Discovery"
        ],
        "tactic": "Discovery",
        "technique": "System Information Discovery",
        "technique_id": "T1082"
      },
      "detections": 0,
      "engage": {
        "activity": "Collect Adversary Behavior",
        "goal": "Understand",
        "rationale": "Commands entered after login reveal adversary discovery interests and operating style.\n"
      },
      "has_attack_mapping": true,
      "has_engage_mapping": true
    },
    "SSH_TOOL_TRANSFER_COMMAND": {
      "attack": {
        "rationale": "Commands such as wget, curl, ftp, scp, chmod, nc, or bash reverse shell patterns may indicate tool staging or payload transfer.\n",
        "tactic": "Command and Control",
        "technique": "Ingress Tool Transfer",
        "technique_id": "T1105"
      },
      "detections": 0,
      "engage": {
        "activity": "Adversary Direction",
        "goal": "Affect",
        "rationale": "A decoy shell can influence attacker behavior and redirect tool staging activity into a controlled observation environment.\n"
      },
      "has_attack_mapping": true,
      "has_engage_mapping": true
    },
    "WEB_HONEYCREDENTIAL_USED": {
      "attack": {
        "rationale": "Submitting a planted credential to the fake web login panel indicates the adversary discovered and attempted to use unsecured credentials.\n",
        "tactic": "Credential Access",
        "technique": "Unsecured Credentials",
        "technique_id": "T1552"
      },
      "detections": 5,
      "engage": {
        "activity": "Credential Collection",
        "goal": "Elicit",
        "rationale": "Planted web credentials elicit credential reuse behavior against the fake admin panel.\n"
      },
      "has_attack_mapping": true,
      "has_engage_mapping": true
    },
    "WEB_HONEYFILE_ACCESS": {
      "attack": {
        "rationale": "Downloading honeyfiles from the fake web panel indicates attempted collection of local or exposed files.\n",
        "tactic": "Collection",
        "technique": "Data from Local System",
        "technique_id": "T1005"
      },
      "detections": 11,
      "engage": {
        "activity": "Collect Adversary Behavior",
        "goal": "Understand",
        "rationale": "Honeyfile download behavior helps identify the attacker's collection interests.\n"
      },
      "has_attack_mapping": true,
      "has_engage_mapping": true
    },
    "WEB_SCANNER_PROBE": {
      "attack": {
        "rationale": "Requests to paths such as /.env, /wp-admin, /phpmyadmin, or /admin indicate web probing or scanning behavior.\n",
        "tactic": "Reconnaissance",
        "technique": "Active Scanning",
        "technique_id": "T1595"
      },
      "detections": 15,
      "engage": {
        "activity": "Expose Decoy Service",
        "goal": "Expose",
        "rationale": "The fake web interface exposes probeable paths to observe scanner behavior.\n"
      },
      "has_attack_mapping": true,
      "has_engage_mapping": true
    }
  },
  "total_detections": 31,
  "triggered_detection_rules": [
    "WEB_HONEYCREDENTIAL_USED",
    "WEB_HONEYFILE_ACCESS",
    "WEB_SCANNER_PROBE"
  ]
}

[+] Generating timeline and final reports...
/opt/deception-lab/parser/generate_timeline.py:93: DeprecationWarning: datetime.datetime.utcnow() is deprecated and scheduled for removal in a future version. Use timezone-aware objects to represent datetimes in UTC: datetime.datetime.now(datetime.UTC).
  lines.append(f"Generated at: {datetime.utcnow().isoformat()}Z")
[+] Timeline written to: /opt/deception-lab/reports/timeline.md
[+] Markdown report written to: /opt/deception-lab/reports/report.md
[+] JSON report written to: /opt/deception-lab/reports/report.json

[+] Report files:
total 52K
drwxrwxr-x 2 lss lss 4.0K May 22 03:11 .
drwxrwxr-x 9 lss lss 4.0K May 23 10:32 ..
-rw-rw-r-- 1 lss lss 2.0K May 23 10:34 mapping_report.md
-rw-rw-r-- 1 lss lss  12K May 23 10:34 report.json
-rw-rw-r-- 1 lss lss 5.4K May 23 10:34 report.md
-rw-rw-r-- 1 lss lss  18K May 23 10:34 timeline.md

[+] Report generation completed.
Main report: /opt/deception-lab/reports/report.md
Timeline:    /opt/deception-lab/reports/timeline.md
JSON report: /opt/deception-lab/reports/report.json
```
- 確認：
```
ls -lah /opt/deception-lab/reports

------------------------------------------------------------------------
# 執行結果：
lss@lss:/opt/deception-lab $ ls -lah /opt/deception-lab/reports
total 52K
drwxrwxr-x 2 lss lss 4.0K May 22 03:11 .
drwxrwxr-x 9 lss lss 4.0K May 23 10:32 ..
-rw-rw-r-- 1 lss lss 2.0K May 23 10:34 mapping_report.md
-rw-rw-r-- 1 lss lss  12K May 23 10:34 report.json
-rw-rw-r-- 1 lss lss 5.4K May 23 10:34 report.md
-rw-rw-r-- 1 lss lss  18K May 23 10:34 timeline.md
```
- 查看 summary：
```
cat /opt/deception-lab/reports/report.json | jq '.summary'

------------------------------------------------------------------------
# 執行結果：
lss@lss:/opt/deception-lab $ cat /opt/deception-lab/reports/report.json | jq '.summary'
{
  "detections_by_rule": {
    "WEB_HONEYCREDENTIAL_USED": 5,
    "WEB_HONEYFILE_ACCESS": 11,
    "WEB_SCANNER_PROBE": 15
  },
  "detections_by_severity": {
    "high": 16,
    "medium": 15
  },
  "events_by_source": {
    "fake-web": 68
  },
  "events_by_type": {
    "web_honeyfile_access": 11,
    "web_login_attempt": 10,
    "web_request": 47
  },
  "honeycredential_detections": 5,
  "honeyfile_detections": 11,
  "total_detections": 31,
  "total_events": 68
}
```

### Step 12.20：執行最終安全檢查
```
/opt/deception-lab/scripts/security_check.sh

#你要確認：
Cowrie Up
Fake Web Up
UFW active
Port 22, 2222, 8080 only
No privileged / docker.sock / host networking
Docker logging limits present
Disk usage healthy
Reports exist

------------------------------------------------------------------------
# 執行結果：
lss@lss:/opt/deception-lab $ /opt/deception-lab/scripts/security_check.sh
=== Security Check: Services ===
NAME                 IMAGE                    COMMAND                  SERVICE    CREATED       STATUS       PORTS
deception-cowrie     cowrie/cowrie:latest     "/cowrie/cowrie-env/…"   cowrie     3 hours ago   Up 3 hours   0.0.0.0:2222->2222/tcp, [::]:2222->2222/tcp, 2223/tcp
deception-fake-web   deception-lab-fake-web   "gunicorn --bind 0.0…"   fake-web   3 hours ago   Up 3 hours   0.0.0.0:8080->8080/tcp, [::]:8080->8080/tcp

=== Security Check: Listening Ports ===
tcp   LISTEN 0      4096         0.0.0.0:2222       0.0.0.0:*    users:(("docker-proxy",pid=7376,fd=8))
tcp   LISTEN 0      4096         0.0.0.0:8080       0.0.0.0:*    users:(("docker-proxy",pid=7458,fd=8))
tcp   LISTEN 0      128          0.0.0.0:22         0.0.0.0:*    users:(("sshd",pid=1188,fd=6))
tcp   LISTEN 0      4096            [::]:2222          [::]:*    users:(("docker-proxy",pid=7384,fd=8))
tcp   LISTEN 0      4096            [::]:8080          [::]:*    users:(("docker-proxy",pid=7464,fd=8))
tcp   LISTEN 0      128             [::]:22            [::]:*    users:(("sshd",pid=1188,fd=7))

=== Security Check: UFW ===
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), deny (routed)
New profiles: skip

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW IN    Anywhere
2222/tcp                   ALLOW IN    Anywhere
8080/tcp                   ALLOW IN    Anywhere
22/tcp (v6)                ALLOW IN    Anywhere (v6)
2222/tcp (v6)              ALLOW IN    Anywhere (v6)
8080/tcp (v6)              ALLOW IN    Anywhere (v6)


=== Security Check: Dangerous Docker Compose Settings ===
[OK] No obvious dangerous settings found.

=== Security Check: Docker Logging Limits ===
[WARN] No logging limit found.

=== Security Check: Important File Permissions ===
drwxrwxr-x 9 lss lss 4096 May 23 10:40 /opt/deception-lab
drwxrwxr-x 9 lss lss 4096 May 22 03:50 /opt/deception-lab/data
drwxrwxr-x 2 lss lss 4096 May 23 07:38 /opt/deception-lab/scripts
-rwxrwxr-x 1 lss lss  787 May 23 07:16 /opt/deception-lab/scripts/apply_lab_firewall_lan_only.sh
-rwxrwxr-x 1 lss lss  642 May 23 07:13 /opt/deception-lab/scripts/apply_lab_firewall.sh
-rwxrwxr-x 1 lss lss  396 May 23 07:35 /opt/deception-lab/scripts/backup_project.sh
-rwxrwxr-x 1 lss lss  352 May 23 07:11 /opt/deception-lab/scripts/backup_ufw_rules.sh
-rwxrwxr-x 1 lss lss 1012 May 13 02:48 /opt/deception-lab/scripts/check_assets.sh
-rwxrwxr-x 1 lss lss  512 May 11 05:20 /opt/deception-lab/scripts/check_env.sh
-rwxrwxr-x 1 lss lss  759 May 23 06:49 /opt/deception-lab/scripts/check_phase11_results.sh
-rwxrwxr-x 1 lss lss  519 May 15 07:14 /opt/deception-lab/scripts/cleanup_old_logs.sh
-rwxrwxr-x 1 lss lss 3019 May 15 06:52 /opt/deception-lab/scripts/collect_logs.sh
-rwxrwxr-x 1 lss lss  678 May 22 03:09 /opt/deception-lab/scripts/generate_report.sh

=== Security Check: Disk Usage ===
Filesystem      Size  Used Avail Use% Mounted on
/dev/mmcblk0p2   58G  5.2G   50G  10% /
26M     /opt/deception-lab
1.2M    /opt/deception-lab/data

=== Security Check: Reports ===
total 52K
drwxrwxr-x 2 lss lss 4.0K May 22 03:11 .
drwxrwxr-x 9 lss lss 4.0K May 23 10:40 ..
-rw-rw-r-- 1 lss lss 2.0K May 23 10:34 mapping_report.md
-rw-rw-r-- 1 lss lss  12K May 23 10:34 report.json
-rw-rw-r-- 1 lss lss 5.4K May 23 10:34 report.md
-rw-rw-r-- 1 lss lss  18K May 23 10:34 timeline.md

=== Security Check Completed ===
```

### Step 12.21：建立第十二階段完成紀錄
```
cat > /opt/deception-lab/PHASE12_READY.md <<'EOF'
# Phase 12 Ready

Security hardening and future expansion planning have been completed.

Completed items:

- SECURITY_NOTES.md created
- UFW backup script created
- Firewall reset scripts created
- Docker Compose dangerous setting check performed
- Docker logging limits added
- File permissions normalized
- Log cleanup script verified
- Weekly maintenance script created
- SD_CARD_NOTES.md created
- Project backup script created
- Security check script created
- FUTURE_WORK.md created
- FINAL_INDEX.md created
- README.md updated
- Final report regenerated
- Final security check performed

Main safety principles:

- Do not expose the lab directly to the Internet.
- Keep the lab isolated from production networks.
- Use only fake credentials and fake files.
- Keep management SSH on port 22 separate from Cowrie on port 2222.
- Keep Fake Web on port 8080 for lab testing only.
- Regularly rotate logs and back up reports.

Main maintenance scripts:

- /opt/deception-lab/scripts/security_check.sh
- /opt/deception-lab/scripts/weekly_maintenance.sh
- /opt/deception-lab/scripts/cleanup_old_logs.sh
- /opt/deception-lab/scripts/backup_project.sh

Main final documents:

- /opt/deception-lab/SECURITY_NOTES.md
- /opt/deception-lab/SD_CARD_NOTES.md
- /opt/deception-lab/FUTURE_WORK.md
- /opt/deception-lab/FINAL_INDEX.md

MVP status:

Completed.
EOF
```
- 確認：
```
cat /opt/deception-lab/PHASE12_READY.md
```

# Step 12.22：最終完成檢查
- 請執行這些指令，貼給我確認：
```
/opt/deception-lab/scripts/security_check.sh
ls -lah /opt/deception-lab
ls -lah /opt/deception-lab/scripts
ls -lah /opt/deception-lab/reports
cat /opt/deception-lab/reports/report.json | jq '.summary'
cat /opt/deception-lab/PHASE12_READY.md
```
- 執行結果：
```
lss@lss:~ $ /opt/deception-lab/scripts/security_check.sh
=== Security Check: Services ===
NAME                 IMAGE                    COMMAND                  SERVICE    CREATED         STATUS         PORTS
deception-cowrie     cowrie/cowrie:latest     "/cowrie/cowrie-env/…"   cowrie     5 minutes ago   Up 5 minutes   0.0.0.0:2222->2222/tcp, [::]:2222->2222/tcp, 2223/tcp
deception-fake-web   deception-lab-fake-web   "gunicorn --bind 0.0…"   fake-web   5 minutes ago   Up 5 minutes   0.0.0.0:8080->8080/tcp, [::]:8080->8080/tcp

=== Security Check: Listening Ports ===
tcp   LISTEN 0      4096         0.0.0.0:2222       0.0.0.0:*    users:(("docker-proxy",pid=10021,fd=8))
tcp   LISTEN 0      4096         0.0.0.0:8080       0.0.0.0:*    users:(("docker-proxy",pid=10101,fd=8))
tcp   LISTEN 0      128          0.0.0.0:22         0.0.0.0:*    users:(("sshd",pid=1188,fd=6))
tcp   LISTEN 0      4096            [::]:2222          [::]:*    users:(("docker-proxy",pid=10027,fd=8))
tcp   LISTEN 0      4096            [::]:8080          [::]:*    users:(("docker-proxy",pid=10108,fd=8))
tcp   LISTEN 0      128             [::]:22            [::]:*    users:(("sshd",pid=1188,fd=7))

=== Security Check: UFW ===
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), deny (routed)
New profiles: skip

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW IN    Anywhere
2222/tcp                   ALLOW IN    Anywhere
8080/tcp                   ALLOW IN    Anywhere
22/tcp (v6)                ALLOW IN    Anywhere (v6)
2222/tcp (v6)              ALLOW IN    Anywhere (v6)
8080/tcp (v6)              ALLOW IN    Anywhere (v6)


=== Security Check: Dangerous Docker Compose Settings ===
[OK] No obvious dangerous settings found.

=== Security Check: Docker Logging Limits ===
17:    logging:
18-      driver: "json-file"
19-      options:
20-        max-size: "10m"
21-        max-file: "3"
--
44:    logging:
45-      driver: "json-file"
46-      options:
47-        max-size: "10m"
48-        max-file: "3"

=== Security Check: Important File Permissions ===
drwxrwxr-x 9 lss lss 4096 May 23 10:47 /opt/deception-lab
drwxrwxr-x 9 lss lss 4096 May 22 03:50 /opt/deception-lab/data
drwxrwxr-x 2 lss lss 4096 May 23 07:38 /opt/deception-lab/scripts
-rwxrwxr-x 1 lss lss  787 May 23 07:16 /opt/deception-lab/scripts/apply_lab_firewall_lan_only.sh
-rwxrwxr-x 1 lss lss  642 May 23 07:13 /opt/deception-lab/scripts/apply_lab_firewall.sh
-rwxrwxr-x 1 lss lss  396 May 23 07:35 /opt/deception-lab/scripts/backup_project.sh
-rwxrwxr-x 1 lss lss  352 May 23 07:11 /opt/deception-lab/scripts/backup_ufw_rules.sh
-rwxrwxr-x 1 lss lss 1012 May 13 02:48 /opt/deception-lab/scripts/check_assets.sh
-rwxrwxr-x 1 lss lss  512 May 11 05:20 /opt/deception-lab/scripts/check_env.sh
-rwxrwxr-x 1 lss lss  759 May 23 06:49 /opt/deception-lab/scripts/check_phase11_results.sh
-rwxrwxr-x 1 lss lss  519 May 15 07:14 /opt/deception-lab/scripts/cleanup_old_logs.sh
-rwxrwxr-x 1 lss lss 3019 May 15 06:52 /opt/deception-lab/scripts/collect_logs.sh
-rwxrwxr-x 1 lss lss  678 May 22 03:09 /opt/deception-lab/scripts/generate_report.sh

=== Security Check: Disk Usage ===
Filesystem      Size  Used Avail Use% Mounted on
/dev/mmcblk0p2   58G  5.2G   50G  10% /
26M     /opt/deception-lab
1.3M    /opt/deception-lab/data

=== Security Check: Reports ===
total 60K
drwxrwxr-x 2 lss lss 4.0K May 22 03:11 .
drwxrwxr-x 9 lss lss 4.0K May 23 10:47 ..
-rw-rw-r-- 1 lss lss 2.3K May 23 10:52 mapping_report.md
-rw-rw-r-- 1 lss lss  13K May 23 10:52 report.json
-rw-rw-r-- 1 lss lss 5.8K May 23 10:52 report.md
-rw-rw-r-- 1 lss lss  22K May 23 10:52 timeline.md

=== Security Check Completed ===
lss@lss:~ $ ls -lah /opt/deception-lab
total 140K
drwxrwxr-x 9 lss  lss  4.0K May 23 10:47 .
drwxr-xr-x 4 root root 4.0K May 11 05:18 ..
drwxrwxr-x 5 lss  lss  4.0K May 23 07:34 backups
drwxrwxr-x 5 lss  lss  4.0K May 12 01:25 cowrie
drwxrwxr-x 9 lss  lss  4.0K May 22 03:50 data
-rw-rw-r-- 1 lss  lss  3.9K May 13 01:47 deception_assets.yml
-rw-rw-r-- 1 lss  lss  1.2K May 23 10:47 docker-compose.before-logging-fix.yml
-rw-rw-r-- 1 lss  lss   991 May 23 07:21 docker-compose.phase11.yml
-rw-rw-r-- 1 lss  lss   313 May 12 01:38 docker-compose.phase3.yml
-rw-rw-r-- 1 lss  lss   772 May 12 02:02 docker-compose.phase4.broken.yml
-rw-rw-r-- 1 lss  lss   419 May 12 05:33 docker-compose.phase4-cowrie-only.yml
-rw-rw-r-- 1 lss  lss   555 May 12 02:21 docker-compose.phase4.permission-error.yml
-rw-rw-r-- 1 lss  lss   922 May 13 02:06 docker-compose.phase5.yml
-rw-rw-r-- 1 lss  lss  1.2K May 23 10:47 docker-compose.yml
-rw-rw-r-- 1 lss  lss   538 May 11 19:35 .env
drwxrwxr-x 5 lss  lss  4.0K May 12 05:32 fake-web
-rw-rw-r-- 1 lss  lss   579 May 23 10:32 FINAL_INDEX.md
-rw-rw-r-- 1 lss  lss  1.5K May 23 10:25 FUTURE_WORK.md
drwxrwxr-x 4 lss  lss  4.0K May 22 03:08 parser
-rw-rw-r-- 1 lss  lss   744 May 22 03:32 PHASE10_READY.md
-rw-rw-r-- 1 lss  lss   789 May 23 06:51 PHASE11_READY.md
-rw-rw-r-- 1 lss  lss  1.4K May 23 10:41 PHASE12_READY.md
-rw-rw-r-- 1 lss  lss   523 May 11 05:19 PHASE2_READY.md
-rw-rw-r-- 1 lss  lss   381 May 11 22:37 PHASE3_READY.md
-rw-rw-r-- 1 lss  lss   578 May 12 02:26 PHASE4_READY.md
-rw-rw-r-- 1 lss  lss   773 May 12 06:13 PHASE5_READY.md
-rw-rw-r-- 1 lss  lss   709 May 13 02:53 PHASE6_READY.md
-rw-rw-r-- 1 lss  lss   718 May 15 07:29 PHASE7_READY.md
-rw-rw-r-- 1 lss  lss   922 May 20 03:54 PHASE8_READY.md
-rw-rw-r-- 1 lss  lss   848 May 21 08:15 PHASE9_READY.md
-rw-rw-r-- 1 lss  lss  1.5K May 23 10:33 README.md
drwxrwxr-x 2 lss  lss  4.0K May 22 03:11 reports
drwxrwxr-x 2 lss  lss  4.0K May 23 07:38 scripts
-rw-rw-r-- 1 lss  lss   613 May 23 07:32 SD_CARD_NOTES.md
-rw-rw-r-- 1 lss  lss  1.1K May 23 07:07 SECURITY_NOTES.md
lss@lss:~ $ ls -lah /opt/deception-lab/scripts
total 104K
drwxrwxr-x 2 lss lss 4.0K May 23 07:38 .
drwxrwxr-x 9 lss lss 4.0K May 23 10:47 ..
-rwxrwxr-x 1 lss lss  787 May 23 07:16 apply_lab_firewall_lan_only.sh
-rwxrwxr-x 1 lss lss  642 May 23 07:13 apply_lab_firewall.sh
-rwxrwxr-x 1 lss lss  396 May 23 07:35 backup_project.sh
-rwxrwxr-x 1 lss lss  352 May 23 07:11 backup_ufw_rules.sh
-rwxrwxr-x 1 lss lss 1012 May 13 02:48 check_assets.sh
-rwxrwxr-x 1 lss lss  512 May 11 05:20 check_env.sh
-rwxrwxr-x 1 lss lss  759 May 23 06:49 check_phase11_results.sh
-rwxrwxr-x 1 lss lss  519 May 15 07:14 cleanup_old_logs.sh
-rwxrwxr-x 1 lss lss 3.0K May 15 06:52 collect_logs.sh
-rwxrwxr-x 1 lss lss  678 May 22 03:09 generate_report.sh
-rwxrwxr-x 1 lss lss  127 May 11 19:52 logs_lab.sh
-rwxrwxr-x 1 lss lss  203 May 11 19:53 restart_lab.sh
-rwxrwxr-x 1 lss lss  880 May 21 08:12 run_mapping.sh
-rwxrwxr-x 1 lss lss  529 May 20 03:26 run_parser.sh
-rwxrwxr-x 1 lss lss 1.1K May 23 07:38 security_check.sh
-rwxrwxr-x 1 lss lss  972 May 15 07:09 show_collected_logs.sh
-rwxrwxr-x 1 lss lss  859 May 20 03:42 show_events.sh
-rwxrwxr-x 1 lss lss 1.1K May 21 08:03 show_mapping.sh
-rwxrwxr-x 1 lss lss  616 May 22 03:20 show_report.sh
-rwxrwxr-x 1 lss lss  181 May 11 19:47 start_lab.sh
-rwxrwxr-x 1 lss lss  637 May 15 07:21 status_lab.sh
-rwxrwxr-x 1 lss lss  151 May 11 19:49 stop_lab.sh
-rwxrwxr-x 1 lss lss 1.3K May 23 06:09 test_web_scenarios.sh
-rwxrwxr-x 1 lss lss  513 May 23 07:30 weekly_maintenance.sh
lss@lss:~ $ ls -lah /opt/deception-lab/reports
total 60K
drwxrwxr-x 2 lss lss 4.0K May 22 03:11 .
drwxrwxr-x 9 lss lss 4.0K May 23 10:47 ..
-rw-rw-r-- 1 lss lss 2.3K May 23 10:52 mapping_report.md
-rw-rw-r-- 1 lss lss  13K May 23 10:52 report.json
-rw-rw-r-- 1 lss lss 5.8K May 23 10:52 report.md
-rw-rw-r-- 1 lss lss  22K May 23 10:52 timeline.md
lss@lss:~ $ cat /opt/deception-lab/reports/report.json | jq '.summary'
{
  "detections_by_rule": {
    "SSH_HONEYCREDENTIAL_LOGIN": 1,
    "SSH_HONEYFILE_ACCESS": 2,
    "SSH_RECON_COMMAND": 3,
    "SSH_TOOL_TRANSFER_COMMAND": 2,
    "WEB_HONEYCREDENTIAL_USED": 5,
    "WEB_HONEYFILE_ACCESS": 11,
    "WEB_SCANNER_PROBE": 15
  },
  "detections_by_severity": {
    "high": 21,
    "medium": 18
  },
  "events_by_source": {
    "cowrie": 11,
    "fake-web": 68
  },
  "events_by_type": {
    "ssh_command": 8,
    "ssh_connection": 1,
    "ssh_login_success": 1,
    "ssh_logout": 1,
    "web_honeyfile_access": 11,
    "web_login_attempt": 10,
    "web_request": 47
  },
  "honeycredential_detections": 6,
  "honeyfile_detections": 13,
  "total_detections": 39,
  "total_events": 79
}
lss@lss:~ $ cat /opt/deception-lab/PHASE12_READY.md
# Phase 12 Ready

Security hardening and future expansion planning have been completed.

Completed items:

- SECURITY_NOTES.md created
- UFW backup script created
- Firewall reset scripts created
- Docker Compose dangerous setting check performed
- Docker logging limits added
- File permissions normalized
- Log cleanup script verified
- Weekly maintenance script created
- SD_CARD_NOTES.md created
- Project backup script created
- Security check script created
- FUTURE_WORK.md created
- FINAL_INDEX.md created
- README.md updated
- Final report regenerated
- Final security check performed

Main safety principles:

- Do not expose the lab directly to the Internet.
- Keep the lab isolated from production networks.
- Use only fake credentials and fake files.
- Keep management SSH on port 22 separate from Cowrie on port 2222.
- Keep Fake Web on port 8080 for lab testing only.
- Regularly rotate logs and back up reports.

Main maintenance scripts:

- /opt/deception-lab/scripts/security_check.sh
- /opt/deception-lab/scripts/weekly_maintenance.sh
- /opt/deception-lab/scripts/cleanup_old_logs.sh
- /opt/deception-lab/scripts/backup_project.sh

Main final documents:

- /opt/deception-lab/SECURITY_NOTES.md
- /opt/deception-lab/SD_CARD_NOTES.md
- /opt/deception-lab/FUTURE_WORK.md
- /opt/deception-lab/FINAL_INDEX.md

MVP status:

Completed.
```
- 確認：Step 12.22 最終完成檢查已通過。
```
你這次重新檢查後，前一次的兩個疑點都已修正：

Docker logging limits 已成功加入
最終 report 已重新包含 Cowrie 與 Fake Web 事件

------------------------------------------------------------------------
1. 服務狀態正常

目前兩個 deception service 都正常執行：

deception-cowrie     Up 5 minutes
deception-fake-web   Up 5 minutes

port 也正常監聽：

22/tcp    真實 Raspberry Pi SSH 管理入口
2222/tcp  Cowrie SSH honeypot
8080/tcp  Fake Web Admin Panel

這代表 Docker Compose、Cowrie、Fake Web 都正常。

------------------------------------------------------------------------
2. UFW 防火牆狀態正常

你的 UFW 是啟用狀態：

Status: active
Default: deny (incoming), allow (outgoing), deny (routed)

目前允許的 inbound port 是：

22/tcp
2222/tcp
8080/tcp

這符合 MVP 設定：管理 SSH、Cowrie honeypot、Fake Web admin panel。

------------------------------------------------------------------------
3. Docker 危險設定檢查通過

安全檢查顯示：

[OK] No obvious dangerous settings found.

代表目前 docker-compose.yml 沒有偵測到明顯高風險設定，例如：

privileged
network_mode
/var/run/docker.sock
cap_add

這點通過。

------------------------------------------------------------------------
4. Docker logging limits 已修正成功

這次已經看到：

logging:
  driver: "json-file"
  options:
    max-size: "10m"
    max-file: "3"

而且 Cowrie 與 Fake Web 兩個 service 都有 logging limit。這代表前一次的 [WARN] No logging limit found 已經修正完成。

------------------------------------------------------------------------
5. 檔案權限與腳本權限正常

專案目錄目前是：

/opt/deception-lab
owner: lss:lss
permission: drwxrwxr-x

/opt/deception-lab/scripts 內的腳本都有可執行權限，例如：

-rwxrwxr-x security_check.sh
-rwxrwxr-x weekly_maintenance.sh
-rwxrwxr-x backup_project.sh
-rwxrwxr-x generate_report.sh
-rwxrwxr-x test_web_scenarios.sh

這代表維護、備份、報告、測試腳本都可以正常執行。

------------------------------------------------------------------------
6. 磁碟空間正常

目前 SD 卡根目錄使用量：

Size: 58G
Used: 5.2G
Avail: 50G
Use%: 10%

目前 /opt/deception-lab 約 26M，data 約 1.3M。對 MVP 實驗來說容量非常充足。

------------------------------------------------------------------------
7. 報告檔案正常產生

目前 reports 目錄已有：

mapping_report.md
report.json
report.md
timeline.md

且時間為最新產生：

May 23 10:52

這代表最終報告已成功重新產生。

------------------------------------------------------------------------
. 最終 report 已包含 Cowrie 與 Fake Web

你的最新 report.json summary 顯示：

"events_by_source": {
  "cowrie": 11,
  "fake-web": 68
}

這代表最終報告已經同時包含：

Cowrie SSH honeypot events
Fake Web events

偵測結果也完整包含：

SSH_HONEYCREDENTIAL_LOGIN
SSH_HONEYFILE_ACCESS
SSH_RECON_COMMAND
SSH_TOOL_TRANSFER_COMMAND
WEB_HONEYCREDENTIAL_USED
WEB_HONEYFILE_ACCESS
WEB_SCANNER_PROBE
目前總計：

total_events: 79
total_detections: 39
honeycredential_detections: 6
honeyfile_detections: 13

這表示第十一階段測試與第十二階段最終報告整合都成功。

------------------------------------------------------------------------
9. PHASE12_READY.md 已存在且內容正確

你的 PHASE12_READY.md 顯示：

MVP status:

Completed.

並且列出已完成：

SECURITY_NOTES.md
UFW backup script
Firewall reset scripts
Docker logging limits
Weekly maintenance script
Project backup script
Security check script
FUTURE_WORK.md
FINAL_INDEX.md
Final report regenerated
Final security check performed

這代表第十二階段完成紀錄也已建立成功。

------------------------------------------------------------------------
最終判定

Step 12.22：最終完成檢查已通過。

整個 Raspberry Pi 5 主動式資安防禦誘捕欺敵 MVP 的 12 個階段已完成：

[完成] 第一階段：系統架構設計
[完成] 第二階段：Raspberry Pi 5 環境準備
[完成] 第三階段：Docker Compose 專案目錄建立
[完成] 第四階段：Cowrie SSH honeypot
[完成] 第五階段：Fake Web Admin Panel
[完成] 第六階段：honeycredential / honeyfile
[完成] 第七階段：集中式 log 收集
[完成] 第八階段：Python parser / detection rules
[完成] 第九階段：MITRE ATT&CK / Engage mapping
[完成] 第十階段：timeline / Markdown / JSON report
[完成] 第十一階段：測試攻擊情境
[完成] 第十二階段：安全加固與後續擴充

你現在的 MVP 可以正式視為完成。
```























