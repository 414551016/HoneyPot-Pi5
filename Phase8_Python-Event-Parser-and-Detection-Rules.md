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

### Step 8.10：檢查輸出檔案
- 執行：
```
ls -lah /opt/deception-lab/data/events

# 執行結果：
lss@lss:/opt/deception-lab/parser $ ls -lah /opt/deception-lab/data/events
total 36K
drwxrwxr-x 2 lss lss 4.0K May 20 03:28 .
drwxrwxr-x 8 lss lss 4.0K May 13 03:01 ..
-rw-rw-r-- 1 lss lss 6.9K May 20 03:28 detections.jsonl
-rw-rw-r-- 1 lss lss  13K May 20 03:28 events.jsonl
-rw-rw-r-- 1 lss lss  631 May 20 03:28 events_summary.json
```

### Step 8.11：查看 events summary
- 執行：
```
cat /opt/deception-lab/data/events/events_summary.json | jq

# 執行結果：你應該會看到類似：JSON
lss@lss:/opt/deception-lab/parser $ cat /opt/deception-lab/data/events/events_summary.json | jq
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
}
```

### Step 8.12：查看標準化事件
- 查看最後 10 筆事件：
```
tail -n 10 /opt/deception-lab/data/events/events.jsonl | jq

# 執行結果：如果 jq 顯示多筆 JSON，代表 events 輸出正常。
lss@lss:/opt/deception-lab/parser $ tail -n 10 /opt/deception-lab/data/events/events.jsonl | jq
{
  "command": "ls",
  "details": {},
  "event_type": "ssh_command",
  "raw": "deception-cowrie  | 2026-05-13T02:27:16+0800 [HoneyPotSSHTransport,0,172.18.0.1] CMD: ls",
  "severity": "medium",
  "source": "cowrie",
  "src_ip": "172.18.0.1",
  "tags": [
    "ssh",
    "command"
  ],
  "timestamp": "2026-05-12T18:27:16+00:00"
}
{
  "command": "ls /home/admin",
  "details": {},
  "event_type": "ssh_command",
  "raw": "deception-cowrie  | 2026-05-13T02:27:30+0800 [HoneyPotSSHTransport,0,172.18.0.1] CMD: ls /home/admin",
  "severity": "medium",
  "source": "cowrie",
  "src_ip": "172.18.0.1",
  "tags": [
    "ssh",
    "command"
  ],
  "timestamp": "2026-05-12T18:27:30+00:00"
}
{
  "command": "ls /home/admin",
  "details": {},
  "event_type": "ssh_command",
  "raw": "deception-cowrie  | 2026-05-13T02:27:46+0800 [HoneyPotSSHTransport,0,172.18.0.1] CMD: ls /home/admin",
  "severity": "medium",
  "source": "cowrie",
  "src_ip": "172.18.0.1",
  "tags": [
    "ssh",
    "command"
  ],
  "timestamp": "2026-05-12T18:27:46+00:00"
}
{
  "command": "cat /home/admin/secrets.txt",
  "details": {},
  "event_type": "ssh_command",
  "raw": "deception-cowrie  | 2026-05-13T02:27:53+0800 [HoneyPotSSHTransport,0,172.18.0.1] CMD: cat /home/admin/secrets.txt",
  "severity": "medium",
  "source": "cowrie",
  "src_ip": "172.18.0.1",
  "tags": [
    "ssh",
    "command"
  ],
  "timestamp": "2026-05-12T18:27:53+00:00"
}
{
  "command": "exit",
  "details": {},
  "event_type": "ssh_command",
  "raw": "deception-cowrie  | 2026-05-13T02:27:57+0800 [HoneyPotSSHTransport,0,172.18.0.1] CMD: exit",
  "severity": "medium",
  "source": "cowrie",
  "src_ip": "172.18.0.1",
  "tags": [
    "ssh",
    "command"
  ],
  "timestamp": "2026-05-12T18:27:57+00:00"
}
{
  "details": {},
  "event_type": "ssh_logout",
  "raw": "deception-cowrie  | 2026-05-13T02:27:57+0800 [HoneyPotSSHTransport,0,172.18.0.1] avatar backup logging out",
  "severity": "info",
  "source": "cowrie",
  "src_ip": "172.18.0.1",
  "tags": [
    "ssh",
    "logout"
  ],
  "timestamp": "2026-05-12T18:27:57+00:00",
  "username": "backup"
}
{
  "details": {},
  "event_type": "web_request",
  "is_scanner_probe": false,
  "method": "POST",
  "path": "/login",
  "query_string": "",
  "severity": "info",
  "source": "fake-web",
  "src_ip": "172.18.0.1",
  "tags": [
    "web",
    "request"
  ],
  "timestamp": "2026-05-12T18:36:26.542718+00:00",
  "user_agent": "curl/8.14.1"
}
{
  "details": {},
  "event_type": "web_login_attempt",
  "honeycredential_used": true,
  "password_length": 11,
  "password_sha256": "a597108ff8384b3be204d08bce36fe3e86bffcc699fdac27604db48ae6f27f71",
  "severity": "high",
  "source": "fake-web",
  "src_ip": "172.18.0.1",
  "tags": [
    "web",
    "login",
    "honeycredential"
  ],
  "timestamp": "2026-05-12T18:36:26.543366+00:00",
  "user_agent": "curl/8.14.1",
  "username": "backup"
}
{
  "details": {},
  "event_type": "web_request",
  "is_scanner_probe": false,
  "method": "POST",
  "path": "/login",
  "query_string": "",
  "severity": "info",
  "source": "fake-web",
  "src_ip": "172.18.0.1",
  "tags": [
    "web",
    "request"
  ],
  "timestamp": "2026-05-12T18:41:52.678577+00:00",
  "user_agent": "curl/8.14.1"
}
{
  "details": {},
  "event_type": "web_login_attempt",
  "honeycredential_used": true,
  "password_length": 11,
  "password_sha256": "a597108ff8384b3be204d08bce36fe3e86bffcc699fdac27604db48ae6f27f71",
  "severity": "high",
  "source": "fake-web",
  "src_ip": "172.18.0.1",
  "tags": [
    "web",
    "login",
    "honeycredential"
  ],
  "timestamp": "2026-05-12T18:41:52.679019+00:00",
  "user_agent": "curl/8.14.1",
  "username": "backup"
}
```
- 如果你想看 Cowrie SSH 指令事件：
```
grep '"event_type": "ssh_command"' /opt/deception-lab/data/events/events.jsonl | jq

# 執行結果：
lss@lss:/opt/deception-lab/parser $ tail -n 10 /opt/deception-lab/data/events/events.jsonl | jq
{
  "command": "ls",
  "details": {},
  "event_type": "ssh_command",
  "raw": "deception-cowrie  | 2026-05-13T02:27:16+0800 [HoneyPotSSHTransport,0,172.18.0.1] CMD: ls",
  "severity": "medium",
  "source": "cowrie",
  "src_ip": "172.18.0.1",
  "tags": [
    "ssh",
    "command"
  ],
  "timestamp": "2026-05-12T18:27:16+00:00"
}
{
  "command": "ls /home/admin",
  "details": {},
  "event_type": "ssh_command",
  "raw": "deception-cowrie  | 2026-05-13T02:27:30+0800 [HoneyPotSSHTransport,0,172.18.0.1] CMD: ls /home/admin",
  "severity": "medium",
  "source": "cowrie",
  "src_ip": "172.18.0.1",
  "tags": [
    "ssh",
    "command"
  ],
  "timestamp": "2026-05-12T18:27:30+00:00"
}
{
  "command": "ls /home/admin",
  "details": {},
  "event_type": "ssh_command",
  "raw": "deception-cowrie  | 2026-05-13T02:27:46+0800 [HoneyPotSSHTransport,0,172.18.0.1] CMD: ls /home/admin",
  "severity": "medium",
  "source": "cowrie",
  "src_ip": "172.18.0.1",
  "tags": [
    "ssh",
    "command"
  ],
  "timestamp": "2026-05-12T18:27:46+00:00"
}
{
  "command": "cat /home/admin/secrets.txt",
  "details": {},
  "event_type": "ssh_command",
  "raw": "deception-cowrie  | 2026-05-13T02:27:53+0800 [HoneyPotSSHTransport,0,172.18.0.1] CMD: cat /home/admin/secrets.txt",
  "severity": "medium",
  "source": "cowrie",
  "src_ip": "172.18.0.1",
  "tags": [
    "ssh",
    "command"
  ],
  "timestamp": "2026-05-12T18:27:53+00:00"
}
{
  "command": "exit",
  "details": {},
  "event_type": "ssh_command",
  "raw": "deception-cowrie  | 2026-05-13T02:27:57+0800 [HoneyPotSSHTransport,0,172.18.0.1] CMD: exit",
  "severity": "medium",
  "source": "cowrie",
  "src_ip": "172.18.0.1",
  "tags": [
    "ssh",
    "command"
  ],
  "timestamp": "2026-05-12T18:27:57+00:00"
}
{
  "details": {},
  "event_type": "ssh_logout",
  "raw": "deception-cowrie  | 2026-05-13T02:27:57+0800 [HoneyPotSSHTransport,0,172.18.0.1] avatar backup logging out",
  "severity": "info",
  "source": "cowrie",
  "src_ip": "172.18.0.1",
  "tags": [
    "ssh",
    "logout"
  ],
  "timestamp": "2026-05-12T18:27:57+00:00",
  "username": "backup"
}
{
  "details": {},
  "event_type": "web_request",
  "is_scanner_probe": false,
  "method": "POST",
  "path": "/login",
  "query_string": "",
  "severity": "info",
  "source": "fake-web",
  "src_ip": "172.18.0.1",
  "tags": [
    "web",
    "request"
  ],
  "timestamp": "2026-05-12T18:36:26.542718+00:00",
  "user_agent": "curl/8.14.1"
}
{
  "details": {},
  "event_type": "web_login_attempt",
  "honeycredential_used": true,
  "password_length": 11,
  "password_sha256": "a597108ff8384b3be204d08bce36fe3e86bffcc699fdac27604db48ae6f27f71",
  "severity": "high",
  "source": "fake-web",
  "src_ip": "172.18.0.1",
  "tags": [
    "web",
    "login",
    "honeycredential"
  ],
  "timestamp": "2026-05-12T18:36:26.543366+00:00",
  "user_agent": "curl/8.14.1",
  "username": "backup"
}
{
  "details": {},
  "event_type": "web_request",
  "is_scanner_probe": false,
  "method": "POST",
  "path": "/login",
  "query_string": "",
  "severity": "info",
  "source": "fake-web",
  "src_ip": "172.18.0.1",
  "tags": [
    "web",
    "request"
  ],
  "timestamp": "2026-05-12T18:41:52.678577+00:00",
  "user_agent": "curl/8.14.1"
}
{
  "details": {},
  "event_type": "web_login_attempt",
  "honeycredential_used": true,
  "password_length": 11,
  "password_sha256": "a597108ff8384b3be204d08bce36fe3e86bffcc699fdac27604db48ae6f27f71",
  "severity": "high",
  "source": "fake-web",
  "src_ip": "172.18.0.1",
  "tags": [
    "web",
    "login",
    "honeycredential"
  ],
  "timestamp": "2026-05-12T18:41:52.679019+00:00",
  "user_agent": "curl/8.14.1",
  "username": "backup"
}
lss@lss:/opt/deception-lab/parser $ ^C
lss@lss:/opt/deception-lab/parser $ grep '"event_type": "ssh_command"' /opt/deception-lab/data/events/events.jsonl | jq
{
  "command": "whoami",
  "details": {},
  "event_type": "ssh_command",
  "raw": "deception-cowrie  | 2026-05-13T02:27:07+0800 [HoneyPotSSHTransport,0,172.18.0.1] CMD: whoami",
  "severity": "medium",
  "source": "cowrie",
  "src_ip": "172.18.0.1",
  "tags": [
    "ssh",
    "command"
  ],
  "timestamp": "2026-05-12T18:27:07+00:00"
}
{
  "command": "pwd",
  "details": {},
  "event_type": "ssh_command",
  "raw": "deception-cowrie  | 2026-05-13T02:27:11+0800 [HoneyPotSSHTransport,0,172.18.0.1] CMD: pwd",
  "severity": "medium",
  "source": "cowrie",
  "src_ip": "172.18.0.1",
  "tags": [
    "ssh",
    "command"
  ],
  "timestamp": "2026-05-12T18:27:11+00:00"
}
{
  "command": "is",
  "details": {},
  "event_type": "ssh_command",
  "raw": "deception-cowrie  | 2026-05-13T02:27:14+0800 [HoneyPotSSHTransport,0,172.18.0.1] CMD: is",
  "severity": "medium",
  "source": "cowrie",
  "src_ip": "172.18.0.1",
  "tags": [
    "ssh",
    "command"
  ],
  "timestamp": "2026-05-12T18:27:14+00:00"
}
{
  "command": "ls",
  "details": {},
  "event_type": "ssh_command",
  "raw": "deception-cowrie  | 2026-05-13T02:27:16+0800 [HoneyPotSSHTransport,0,172.18.0.1] CMD: ls",
  "severity": "medium",
  "source": "cowrie",
  "src_ip": "172.18.0.1",
  "tags": [
    "ssh",
    "command"
  ],
  "timestamp": "2026-05-12T18:27:16+00:00"
}
{
  "command": "ls /home/admin",
  "details": {},
  "event_type": "ssh_command",
  "raw": "deception-cowrie  | 2026-05-13T02:27:30+0800 [HoneyPotSSHTransport,0,172.18.0.1] CMD: ls /home/admin",
  "severity": "medium",
  "source": "cowrie",
  "src_ip": "172.18.0.1",
  "tags": [
    "ssh",
    "command"
  ],
  "timestamp": "2026-05-12T18:27:30+00:00"
}
{
  "command": "ls /home/admin",
  "details": {},
  "event_type": "ssh_command",
  "raw": "deception-cowrie  | 2026-05-13T02:27:46+0800 [HoneyPotSSHTransport,0,172.18.0.1] CMD: ls /home/admin",
  "severity": "medium",
  "source": "cowrie",
  "src_ip": "172.18.0.1",
  "tags": [
    "ssh",
    "command"
  ],
  "timestamp": "2026-05-12T18:27:46+00:00"
}
{
  "command": "cat /home/admin/secrets.txt",
  "details": {},
  "event_type": "ssh_command",
  "raw": "deception-cowrie  | 2026-05-13T02:27:53+0800 [HoneyPotSSHTransport,0,172.18.0.1] CMD: cat /home/admin/secrets.txt",
  "severity": "medium",
  "source": "cowrie",
  "src_ip": "172.18.0.1",
  "tags": [
    "ssh",
    "command"
  ],
  "timestamp": "2026-05-12T18:27:53+00:00"
}
{
  "command": "exit",
  "details": {},
  "event_type": "ssh_command",
  "raw": "deception-cowrie  | 2026-05-13T02:27:57+0800 [HoneyPotSSHTransport,0,172.18.0.1] CMD: exit",
  "severity": "medium",
  "source": "cowrie",
  "src_ip": "172.18.0.1",
  "tags": [
    "ssh",
    "command"
  ],
  "timestamp": "2026-05-12T18:27:57+00:00"
}
```
- 如果你想看 Web honeycredential 事件：
```
grep '"honeycredential_used": true' /opt/deception-lab/data/events/events.jsonl | jq

# 執行結果：
lss@lss:/opt/deception-lab/parser $ grep '"event_type": "ssh_command"' /opt/deception-lab/data/events/events.jsonl | jq
{
  "command": "whoami",
  "details": {},
  "event_type": "ssh_command",
  "raw": "deception-cowrie  | 2026-05-13T02:27:07+0800 [HoneyPotSSHTransport,0,172.18.0.1] CMD: whoami",
  "severity": "medium",
  "source": "cowrie",
  "src_ip": "172.18.0.1",
  "tags": [
    "ssh",
    "command"
  ],
  "timestamp": "2026-05-12T18:27:07+00:00"
}
{
  "command": "pwd",
  "details": {},
  "event_type": "ssh_command",
  "raw": "deception-cowrie  | 2026-05-13T02:27:11+0800 [HoneyPotSSHTransport,0,172.18.0.1] CMD: pwd",
  "severity": "medium",
  "source": "cowrie",
  "src_ip": "172.18.0.1",
  "tags": [
    "ssh",
    "command"
  ],
  "timestamp": "2026-05-12T18:27:11+00:00"
}
{
  "command": "is",
  "details": {},
  "event_type": "ssh_command",
  "raw": "deception-cowrie  | 2026-05-13T02:27:14+0800 [HoneyPotSSHTransport,0,172.18.0.1] CMD: is",
  "severity": "medium",
  "source": "cowrie",
  "src_ip": "172.18.0.1",
  "tags": [
    "ssh",
    "command"
  ],
  "timestamp": "2026-05-12T18:27:14+00:00"
}
{
  "command": "ls",
  "details": {},
  "event_type": "ssh_command",
  "raw": "deception-cowrie  | 2026-05-13T02:27:16+0800 [HoneyPotSSHTransport,0,172.18.0.1] CMD: ls",
  "severity": "medium",
  "source": "cowrie",
  "src_ip": "172.18.0.1",
  "tags": [
    "ssh",
    "command"
  ],
  "timestamp": "2026-05-12T18:27:16+00:00"
}
{
  "command": "ls /home/admin",
  "details": {},
  "event_type": "ssh_command",
  "raw": "deception-cowrie  | 2026-05-13T02:27:30+0800 [HoneyPotSSHTransport,0,172.18.0.1] CMD: ls /home/admin",
  "severity": "medium",
  "source": "cowrie",
  "src_ip": "172.18.0.1",
  "tags": [
    "ssh",
    "command"
  ],
  "timestamp": "2026-05-12T18:27:30+00:00"
}
{
  "command": "ls /home/admin",
  "details": {},
  "event_type": "ssh_command",
  "raw": "deception-cowrie  | 2026-05-13T02:27:46+0800 [HoneyPotSSHTransport,0,172.18.0.1] CMD: ls /home/admin",
  "severity": "medium",
  "source": "cowrie",
  "src_ip": "172.18.0.1",
  "tags": [
    "ssh",
    "command"
  ],
  "timestamp": "2026-05-12T18:27:46+00:00"
}
{
  "command": "cat /home/admin/secrets.txt",
  "details": {},
  "event_type": "ssh_command",
  "raw": "deception-cowrie  | 2026-05-13T02:27:53+0800 [HoneyPotSSHTransport,0,172.18.0.1] CMD: cat /home/admin/secrets.txt",
  "severity": "medium",
  "source": "cowrie",
  "src_ip": "172.18.0.1",
  "tags": [
    "ssh",
    "command"
  ],
  "timestamp": "2026-05-12T18:27:53+00:00"
}
{
  "command": "exit",
  "details": {},
  "event_type": "ssh_command",
  "raw": "deception-cowrie  | 2026-05-13T02:27:57+0800 [HoneyPotSSHTransport,0,172.18.0.1] CMD: exit",
  "severity": "medium",
  "source": "cowrie",
  "src_ip": "172.18.0.1",
  "tags": [
    "ssh",
    "command"
  ],
  "timestamp": "2026-05-12T18:27:57+00:00"
}
lss@lss:/opt/deception-lab/parser $ ^C
lss@lss:/opt/deception-lab/parser $ grep '"honeycredential_used": true' /opt/deception-lab/data/events/events.jsonl | jq
{
  "details": {
    "honeycredential_id": "HC_BACKUP_001"
  },
  "event_type": "ssh_login_success",
  "honeycredential_used": true,
  "password_length": 11,
  "password_sha256": "a597108ff8384b3be204d08bce36fe3e86bffcc699fdac27604db48ae6f27f71",
  "raw": "deception-cowrie  | 2026-05-13T02:26:42+0800 [HoneyPotSSHTransport,0,172.18.0.1] login attempt [b'backup'/b'Backup2026!'] succeeded",
  "severity": "high",
  "source": "cowrie",
  "src_ip": "172.18.0.1",
  "tags": [
    "ssh",
    "login",
    "login-success",
    "honeycredential"
  ],
  "timestamp": "2026-05-12T18:26:42+00:00",
  "username": "backup"
}
{
  "details": {},
  "event_type": "web_login_attempt",
  "honeycredential_used": true,
  "password_length": 11,
  "password_sha256": "a597108ff8384b3be204d08bce36fe3e86bffcc699fdac27604db48ae6f27f71",
  "severity": "high",
  "source": "fake-web",
  "src_ip": "172.18.0.1",
  "tags": [
    "web",
    "login",
    "honeycredential"
  ],
  "timestamp": "2026-05-12T18:36:26.543366+00:00",
  "user_agent": "curl/8.14.1",
  "username": "backup"
}
{
  "details": {},
  "event_type": "web_login_attempt",
  "honeycredential_used": true,
  "password_length": 11,
  "password_sha256": "a597108ff8384b3be204d08bce36fe3e86bffcc699fdac27604db48ae6f27f71",
  "severity": "high",
  "source": "fake-web",
  "src_ip": "172.18.0.1",
  "tags": [
    "web",
    "login",
    "honeycredential"
  ],
  "timestamp": "2026-05-12T18:41:52.679019+00:00",
  "user_agent": "curl/8.14.1",
  "username": "backup"
}
```

