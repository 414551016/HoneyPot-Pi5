# 第七階段：集中式 log 收集
```
# 目前你的 log 來源有三種：
1. Cowrie Docker logs  # 這是 Cowrie honeypot（SSH/Telnet）產生的 log，記錄攻擊者在「假 SSH 系統」內的所有行為{ cowrie.json（最重要）、 cowrie.log（人類閱讀）}。例如：登入相關、互動行為、檔案行為
  ✅ 登入相關：username / password、登入成功 / 失敗、來源 IP
  ✅ 互動行為：執行的 command（whoami, wget, cat...）、shell 操作、session 開始 / 結束
  ✅ 檔案行為：上傳檔案（SCP / SFTP）、下載檔案（wget / curl）、存取 honeyfile
2. Fake Web access log  # 這是 Fake Web Admin（Flask）記錄所有 HTTP request 的 log
3. Fake Web auth log    # 這是專門記錄「登入相關行為」的 log（從 access log 分離出來）
  ✅ 1. Web brute force 偵測：多次 POST /login、不同帳密嘗試。👉 對應：T1110 Brute Force
  ✅ 2. Honeycredential 命中（最重要）：👉 代表：攻擊者讀過你的誘餌資料、已進入「誘捕成功」階段
  ✅ 3. 攻擊意圖判定：dictionary attack、credential stuffing、targeted login（只試特定帳號）
#目前位置大致是：
Cowrie:
  docker compose logs cowrie
  /opt/deception-lab/data/logs/cowrie/cowrie-docker.log

Fake Web:
  /opt/deception-lab/data/logs/web/web_access.jsonl
  /opt/deception-lab/data/logs/web/web_auth.jsonl

# 第七階段會整理成：
/opt/deception-lab/data/collected/
├── cowrie-docker.log
├── web_access.jsonl
├── web_auth.jsonl
├── source_manifest.json
└── collection_summary.txt
```

### Step 7.1：進入專案目錄
```
cd /opt/deception-lab
```

### Step 7.2：建立集中 log 目錄
- 執行：
  ```
  mkdir -p \
    /opt/deception-lab/data/collected \
    /opt/deception-lab/data/archive \
    /opt/deception-lab/data/raw
  ```
  你應該看到類似：
  ```
  lss@lss:/opt/deception-lab $ tree -L 2 /opt/deception-lab/data
  /opt/deception-lab/data
  ├── archive
  ├── collected
  ├── events
  ├── logs
  │   ├── cowrie
  │   └── web
  ├── raw
  └── samples
      └── uploads
  
  10 directories, 0 files
  ```

