# 第八階段：Python event parser 與 detection rules
這一階段會把第七階段集中好的 log：
```
/opt/deception-lab/data/collected/cowrie-docker.log
/opt/deception-lab/data/collected/web_access.jsonl
/opt/deception-lab/data/collected/web_auth.jsonl
轉換成：
/opt/deception-lab/data/events/events.jsonl
/opt/deception-lab/data/events/detections.jsonl
/opt/deception-lab/data/events/events_summary.json
也就是把原始 log 變成「標準化事件」和「偵測結果」。
```
## 第八階段目標完成後，你會有：
```
[完成] Python parser
[完成] detection_rules.yml
[完成] 解析 Cowrie Docker log
[完成] 解析 Fake Web access log
[完成] 解析 Fake Web auth log
[完成] 偵測 SSH honeycredential 登入
[完成] 偵測 SSH 指令行為
[完成] 偵測 SSH honeyfile 存取嘗試
[完成] 偵測 Web honeycredential 使用
[完成] 偵測 Web honeyfile 存取
[完成] 輸出 events.jsonl
[完成] 輸出 detections.jsonl
[完成] 建立 run_parser.sh
```

### Step 8.1：進入專案目錄
```
cd /opt/deception-lab
```

### Step 8.2：建立 parser 目錄結構
- 執行：
  ```
  mkdir -p \
    /opt/deception-lab/parser/rules \
    /opt/deception-lab/data/events
  ```
- 確認：
  ```
  tree -L 3 /opt/deception-lab/parser
  # 執行結果
  lss@lss:/opt/deception-lab $ tree -L 3 /opt/deception-lab/parser
  /opt/deception-lab/parser
  └── rules
  
  lss@lss:/opt/deception-lab $ tree -L 3
  .
  ├── cowrie
  │   ├── etc
  │   │   └── userdb.txt
  │   ├── honeyfs
  │   │   ├── etc
  │   │   ├── home
  │   │   ├── opt
  │   │   ├── root
  │   │   ├── tmp
  │   │   └── var
  │   └── var
  │       ├── lib
  │       └── log
  ├── data
  │   ├── archive
  │   │   ├── 20260514T225346Z
  │   │   └── 20260514T232451Z
  │   ├── collected
  │   │   ├── collection_summary.txt
  │   │   ├── cowrie-docker.log
  │   │   ├── README.md
  │   │   ├── source_manifest.json
  │   │   ├── web_access.jsonl
  │   │   └── web_auth.jsonl
  │   ├── events
  │   ├── logs
  │   │   ├── cowrie
  │   │   └── web
  │   ├── raw
  │   └── samples
  │       └── uploads
  ├── deception_assets.yml
  ├── docker-compose.phase3.yml
  ├── docker-compose.phase4.broken.yml
  ├── docker-compose.phase4-cowrie-only.yml
  ├── docker-compose.phase4.permission-error.yml
  ├── docker-compose.phase5.yml
  ├── docker-compose.yml
  ├── fake-web
  │   ├── app.py
  │   ├── Dockerfile
  │   ├── honeyfiles
  │   │   ├── backup_config.ini
  │   │   ├── database_passwords.txt
  │   │   ├── secrets.txt
  │   │   ├── ssh_keys_backup.txt
  │   │   └── vpn_users.csv
  │   ├── requirements.txt
  │   ├── static
  │   │   └── style.css
  │   └── templates
  │       ├── 404.html
  │       ├── backup.html
  │       ├── config.html
  │       ├── dashboard.html
  │       └── login.html
  ├── parser
  │   └── rules
  ├── PHASE2_READY.md
  ├── PHASE3_READY.md
  ├── PHASE4_READY.md
  ├── PHASE5_READY.md
  ├── PHASE6_READY.md
  ├── PHASE7_READY.md
  ├── README.md
  ├── reports
  └── scripts
      ├── check_assets.sh
      ├── check_env.sh
      ├── cleanup_old_logs.sh
      ├── collect_logs.sh
      ├── logs_lab.sh
      ├── restart_lab.sh
      ├── show_collected_logs.sh
      ├── start_lab.sh
      ├── status_lab.sh
      └── stop_lab.sh
  
  33 directories, 45 files
  ```