### Step 8.13：查看 detections
- 查看全部 detection：
```
cat /opt/deception-lab/data/events/detections.jsonl | jq

# 執行結果：
lss@lss:/opt/deception-lab/parser $ cat /opt/deception-lab/data/events/detections.jsonl | jq
{
  "attack_mapping": {
    "tactic": "Collection",
    "technique": "Data from Local System",
    "technique_id": "T1005"
  },
  "command": null,
  "description": "User accessed or downloaded a honeyfile from the fake web panel.",
  "engage_mapping": {
    "activity": "Collect Adversary Behavior",
    "goal": "Understand"
  },
  "event_index": 12,
  "event_type": "web_honeyfile_access",
  "filename": "secrets.txt",
  "path": "/download/secrets.txt",
  "rule_id": "WEB_HONEYFILE_ACCESS",
  "rule_name": "Web honeyfile accessed",
  "severity": "high",
  "source": "fake-web",
  "src_ip": "192.168.1.1",
  "tags": [
    "download",
    "honeyfile",
    "web"
  ],
  "timestamp": "2026-05-11T21:57:12.514844+00:00",
  "username": null
}
{
  "attack_mapping": {
    "tactic": "Collection",
    "technique": "Data from Local System",
    "technique_id": "T1005"
  },
  "command": null,
  "description": "User accessed or downloaded a honeyfile from the fake web panel.",
  "engage_mapping": {
    "activity": "Collect Adversary Behavior",
    "goal": "Understand"
  },
  "event_index": 14,
  "event_type": "web_honeyfile_access",
  "filename": "vpn_users.csv",
  "path": "/download/vpn_users.csv",
  "rule_id": "WEB_HONEYFILE_ACCESS",
  "rule_name": "Web honeyfile accessed",
  "severity": "high",
  "source": "fake-web",
  "src_ip": "192.168.1.1",
  "tags": [
    "download",
    "honeyfile",
    "web"
  ],
  "timestamp": "2026-05-11T21:57:19.331247+00:00",
  "username": null
}
{
  "attack_mapping": {
    "tactic": "Credential Access",
    "technique": "Unsecured Credentials",
    "technique_id": "T1552"
  },
  "command": null,
  "description": "Attacker successfully logged into Cowrie using a known honeycredential.",
  "engage_mapping": {
    "activity": "Credential Collection",
    "goal": "Elicit"
  },
  "event_index": 20,
  "event_type": "ssh_login_success",
  "filename": null,
  "path": null,
  "rule_id": "SSH_HONEYCREDENTIAL_LOGIN",
  "rule_name": "SSH honeycredential login succeeded",
  "severity": "high",
  "source": "cowrie",
  "src_ip": "172.18.0.1",
  "tags": [
    "deception",
    "honeycredential",
    "login",
    "login-success",
    "ssh"
  ],
  "timestamp": "2026-05-12T18:26:42+00:00",
  "username": "backup"
}
{
  "attack_mapping": {
    "tactic": "Discovery",
    "technique": "System Information Discovery",
    "technique_id": "T1082"
  },
  "command": "whoami",
  "description": "Attacker executed common discovery or reconnaissance commands.",
  "engage_mapping": {
    "activity": "Collect Adversary Behavior",
    "goal": "Understand"
  },
  "event_index": 21,
  "event_type": "ssh_command",
  "filename": null,
  "path": null,
  "rule_id": "SSH_RECON_COMMAND",
  "rule_name": "SSH reconnaissance command",
  "severity": "medium",
  "source": "cowrie",
  "src_ip": "172.18.0.1",
  "tags": [
    "command",
    "discovery",
    "ssh"
  ],
  "timestamp": "2026-05-12T18:27:07+00:00",
  "username": null
}
{
  "attack_mapping": {
    "tactic": "Discovery",
    "technique": "System Information Discovery",
    "technique_id": "T1082"
  },
  "command": "pwd",
  "description": "Attacker executed common discovery or reconnaissance commands.",
  "engage_mapping": {
    "activity": "Collect Adversary Behavior",
    "goal": "Understand"
  },
  "event_index": 22,
  "event_type": "ssh_command",
  "filename": null,
  "path": null,
  "rule_id": "SSH_RECON_COMMAND",
  "rule_name": "SSH reconnaissance command",
  "severity": "medium",
  "source": "cowrie",
  "src_ip": "172.18.0.1",
  "tags": [
    "command",
    "discovery",
    "ssh"
  ],
  "timestamp": "2026-05-12T18:27:11+00:00",
  "username": null
}
{
  "attack_mapping": {
    "tactic": "Discovery",
    "technique": "System Information Discovery",
    "technique_id": "T1082"
  },
  "command": "ls",
  "description": "Attacker executed common discovery or reconnaissance commands.",
  "engage_mapping": {
    "activity": "Collect Adversary Behavior",
    "goal": "Understand"
  },
  "event_index": 24,
  "event_type": "ssh_command",
  "filename": null,
  "path": null,
  "rule_id": "SSH_RECON_COMMAND",
  "rule_name": "SSH reconnaissance command",
  "severity": "medium",
  "source": "cowrie",
  "src_ip": "172.18.0.1",
  "tags": [
    "command",
    "discovery",
    "ssh"
  ],
  "timestamp": "2026-05-12T18:27:16+00:00",
  "username": null
}
{
  "attack_mapping": {
    "tactic": "Discovery",
    "technique": "System Information Discovery",
    "technique_id": "T1082"
  },
  "command": "ls /home/admin",
  "description": "Attacker executed common discovery or reconnaissance commands.",
  "engage_mapping": {
    "activity": "Collect Adversary Behavior",
    "goal": "Understand"
  },
  "event_index": 25,
  "event_type": "ssh_command",
  "filename": null,
  "path": null,
  "rule_id": "SSH_RECON_COMMAND",
  "rule_name": "SSH reconnaissance command",
  "severity": "medium",
  "source": "cowrie",
  "src_ip": "172.18.0.1",
  "tags": [
    "command",
    "discovery",
    "ssh"
  ],
  "timestamp": "2026-05-12T18:27:30+00:00",
  "username": null
}
{
  "attack_mapping": {
    "tactic": "Discovery",
    "technique": "System Information Discovery",
    "technique_id": "T1082"
  },
  "command": "ls /home/admin",
  "description": "Attacker executed common discovery or reconnaissance commands.",
  "engage_mapping": {
    "activity": "Collect Adversary Behavior",
    "goal": "Understand"
  },
  "event_index": 26,
  "event_type": "ssh_command",
  "filename": null,
  "path": null,
  "rule_id": "SSH_RECON_COMMAND",
  "rule_name": "SSH reconnaissance command",
  "severity": "medium",
  "source": "cowrie",
  "src_ip": "172.18.0.1",
  "tags": [
    "command",
    "discovery",
    "ssh"
  ],
  "timestamp": "2026-05-12T18:27:46+00:00",
  "username": null
}
{
  "attack_mapping": {
    "tactic": "Collection",
    "technique": "Data from Local System",
    "technique_id": "T1005"
  },
  "command": "cat /home/admin/secrets.txt",
  "description": "Attacker attempted to access a known honeyfile inside Cowrie.",
  "engage_mapping": {
    "activity": "Reveal Adversary Intent",
    "goal": "Elicit"
  },
  "event_index": 27,
  "event_type": "ssh_command",
  "filename": null,
  "path": null,
  "rule_id": "SSH_HONEYFILE_ACCESS",
  "rule_name": "SSH honeyfile access attempt",
  "severity": "high",
  "source": "cowrie",
  "src_ip": "172.18.0.1",
  "tags": [
    "collection",
    "command",
    "honeyfile",
    "ssh"
  ],
  "timestamp": "2026-05-12T18:27:53+00:00",
  "username": null
}
{
  "attack_mapping": {
    "tactic": "Credential Access",
    "technique": "Unsecured Credentials",
    "technique_id": "T1552"
  },
  "command": null,
  "description": "User submitted a known honeycredential to the fake web login panel.",
  "engage_mapping": {
    "activity": "Credential Collection",
    "goal": "Elicit"
  },
  "event_index": 31,
  "event_type": "web_login_attempt",
  "filename": null,
  "path": null,
  "rule_id": "WEB_HONEYCREDENTIAL_USED",
  "rule_name": "Web honeycredential used",
  "severity": "high",
  "source": "fake-web",
  "src_ip": "172.18.0.1",
  "tags": [
    "honeycredential",
    "login",
    "web"
  ],
  "timestamp": "2026-05-12T18:36:26.543366+00:00",
  "username": "backup"
}
{
  "attack_mapping": {
    "tactic": "Credential Access",
    "technique": "Unsecured Credentials",
    "technique_id": "T1552"
  },
  "command": null,
  "description": "User submitted a known honeycredential to the fake web login panel.",
  "engage_mapping": {
    "activity": "Credential Collection",
    "goal": "Elicit"
  },
  "event_index": 33,
  "event_type": "web_login_attempt",
  "filename": null,
  "path": null,
  "rule_id": "WEB_HONEYCREDENTIAL_USED",
  "rule_name": "Web honeycredential used",
  "severity": "high",
  "source": "fake-web",
  "src_ip": "172.18.0.1",
  "tags": [
    "honeycredential",
    "login",
    "web"
  ],
  "timestamp": "2026-05-12T18:41:52.679019+00:00",
  "username": "backup"
}
```
- 如果輸出太多，可以看最後 20 筆：
```
tail -n 20 /opt/deception-lab/data/events/detections.jsonl | jq

# 執行結果：
lss@lss:/opt/deception-lab/parser $ tail -n 20 /opt/deception-lab/data/events/detections.jsonl | jq
{
  "attack_mapping": {
    "tactic": "Collection",
    "technique": "Data from Local System",
    "technique_id": "T1005"
  },
  "command": null,
  "description": "User accessed or downloaded a honeyfile from the fake web panel.",
  "engage_mapping": {
    "activity": "Collect Adversary Behavior",
    "goal": "Understand"
  },
  "event_index": 12,
  "event_type": "web_honeyfile_access",
  "filename": "secrets.txt",
  "path": "/download/secrets.txt",
  "rule_id": "WEB_HONEYFILE_ACCESS",
  "rule_name": "Web honeyfile accessed",
  "severity": "high",
  "source": "fake-web",
  "src_ip": "192.168.1.1",
  "tags": [
    "download",
    "honeyfile",
    "web"
  ],
  "timestamp": "2026-05-11T21:57:12.514844+00:00",
  "username": null
}
{
  "attack_mapping": {
    "tactic": "Collection",
    "technique": "Data from Local System",
    "technique_id": "T1005"
  },
  "command": null,
  "description": "User accessed or downloaded a honeyfile from the fake web panel.",
  "engage_mapping": {
    "activity": "Collect Adversary Behavior",
    "goal": "Understand"
  },
  "event_index": 14,
  "event_type": "web_honeyfile_access",
  "filename": "vpn_users.csv",
  "path": "/download/vpn_users.csv",
  "rule_id": "WEB_HONEYFILE_ACCESS",
  "rule_name": "Web honeyfile accessed",
  "severity": "high",
  "source": "fake-web",
  "src_ip": "192.168.1.1",
  "tags": [
    "download",
    "honeyfile",
    "web"
  ],
  "timestamp": "2026-05-11T21:57:19.331247+00:00",
  "username": null
}
{
  "attack_mapping": {
    "tactic": "Credential Access",
    "technique": "Unsecured Credentials",
    "technique_id": "T1552"
  },
  "command": null,
  "description": "Attacker successfully logged into Cowrie using a known honeycredential.",
  "engage_mapping": {
    "activity": "Credential Collection",
    "goal": "Elicit"
  },
  "event_index": 20,
  "event_type": "ssh_login_success",
  "filename": null,
  "path": null,
  "rule_id": "SSH_HONEYCREDENTIAL_LOGIN",
  "rule_name": "SSH honeycredential login succeeded",
  "severity": "high",
  "source": "cowrie",
  "src_ip": "172.18.0.1",
  "tags": [
    "deception",
    "honeycredential",
    "login",
    "login-success",
    "ssh"
  ],
  "timestamp": "2026-05-12T18:26:42+00:00",
  "username": "backup"
}
{
  "attack_mapping": {
    "tactic": "Discovery",
    "technique": "System Information Discovery",
    "technique_id": "T1082"
  },
  "command": "whoami",
  "description": "Attacker executed common discovery or reconnaissance commands.",
  "engage_mapping": {
    "activity": "Collect Adversary Behavior",
    "goal": "Understand"
  },
  "event_index": 21,
  "event_type": "ssh_command",
  "filename": null,
  "path": null,
  "rule_id": "SSH_RECON_COMMAND",
  "rule_name": "SSH reconnaissance command",
  "severity": "medium",
  "source": "cowrie",
  "src_ip": "172.18.0.1",
  "tags": [
    "command",
    "discovery",
    "ssh"
  ],
  "timestamp": "2026-05-12T18:27:07+00:00",
  "username": null
}
{
  "attack_mapping": {
    "tactic": "Discovery",
    "technique": "System Information Discovery",
    "technique_id": "T1082"
  },
  "command": "pwd",
  "description": "Attacker executed common discovery or reconnaissance commands.",
  "engage_mapping": {
    "activity": "Collect Adversary Behavior",
    "goal": "Understand"
  },
  "event_index": 22,
  "event_type": "ssh_command",
  "filename": null,
  "path": null,
  "rule_id": "SSH_RECON_COMMAND",
  "rule_name": "SSH reconnaissance command",
  "severity": "medium",
  "source": "cowrie",
  "src_ip": "172.18.0.1",
  "tags": [
    "command",
    "discovery",
    "ssh"
  ],
  "timestamp": "2026-05-12T18:27:11+00:00",
  "username": null
}
{
  "attack_mapping": {
    "tactic": "Discovery",
    "technique": "System Information Discovery",
    "technique_id": "T1082"
  },
  "command": "ls",
  "description": "Attacker executed common discovery or reconnaissance commands.",
  "engage_mapping": {
    "activity": "Collect Adversary Behavior",
    "goal": "Understand"
  },
  "event_index": 24,
  "event_type": "ssh_command",
  "filename": null,
  "path": null,
  "rule_id": "SSH_RECON_COMMAND",
  "rule_name": "SSH reconnaissance command",
  "severity": "medium",
  "source": "cowrie",
  "src_ip": "172.18.0.1",
  "tags": [
    "command",
    "discovery",
    "ssh"
  ],
  "timestamp": "2026-05-12T18:27:16+00:00",
  "username": null
}
{
  "attack_mapping": {
    "tactic": "Discovery",
    "technique": "System Information Discovery",
    "technique_id": "T1082"
  },
  "command": "ls /home/admin",
  "description": "Attacker executed common discovery or reconnaissance commands.",
  "engage_mapping": {
    "activity": "Collect Adversary Behavior",
    "goal": "Understand"
  },
  "event_index": 25,
  "event_type": "ssh_command",
  "filename": null,
  "path": null,
  "rule_id": "SSH_RECON_COMMAND",
  "rule_name": "SSH reconnaissance command",
  "severity": "medium",
  "source": "cowrie",
  "src_ip": "172.18.0.1",
  "tags": [
    "command",
    "discovery",
    "ssh"
  ],
  "timestamp": "2026-05-12T18:27:30+00:00",
  "username": null
}
{
  "attack_mapping": {
    "tactic": "Discovery",
    "technique": "System Information Discovery",
    "technique_id": "T1082"
  },
  "command": "ls /home/admin",
  "description": "Attacker executed common discovery or reconnaissance commands.",
  "engage_mapping": {
    "activity": "Collect Adversary Behavior",
    "goal": "Understand"
  },
  "event_index": 26,
  "event_type": "ssh_command",
  "filename": null,
  "path": null,
  "rule_id": "SSH_RECON_COMMAND",
  "rule_name": "SSH reconnaissance command",
  "severity": "medium",
  "source": "cowrie",
  "src_ip": "172.18.0.1",
  "tags": [
    "command",
    "discovery",
    "ssh"
  ],
  "timestamp": "2026-05-12T18:27:46+00:00",
  "username": null
}
{
  "attack_mapping": {
    "tactic": "Collection",
    "technique": "Data from Local System",
    "technique_id": "T1005"
  },
  "command": "cat /home/admin/secrets.txt",
  "description": "Attacker attempted to access a known honeyfile inside Cowrie.",
  "engage_mapping": {
    "activity": "Reveal Adversary Intent",
    "goal": "Elicit"
  },
  "event_index": 27,
  "event_type": "ssh_command",
  "filename": null,
  "path": null,
  "rule_id": "SSH_HONEYFILE_ACCESS",
  "rule_name": "SSH honeyfile access attempt",
  "severity": "high",
  "source": "cowrie",
  "src_ip": "172.18.0.1",
  "tags": [
    "collection",
    "command",
    "honeyfile",
    "ssh"
  ],
  "timestamp": "2026-05-12T18:27:53+00:00",
  "username": null
}
{
  "attack_mapping": {
    "tactic": "Credential Access",
    "technique": "Unsecured Credentials",
    "technique_id": "T1552"
  },
  "command": null,
  "description": "User submitted a known honeycredential to the fake web login panel.",
  "engage_mapping": {
    "activity": "Credential Collection",
    "goal": "Elicit"
  },
  "event_index": 31,
  "event_type": "web_login_attempt",
  "filename": null,
  "path": null,
  "rule_id": "WEB_HONEYCREDENTIAL_USED",
  "rule_name": "Web honeycredential used",
  "severity": "high",
  "source": "fake-web",
  "src_ip": "172.18.0.1",
  "tags": [
    "honeycredential",
    "login",
    "web"
  ],
  "timestamp": "2026-05-12T18:36:26.543366+00:00",
  "username": "backup"
}
{
  "attack_mapping": {
    "tactic": "Credential Access",
    "technique": "Unsecured Credentials",
    "technique_id": "T1552"
  },
  "command": null,
  "description": "User submitted a known honeycredential to the fake web login panel.",
  "engage_mapping": {
    "activity": "Credential Collection",
    "goal": "Elicit"
  },
  "event_index": 33,
  "event_type": "web_login_attempt",
  "filename": null,
  "path": null,
  "rule_id": "WEB_HONEYCREDENTIAL_USED",
  "rule_name": "Web honeycredential used",
  "severity": "high",
  "source": "fake-web",
  "src_ip": "172.18.0.1",
  "tags": [
    "honeycredential",
    "login",
    "web"
  ],
  "timestamp": "2026-05-12T18:41:52.679019+00:00",
  "username": "backup"
}
```
- 你應該看到 detection，例如：
```
SSH_HONEYCREDENTIAL_LOGIN
SSH_RECON_COMMAND
SSH_HONEYFILE_ACCESS
WEB_HONEYCREDENTIAL_USED
WEB_HONEYFILE_ACCESS
```