### Step 7.3：建立 log 收集腳本 collect_logs.sh
這個腳本會做幾件事：
```
1. 匯出 Cowrie Docker logs
2. 複製 Web access log
3. 複製 Web auth log
4. 產生 source_manifest.json
5. 產生 collection_summary.txt
6. 保留一份 timestamp archive
```
- 執行：
  ```
  cat > /opt/deception-lab/scripts/collect_logs.sh <<'EOF'
  #!/usr/bin/env bash
  set -euo pipefail
  
  PROJECT_ROOT="/opt/deception-lab"
  COLLECTED_DIR="$PROJECT_ROOT/data/collected"
  ARCHIVE_DIR="$PROJECT_ROOT/data/archive"
  WEB_LOG_DIR="$PROJECT_ROOT/data/logs/web"
  COWRIE_LOG_DIR="$PROJECT_ROOT/data/logs/cowrie"
  
  TS="$(date -u +"%Y%m%dT%H%M%SZ")"
  ARCHIVE_RUN_DIR="$ARCHIVE_DIR/$TS"
  
  mkdir -p "$COLLECTED_DIR"
  mkdir -p "$ARCHIVE_RUN_DIR"
  mkdir -p "$COWRIE_LOG_DIR"
  mkdir -p "$WEB_LOG_DIR"
  
  echo "[+] Collecting logs at UTC time: $TS"
  
  echo "[+] Exporting Cowrie Docker logs..."
  cd "$PROJECT_ROOT"
  docker compose logs --no-color cowrie > "$COLLECTED_DIR/cowrie-docker.log" || true
  cp "$COLLECTED_DIR/cowrie-docker.log" "$COWRIE_LOG_DIR/cowrie-docker.log" || true
  
  echo "[+] Collecting Fake Web access log..."
  if [ -f "$WEB_LOG_DIR/web_access.jsonl" ]; then
    cp "$WEB_LOG_DIR/web_access.jsonl" "$COLLECTED_DIR/web_access.jsonl"
  else
    : > "$COLLECTED_DIR/web_access.jsonl"
  fi
  
  echo "[+] Collecting Fake Web auth log..."
  if [ -f "$WEB_LOG_DIR/web_auth.jsonl" ]; then
    cp "$WEB_LOG_DIR/web_auth.jsonl" "$COLLECTED_DIR/web_auth.jsonl"
  else
    : > "$COLLECTED_DIR/web_auth.jsonl"
  fi
  
  echo "[+] Creating source manifest..."
  cat > "$COLLECTED_DIR/source_manifest.json" <<MANIFEST
  {
    "collection_time_utc": "$TS",
    "project_root": "$PROJECT_ROOT",
    "sources": [
      {
        "name": "cowrie-docker",
        "type": "docker-logs",
        "service": "cowrie",
        "collected_file": "$COLLECTED_DIR/cowrie-docker.log"
      },
      {
        "name": "fake-web-access",
        "type": "jsonl",
        "service": "fake-web",
        "source_file": "$WEB_LOG_DIR/web_access.jsonl",
        "collected_file": "$COLLECTED_DIR/web_access.jsonl"
      },
      {
        "name": "fake-web-auth",
        "type": "jsonl",
        "service": "fake-web",
        "source_file": "$WEB_LOG_DIR/web_auth.jsonl",
        "collected_file": "$COLLECTED_DIR/web_auth.jsonl"
      }
    ]
  }
  MANIFEST
  
  echo "[+] Creating collection summary..."
  {
    echo "Collection time UTC: $TS"
    echo
    echo "Collected files:"
    echo
  
    for f in \
      "$COLLECTED_DIR/cowrie-docker.log" \
      "$COLLECTED_DIR/web_access.jsonl" \
      "$COLLECTED_DIR/web_auth.jsonl" \
      "$COLLECTED_DIR/source_manifest.json"
    do
      if [ -f "$f" ]; then
        echo "- $f"
        echo "  size: $(du -h "$f" | awk '{print $1}')"
        echo "  lines: $(wc -l < "$f")"
      else
        echo "- $f"
        echo "  missing"
      fi
    done
  } > "$COLLECTED_DIR/collection_summary.txt"
  
  echo "[+] Archiving collected logs..."
  cp "$COLLECTED_DIR/cowrie-docker.log" "$ARCHIVE_RUN_DIR/cowrie-docker.log"
  cp "$COLLECTED_DIR/web_access.jsonl" "$ARCHIVE_RUN_DIR/web_access.jsonl"
  cp "$COLLECTED_DIR/web_auth.jsonl" "$ARCHIVE_RUN_DIR/web_auth.jsonl"
  cp "$COLLECTED_DIR/source_manifest.json" "$ARCHIVE_RUN_DIR/source_manifest.json"
  cp "$COLLECTED_DIR/collection_summary.txt" "$ARCHIVE_RUN_DIR/collection_summary.txt"
  
  echo
  echo "[+] Collection completed."
  echo "[+] Current collected directory:"
  ls -lah "$COLLECTED_DIR"
  
  echo
  echo "[+] Summary:"
  cat "$COLLECTED_DIR/collection_summary.txt"
  EOF
  ```
- 設定可執行：
  ```
  chmod +x /opt/deception-lab/scripts/collect_logs.sh
  ```

### Step 7.4：執行第一次集中 log 收集
- 執行：
  ```
  /opt/deception-lab/scripts/collect_logs.sh
  ```
- 執行結果：
  ```
  lss@lss:/opt/deception-lab $ chmod +x /opt/deception-lab/scripts/collect_logs.sh
  lss@lss:/opt/deception-lab $ /opt/deception-lab/scripts/collect_logs.sh
  [+] Collecting logs at UTC time: 20260514T225346Z
  [+] Exporting Cowrie Docker logs...
  [+] Collecting Fake Web access log...
  [+] Collecting Fake Web auth log...
  [+] Creating source manifest...
  [+] Creating collection summary...
  [+] Archiving collected logs...
  
  [+] Collection completed.
  [+] Current collected directory:
  total 36K
  drwxrwxr-x 2 lss lss 4.0K May 15 06:53 .
  drwxrwxr-x 8 lss lss 4.0K May 13 03:01 ..
  -rw-rw-r-- 1 lss lss  371 May 15 06:53 collection_summary.txt
  -rw-rw-r-- 1 lss lss 7.7K May 15 06:53 cowrie-docker.log
  -rw-rw-r-- 1 lss lss  777 May 15 06:53 source_manifest.json
  -rwxrwxr-x 1 lss lss 6.5K May 15 06:53 web_access.jsonl
  -rwxrwxr-x 1 lss lss 1.7K May 15 06:53 web_auth.jsonl
  
  [+] Summary:
  Collection time UTC: 20260514T225346Z
  
  Collected files:
  
  - /opt/deception-lab/data/collected/cowrie-docker.log
    size: 8.0K
    lines: 63
  - /opt/deception-lab/data/collected/web_access.jsonl
    size: 8.0K
    lines: 19
  - /opt/deception-lab/data/collected/web_auth.jsonl
    size: 4.0K
    lines: 4
  - /opt/deception-lab/data/collected/source_manifest.json
    size: 4.0K
    lines: 26
  
  ```