### Step 8.3：安裝 Python venv 工具
- 先確認 Python 版本：
```
python3 --version

lss@lss:/opt/deception-lab $ python3 --version
Python 3.13.5
```
- 安裝 venv 與 pip：
```
sudo apt install -y python3-venv python3-pip
```

### Step 8.4：建立 parser Python 虛擬環境
- 執行：
```
cd /opt/deception-lab/parser
python3 -m venv venv
```
- 啟用虛擬環境：
```
source /opt/deception-lab/parser/venv/bin/activate

# 執行結果：你應該會看到 shell 前面多出：(venv)
lss@lss:/opt/deception-lab/parser $ source /opt/deception-lab/parser/venv/bin/activate
(venv) lss@lss:/opt/deception-lab/parser $
```

### Step 8.5：建立 parser requirements 解析器要求
- 執行：
```
cat > /opt/deception-lab/parser/requirements.txt <<'EOF'
PyYAML==6.0.2
EOF
```
- 安裝套件：
```
pip install -r /opt/deception-lab/parser/requirements.txt
```
- 安裝完成後可以先離開 venv：
```
deactivate
```

### Step 8.6：建立 detection rules
這份檔案定義第八階段要偵測的攻擊行為。
- 執行：
```
cat > /opt/deception-lab/parser/rules/detection_rules.yml <<'EOF'
rules:
  - id: SSH_HONEYCREDENTIAL_LOGIN
    name: SSH honeycredential login succeeded
    source: cowrie
    event_type: ssh_login_success
    severity: high
    description: Attacker successfully logged into Cowrie using a known honeycredential.
    tags:
      - ssh
      - honeycredential
      - deception
    attack:
      tactic: Credential Access
      technique_id: T1552
      technique: Unsecured Credentials
    engage:
      goal: Elicit
      activity: Credential Collection

  - id: SSH_LOGIN_FAILED
    name: SSH login failed
    source: cowrie
    event_type: ssh_login_failed
    severity: medium
    description: Failed SSH login attempt observed by Cowrie.
    tags:
      - ssh
      - login-failed
      - brute-force
    attack:
      tactic: Credential Access
      technique_id: T1110
      technique: Brute Force
    engage:
      goal: Expose
      activity: Expose Decoy Service

  - id: SSH_RECON_COMMAND
    name: SSH reconnaissance command
    source: cowrie
    event_type: ssh_command
    severity: medium
    description: Attacker executed common discovery or reconnaissance commands.
    command_keywords:
      - whoami
      - id
      - uname
      - pwd
      - ls
      - ip addr
      - ifconfig
      - ps
      - netstat
    tags:
      - ssh
      - command
      - discovery
    attack:
      tactic: Discovery
      technique_id: T1082
      technique: System Information Discovery
    engage:
      goal: Understand
      activity: Collect Adversary Behavior

  - id: SSH_TOOL_TRANSFER_COMMAND
    name: SSH tool transfer command
    source: cowrie
    event_type: ssh_command
    severity: high
    description: Attacker attempted to download or stage tools.
    command_keywords:
      - wget
      - curl
      - ftp
      - tftp
      - scp
      - chmod
      - bash -i
      - nc
      - netcat
    tags:
      - ssh
      - tool-transfer
      - payload
    attack:
      tactic: Command and Control
      technique_id: T1105
      technique: Ingress Tool Transfer
    engage:
      goal: Affect
      activity: Adversary Direction

  - id: SSH_HONEYFILE_ACCESS
    name: SSH honeyfile access attempt
    source: cowrie
    event_type: ssh_command
    severity: high
    description: Attacker attempted to access a known honeyfile inside Cowrie.
    command_keywords:
      - secrets.txt
      - backup_config.ini
      - vpn_users.csv
      - ssh_keys_backup.txt
      - database_passwords.txt
      - backup_jobs.txt
      - maintenance_notes.txt
      - app.conf
    tags:
      - ssh
      - honeyfile
      - collection
    attack:
      tactic: Collection
      technique_id: T1005
      technique: Data from Local System
    engage:
      goal: Elicit
      activity: Reveal Adversary Intent

  - id: WEB_HONEYCREDENTIAL_USED
    name: Web honeycredential used
    source: fake-web
    event_type: web_login_attempt
    severity: high
    description: User submitted a known honeycredential to the fake web login panel.
    require_field:
      field: honeycredential_used
      equals: true
    tags:
      - web
      - login
      - honeycredential
    attack:
      tactic: Credential Access
      technique_id: T1552
      technique: Unsecured Credentials
    engage:
      goal: Elicit
      activity: Credential Collection

  - id: WEB_HONEYFILE_ACCESS
    name: Web honeyfile accessed
    source: fake-web
    event_type: web_honeyfile_access
    severity: high
    description: User accessed or downloaded a honeyfile from the fake web panel.
    tags:
      - web
      - honeyfile
      - download
    attack:
      tactic: Collection
      technique_id: T1005
      technique: Data from Local System
    engage:
      goal: Understand
      activity: Collect Adversary Behavior

  - id: WEB_SCANNER_PROBE
    name: Web scanner probe
    source: fake-web
    event_type: web_request
    severity: medium
    description: User requested a path commonly associated with scanning or probing.
    require_field:
      field: is_scanner_probe
      equals: true
    tags:
      - web
      - scanner
      - discovery
    attack:
      tactic: Reconnaissance
      technique_id: T1595
      technique: Active Scanning
    engage:
      goal: Expose
      activity: Expose Decoy Service
EOF
```
- 確認：
```
cat /opt/deception-lab/parser/rules/detection_rules.yml
```