### Step 8.14：用 grep 快速看偵測種類
- 執行：
```
grep -o '"rule_id": "[^"]*"' /opt/deception-lab/data/events/detections.jsonl | sort | uniq -c

# 執行結果：你可能會看到類似：
lss@lss:/opt/deception-lab/parser $ grep -o '"rule_id": "[^"]*"' /opt/deception-lab/data/events/detections.jsonl | sort | uniq -c
      1 "rule_id": "SSH_HONEYCREDENTIAL_LOGIN"
      1 "rule_id": "SSH_HONEYFILE_ACCESS"
      5 "rule_id": "SSH_RECON_COMMAND"
      2 "rule_id": "WEB_HONEYCREDENTIAL_USED"
      2 "rule_id": "WEB_HONEYFILE_ACCESS"
```

### Step 8.15：建立事件查看腳本
建立一個快速查看 parser 結果的腳本。
- 執行：
```
cat > /opt/deception-lab/scripts/show_events.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

EVENT_DIR="/opt/deception-lab/data/events"

echo "=== Event Files ==="
ls -lah "$EVENT_DIR"

echo
echo "=== Summary ==="
if [ -f "$EVENT_DIR/events_summary.json" ]; then
  cat "$EVENT_DIR/events_summary.json" | jq
else
  echo "No events_summary.json found."
fi

echo
echo "=== Detection Rule Counts ==="
if [ -f "$EVENT_DIR/detections.jsonl" ]; then
  grep -o '"rule_id": "[^"]*"' "$EVENT_DIR/detections.jsonl" | sort | uniq -c || true
else
  echo "No detections.jsonl found."
fi

echo
echo "=== Last 10 Events ==="
if [ -f "$EVENT_DIR/events.jsonl" ]; then
  tail -n 10 "$EVENT_DIR/events.jsonl" | jq
else
  echo "No events.jsonl found."
fi

echo
echo "=== Last 10 Detections ==="
if [ -f "$EVENT_DIR/detections.jsonl" ]; then
  tail -n 10 "$EVENT_DIR/detections.jsonl" | jq
else
  echo "No detections.jsonl found."
fi
EOF
```
- 設定可執行：
```
chmod +x /opt/deception-lab/scripts/show_events.sh
```
- 執行：
```
/opt/deception-lab/scripts/show_events.sh

# 執行結果：
lss@lss:/opt/deception-lab/parser $ /opt/deception-lab/scripts/show_events.sh
=== Event Files ===
total 36K
drwxrwxr-x 2 lss lss 4.0K May 20 03:28 .
drwxrwxr-x 8 lss lss 4.0K May 13 03:01 ..
-rw-rw-r-- 1 lss lss 6.9K May 20 03:28 detections.jsonl
-rw-rw-r-- 1 lss lss  13K May 20 03:28 events.jsonl
-rw-rw-r-- 1 lss lss  631 May 20 03:28 events_summary.json

=== Summary ===
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
}

=== Detection Rule Counts ===
      1 "rule_id": "SSH_HONEYCREDENTIAL_LOGIN"
      1 "rule_id": "SSH_HONEYFILE_ACCESS"
      5 "rule_id": "SSH_RECON_COMMAND"
      2 "rule_id": "WEB_HONEYCREDENTIAL_USED"
      2 "rule_id": "WEB_HONEYFILE_ACCESS"

=== Last 10 Events ===
{
  "command": "ls",
  "details": {},
  "event_type": "ssh_command",
  "raw": "deception-cowrie  | 2026-05-13T02:27:16+0800 [HoneyPotSSHTransport,0,172.18.0.1] CMD: ls",
  "severity": "medium",
  "source": "cowrie",
  "src_ip": "172.18.0.1",
  "tags": [
    "ssh",
    "command"
  ],
  "timestamp": "2026-05-12T18:27:16+00:00"
}
{
  "command": "ls /home/admin",
  "details": {},
  "event_type": "ssh_command",
  "raw": "deception-cowrie  | 2026-05-13T02:27:30+0800 [HoneyPotSSHTransport,0,172.18.0.1] CMD: ls /home/admin",
  "severity": "medium",
  "source": "cowrie",
  "src_ip": "172.18.0.1",
  "tags": [
    "ssh",
    "command"
  ],
  "timestamp": "2026-05-12T18:27:30+00:00"
}
{
  "command": "ls /home/admin",
  "details": {},
  "event_type": "ssh_command",
  "raw": "deception-cowrie  | 2026-05-13T02:27:46+0800 [HoneyPotSSHTransport,0,172.18.0.1] CMD: ls /home/admin",
  "severity": "medium",
  "source": "cowrie",
  "src_ip": "172.18.0.1",
  "tags": [
    "ssh",
    "command"
  ],
  "timestamp": "2026-05-12T18:27:46+00:00"
}
{
  "command": "cat /home/admin/secrets.txt",
  "details": {},
  "event_type": "ssh_command",
  "raw": "deception-cowrie  | 2026-05-13T02:27:53+0800 [HoneyPotSSHTransport,0,172.18.0.1] CMD: cat /home/admin/secrets.txt",
  "severity": "medium",
  "source": "cowrie",
  "src_ip": "172.18.0.1",
  "tags": [
    "ssh",
    "command"
  ],
  "timestamp": "2026-05-12T18:27:53+00:00"
}
{
  "command": "exit",
  "details": {},
  "event_type": "ssh_command",
  "raw": "deception-cowrie  | 2026-05-13T02:27:57+0800 [HoneyPotSSHTransport,0,172.18.0.1] CMD: exit",
  "severity": "medium",
  "source": "cowrie",
  "src_ip": "172.18.0.1",
  "tags": [
    "ssh",
    "command"
  ],
  "timestamp": "2026-05-12T18:27:57+00:00"
}
{
  "details": {},
  "event_type": "ssh_logout",
  "raw": "deception-cowrie  | 2026-05-13T02:27:57+0800 [HoneyPotSSHTransport,0,172.18.0.1] avatar backup logging out",
  "severity": "info",
  "source": "cowrie",
  "src_ip": "172.18.0.1",
  "tags": [
    "ssh",
    "logout"
  ],
  "timestamp": "2026-05-12T18:27:57+00:00",
  "username": "backup"
}
{
  "details": {},
  "event_type": "web_request",
  "is_scanner_probe": false,
  "method": "POST",
  "path": "/login",
  "query_string": "",
  "severity": "info",
  "source": "fake-web",
  "src_ip": "172.18.0.1",
  "tags": [
    "web",
    "request"
  ],
  "timestamp": "2026-05-12T18:36:26.542718+00:00",
  "user_agent": "curl/8.14.1"
}
{
  "details": {},
  "event_type": "web_login_attempt",
  "honeycredential_used": true,
  "password_length": 11,
  "password_sha256": "a597108ff8384b3be204d08bce36fe3e86bffcc699fdac27604db48ae6f27f71",
  "severity": "high",
  "source": "fake-web",
  "src_ip": "172.18.0.1",
  "tags": [
    "web",
    "login",
    "honeycredential"
  ],
  "timestamp": "2026-05-12T18:36:26.543366+00:00",
  "user_agent": "curl/8.14.1",
  "username": "backup"
}
{
  "details": {},
  "event_type": "web_request",
  "is_scanner_probe": false,
  "method": "POST",
  "path": "/login",
  "query_string": "",
  "severity": "info",
  "source": "fake-web",
  "src_ip": "172.18.0.1",
  "tags": [
    "web",
    "request"
  ],
  "timestamp": "2026-05-12T18:41:52.678577+00:00",
  "user_agent": "curl/8.14.1"
}
{
  "details": {},
  "event_type": "web_login_attempt",
  "honeycredential_used": true,
  "password_length": 11,
  "password_sha256": "a597108ff8384b3be204d08bce36fe3e86bffcc699fdac27604db48ae6f27f71",
  "severity": "high",
  "source": "fake-web",
  "src_ip": "172.18.0.1",
  "tags": [
    "web",
    "login",
    "honeycredential"
  ],
  "timestamp": "2026-05-12T18:41:52.679019+00:00",
  "user_agent": "curl/8.14.1",
  "username": "backup"
}

=== Last 10 Detections ===
{
  "attack_mapping": {
    "tactic": "Collection",
    "technique": "Data from Local System",
    "technique_id": "T1005"
  },
  "command": null,
  "description": "User accessed or downloaded a honeyfile from the fake web panel.",
  "engage_mapping": {
    "activity": "Collect Adversary Behavior",
    "goal": "Understand"
  },
  "event_index": 14,
  "event_type": "web_honeyfile_access",
  "filename": "vpn_users.csv",
  "path": "/download/vpn_users.csv",
  "rule_id": "WEB_HONEYFILE_ACCESS",
  "rule_name": "Web honeyfile accessed",
  "severity": "high",
  "source": "fake-web",
  "src_ip": "192.168.1.1",
  "tags": [
    "download",
    "honeyfile",
    "web"
  ],
  "timestamp": "2026-05-11T21:57:19.331247+00:00",
  "username": null
}
{
  "attack_mapping": {
    "tactic": "Credential Access",
    "technique": "Unsecured Credentials",
    "technique_id": "T1552"
  },
  "command": null,
  "description": "Attacker successfully logged into Cowrie using a known honeycredential.",
  "engage_mapping": {
    "activity": "Credential Collection",
    "goal": "Elicit"
  },
  "event_index": 20,
  "event_type": "ssh_login_success",
  "filename": null,
  "path": null,
  "rule_id": "SSH_HONEYCREDENTIAL_LOGIN",
  "rule_name": "SSH honeycredential login succeeded",
  "severity": "high",
  "source": "cowrie",
  "src_ip": "172.18.0.1",
  "tags": [
    "deception",
    "honeycredential",
    "login",
    "login-success",
    "ssh"
  ],
  "timestamp": "2026-05-12T18:26:42+00:00",
  "username": "backup"
}
{
  "attack_mapping": {
    "tactic": "Discovery",
    "technique": "System Information Discovery",
    "technique_id": "T1082"
  },
  "command": "whoami",
  "description": "Attacker executed common discovery or reconnaissance commands.",
  "engage_mapping": {
    "activity": "Collect Adversary Behavior",
    "goal": "Understand"
  },
  "event_index": 21,
  "event_type": "ssh_command",
  "filename": null,
  "path": null,
  "rule_id": "SSH_RECON_COMMAND",
  "rule_name": "SSH reconnaissance command",
  "severity": "medium",
  "source": "cowrie",
  "src_ip": "172.18.0.1",
  "tags": [
    "command",
    "discovery",
    "ssh"
  ],
  "timestamp": "2026-05-12T18:27:07+00:00",
  "username": null
}
{
  "attack_mapping": {
    "tactic": "Discovery",
    "technique": "System Information Discovery",
    "technique_id": "T1082"
  },
  "command": "pwd",
  "description": "Attacker executed common discovery or reconnaissance commands.",
  "engage_mapping": {
    "activity": "Collect Adversary Behavior",
    "goal": "Understand"
  },
  "event_index": 22,
  "event_type": "ssh_command",
  "filename": null,
  "path": null,
  "rule_id": "SSH_RECON_COMMAND",
  "rule_name": "SSH reconnaissance command",
  "severity": "medium",
  "source": "cowrie",
  "src_ip": "172.18.0.1",
  "tags": [
    "command",
    "discovery",
    "ssh"
  ],
  "timestamp": "2026-05-12T18:27:11+00:00",
  "username": null
}
{
  "attack_mapping": {
    "tactic": "Discovery",
    "technique": "System Information Discovery",
    "technique_id": "T1082"
  },
  "command": "ls",
  "description": "Attacker executed common discovery or reconnaissance commands.",
  "engage_mapping": {
    "activity": "Collect Adversary Behavior",
    "goal": "Understand"
  },
  "event_index": 24,
  "event_type": "ssh_command",
  "filename": null,
  "path": null,
  "rule_id": "SSH_RECON_COMMAND",
  "rule_name": "SSH reconnaissance command",
  "severity": "medium",
  "source": "cowrie",
  "src_ip": "172.18.0.1",
  "tags": [
    "command",
    "discovery",
    "ssh"
  ],
  "timestamp": "2026-05-12T18:27:16+00:00",
  "username": null
}
{
  "attack_mapping": {
    "tactic": "Discovery",
    "technique": "System Information Discovery",
    "technique_id": "T1082"
  },
  "command": "ls /home/admin",
  "description": "Attacker executed common discovery or reconnaissance commands.",
  "engage_mapping": {
    "activity": "Collect Adversary Behavior",
    "goal": "Understand"
  },
  "event_index": 25,
  "event_type": "ssh_command",
  "filename": null,
  "path": null,
  "rule_id": "SSH_RECON_COMMAND",
  "rule_name": "SSH reconnaissance command",
  "severity": "medium",
  "source": "cowrie",
  "src_ip": "172.18.0.1",
  "tags": [
    "command",
    "discovery",
    "ssh"
  ],
  "timestamp": "2026-05-12T18:27:30+00:00",
  "username": null
}
{
  "attack_mapping": {
    "tactic": "Discovery",
    "technique": "System Information Discovery",
    "technique_id": "T1082"
  },
  "command": "ls /home/admin",
  "description": "Attacker executed common discovery or reconnaissance commands.",
  "engage_mapping": {
    "activity": "Collect Adversary Behavior",
    "goal": "Understand"
  },
  "event_index": 26,
  "event_type": "ssh_command",
  "filename": null,
  "path": null,
  "rule_id": "SSH_RECON_COMMAND",
  "rule_name": "SSH reconnaissance command",
  "severity": "medium",
  "source": "cowrie",
  "src_ip": "172.18.0.1",
  "tags": [
    "command",
    "discovery",
    "ssh"
  ],
  "timestamp": "2026-05-12T18:27:46+00:00",
  "username": null
}
{
  "attack_mapping": {
    "tactic": "Collection",
    "technique": "Data from Local System",
    "technique_id": "T1005"
  },
  "command": "cat /home/admin/secrets.txt",
  "description": "Attacker attempted to access a known honeyfile inside Cowrie.",
  "engage_mapping": {
    "activity": "Reveal Adversary Intent",
    "goal": "Elicit"
  },
  "event_index": 27,
  "event_type": "ssh_command",
  "filename": null,
  "path": null,
  "rule_id": "SSH_HONEYFILE_ACCESS",
  "rule_name": "SSH honeyfile access attempt",
  "severity": "high",
  "source": "cowrie",
  "src_ip": "172.18.0.1",
  "tags": [
    "collection",
    "command",
    "honeyfile",
    "ssh"
  ],
  "timestamp": "2026-05-12T18:27:53+00:00",
  "username": null
}
{
  "attack_mapping": {
    "tactic": "Credential Access",
    "technique": "Unsecured Credentials",
    "technique_id": "T1552"
  },
  "command": null,
  "description": "User submitted a known honeycredential to the fake web login panel.",
  "engage_mapping": {
    "activity": "Credential Collection",
    "goal": "Elicit"
  },
  "event_index": 31,
  "event_type": "web_login_attempt",
  "filename": null,
  "path": null,
  "rule_id": "WEB_HONEYCREDENTIAL_USED",
  "rule_name": "Web honeycredential used",
  "severity": "high",
  "source": "fake-web",
  "src_ip": "172.18.0.1",
  "tags": [
    "honeycredential",
    "login",
    "web"
  ],
  "timestamp": "2026-05-12T18:36:26.543366+00:00",
  "username": "backup"
}
{
  "attack_mapping": {
    "tactic": "Credential Access",
    "technique": "Unsecured Credentials",
    "technique_id": "T1552"
  },
  "command": null,
  "description": "User submitted a known honeycredential to the fake web login panel.",
  "engage_mapping": {
    "activity": "Credential Collection",
    "goal": "Elicit"
  },
  "event_index": 33,
  "event_type": "web_login_attempt",
  "filename": null,
  "path": null,
  "rule_id": "WEB_HONEYCREDENTIAL_USED",
  "rule_name": "Web honeycredential used",
  "severity": "high",
  "source": "fake-web",
  "src_ip": "172.18.0.1",
  "tags": [
    "honeycredential",
    "login",
    "web"
  ],
  "timestamp": "2026-05-12T18:41:52.679019+00:00",
  "username": "backup"
}
```