### Step 7.5：確認 collected 目錄
- 執行：
  ```
  ls -lah /opt/deception-lab/data/collected
  ```
- 執行結果：
  ```
  lss@lss:/opt/deception-lab $ ls -lah /opt/deception-lab/data/collected
  total 36K
  drwxrwxr-x 2 lss lss 4.0K May 15 06:53 .
  drwxrwxr-x 8 lss lss 4.0K May 13 03:01 ..
  -rw-rw-r-- 1 lss lss  371 May 15 06:53 collection_summary.txt
  -rw-rw-r-- 1 lss lss 7.7K May 15 06:53 cowrie-docker.log
  -rw-rw-r-- 1 lss lss  777 May 15 06:53 source_manifest.json
  -rwxrwxr-x 1 lss lss 6.5K May 15 06:53 web_access.jsonl
  -rwxrwxr-x 1 lss lss 1.7K May 15 06:53 web_auth.jsonl
  
  ```

### Step 7.6：查看 collection summary
- 執行：
  ```
  lss@lss:/opt/deception-lab $ cat /opt/deception-lab/data/collected/collection_summary.txt
  Collection time UTC: 20260514T225346Z
  
  Collected files:
  
  - /opt/deception-lab/data/collected/cowrie-docker.log
    size: 8.0K
    lines: 63
  - /opt/deception-lab/data/collected/web_access.jsonl
    size: 8.0K
    lines: 19
  - /opt/deception-lab/data/collected/web_auth.jsonl
    size: 4.0K
    lines: 4
  - /opt/deception-lab/data/collected/source_manifest.json
    size: 4.0K
    lines: 26
  
  ```

### Step 7.7：查看 source manifest
- 執行：
  ```
  lss@lss:/opt/deception-lab $ cat /opt/deception-lab/data/collected/source_manifest.json | jq
  {
    "collection_time_utc": "20260514T225346Z",
    "project_root": "/opt/deception-lab",
    "sources": [
      {
        "name": "cowrie-docker",
        "type": "docker-logs",
        "service": "cowrie",
        "collected_file": "/opt/deception-lab/data/collected/cowrie-docker.log"
      },
      {
        "name": "fake-web-access",
        "type": "jsonl",
        "service": "fake-web",
        "source_file": "/opt/deception-lab/data/logs/web/web_access.jsonl",
        "collected_file": "/opt/deception-lab/data/collected/web_access.jsonl"
      },
      {
        "name": "fake-web-auth",
        "type": "jsonl",
        "service": "fake-web",
        "source_file": "/opt/deception-lab/data/logs/web/web_auth.jsonl",
        "collected_file": "/opt/deception-lab/data/collected/web_auth.jsonl"
      }
    ]
  }
  
  # 這份檔案的用途是記錄：
  這次收集了哪些 log
  來源在哪裡
  集中後放在哪裡
  收集時間是什麼
  ```

### Step 7.8：確認 Cowrie log 內容
- 執行：
  ```
  grep -E "login attempt|CMD:|New connection|avatar .* logging out" \
    /opt/deception-lab/data/collected/cowrie-docker.log | tail -n 30
  ```
- 執行結果：
  ```
  lss@lss:/opt/deception-lab $ grep -E "login attempt|CMD:|New connection|avatar .* logging out" \
    /opt/deception-lab/data/collected/cowrie-docker.log | tail -n 30
  deception-cowrie  | 2026-05-13T02:26:21+0800 [cowrie.ssh.factory.CowrieSSHFactory] New connection: 172.18.0.1:43902 (172.18.0.2:2222) [ses       sion: 221da1bdefb7]
  deception-cowrie  | 2026-05-13T02:26:42+0800 [HoneyPotSSHTransport,0,172.18.0.1] login attempt [b'backup'/b'Backup2026!'] succeeded
  deception-cowrie  | 2026-05-13T02:27:07+0800 [HoneyPotSSHTransport,0,172.18.0.1] CMD: whoami
  deception-cowrie  | 2026-05-13T02:27:11+0800 [HoneyPotSSHTransport,0,172.18.0.1] CMD: pwd
  deception-cowrie  | 2026-05-13T02:27:14+0800 [HoneyPotSSHTransport,0,172.18.0.1] CMD: is
  deception-cowrie  | 2026-05-13T02:27:16+0800 [HoneyPotSSHTransport,0,172.18.0.1] CMD: ls
  deception-cowrie  | 2026-05-13T02:27:30+0800 [HoneyPotSSHTransport,0,172.18.0.1] CMD: ls /home/admin
  deception-cowrie  | 2026-05-13T02:27:46+0800 [HoneyPotSSHTransport,0,172.18.0.1] CMD: ls /home/admin
  deception-cowrie  | 2026-05-13T02:27:53+0800 [HoneyPotSSHTransport,0,172.18.0.1] CMD: cat /home/admin/secrets.txt
  deception-cowrie  | 2026-05-13T02:27:57+0800 [HoneyPotSSHTransport,0,172.18.0.1] CMD: exit
  deception-cowrie  | 2026-05-13T02:27:57+0800 [HoneyPotSSHTransport,0,172.18.0.1] avatar backup logging out
  
  你應該會看到類似：
  login attempt [b'backup'/b'Backup2026!'] succeeded
  CMD: whoami
  CMD: pwd
  CMD: ls
  CMD: cat /home/admin/secrets.txt
  avatar backup logging out
  這代表 Cowrie 互動 log 已經被集中收集。
  ```