### Step 8.7：建立 Python event parser
```
# 這支程式會做：
1. 讀取 collected/cowrie-docker.log
2. 讀取 collected/web_access.jsonl
3. 讀取 collected/web_auth.jsonl
4. 正規化成 events
5. 套用 detection_rules.yml
6. 輸出 events.jsonl
7. 輸出 detections.jsonl
8. 輸出 events_summary.json
```
- 執行：
```
cat > /opt/deception-lab/parser/parse_events.py <<'EOF'
#!/usr/bin/env python3
import argparse
import hashlib
import json
import re
from collections import Counter
from datetime import datetime, timezone
from pathlib import Path

import yaml


COWRIE_TIME_RE = re.compile(
    r"(?P<ts>\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}[+-]\d{4})"
)

COWRIE_NEW_CONN_RE = re.compile(
    r"New connection: (?P<src_ip>[0-9a-fA-F:\.]+):(?P<src_port>\d+)"
)

COWRIE_TRANSPORT_IP_RE = re.compile(
    r"\[HoneyPotSSHTransport,\d+,(?P<src_ip>[0-9a-fA-F:\.]+)\]"
)

COWRIE_LOGIN_RE = re.compile(
    r"login attempt \[b'(?P<username>[^']*)'/b'(?P<password>[^']*)'\] (?P<result>succeeded|failed)"
)

COWRIE_CMD_RE = re.compile(
    r"CMD: (?P<command>.*)$"
)

COWRIE_LOGOUT_RE = re.compile(
    r"avatar (?P<username>\S+) logging out"
)


def parse_args():
    parser = argparse.ArgumentParser(
        description="Parse deception lab logs into normalized events and detections."
    )
    parser.add_argument(
        "--project-root",
        default="/opt/deception-lab",
        help="Project root directory."
    )
    return parser.parse_args()


def parse_cowrie_timestamp(line):
    match = COWRIE_TIME_RE.search(line)
    if not match:
        return None

    value = match.group("ts")
    try:
        dt = datetime.strptime(value, "%Y-%m-%dT%H:%M:%S%z")
        return dt.astimezone(timezone.utc).isoformat()
    except ValueError:
        return None


def extract_cowrie_src_ip(line):
    match = COWRIE_NEW_CONN_RE.search(line)
    if match:
        return match.group("src_ip")

    match = COWRIE_TRANSPORT_IP_RE.search(line)
    if match:
        return match.group("src_ip")

    return None


def sha256_text(value):
    return hashlib.sha256(value.encode("utf-8")).hexdigest()


def load_yaml(path):
    with open(path, "r", encoding="utf-8") as f:
        return yaml.safe_load(f)


def load_honeycredentials(assets_path):
    if not assets_path.exists():
        return {}

    data = load_yaml(assets_path) or {}
    creds = {}

    for item in data.get("honeycredentials", []):
        username = item.get("username")
        password = item.get("password")
        if username and password:
            creds[(username, password)] = item

    return creds


def normalize_base(timestamp, source, event_type, src_ip=None):
    return {
        "timestamp": timestamp or datetime.now(timezone.utc).isoformat(),
        "source": source,
        "event_type": event_type,
        "src_ip": src_ip,
        "severity": "info",
        "tags": [],
        "details": {}
    }


def parse_cowrie_log(path, honeycredentials):
    events = []

    if not path.exists():
        return events

    for line in path.read_text(encoding="utf-8", errors="ignore").splitlines():
        timestamp = parse_cowrie_timestamp(line)
        src_ip = extract_cowrie_src_ip(line)

        if "New connection:" in line:
            match = COWRIE_NEW_CONN_RE.search(line)
            event = normalize_base(timestamp, "cowrie", "ssh_connection", src_ip)
            if match:
                event["src_port"] = int(match.group("src_port"))
            event["severity"] = "info"
            event["tags"] = ["ssh", "connection"]
            event["raw"] = line
            events.append(event)
            continue

        login_match = COWRIE_LOGIN_RE.search(line)
        if login_match:
            username = login_match.group("username")
            password = login_match.group("password")
            result = login_match.group("result")

            event_type = "ssh_login_success" if result == "succeeded" else "ssh_login_failed"
            event = normalize_base(timestamp, "cowrie", event_type, src_ip)
            event["username"] = username
            event["password_sha256"] = sha256_text(password)
            event["password_length"] = len(password)
            event["honeycredential_used"] = (username, password) in honeycredentials
            event["severity"] = "high" if event["honeycredential_used"] and result == "succeeded" else "medium"
            event["tags"] = ["ssh", "login"]
            if result == "succeeded":
                event["tags"].append("login-success")
            else:
                event["tags"].append("login-failed")
            if event["honeycredential_used"]:
                event["tags"].append("honeycredential")
                event["details"]["honeycredential_id"] = honeycredentials[(username, password)].get("id")
            event["raw"] = line
            events.append(event)
            continue

        cmd_match = COWRIE_CMD_RE.search(line)
        if cmd_match:
            command = cmd_match.group("command").strip()
            event = normalize_base(timestamp, "cowrie", "ssh_command", src_ip)
            event["command"] = command
            event["severity"] = "medium"
            event["tags"] = ["ssh", "command"]
            event["raw"] = line
            events.append(event)
            continue

        logout_match = COWRIE_LOGOUT_RE.search(line)
        if logout_match:
            event = normalize_base(timestamp, "cowrie", "ssh_logout", src_ip)
            event["username"] = logout_match.group("username")
            event["severity"] = "info"
            event["tags"] = ["ssh", "logout"]
            event["raw"] = line
            events.append(event)
            continue

    return events


def parse_jsonl(path):
    events = []

    if not path.exists():
        return events

    for line_no, line in enumerate(path.read_text(encoding="utf-8", errors="ignore").splitlines(), start=1):
        line = line.strip()
        if not line:
            continue

        try:
            obj = json.loads(line)
        except json.JSONDecodeError:
            events.append({
                "timestamp": datetime.now(timezone.utc).isoformat(),
                "source": "parser",
                "event_type": "parse_error",
                "severity": "low",
                "tags": ["parse-error"],
                "details": {
                    "file": str(path),
                    "line_no": line_no,
                    "line": line
                }
            })
            continue

        obj.setdefault("source", "fake-web")
        obj.setdefault("timestamp", datetime.now(timezone.utc).isoformat())
        obj.setdefault("severity", "info")
        obj.setdefault("tags", [])

        if "details" not in obj:
            obj["details"] = {}

        events.append(obj)

    return events


def field_matches(event, requirement):
    if not requirement:
        return True

    field = requirement.get("field")
    expected = requirement.get("equals")

    return event.get(field) == expected


def command_matches(event, keywords):
    command = event.get("command", "")
    command_lower = command.lower()

    for keyword in keywords or []:
        if keyword.lower() in command_lower:
            return True

    return False


def apply_detection_rules(events, rules):
    detections = []

    for idx, event in enumerate(events):
        for rule in rules:
            if rule.get("source") != event.get("source"):
                continue

            if rule.get("event_type") != event.get("event_type"):
                continue

            if not field_matches(event, rule.get("require_field")):
                continue

            if "command_keywords" in rule:
                if not command_matches(event, rule.get("command_keywords")):
                    continue

            detection = {
                "timestamp": event.get("timestamp"),
                "rule_id": rule.get("id"),
                "rule_name": rule.get("name"),
                "source": event.get("source"),
                "event_type": event.get("event_type"),
                "src_ip": event.get("src_ip"),
                "username": event.get("username"),
                "command": event.get("command"),
                "path": event.get("path"),
                "filename": event.get("filename"),
                "severity": rule.get("severity", event.get("severity", "info")),
                "description": rule.get("description"),
                "tags": sorted(set(event.get("tags", []) + rule.get("tags", []))),
                "event_index": idx,
                "attack_mapping": rule.get("attack", {}),
                "engage_mapping": rule.get("engage", {})
            }

            detections.append(detection)

    return detections


def write_jsonl(path, records):
    path.parent.mkdir(parents=True, exist_ok=True)

    with open(path, "w", encoding="utf-8") as f:
        for record in records:
            f.write(json.dumps(record, ensure_ascii=False, sort_keys=True) + "\n")


def write_summary(path, events, detections):
    summary = {
        "generated_at": datetime.now(timezone.utc).isoformat(),
        "total_events": len(events),
        "total_detections": len(detections),
        "events_by_source": Counter(e.get("source") for e in events),
        "events_by_type": Counter(e.get("event_type") for e in events),
        "detections_by_rule": Counter(d.get("rule_id") for d in detections),
        "detections_by_severity": Counter(d.get("severity") for d in detections),
    }

    serializable = {
        k: dict(v) if isinstance(v, Counter) else v
        for k, v in summary.items()
    }

    path.parent.mkdir(parents=True, exist_ok=True)
    path.write_text(
        json.dumps(serializable, ensure_ascii=False, indent=2, sort_keys=True),
        encoding="utf-8"
    )


def main():
    args = parse_args()
    project_root = Path(args.project_root)

    collected_dir = project_root / "data" / "collected"
    events_dir = project_root / "data" / "events"
    rules_path = project_root / "parser" / "rules" / "detection_rules.yml"
    assets_path = project_root / "deception_assets.yml"

    cowrie_log = collected_dir / "cowrie-docker.log"
    web_access = collected_dir / "web_access.jsonl"
    web_auth = collected_dir / "web_auth.jsonl"

    honeycredentials = load_honeycredentials(assets_path)
    rules_data = load_yaml(rules_path) or {}
    rules = rules_data.get("rules", [])

    events = []
    events.extend(parse_cowrie_log(cowrie_log, honeycredentials))
    events.extend(parse_jsonl(web_access))
    events.extend(parse_jsonl(web_auth))

    events.sort(key=lambda e: e.get("timestamp", ""))

    detections = apply_detection_rules(events, rules)
    detections.sort(key=lambda d: d.get("timestamp", ""))

    write_jsonl(events_dir / "events.jsonl", events)
    write_jsonl(events_dir / "detections.jsonl", detections)
    write_summary(events_dir / "events_summary.json", events, detections)

    print("[+] Parsing completed.")
    print(f"[+] Events written to: {events_dir / 'events.jsonl'}")
    print(f"[+] Detections written to: {events_dir / 'detections.jsonl'}")
    print(f"[+] Summary written to: {events_dir / 'events_summary.json'}")
    print()
    print(f"Total events: {len(events)}")
    print(f"Total detections: {len(detections)}")


if __name__ == "__main__":
    main()
EOF
```
- 設定可執行：
```
chmod +x /opt/deception-lab/parser/parse_events.py
```