### Step 8.16：建立 Parser README
- 執行：
```
cat > /opt/deception-lab/parser/README.md <<'EOF'
# Phase 8 Parser

This parser reads centralized logs from:

- /opt/deception-lab/data/collected/cowrie-docker.log
- /opt/deception-lab/data/collected/web_access.jsonl
- /opt/deception-lab/data/collected/web_auth.jsonl

It outputs normalized events and detections to:

- /opt/deception-lab/data/events/events.jsonl
- /opt/deception-lab/data/events/detections.jsonl
- /opt/deception-lab/data/events/events_summary.json

Main scripts:

- /opt/deception-lab/scripts/run_parser.sh
- /opt/deception-lab/scripts/show_events.sh

Detection rules:

- /opt/deception-lab/parser/rules/detection_rules.yml
EOF
```

### Step 8.17：建立第八階段完成紀錄
如果 parser 成功執行，建立：
```
cat > /opt/deception-lab/PHASE8_READY.md <<'EOF'
# Phase 8 Ready

Python event parser and detection rules have been implemented.

Completed items:

- parser Python virtual environment created
- PyYAML installed
- detection_rules.yml created
- parse_events.py created
- run_parser.sh created
- show_events.sh created
- Cowrie Docker logs parsed
- Fake Web access logs parsed
- Fake Web auth logs parsed
- events.jsonl generated
- detections.jsonl generated
- events_summary.json generated

Main input directory:

/opt/deception-lab/data/collected

Main output directory:

/opt/deception-lab/data/events

Output files:

- events.jsonl
- detections.jsonl
- events_summary.json

Next phase:

Phase 9 - MITRE ATT&CK / MITRE Engage mapping refinement.
EOF
```
- 確認：
```
cat /opt/deception-lab/PHASE8_READY.md
```

