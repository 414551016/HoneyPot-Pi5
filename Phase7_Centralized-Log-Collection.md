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