### Step 7.9：確認 Fake Web access log
- 執行：
  ```
  tail -n 5 /opt/deception-lab/data/collected/web_access.jsonl
  ```
- 執行結果：
  ```
  lss@lss:/opt/deception-lab $ tail -n 5 /opt/deception-lab/data/collected/web_access.jsonl
  {"timestamp": "2026-05-11T22:10:43.882087+00:00", "source": "fake-web", "event_type": "web_request", "src_ip": "172.18.0.1", "method": "HEAD", "path": "/login", "query_string": "", "user_agent": "curl/8.14.1", "is_scanner_probe": false, "tags": ["web", "request"]}
  {"timestamp": "2026-05-11T22:18:07.650530+00:00", "source": "fake-web", "event_type": "web_request", "src_ip": "192.168.1.1", "method": "POST", "path": "/login", "query_string": "", "user_agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.0.0 Safari/537.36", "is_scanner_probe": false, "tags": ["web", "request"]}
  {"timestamp": "2026-05-11T22:18:07.668441+00:00", "source": "fake-web", "event_type": "web_request", "src_ip": "192.168.1.1", "method": "GET", "path": "/static/style.css", "query_string": "", "user_agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.0.0 Safari/537.36", "is_scanner_probe": false, "tags": ["web", "request"]}
  {"timestamp": "2026-05-12T18:36:26.542718+00:00", "source": "fake-web", "event_type": "web_request", "src_ip": "172.18.0.1", "method": "POST", "path": "/login", "query_string": "", "user_agent": "curl/8.14.1", "is_scanner_probe": false, "tags": ["web", "request"]}
  {"timestamp": "2026-05-12T18:41:52.678577+00:00", "source": "fake-web", "event_type": "web_request", "src_ip": "172.18.0.1", "method": "POST", "path": "/login", "query_string": "", "user_agent": "curl/8.14.1", "is_scanner_probe": false, "tags": ["web", "request"]}
  
  你應該會看到 JSONL 格式事件，例如：
  {"timestamp": "...", "source": "fake-web", "event_type": "web_request", ...}
  {"timestamp": "...", "source": "fake-web", "event_type": "web_honeyfile_access", ...}
  ```

### Step 7.10：確認 Fake Web auth log
- 執行：
  ```
  tail -n 5 /opt/deception-lab/data/collected/web_auth.jsonl
  ```
- 執行結果：
  ```
  lss@lss:/opt/deception-lab $ tail -n 5 /opt/deception-lab/data/collected/web_auth.jsonl
  {"timestamp": "2026-05-11T21:46:56.695098+00:00", "source": "fake-web", "event_type": "web_login_attempt", "src_ip": "192.168.1.1", "username": "admin", "password_sha256": "8d969eef6ecad3c29a3a629280e686cf0c3f5d5a86aff3ca12020c923adc6c92", "password_length": 6, "honeycredential_used": false, "user_agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.0.0 Safari/537.36", "severity": "medium", "tags": ["web", "login"]}
  {"timestamp": "2026-05-11T22:18:07.651474+00:00", "source": "fake-web", "event_type": "web_login_attempt", "src_ip": "192.168.1.1", "username": "backup", "password_sha256": "0ecba7213823d57d8b9c6510186aa9ba9e401b2f4508e792f8f3ca4aec6394e1", "password_length": 12, "honeycredential_used": false, "user_agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.0.0 Safari/537.36", "severity": "medium", "tags": ["web", "login"]}
  {"timestamp": "2026-05-12T18:36:26.543366+00:00", "source": "fake-web", "event_type": "web_login_attempt", "src_ip": "172.18.0.1", "username": "backup", "password_sha256": "a597108ff8384b3be204d08bce36fe3e86bffcc699fdac27604db48ae6f27f71", "password_length": 11, "honeycredential_used": true, "user_agent": "curl/8.14.1", "severity": "high", "tags": ["web", "login", "honeycredential"]}
  {"timestamp": "2026-05-12T18:41:52.679019+00:00", "source": "fake-web", "event_type": "web_login_attempt", "src_ip": "172.18.0.1", "username": "backup", "password_sha256": "a597108ff8384b3be204d08bce36fe3e86bffcc699fdac27604db48ae6f27f71", "password_length": 11, "honeycredential_used": true, "user_agent": "curl/8.14.1", "severity": "high", "tags": ["web", "login", "honeycredential"]}
  
  你應該會看到：
  {"timestamp": "...", "source": "fake-web", "event_type": "web_login_attempt", ...}
  其中應該包含："honeycredential_used": true
  ```