### Step 8.8：建立執行 parser 的腳本
- 執行：
```
cat > /opt/deception-lab/scripts/run_parser.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

PROJECT_ROOT="/opt/deception-lab"
PARSER_VENV="$PROJECT_ROOT/parser/venv"
PARSER="$PROJECT_ROOT/parser/parse_events.py"

echo "[+] Collecting latest logs..."
"$PROJECT_ROOT/scripts/collect_logs.sh"

echo
echo "[+] Running event parser..."
source "$PARSER_VENV/bin/activate"
python "$PARSER" --project-root "$PROJECT_ROOT"
deactivate

echo
echo "[+] Event output files:"
ls -lah "$PROJECT_ROOT/data/events"

echo
echo "[+] Event summary:"
cat "$PROJECT_ROOT/data/events/events_summary.json"
EOF
```
- 設定可執行：
```
chmod +x /opt/deception-lab/scripts/run_parser.sh
```

### Step 8.9：執行 parser
- 現在執行：
```
/opt/deception-lab/scripts/run_parser.sh

# 成功的話會看到類似：
[+] Parsing completed.
[+] Events written to: /opt/deception-lab/data/events/events.jsonl
[+] Detections written to: /opt/deception-lab/data/events/detections.jsonl
[+] Summary written to: /opt/deception-lab/data/events/events_summary.json

Total events: ...
Total detections: ...

# 執行結果：
lss@lss:/opt/deception-lab/parser $ /opt/deception-lab/scripts/run_parser.sh
[+] Collecting latest logs...
[+] Collecting logs at UTC time: 20260519T192850Z
[+] Exporting Cowrie Docker logs...
[+] Collecting Fake Web access log...
[+] Collecting Fake Web auth log...
[+] Creating source manifest...
[+] Creating collection summary...
[+] Archiving collected logs...

[+] Collection completed.
[+] Current collected directory:
total 40K
drwxrwxr-x 2 lss lss 4.0K May 15 07:18 .
drwxrwxr-x 8 lss lss 4.0K May 13 03:01 ..
-rw-rw-r-- 1 lss lss  371 May 20 03:28 collection_summary.txt
-rw-rw-r-- 1 lss lss 7.7K May 20 03:28 cowrie-docker.log
-rw-rw-r-- 1 lss lss  579 May 15 07:18 README.md
-rw-rw-r-- 1 lss lss  777 May 20 03:28 source_manifest.json
-rwxrwxr-x 1 lss lss 6.5K May 20 03:28 web_access.jsonl
-rwxrwxr-x 1 lss lss 1.7K May 20 03:28 web_auth.jsonl

[+] Summary:
Collection time UTC: 20260519T192850Z

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

[+] Running event parser...
[+] Parsing completed.
[+] Events written to: /opt/deception-lab/data/events/events.jsonl
[+] Detections written to: /opt/deception-lab/data/events/detections.jsonl
[+] Summary written to: /opt/deception-lab/data/events/events_summary.json

Total events: 34
Total detections: 11

[+] Event output files:
total 36K
drwxrwxr-x 2 lss lss 4.0K May 20 03:28 .
drwxrwxr-x 8 lss lss 4.0K May 13 03:01 ..
-rw-rw-r-- 1 lss lss 6.9K May 20 03:28 detections.jsonl
-rw-rw-r-- 1 lss lss  13K May 20 03:28 events.jsonl
-rw-rw-r-- 1 lss lss  631 May 20 03:28 events_summary.json

[+] Event summary:
{
  "detections_by_rule": {
    "SSH_HONEYCREDENTIAL_LOGIN": 1,
    "SSH_HONEYFILE_ACCESS": 1,
    "SSH_RECON_COMMAND": 5,
    "WEB_HONEYCREDENTIAL_USED": 2,
    "WEB_HONEYFILE_ACCESS": 2
  },
  "detections_by_severity": {
    "high": 6,
    "medium": 5
  },
  "events_by_source": {
    "cowrie": 11,
    "fake-web": 23
  },
  "events_by_type": {
    "ssh_command": 8,
    "ssh_connection": 1,
    "ssh_login_success": 1,
    "ssh_logout": 1,
    "web_honeyfile_access": 2,
    "web_login_attempt": 4,
    "web_request": 17
  },
  "generated_at": "2026-05-19T19:28:50.271764+00:00",
  "total_detections": 11,
  "total_events": 34

```
