## 第八階段完成檢查
最後請執行以下指令，把結果貼給我確認：
```
/opt/deception-lab/scripts/run_parser.sh
# 你現在的 parser 已經可以正常完成以下流程：
collect_logs.sh
    ↓
讀取集中式 log
    ↓
parse_events.py
    ↓
產生標準化事件 events.jsonl
    ↓
產生偵測結果 detections.jsonl
    ↓
產生摘要 events_summary.json

ls -lah /opt/deception-lab/data/events
# 你輸出的重點如下：
Total events: 34
Total detections: 11

cat /opt/deception-lab/data/events/events_summary.json | jq
# 而且偵測規則也成功命中：
1  SSH_HONEYCREDENTIAL_LOGIN
1  SSH_HONEYFILE_ACCESS
5  SSH_RECON_COMMAND
2  WEB_HONEYCREDENTIAL_USED
2  WEB_HONEYFILE_ACCESS

grep -o '"rule_id": "[^"]*"' /opt/deception-lab/data/events/detections.jsonl | sort | uniq -c
# 這代表你的第八階段 parser 已經能偵測：
[成功] Cowrie honeycredential 登入
[成功] Cowrie 偵察指令
[成功] Cowrie honeyfile 存取嘗試
[成功] Fake Web honeycredential 使用
[成功] Fake Web honeyfile 存取

/opt/deception-lab/scripts/show_events.sh
# 你也已經成功產生三個核心輸出檔案：
/opt/deception-lab/data/events/events.jsonl
/opt/deception-lab/data/events/detections.jsonl
/opt/deception-lab/data/events/events_summary.json
```

## 第八階段完成後的狀態
完成後，你的平台資料流會變成：
你的 show_events.sh 輸出也清楚顯示了 backup / Backup2026! 成功登入 Cowrie、cat /home/admin/secrets.txt 被判定為 SSH honeyfile access，以及 Fake Web 的 WEB_HONEYCREDENTIAL_USED 偵測結果。
```
# 目前你的平台已經具備：
[完成] SSH honeypot
[完成] Fake Web Admin
[完成] honeycredential
[完成] honeyfile
[完成] 集中式 log 收集
[完成] Python event parser
[完成] detection rules
[完成] events.jsonl
[完成] detections.jsonl
[完成] events_summary.json

# 目前資料流如下：
Cowrie Docker logs
Fake Web access log
Fake Web auth log
        ↓
/opt/deception-lab/scripts/collect_logs.sh
        ↓
/opt/deception-lab/data/collected/
        ↓
/opt/deception-lab/scripts/run_parser.sh
        ↓
/opt/deception-lab/data/events/
        ├── events.jsonl
        ├── detections.jsonl
        └── events_summary.json
```