### Step 7.11：建立 log 快速查看腳本
這個腳本會快速顯示目前三種 log 的摘要。
- 執行：
  ```
  cat > /opt/deception-lab/scripts/show_collected_logs.sh <<'EOF'
  #!/usr/bin/env bash
  set -euo pipefail
  
  COLLECTED_DIR="/opt/deception-lab/data/collected"
  
  echo "=== Collected Directory ==="
  ls -lah "$COLLECTED_DIR"
  
  echo
  echo "=== Collection Summary ==="
  if [ -f "$COLLECTED_DIR/collection_summary.txt" ]; then
    cat "$COLLECTED_DIR/collection_summary.txt"
  else
    echo "No collection_summary.txt found."
  fi
  
  echo
  echo "=== Cowrie Interesting Lines ==="
  if [ -f "$COLLECTED_DIR/cowrie-docker.log" ]; then
    grep -E "login attempt|CMD:|New connection|avatar .* logging out" "$COLLECTED_DIR/cowrie-docker.log" | tail -n 30 || true
  else
    echo "No cowrie-docker.log found."
  fi
  
  echo
  echo "=== Fake Web Auth Last 10 ==="
  if [ -f "$COLLECTED_DIR/web_auth.jsonl" ]; then
    tail -n 10 "$COLLECTED_DIR/web_auth.jsonl"
  else
    echo "No web_auth.jsonl found."
  fi
  
  echo
  echo "=== Fake Web Access Last 10 ==="
  if [ -f "$COLLECTED_DIR/web_access.jsonl" ]; then
    tail -n 10 "$COLLECTED_DIR/web_access.jsonl"
  else
    echo "No web_access.jsonl found."
  fi
  EOF
  ```
- 設定可執行：
  ```
  chmod +x /opt/deception-lab/scripts/show_collected_logs.sh
  ```
- 執行測試：
  ```
  /opt/deception-lab/scripts/show_collected_logs.sh
  ```
- 執行結果：
  ```
  lss@lss:/opt/deception-lab $ /opt/deception-lab/scripts/show_collected_logs.sh
  === Collected Directory ===
  total 36K
  drwxrwxr-x 2 lss lss 4.0K May 15 06:53 .
  drwxrwxr-x 8 lss lss 4.0K May 13 03:01 ..
  -rw-rw-r-- 1 lss lss  371 May 15 06:53 collection_summary.txt
  -rw-rw-r-- 1 lss lss 7.7K May 15 06:53 cowrie-docker.log
  -rw-rw-r-- 1 lss lss  777 May 15 06:53 source_manifest.json
  -rwxrwxr-x 1 lss lss 6.5K May 15 06:53 web_access.jsonl
  -rwxrwxr-x 1 lss lss 1.7K May 15 06:53 web_auth.jsonl
  
  === Collection Summary ===
  Collection time UTC: 20260514T225346Z
  
  Collected files:
  
  - /opt/deception-lab/data/collected/cowrie-docker.log
    size: 8.0K
    lines: 63
  - /opt/deception-lab/data/collected/web_access.jsonl
    size: 8.0K
    lines: 19
  - /opt/deception-lab/data/collected/web_auth.jsonl
    size: 4.0K
    lines: 4
  - /opt/deception-lab/data/collected/source_manifest.json
    size: 4.0K
    lines: 26
  
  === Cowrie Interesting Lines ===
  deception-cowrie  | 2026-05-13T02:26:21+0800 [cowrie.ssh.factory.CowrieSSHFactory] New connection: 172.18.0.1:43902 (172.18.0.2:2222) [session: 221da1bdefb7]
  deception-cowrie  | 2026-05-13T02:26:42+0800 [HoneyPotSSHTransport,0,172.18.0.1] login attempt [b'backup'/b'Backup2026!'] succeeded
  deception-cowrie  | 2026-05-13T02:27:07+0800 [HoneyPotSSHTransport,0,172.18.0.1] CMD: whoami
  deception-cowrie  | 2026-05-13T02:27:11+0800 [HoneyPotSSHTransport,0,172.18.0.1] CMD: pwd
  deception-cowrie  | 2026-05-13T02:27:14+0800 [HoneyPotSSHTransport,0,172.18.0.1] CMD: is
  deception-cowrie  | 2026-05-13T02:27:16+0800 [HoneyPotSSHTransport,0,172.18.0.1] CMD: ls
  deception-cowrie  | 2026-05-13T02:27:30+0800 [HoneyPotSSHTransport,0,172.18.0.1] CMD: ls /home/admin
  deception-cowrie  | 2026-05-13T02:27:46+0800 [HoneyPotSSHTransport,0,172.18.0.1] CMD: ls /home/admin
  deception-cowrie  | 2026-05-13T02:27:53+0800 [HoneyPotSSHTransport,0,172.18.0.1] CMD: cat /home/admin/secrets.txt
  deception-cowrie  | 2026-05-13T02:27:57+0800 [HoneyPotSSHTransport,0,172.18.0.1] CMD: exit
  deception-cowrie  | 2026-05-13T02:27:57+0800 [HoneyPotSSHTransport,0,172.18.0.1] avatar backup logging out
  
  === Fake Web Auth Last 10 ===
  {"timestamp": "2026-05-11T21:46:56.695098+00:00", "source": "fake-web", "event_type": "web_login_attempt", "src_ip": "192.168.1.1", "username": "admin", "password_sha256": "8d969eef6ecad3c29a3a629280e686cf0c3f5d5a86aff3ca12020c923adc6c92", "password_length": 6, "honeycredential_used": false, "user_agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.0.0 Safari/537.36", "severity": "medium", "tags": ["web", "login"]}
  {"timestamp": "2026-05-11T22:18:07.651474+00:00", "source": "fake-web", "event_type": "web_login_attempt", "src_ip": "192.168.1.1", "username": "backup", "password_sha256": "0ecba7213823d57d8b9c6510186aa9ba9e401b2f4508e792f8f3ca4aec6394e1", "password_length": 12, "honeycredential_used": false, "user_agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.0.0 Safari/537.36", "severity": "medium", "tags": ["web", "login"]}
  {"timestamp": "2026-05-12T18:36:26.543366+00:00", "source": "fake-web", "event_type": "web_login_attempt", "src_ip": "172.18.0.1", "username": "backup", "password_sha256": "a597108ff8384b3be204d08bce36fe3e86bffcc699fdac27604db48ae6f27f71", "password_length": 11, "honeycredential_used": true, "user_agent": "curl/8.14.1", "severity": "high", "tags": ["web", "login", "honeycredential"]}
  {"timestamp": "2026-05-12T18:41:52.679019+00:00", "source": "fake-web", "event_type": "web_login_attempt", "src_ip": "172.18.0.1", "username": "backup", "password_sha256": "a597108ff8384b3be204d08bce36fe3e86bffcc699fdac27604db48ae6f27f71", "password_length": 11, "honeycredential_used": true, "user_agent": "curl/8.14.1", "severity": "high", "tags": ["web", "login", "honeycredential"]}
  
  === Fake Web Access Last 10 ===
  {"timestamp": "2026-05-11T21:57:09.331926+00:00", "source": "fake-web", "event_type": "web_request", "src_ip": "192.168.1.1", "method": "GET", "path": "/static/style.css", "query_string": "", "user_agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.0.0 Safari/537.36", "is_scanner_probe": false, "tags": ["web", "request"]}
  {"timestamp": "2026-05-11T21:57:12.514618+00:00", "source": "fake-web", "event_type": "web_request", "src_ip": "192.168.1.1", "method": "GET", "path": "/download/secrets.txt", "query_string": "", "user_agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.0.0 Safari/537.36", "is_scanner_probe": false, "tags": ["web", "request"]}
  {"timestamp": "2026-05-11T21:57:12.514844+00:00", "source": "fake-web", "event_type": "web_honeyfile_access", "src_ip": "192.168.1.1", "filename": "secrets.txt", "path": "/download/secrets.txt", "user_agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.0.0 Safari/537.36", "severity": "high", "tags": ["web", "honeyfile", "download"]}
  {"timestamp": "2026-05-11T21:57:19.330990+00:00", "source": "fake-web", "event_type": "web_request", "src_ip": "192.168.1.1", "method": "GET", "path": "/download/vpn_users.csv", "query_string": "", "user_agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.0.0 Safari/537.36", "is_scanner_probe": false, "tags": ["web", "request"]}
  {"timestamp": "2026-05-11T21:57:19.331247+00:00", "source": "fake-web", "event_type": "web_honeyfile_access", "src_ip": "192.168.1.1", "filename": "vpn_users.csv", "path": "/download/vpn_users.csv", "user_agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.0.0 Safari/537.36", "severity": "high", "tags": ["web", "honeyfile", "download"]}
  {"timestamp": "2026-05-11T22:10:43.882087+00:00", "source": "fake-web", "event_type": "web_request", "src_ip": "172.18.0.1", "method": "HEAD", "path": "/login", "query_string": "", "user_agent": "curl/8.14.1", "is_scanner_probe": false, "tags": ["web", "request"]}
  {"timestamp": "2026-05-11T22:18:07.650530+00:00", "source": "fake-web", "event_type": "web_request", "src_ip": "192.168.1.1", "method": "POST", "path": "/login", "query_string": "", "user_agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.0.0 Safari/537.36", "is_scanner_probe": false, "tags": ["web", "request"]}
  {"timestamp": "2026-05-11T22:18:07.668441+00:00", "source": "fake-web", "event_type": "web_request", "src_ip": "192.168.1.1", "method": "GET", "path": "/static/style.css", "query_string": "", "user_agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.0.0 Safari/537.36", "is_scanner_probe": false, "tags": ["web", "request"]}
  {"timestamp": "2026-05-12T18:36:26.542718+00:00", "source": "fake-web", "event_type": "web_request", "src_ip": "172.18.0.1", "method": "POST", "path": "/login", "query_string": "", "user_agent": "curl/8.14.1", "is_scanner_probe": false, "tags": ["web", "request"]}
  {"timestamp": "2026-05-12T18:41:52.678577+00:00", "source": "fake-web", "event_type": "web_request", "src_ip": "172.18.0.1", "method": "POST", "path": "/login", "query_string": "", "user_agent": "curl/8.14.1", "is_scanner_probe": false, "tags": ["web", "request"]}  
  ```

### Step 7.12：建立簡易 log rotation 腳本
honeypot 會一直寫 log，所以要避免 SD 卡被塞滿。
```
這個腳本會：
1. 保留 archive 最近 14 天
2. 顯示 data 目錄用量
3. 不會刪除目前 collected 目錄
```
- 執行：
  ```
  cat > /opt/deception-lab/scripts/cleanup_old_logs.sh <<'EOF'
  #!/usr/bin/env bash
  set -euo pipefail
  
  ARCHIVE_DIR="/opt/deception-lab/data/archive"
  
  echo "=== Disk usage before cleanup ==="
  df -h /
  echo
  du -sh /opt/deception-lab/data || true
  
  echo
  echo "=== Removing archive directories older than 14 days ==="
  if [ -d "$ARCHIVE_DIR" ]; then
    find "$ARCHIVE_DIR" -mindepth 1 -maxdepth 1 -type d -mtime +14 -print -exec rm -rf {} \;
  else
    echo "Archive directory does not exist."
  fi
  
  echo
  echo "=== Disk usage after cleanup ==="
  df -h /
  echo
  du -sh /opt/deception-lab/data || true
  EOF
  ```
- 設定可執行：
  ```
  chmod +x /opt/deception-lab/scripts/cleanup_old_logs.sh
  ```
- 執行測試：
  ```
  /opt/deception-lab/scripts/cleanup_old_logs.sh
  ```
- 執行結果：
  ```
  lss@lss:/opt/deception-lab $ /opt/deception-lab/scripts/cleanup_old_logs.sh
  === Disk usage before cleanup ===
  Filesystem      Size  Used Avail Use% Mounted on
  /dev/mmcblk0p2   58G  5.2G   50G  10% /
  
  120K    /opt/deception-lab/data
  
  === Removing archive directories older than 14 days ===
  
  === Disk usage after cleanup ===
  Filesystem      Size  Used Avail Use% Mounted on
  /dev/mmcblk0p2   58G  5.2G   50G  10% /
  
  120K    /opt/deception-lab/data
  ```

### Step 7.13：建立集中 log 目錄說明檔
- 執行：
```
cat > /opt/deception-lab/data/collected/README.md <<'EOF'
# Collected Logs

This directory contains centralized logs collected from the deception lab.

Files:

- cowrie-docker.log
  - Exported Docker logs from the Cowrie SSH honeypot container.

- web_access.jsonl
  - JSONL access events from the Fake Web Admin Panel.

- web_auth.jsonl
  - JSONL login/authentication events from the Fake Web Admin Panel.

- source_manifest.json
  - Metadata describing log sources and collection time.

- collection_summary.txt
  - Human-readable summary of collected log files.

The Python event parser in Phase 8 will read logs from this directory.
EOF
```

### Step 7.14：更新 status 腳本，加入 collected 目錄狀態
- 現在更新：
```
/opt/deception-lab/scripts/status_lab.sh
```
- 執行：
```
cat > /opt/deception-lab/scripts/status_lab.sh <<'EOF'
#!/usr/bin/env bash
set -e

cd /opt/deception-lab

echo "=== Docker Compose Services ==="
docker compose ps

echo
echo "=== Docker Networks ==="
docker network ls | grep deception || true

echo
echo "=== Listening Ports ==="
sudo ss -tulpn | grep -E ':22|:2222|:8080' || true

echo
echo "=== Disk Usage ==="
df -h /

echo
echo "=== Log Directories ==="
du -sh /opt/deception-lab/data/logs/* 2>/dev/null || true

echo
echo "=== Collected Logs ==="
ls -lah /opt/deception-lab/data/collected 2>/dev/null || true

echo
echo "=== Latest Collection Summary ==="
cat /opt/deception-lab/data/collected/collection_summary.txt 2>/dev/null || true
EOF
```
- 設定權限：
```
chmod +x /opt/deception-lab/scripts/status_lab.sh
```
- 測試：
```
/opt/deception-lab/scripts/status_lab.sh
```
- 執行結果：
```
lss@lss:/opt/deception-lab $ /opt/deception-lab/scripts/status_lab.sh
=== Docker Compose Services ===
NAME                 IMAGE                    COMMAND                  SERVICE    CREATED      STATUS      PORTS
deception-cowrie     cowrie/cowrie:latest     "/cowrie/cowrie-env/…"   cowrie     2 days ago   Up 2 days   0.0.0.0:2222->2222/tcp, [::]:2222->2222/tcp, 2223/tcp
deception-fake-web   deception-lab-fake-web   "gunicorn --bind 0.0…"   fake-web   2 days ago   Up 2 days   0.0.0.0:8080->8080/tcp, [::]:8080->8080/tcp

=== Docker Networks ===
e3ff41c4fcca   deception-lab_deception_net   bridge    local

=== Listening Ports ===
tcp   LISTEN 0      4096         0.0.0.0:8080       0.0.0.0:*    users:(("docker-proxy",pid=10664,fd=8))
tcp   LISTEN 0      4096         0.0.0.0:2222       0.0.0.0:*    users:(("docker-proxy",pid=10581,fd=8))
tcp   LISTEN 0      128          0.0.0.0:22         0.0.0.0:*    users:(("sshd",pid=1136,fd=6))
tcp   LISTEN 0      4096            [::]:8080          [::]:*    users:(("docker-proxy",pid=10670,fd=8))
tcp   LISTEN 0      4096            [::]:2222          [::]:*    users:(("docker-proxy",pid=10589,fd=8))
tcp   LISTEN 0      128             [::]:22            [::]:*    users:(("sshd",pid=1136,fd=7))

=== Disk Usage ===
Filesystem      Size  Used Avail Use% Mounted on
/dev/mmcblk0p2   58G  5.2G   50G  10% /

=== Log Directories ===
12K     /opt/deception-lab/data/logs/cowrie
16K     /opt/deception-lab/data/logs/web

=== Collected Logs ===
total 40K
drwxrwxr-x 2 lss lss 4.0K May 15 07:18 .
drwxrwxr-x 8 lss lss 4.0K May 13 03:01 ..
-rw-rw-r-- 1 lss lss  371 May 15 06:53 collection_summary.txt
-rw-rw-r-- 1 lss lss 7.7K May 15 06:53 cowrie-docker.log
-rw-rw-r-- 1 lss lss  579 May 15 07:18 README.md
-rw-rw-r-- 1 lss lss  777 May 15 06:53 source_manifest.json
-rwxrwxr-x 1 lss lss 6.5K May 15 06:53 web_access.jsonl
-rwxrwxr-x 1 lss lss 1.7K May 15 06:53 web_auth.jsonl

=== Latest Collection Summary ===
Collection time UTC: 20260514T225346Z

Collected files:

- /opt/deception-lab/data/collected/cowrie-docker.log
  size: 8.0K
  lines: 63
- /opt/deception-lab/data/collected/web_access.jsonl
  size: 8.0K
  lines: 19
- /opt/deception-lab/data/collected/web_auth.jsonl
  size: 4.0K
  lines: 4
- /opt/deception-lab/data/collected/source_manifest.json
  size: 4.0K
  lines: 26
```

## Step 7.15：建立第七階段完成紀錄
```
cat > /opt/deception-lab/PHASE7_READY.md <<'EOF'
# Phase 7 Ready

Centralized log collection has been implemented.

Completed items:

- data/collected directory created
- data/archive directory created
- collect_logs.sh created
- show_collected_logs.sh created
- cleanup_old_logs.sh created
- Cowrie Docker logs collected
- Fake Web access logs collected
- Fake Web auth logs collected
- source_manifest.json created
- collection_summary.txt created
- collected logs archived by timestamp
- status_lab.sh updated

Collected log directory:

/opt/deception-lab/data/collected

Main collected files:

- cowrie-docker.log
- web_access.jsonl
- web_auth.jsonl
- source_manifest.json
- collection_summary.txt

Next phase:

Phase 8 - Python event parser and detection rules.
EOF
```
















