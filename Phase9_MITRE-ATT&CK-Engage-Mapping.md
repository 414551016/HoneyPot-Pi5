# 第九階段：MITRE ATT&CK / MITRE Engage mapping
```
第九階段要做的是把 detection 結果中的 mapping 正式整理出來，產生：
/opt/deception-lab/parser/rules/attack_mapping.yml
/opt/deception-lab/parser/rules/engage_mapping.yml
/opt/deception-lab/data/events/mapping_summary.json
/opt/deception-lab/data/events/attack_coverage.json
/opt/deception-lab/data/events/engage_coverage.json

# 完成後，你會有：
[完成] MITRE ATT&CK mapping 獨立設定檔
[完成] MITRE Engage mapping 獨立設定檔
[完成] mapping analyzer
[完成] mapping summary
[完成] ATT&CK coverage summary
[完成] Engage coverage summary
[完成] mapping 檢查腳本
[完成] PHASE9_READY.md
```

### Step 9.1：進入專案目錄
```
cd /opt/deception-lab
```

### Step 9.2：建立 ATT&CK mapping 設定檔
- 執行
```
cat > /opt/deception-lab/parser/rules/attack_mapping.yml <<'EOF'
framework:
  name: MITRE ATT&CK
  matrix: Enterprise
  version_note: MVP mapping for Raspberry Pi deception lab

mappings:
  SSH_HONEYCREDENTIAL_LOGIN:
    tactic: Credential Access
    technique_id: T1552
    technique: Unsecured Credentials
    rationale: >
      The attacker used a credential intentionally planted in deception assets.
      This indicates exposure to unsecured credential material.

  SSH_LOGIN_FAILED:
    tactic: Credential Access
    technique_id: T1110
    technique: Brute Force
    rationale: >
      Failed SSH authentication attempts against the honeypot indicate password
      guessing or brute-force behavior.

  SSH_RECON_COMMAND:
    tactic: Discovery
    technique_id: T1082
    technique: System Information Discovery
    related_techniques:
      - T1033 System Owner/User Discovery
      - T1016 System Network Configuration Discovery
      - T1057 Process Discovery
    rationale: >
      Commands such as whoami, pwd, ls, uname, ip addr, ifconfig, ps, and netstat
      indicate post-login discovery activity.

  SSH_TOOL_TRANSFER_COMMAND:
    tactic: Command and Control
    technique_id: T1105
    technique: Ingress Tool Transfer
    rationale: >
      Commands such as wget, curl, ftp, scp, chmod, nc, or bash reverse shell
      patterns may indicate tool staging or payload transfer.

  SSH_HONEYFILE_ACCESS:
    tactic: Collection
    technique_id: T1005
    technique: Data from Local System
    rationale: >
      Attempting to read honeyfiles such as secrets.txt or backup_config.ini
      indicates interest in local sensitive data.

  WEB_HONEYCREDENTIAL_USED:
    tactic: Credential Access
    technique_id: T1552
    technique: Unsecured Credentials
    rationale: >
      Submitting a planted credential to the fake web login panel indicates the
      adversary discovered and attempted to use unsecured credentials.

  WEB_HONEYFILE_ACCESS:
    tactic: Collection
    technique_id: T1005
    technique: Data from Local System
    rationale: >
      Downloading honeyfiles from the fake web panel indicates attempted
      collection of local or exposed files.

  WEB_SCANNER_PROBE:
    tactic: Reconnaissance
    technique_id: T1595
    technique: Active Scanning
    rationale: >
      Requests to paths such as /.env, /wp-admin, /phpmyadmin, or /admin indicate
      web probing or scanning behavior.
EOF
```
- 確認：
```
cat /opt/deception-lab/parser/rules/attack_mapping.yml
```

### Step 9.3：建立 MITRE Engage mapping 設定檔
```
這個檔案會把 detection rule 對應到 Engage 的五個戰略目標：
Prepare
Expose
Affect
Elicit
Understand
```
- 執行：
```
cat > /opt/deception-lab/parser/rules/engage_mapping.yml <<'EOF'
framework:
  name: MITRE Engage
  version_note: MVP mapping for Raspberry Pi deception lab

goals:
  Prepare:
    description: Prepare the deception environment, assets, logging, and safety boundaries.

  Expose:
    description: Expose decoy services and deception artifacts to adversary interaction.

  Affect:
    description: Influence adversary perception, decision-making, and behavior.

  Elicit:
    description: Elicit observable adversary behavior by placing credentials, files, and services.

  Understand:
    description: Collect and analyze adversary behavior to produce defensive knowledge.

mappings:
  SSH_HONEYCREDENTIAL_LOGIN:
    goal: Elicit
    activity: Credential Collection
    rationale: >
      A planted SSH credential caused the adversary to reveal credential-use behavior.

  SSH_LOGIN_FAILED:
    goal: Expose
    activity: Expose Decoy Service
    rationale: >
      The SSH honeypot exposes a fake login surface for observing authentication attempts.

  SSH_RECON_COMMAND:
    goal: Understand
    activity: Collect Adversary Behavior
    rationale: >
      Commands entered after login reveal adversary discovery interests and operating style.

  SSH_TOOL_TRANSFER_COMMAND:
    goal: Affect
    activity: Adversary Direction
    rationale: >
      A decoy shell can influence attacker behavior and redirect tool staging activity
      into a controlled observation environment.

  SSH_HONEYFILE_ACCESS:
    goal: Elicit
    activity: Reveal Adversary Intent
    rationale: >
      Honeyfile access attempts reveal interest in sensitive files, backup data,
      credentials, and local configuration.

  WEB_HONEYCREDENTIAL_USED:
    goal: Elicit
    activity: Credential Collection
    rationale: >
      Planted web credentials elicit credential reuse behavior against the fake admin panel.

  WEB_HONEYFILE_ACCESS:
    goal: Understand
    activity: Collect Adversary Behavior
    rationale: >
      Honeyfile download behavior helps identify the attacker's collection interests.

  WEB_SCANNER_PROBE:
    goal: Expose
    activity: Expose Decoy Service
    rationale: >
      The fake web interface exposes probeable paths to observe scanner behavior.
EOF
```
- 確認：
```
cat /opt/deception-lab/parser/rules/engage_mapping.yml
```

### Step 9.4：建立 mapping analyzer
```
# 這個程式會讀取：
/opt/deception-lab/data/events/detections.jsonl
/opt/deception-lab/parser/rules/attack_mapping.yml
/opt/deception-lab/parser/rules/engage_mapping.yml

# 然後產生：
mapping_summary.json
attack_coverage.json
engage_coverage.json
```
- 執行：
```
cat > /opt/deception-lab/parser/analyze_mapping.py <<'EOF'
#!/usr/bin/env python3
import argparse
import json
from collections import Counter, defaultdict
from datetime import datetime, timezone
from pathlib import Path

import yaml


def parse_args():
    parser = argparse.ArgumentParser(
        description="Analyze MITRE ATT&CK and MITRE Engage mapping coverage."
    )
    parser.add_argument(
        "--project-root",
        default="/opt/deception-lab",
        help="Project root directory."
    )
    return parser.parse_args()


def load_yaml(path):
    if not path.exists():
        return {}
    with open(path, "r", encoding="utf-8") as f:
        return yaml.safe_load(f) or {}


def load_jsonl(path):
    records = []
    if not path.exists():
        return records

    with open(path, "r", encoding="utf-8", errors="ignore") as f:
        for line_no, line in enumerate(f, start=1):
            line = line.strip()
            if not line:
                continue
            try:
                records.append(json.loads(line))
            except json.JSONDecodeError:
                records.append({
                    "parse_error": True,
                    "line_no": line_no,
                    "raw": line
                })
    return records


def write_json(path, data):
    path.parent.mkdir(parents=True, exist_ok=True)
    path.write_text(
        json.dumps(data, ensure_ascii=False, indent=2, sort_keys=True),
        encoding="utf-8"
    )


def main():
    args = parse_args()
    root = Path(args.project_root)

    detections_path = root / "data" / "events" / "detections.jsonl"
    events_dir = root / "data" / "events"

    attack_path = root / "parser" / "rules" / "attack_mapping.yml"
    engage_path = root / "parser" / "rules" / "engage_mapping.yml"
    detection_rules_path = root / "parser" / "rules" / "detection_rules.yml"

    detections = load_jsonl(detections_path)
    attack_cfg = load_yaml(attack_path)
    engage_cfg = load_yaml(engage_path)
    rules_cfg = load_yaml(detection_rules_path)

    attack_mappings = attack_cfg.get("mappings", {})
    engage_mappings = engage_cfg.get("mappings", {})
    defined_rules = [r.get("id") for r in rules_cfg.get("rules", []) if r.get("id")]

    detection_rule_counter = Counter(d.get("rule_id") for d in detections if d.get("rule_id"))

    attack_by_tactic = Counter()
    attack_by_technique = Counter()
    engage_by_goal = Counter()
    engage_by_activity = Counter()

    unmapped_attack_rules = []
    unmapped_engage_rules = []

    per_rule = {}

    for rule_id in defined_rules:
        attack = attack_mappings.get(rule_id)
        engage = engage_mappings.get(rule_id)
        count = detection_rule_counter.get(rule_id, 0)

        if not attack:
            unmapped_attack_rules.append(rule_id)

        if not engage:
            unmapped_engage_rules.append(rule_id)

        if attack:
            tactic = attack.get("tactic", "Unknown")
            technique_id = attack.get("technique_id", "Unknown")
            technique = attack.get("technique", "Unknown")
            if count:
                attack_by_tactic[tactic] += count
                attack_by_technique[f"{technique_id} {technique}"] += count

        if engage:
            goal = engage.get("goal", "Unknown")
            activity = engage.get("activity", "Unknown")
            if count:
                engage_by_goal[goal] += count
                engage_by_activity[f"{goal}: {activity}"] += count

        per_rule[rule_id] = {
            "detections": count,
            "has_attack_mapping": bool(attack),
            "has_engage_mapping": bool(engage),
            "attack": attack or {},
            "engage": engage or {}
        }

    triggered_rules = sorted([r for r, c in detection_rule_counter.items() if c > 0])
    triggered_without_attack_mapping = [
        r for r in triggered_rules if r not in attack_mappings
    ]
    triggered_without_engage_mapping = [
        r for r in triggered_rules if r not in engage_mappings
    ]

    total_defined = len(defined_rules)
    attack_mapped_defined = total_defined - len(unmapped_attack_rules)
    engage_mapped_defined = total_defined - len(unmapped_engage_rules)

    mapping_summary = {
        "generated_at": datetime.now(timezone.utc).isoformat(),
        "input_detections": str(detections_path),
        "total_detections": len(detections),
        "defined_detection_rules": total_defined,
        "triggered_detection_rules": triggered_rules,
        "attack_mapping_coverage": {
            "mapped_defined_rules": attack_mapped_defined,
            "total_defined_rules": total_defined,
            "coverage_percent": round((attack_mapped_defined / total_defined) * 100, 2) if total_defined else 0,
            "unmapped_defined_rules": unmapped_attack_rules,
            "triggered_without_mapping": triggered_without_attack_mapping
        },
        "engage_mapping_coverage": {
            "mapped_defined_rules": engage_mapped_defined,
            "total_defined_rules": total_defined,
            "coverage_percent": round((engage_mapped_defined / total_defined) * 100, 2) if total_defined else 0,
            "unmapped_defined_rules": unmapped_engage_rules,
            "triggered_without_mapping": triggered_without_engage_mapping
        },
        "detections_by_rule": dict(detection_rule_counter),
        "per_rule": per_rule
    }

    attack_coverage = {
        "generated_at": datetime.now(timezone.utc).isoformat(),
        "framework": attack_cfg.get("framework", {}),
        "detections_by_tactic": dict(attack_by_tactic),
        "detections_by_technique": dict(attack_by_technique),
        "mappings": attack_mappings
    }

    engage_coverage = {
        "generated_at": datetime.now(timezone.utc).isoformat(),
        "framework": engage_cfg.get("framework", {}),
        "detections_by_goal": dict(engage_by_goal),
        "detections_by_activity": dict(engage_by_activity),
        "mappings": engage_mappings
    }

    write_json(events_dir / "mapping_summary.json", mapping_summary)
    write_json(events_dir / "attack_coverage.json", attack_coverage)
    write_json(events_dir / "engage_coverage.json", engage_coverage)

    print("[+] Mapping analysis completed.")
    print(f"[+] Mapping summary: {events_dir / 'mapping_summary.json'}")
    print(f"[+] ATT&CK coverage: {events_dir / 'attack_coverage.json'}")
    print(f"[+] Engage coverage: {events_dir / 'engage_coverage.json'}")
    print()
    print(f"Defined detection rules: {total_defined}")
    print(f"Total detections: {len(detections)}")
    print(f"ATT&CK mapping coverage: {mapping_summary['attack_mapping_coverage']['coverage_percent']}%")
    print(f"Engage mapping coverage: {mapping_summary['engage_mapping_coverage']['coverage_percent']}%")


if __name__ == "__main__":
    main()
EOF
```
- 設定可執行：
```
chmod +x /opt/deception-lab/parser/analyze_mapping.py
```

### Step 9.5：建立執行 mapping analyzer 的腳本
-執行：
```
cat > /opt/deception-lab/scripts/run_mapping.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

PROJECT_ROOT="/opt/deception-lab"
PARSER_VENV="$PROJECT_ROOT/parser/venv"
MAPPING_ANALYZER="$PROJECT_ROOT/parser/analyze_mapping.py"

echo "[+] Running parser first to refresh detections..."
"$PROJECT_ROOT/scripts/run_parser.sh"

echo
echo "[+] Running MITRE mapping analyzer..."
source "$PARSER_VENV/bin/activate"
python "$MAPPING_ANALYZER" --project-root "$PROJECT_ROOT"
deactivate

echo
echo "[+] Mapping output files:"
ls -lah "$PROJECT_ROOT/data/events" | grep -E 'mapping_summary|attack_coverage|engage_coverage' || true

echo
echo "[+] Mapping summary:"
cat "$PROJECT_ROOT/data/events/mapping_summary.json" | jq
EOF
```
- 設定可執行：
```
chmod +x /opt/deception-lab/scripts/run_mapping.sh
```

### Step 9.6：執行 mapping analyzer
- 請執行：
```
/opt/deception-lab/scripts/run_mapping.sh

# 成功時應該看到類似：
[+] Mapping analysis completed.
[+] Mapping summary: /opt/deception-lab/data/events/mapping_summary.json
[+] ATT&CK coverage: /opt/deception-lab/data/events/attack_coverage.json
[+] Engage coverage: /opt/deception-lab/data/events/engage_coverage.json

Defined detection rules: 8
Total detections: 11
ATT&CK mapping coverage: 100.0%
Engage mapping coverage: 100.0%

# 執行結果：
lss@lss:/opt/deception-lab $ /opt/deception-lab/scripts/run_mapping.sh
[+] Running parser first to refresh detections...
[+] Collecting latest logs...
[+] Collecting logs at UTC time: 20260520T235317Z
[+] Exporting Cowrie Docker logs...
[+] Collecting Fake Web access log...
[+] Collecting Fake Web auth log...
[+] Creating source manifest...
[+] Creating collection summary...
[+] Archiving collected logs...

[+] Collection completed.
[+] Current collected directory:
total 44K
drwxrwxr-x 2 lss lss 4.0K May 15 07:18 .
drwxrwxr-x 8 lss lss 4.0K May 13 03:01 ..
-rw-rw-r-- 1 lss lss  370 May 21 07:53 collection_summary.txt
-rw-rw-r-- 1 lss lss  12K May 21 07:53 cowrie-docker.log
-rw-rw-r-- 1 lss lss  579 May 15 07:18 README.md
-rw-rw-r-- 1 lss lss  777 May 21 07:53 source_manifest.json
-rwxrwxr-x 1 lss lss 6.5K May 21 07:53 web_access.jsonl
-rwxrwxr-x 1 lss lss 1.7K May 21 07:53 web_auth.jsonl

[+] Summary:
Collection time UTC: 20260520T235317Z

Collected files:

- /opt/deception-lab/data/collected/cowrie-docker.log
  size: 12K
  lines: 93
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
-rw-rw-r-- 1 lss lss 6.9K May 21 07:53 detections.jsonl
-rw-rw-r-- 1 lss lss  13K May 21 07:53 events.jsonl
-rw-rw-r-- 1 lss lss  631 May 21 07:53 events_summary.json

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
  "generated_at": "2026-05-20T23:53:18.174017+00:00",
  "total_detections": 11,
  "total_events": 34
}
[+] Running MITRE mapping analyzer...
[+] Mapping analysis completed.
[+] Mapping summary: /opt/deception-lab/data/events/mapping_summary.json
[+] ATT&CK coverage: /opt/deception-lab/data/events/attack_coverage.json
[+] Engage coverage: /opt/deception-lab/data/events/engage_coverage.json

Defined detection rules: 8
Total detections: 11
ATT&CK mapping coverage: 100.0%
Engage mapping coverage: 100.0%

[+] Mapping output files:
-rw-rw-r-- 1 lss lss 2.9K May 21 07:53 attack_coverage.json
-rw-rw-r-- 1 lss lss 2.2K May 21 07:53 engage_coverage.json
-rw-rw-r-- 1 lss lss 6.1K May 21 07:53 mapping_summary.json

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
    "SSH_HONEYCREDENTIAL_LOGIN": 1,
    "SSH_HONEYFILE_ACCESS": 1,
    "SSH_RECON_COMMAND": 5,
    "WEB_HONEYCREDENTIAL_USED": 2,
    "WEB_HONEYFILE_ACCESS": 2
  },
  "engage_mapping_coverage": {
    "coverage_percent": 100.0,
    "mapped_defined_rules": 8,
    "total_defined_rules": 8,
    "triggered_without_mapping": [],
    "unmapped_defined_rules": []
  },
  "generated_at": "2026-05-20T23:53:18.251815+00:00",
  "input_detections": "/opt/deception-lab/data/events/detections.jsonl",
  "per_rule": {
    "SSH_HONEYCREDENTIAL_LOGIN": {
      "attack": {
        "rationale": "The attacker used a credential intentionally planted in deception assets. This indicates exposure to unsecured credential material.\n",
        "tactic": "Credential Access",
        "technique": "Unsecured Credentials",
        "technique_id": "T1552"
      },
      "detections": 1,
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
      "detections": 1,
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
      "detections": 5,
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
      "detections": 2,
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
      "detections": 2,
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
      "detections": 0,
      "engage": {
        "activity": "Expose Decoy Service",
        "goal": "Expose",
        "rationale": "The fake web interface exposes probeable paths to observe scanner behavior.\n"
      },
      "has_attack_mapping": true,
      "has_engage_mapping": true
    }
  },
  "total_detections": 11,
  "triggered_detection_rules": [
    "SSH_HONEYCREDENTIAL_LOGIN",
    "SSH_HONEYFILE_ACCESS",
    "SSH_RECON_COMMAND",
    "WEB_HONEYCREDENTIAL_USED",
    "WEB_HONEYFILE_ACCESS"
  ]
}
```

### Step 9.7：查看 mapping summary
- 請執行：
```
cat /opt/deception-lab/data/events/mapping_summary.json | jq

# 你要確認：
attack_mapping_coverage coverage_percent 是 100.0
engage_mapping_coverage coverage_percent 是 100.0
triggered_without_mapping 是 []

# 執行結果：
lss@lss:/opt/deception-lab $ cat /opt/deception-lab/data/events/mapping_summary.json | jq
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
    "SSH_HONEYCREDENTIAL_LOGIN": 1,
    "SSH_HONEYFILE_ACCESS": 1,
    "SSH_RECON_COMMAND": 5,
    "WEB_HONEYCREDENTIAL_USED": 2,
    "WEB_HONEYFILE_ACCESS": 2
  },
  "engage_mapping_coverage": {
    "coverage_percent": 100.0,
    "mapped_defined_rules": 8,
    "total_defined_rules": 8,
    "triggered_without_mapping": [],
    "unmapped_defined_rules": []
  },
  "generated_at": "2026-05-20T23:53:18.251815+00:00",
  "input_detections": "/opt/deception-lab/data/events/detections.jsonl",
  "per_rule": {
    "SSH_HONEYCREDENTIAL_LOGIN": {
      "attack": {
        "rationale": "The attacker used a credential intentionally planted in deception assets. This indicates exposure to unsecured credential material.\n",
        "tactic": "Credential Access",
        "technique": "Unsecured Credentials",
        "technique_id": "T1552"
      },
      "detections": 1,
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
      "detections": 1,
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
      "detections": 5,
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
      "detections": 2,
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
      "detections": 2,
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
      "detections": 0,
      "engage": {
        "activity": "Expose Decoy Service",
        "goal": "Expose",
        "rationale": "The fake web interface exposes probeable paths to observe scanner behavior.\n"
      },
      "has_attack_mapping": true,
      "has_engage_mapping": true
    }
  },
  "total_detections": 11,
  "triggered_detection_rules": [
    "SSH_HONEYCREDENTIAL_LOGIN",
    "SSH_HONEYFILE_ACCESS",
    "SSH_RECON_COMMAND",
    "WEB_HONEYCREDENTIAL_USED",
    "WEB_HONEYFILE_ACCESS"
  ]
}
```

### Step 9.8：查看 ATT&CK coverage
- 請執行：
```
cat /opt/deception-lab/data/events/attack_coverage.json | jq

# 你應該會看到類似：實際數字會依你目前 log 而定。
"detections_by_tactic": {
  "Collection": 3,
  "Credential Access": 3,
  "Discovery": 5
}
及
"detections_by_technique": {
  "T1005 Data from Local System": 3,
  "T1552 Unsecured Credentials": 3,
  "T1082 System Information Discovery": 5
}

# 執行結果：
lss@lss:/opt/deception-lab $ cat /opt/deception-lab/data/events/attack_coverage.json | jq
{
  "detections_by_tactic": {
    "Collection": 3,
    "Credential Access": 3,
    "Discovery": 5
  },
  "detections_by_technique": {
    "T1005 Data from Local System": 3,
    "T1082 System Information Discovery": 5,
    "T1552 Unsecured Credentials": 3
  },
  "framework": {
    "matrix": "Enterprise",
    "name": "MITRE ATT&CK",
    "version_note": "MVP mapping for Raspberry Pi deception lab"
  },
  "generated_at": "2026-05-20T23:53:18.251850+00:00",
  "mappings": {
    "SSH_HONEYCREDENTIAL_LOGIN": {
      "rationale": "The attacker used a credential intentionally planted in deception assets. This indicates exposure to unsecured credential material.\n",
      "tactic": "Credential Access",
      "technique": "Unsecured Credentials",
      "technique_id": "T1552"
    },
    "SSH_HONEYFILE_ACCESS": {
      "rationale": "Attempting to read honeyfiles such as secrets.txt or backup_config.ini indicates interest in local sensitive data.\n",
      "tactic": "Collection",
      "technique": "Data from Local System",
      "technique_id": "T1005"
    },
    "SSH_LOGIN_FAILED": {
      "rationale": "Failed SSH authentication attempts against the honeypot indicate password guessing or brute-force behavior.\n",
      "tactic": "Credential Access",
      "technique": "Brute Force",
      "technique_id": "T1110"
    },
    "SSH_RECON_COMMAND": {
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
    "SSH_TOOL_TRANSFER_COMMAND": {
      "rationale": "Commands such as wget, curl, ftp, scp, chmod, nc, or bash reverse shell patterns may indicate tool staging or payload transfer.\n",
      "tactic": "Command and Control",
      "technique": "Ingress Tool Transfer",
      "technique_id": "T1105"
    },
    "WEB_HONEYCREDENTIAL_USED": {
      "rationale": "Submitting a planted credential to the fake web login panel indicates the adversary discovered and attempted to use unsecured credentials.\n",
      "tactic": "Credential Access",
      "technique": "Unsecured Credentials",
      "technique_id": "T1552"
    },
    "WEB_HONEYFILE_ACCESS": {
      "rationale": "Downloading honeyfiles from the fake web panel indicates attempted collection of local or exposed files.\n",
      "tactic": "Collection",
      "technique": "Data from Local System",
      "technique_id": "T1005"
    },
    "WEB_SCANNER_PROBE": {
      "rationale": "Requests to paths such as /.env, /wp-admin, /phpmyadmin, or /admin indicate web probing or scanning behavior.\n",
      "tactic": "Reconnaissance",
      "technique": "Active Scanning",
      "technique_id": "T1595"
    }
  }
}
```

### Step 9.9：查看 Engage coverage
- 請執行：
```
cat /opt/deception-lab/data/events/engage_coverage.json | jq

# 你應該會看到類似：
"detections_by_goal": {
  "Elicit": 4,
  "Understand": 7
}
及
"detections_by_activity": {
  "Elicit: Credential Collection": 3,
  "Elicit: Reveal Adversary Intent": 1,
  "Understand: Collect Adversary Behavior": 7
}

# 執行結果：
lss@lss:/opt/deception-lab $ cat /opt/deception-lab/data/events/engage_coverage.json | jq
{
  "detections_by_activity": {
    "Elicit: Credential Collection": 3,
    "Elicit: Reveal Adversary Intent": 1,
    "Understand: Collect Adversary Behavior": 7
  },
  "detections_by_goal": {
    "Elicit": 4,
    "Understand": 7
  },
  "framework": {
    "name": "MITRE Engage",
    "version_note": "MVP mapping for Raspberry Pi deception lab"
  },
  "generated_at": "2026-05-20T23:53:18.251855+00:00",
  "mappings": {
    "SSH_HONEYCREDENTIAL_LOGIN": {
      "activity": "Credential Collection",
      "goal": "Elicit",
      "rationale": "A planted SSH credential caused the adversary to reveal credential-use behavior.\n"
    },
    "SSH_HONEYFILE_ACCESS": {
      "activity": "Reveal Adversary Intent",
      "goal": "Elicit",
      "rationale": "Honeyfile access attempts reveal interest in sensitive files, backup data, credentials, and local configuration.\n"
    },
    "SSH_LOGIN_FAILED": {
      "activity": "Expose Decoy Service",
      "goal": "Expose",
      "rationale": "The SSH honeypot exposes a fake login surface for observing authentication attempts.\n"
    },
    "SSH_RECON_COMMAND": {
      "activity": "Collect Adversary Behavior",
      "goal": "Understand",
      "rationale": "Commands entered after login reveal adversary discovery interests and operating style.\n"
    },
    "SSH_TOOL_TRANSFER_COMMAND": {
      "activity": "Adversary Direction",
      "goal": "Affect",
      "rationale": "A decoy shell can influence attacker behavior and redirect tool staging activity into a controlled observation environment.\n"
    },
    "WEB_HONEYCREDENTIAL_USED": {
      "activity": "Credential Collection",
      "goal": "Elicit",
      "rationale": "Planted web credentials elicit credential reuse behavior against the fake admin panel.\n"
    },
    "WEB_HONEYFILE_ACCESS": {
      "activity": "Collect Adversary Behavior",
      "goal": "Understand",
      "rationale": "Honeyfile download behavior helps identify the attacker's collection interests.\n"
    },
    "WEB_SCANNER_PROBE": {
      "activity": "Expose Decoy Service",
      "goal": "Expose",
      "rationale": "The fake web interface exposes probeable paths to observe scanner behavior.\n"
    }
  }
}
```

### Step 9.10：建立 mapping 快速查看腳本
- 請執行：
```
cat > /opt/deception-lab/scripts/show_mapping.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

EVENT_DIR="/opt/deception-lab/data/events"

echo "=== Mapping Files ==="
ls -lah "$EVENT_DIR" | grep -E 'mapping_summary|attack_coverage|engage_coverage' || true

echo
echo "=== Mapping Summary ==="
if [ -f "$EVENT_DIR/mapping_summary.json" ]; then
  cat "$EVENT_DIR/mapping_summary.json" | jq
else
  echo "No mapping_summary.json found."
fi

echo
echo "=== ATT&CK Detection Coverage ==="
if [ -f "$EVENT_DIR/attack_coverage.json" ]; then
  echo "--- Tactics ---"
  cat "$EVENT_DIR/attack_coverage.json" | jq '.detections_by_tactic'
  echo
  echo "--- Techniques ---"
  cat "$EVENT_DIR/attack_coverage.json" | jq '.detections_by_technique'
else
  echo "No attack_coverage.json found."
fi

echo
echo "=== Engage Detection Coverage ==="
if [ -f "$EVENT_DIR/engage_coverage.json" ]; then
  echo "--- Goals ---"
  cat "$EVENT_DIR/engage_coverage.json" | jq '.detections_by_goal'
  echo
  echo "--- Activities ---"
  cat "$EVENT_DIR/engage_coverage.json" | jq '.detections_by_activity'
else
  echo "No engage_coverage.json found."
fi
EOF
```
- 設定可執行：
```
chmod +x /opt/deception-lab/scripts/show_mapping.sh
```
執行：
```
/opt/deception-lab/scripts/show_mapping.sh

# 執行結果：
lss@lss:/opt/deception-lab $ /opt/deception-lab/scripts/show_mapping.sh
=== Mapping Files ===
-rw-rw-r-- 1 lss lss 2.9K May 21 07:53 attack_coverage.json
-rw-rw-r-- 1 lss lss 2.2K May 21 07:53 engage_coverage.json
-rw-rw-r-- 1 lss lss 6.1K May 21 07:53 mapping_summary.json

=== Mapping Summary ===
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
    "SSH_HONEYCREDENTIAL_LOGIN": 1,
    "SSH_HONEYFILE_ACCESS": 1,
    "SSH_RECON_COMMAND": 5,
    "WEB_HONEYCREDENTIAL_USED": 2,
    "WEB_HONEYFILE_ACCESS": 2
  },
  "engage_mapping_coverage": {
    "coverage_percent": 100.0,
    "mapped_defined_rules": 8,
    "total_defined_rules": 8,
    "triggered_without_mapping": [],
    "unmapped_defined_rules": []
  },
  "generated_at": "2026-05-20T23:53:18.251815+00:00",
  "input_detections": "/opt/deception-lab/data/events/detections.jsonl",
  "per_rule": {
    "SSH_HONEYCREDENTIAL_LOGIN": {
      "attack": {
        "rationale": "The attacker used a credential intentionally planted in deception assets. This indicates exposure to unsecured credential material.\n",
        "tactic": "Credential Access",
        "technique": "Unsecured Credentials",
        "technique_id": "T1552"
      },
      "detections": 1,
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
      "detections": 1,
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
      "detections": 5,
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
      "detections": 2,
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
      "detections": 2,
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
      "detections": 0,
      "engage": {
        "activity": "Expose Decoy Service",
        "goal": "Expose",
        "rationale": "The fake web interface exposes probeable paths to observe scanner behavior.\n"
      },
      "has_attack_mapping": true,
      "has_engage_mapping": true
    }
  },
  "total_detections": 11,
  "triggered_detection_rules": [
    "SSH_HONEYCREDENTIAL_LOGIN",
    "SSH_HONEYFILE_ACCESS",
    "SSH_RECON_COMMAND",
    "WEB_HONEYCREDENTIAL_USED",
    "WEB_HONEYFILE_ACCESS"
  ]
}

=== ATT&CK Detection Coverage ===
--- Tactics ---
{
  "Collection": 3,
  "Credential Access": 3,
  "Discovery": 5
}

--- Techniques ---
{
  "T1005 Data from Local System": 3,
  "T1082 System Information Discovery": 5,
  "T1552 Unsecured Credentials": 3
}

=== Engage Detection Coverage ===
--- Goals ---
{
  "Elicit": 4,
  "Understand": 7
}

--- Activities ---
{
  "Elicit: Credential Collection": 3,
  "Elicit: Reveal Adversary Intent": 1,
  "Understand: Collect Adversary Behavior": 7
}
```

### Step 9.11：產生 Markdown mapping 報告
這份報告先給你人工閱讀，之後第十階段會整合進正式 report.md。
- 請建立程式：
```
cat > /opt/deception-lab/parser/generate_mapping_markdown.py <<'EOF'
#!/usr/bin/env python3
import json
from datetime import datetime, timezone
from pathlib import Path


ROOT = Path("/opt/deception-lab")
EVENT_DIR = ROOT / "data" / "events"
REPORT_DIR = ROOT / "reports"

MAPPING_SUMMARY = EVENT_DIR / "mapping_summary.json"
ATTACK_COVERAGE = EVENT_DIR / "attack_coverage.json"
ENGAGE_COVERAGE = EVENT_DIR / "engage_coverage.json"

OUTPUT = REPORT_DIR / "mapping_report.md"


def load_json(path):
    if not path.exists():
        return {}
    return json.loads(path.read_text(encoding="utf-8"))


def markdown_table(headers, rows):
    lines = []
    lines.append("| " + " | ".join(headers) + " |")
    lines.append("| " + " | ".join(["---"] * len(headers)) + " |")
    for row in rows:
        lines.append("| " + " | ".join(str(x) for x in row) + " |")
    return "\n".join(lines)


def main():
    REPORT_DIR.mkdir(parents=True, exist_ok=True)

    summary = load_json(MAPPING_SUMMARY)
    attack = load_json(ATTACK_COVERAGE)
    engage = load_json(ENGAGE_COVERAGE)

    lines = []
    lines.append("# MITRE Mapping Report")
    lines.append("")
    lines.append(f"Generated at: {datetime.now(timezone.utc).isoformat()}")
    lines.append("")

    lines.append("## Mapping Coverage")
    lines.append("")
    attack_cov = summary.get("attack_mapping_coverage", {})
    engage_cov = summary.get("engage_mapping_coverage", {})

    lines.append(markdown_table(
        ["Framework", "Mapped Rules", "Total Rules", "Coverage"],
        [
            [
                "MITRE ATT&CK",
                attack_cov.get("mapped_defined_rules", 0),
                attack_cov.get("total_defined_rules", 0),
                f"{attack_cov.get('coverage_percent', 0)}%"
            ],
            [
                "MITRE Engage",
                engage_cov.get("mapped_defined_rules", 0),
                engage_cov.get("total_defined_rules", 0),
                f"{engage_cov.get('coverage_percent', 0)}%"
            ]
        ]
    ))
    lines.append("")

    lines.append("## Detection Counts by Rule")
    lines.append("")
    detections_by_rule = summary.get("detections_by_rule", {})
    lines.append(markdown_table(
        ["Rule ID", "Count"],
        sorted(detections_by_rule.items())
    ))
    lines.append("")

    lines.append("## ATT&CK Coverage by Tactic")
    lines.append("")
    attack_tactics = attack.get("detections_by_tactic", {})
    lines.append(markdown_table(
        ["Tactic", "Detection Count"],
        sorted(attack_tactics.items())
    ))
    lines.append("")

    lines.append("## ATT&CK Coverage by Technique")
    lines.append("")
    attack_techniques = attack.get("detections_by_technique", {})
    lines.append(markdown_table(
        ["Technique", "Detection Count"],
        sorted(attack_techniques.items())
    ))
    lines.append("")

    lines.append("## MITRE Engage Coverage by Goal")
    lines.append("")
    engage_goals = engage.get("detections_by_goal", {})
    lines.append(markdown_table(
        ["Goal", "Detection Count"],
        sorted(engage_goals.items())
    ))
    lines.append("")

    lines.append("## MITRE Engage Coverage by Activity")
    lines.append("")
    engage_activities = engage.get("detections_by_activity", {})
    lines.append(markdown_table(
        ["Activity", "Detection Count"],
        sorted(engage_activities.items())
    ))
    lines.append("")

    lines.append("## Per-rule Mapping")
    lines.append("")
    per_rule = summary.get("per_rule", {})
    rows = []
    for rule_id, item in sorted(per_rule.items()):
        attack_item = item.get("attack", {})
        engage_item = item.get("engage", {})
        rows.append([
            rule_id,
            item.get("detections", 0),
            attack_item.get("technique_id", ""),
            attack_item.get("technique", ""),
            engage_item.get("goal", ""),
            engage_item.get("activity", "")
        ])

    lines.append(markdown_table(
        ["Rule ID", "Detections", "ATT&CK ID", "ATT&CK Technique", "Engage Goal", "Engage Activity"],
        rows
    ))
    lines.append("")

    OUTPUT.write_text("\n".join(lines), encoding="utf-8")
    print(f"[+] Mapping markdown report written to: {OUTPUT}")


if __name__ == "__main__":
    main()
EOF
```
- 設定可執行：
```
chmod +x /opt/deception-lab/parser/generate_mapping_markdown.py
```
- 執行：
```
source /opt/deception-lab/parser/venv/bin/activate
python /opt/deception-lab/parser/generate_mapping_markdown.py
deactivate
```
- 執行結果：
```
lss@lss:/opt/deception-lab $ source /opt/deception-lab/parser/venv/bin/activate
python /opt/deception-lab/parser/generate_mapping_markdown.py
deactivate
[+] Mapping markdown report written to: /opt/deception-lab/reports/mapping_report.md

lss@lss:/opt/deception-lab $ source /opt/deception-lab/parser/venv/bin/activate
(venv) lss@lss:/opt/deception-lab $ python /opt/deception-lab/parser/generate_mapping_markdown.py
[+] Mapping markdown report written to: /opt/deception-lab/reports/mapping_report.md

(venv) lss@lss:/opt/deception-lab $ deactivate
lss@lss:/opt/deception-lab $
```
- 確認：
```
cat /opt/deception-lab/reports/mapping_report.md

# 執行結果：
lss@lss:/opt/deception-lab $ cat /opt/deception-lab/reports/mapping_report.md
# MITRE Mapping Report

Generated at: 2026-05-21T00:07:51.586744+00:00

## Mapping Coverage

| Framework | Mapped Rules | Total Rules | Coverage |
| --- | --- | --- | --- |
| MITRE ATT&CK | 8 | 8 | 100.0% |
| MITRE Engage | 8 | 8 | 100.0% |

## Detection Counts by Rule

| Rule ID | Count |
| --- | --- |
| SSH_HONEYCREDENTIAL_LOGIN | 1 |
| SSH_HONEYFILE_ACCESS | 1 |
| SSH_RECON_COMMAND | 5 |
| WEB_HONEYCREDENTIAL_USED | 2 |
| WEB_HONEYFILE_ACCESS | 2 |

## ATT&CK Coverage by Tactic

| Tactic | Detection Count |
| --- | --- |
| Collection | 3 |
| Credential Access | 3 |
| Discovery | 5 |

## ATT&CK Coverage by Technique

| Technique | Detection Count |
| --- | --- |
| T1005 Data from Local System | 3 |
| T1082 System Information Discovery | 5 |
| T1552 Unsecured Credentials | 3 |

## MITRE Engage Coverage by Goal

| Goal | Detection Count |
| --- | --- |
| Elicit | 4 |
| Understand | 7 |

## MITRE Engage Coverage by Activity

| Activity | Detection Count |
| --- | --- |
| Elicit: Credential Collection | 3 |
| Elicit: Reveal Adversary Intent | 1 |
| Understand: Collect Adversary Behavior | 7 |

## Per-rule Mapping

| Rule ID | Detections | ATT&CK ID | ATT&CK Technique | Engage Goal | Engage Activity |
| --- | --- | --- | --- | --- | --- |
| SSH_HONEYCREDENTIAL_LOGIN | 1 | T1552 | Unsecured Credentials | Elicit | Credential Collection |
| SSH_HONEYFILE_ACCESS | 1 | T1005 | Data from Local System | Elicit | Reveal Adversary Intent |
| SSH_LOGIN_FAILED | 0 | T1110 | Brute Force | Expose | Expose Decoy Service |
| SSH_RECON_COMMAND | 5 | T1082 | System Information Discovery | Understand | Collect Adversary Behavior |
| SSH_TOOL_TRANSFER_COMMAND | 0 | T1105 | Ingress Tool Transfer | Affect | Adversary Direction |
| WEB_HONEYCREDENTIAL_USED | 2 | T1552 | Unsecured Credentials | Elicit | Credential Collection |
| WEB_HONEYFILE_ACCESS | 2 | T1005 | Data from Local System | Understand | Collect Adversary Behavior |
| WEB_SCANNER_PROBE | 0 | T1595 | Active Scanning | Expose | Expose Decoy Service |
lss@lss:/opt/deception-lab $
```

### Step 9.12：更新 run_mapping.sh，自動產生 Markdown 報告
- 請覆蓋原本 run_mapping.sh：
```
cat > /opt/deception-lab/scripts/run_mapping.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

PROJECT_ROOT="/opt/deception-lab"
PARSER_VENV="$PROJECT_ROOT/parser/venv"
MAPPING_ANALYZER="$PROJECT_ROOT/parser/analyze_mapping.py"
MAPPING_MD="$PROJECT_ROOT/parser/generate_mapping_markdown.py"

echo "[+] Running parser first to refresh detections..."
"$PROJECT_ROOT/scripts/run_parser.sh"

echo
echo "[+] Running MITRE mapping analyzer..."
source "$PARSER_VENV/bin/activate"
python "$MAPPING_ANALYZER" --project-root "$PROJECT_ROOT"

echo
echo "[+] Generating mapping Markdown report..."
python "$MAPPING_MD"
deactivate

echo
echo "[+] Mapping output files:"
ls -lah "$PROJECT_ROOT/data/events" | grep -E 'mapping_summary|attack_coverage|engage_coverage' || true

echo
echo "[+] Mapping report:"
ls -lah "$PROJECT_ROOT/reports/mapping_report.md"

echo
echo "[+] Mapping summary:"
cat "$PROJECT_ROOT/data/events/mapping_summary.json" | jq
EOF
```
- 設定可執行：
```
chmod +x /opt/deception-lab/scripts/run_mapping.sh
```
- 重新執行：
```
/opt/deception-lab/scripts/run_mapping.sh

# 執行結果：
lss@lss:/opt/deception-lab $ /opt/deception-lab/scripts/run_mapping.sh
[+] Running parser first to refresh detections...
[+] Collecting latest logs...
[+] Collecting logs at UTC time: 20260521T001319Z
[+] Exporting Cowrie Docker logs...
[+] Collecting Fake Web access log...
[+] Collecting Fake Web auth log...
[+] Creating source manifest...
[+] Creating collection summary...
[+] Archiving collected logs...

[+] Collection completed.
[+] Current collected directory:
total 44K
drwxrwxr-x 2 lss lss 4.0K May 15 07:18 .
drwxrwxr-x 8 lss lss 4.0K May 13 03:01 ..
-rw-rw-r-- 1 lss lss  370 May 21 08:13 collection_summary.txt
-rw-rw-r-- 1 lss lss  12K May 21 08:13 cowrie-docker.log
-rw-rw-r-- 1 lss lss  579 May 15 07:18 README.md
-rw-rw-r-- 1 lss lss  777 May 21 08:13 source_manifest.json
-rwxrwxr-x 1 lss lss 6.5K May 21 08:13 web_access.jsonl
-rwxrwxr-x 1 lss lss 1.7K May 21 08:13 web_auth.jsonl

[+] Summary:
Collection time UTC: 20260521T001319Z

Collected files:

- /opt/deception-lab/data/collected/cowrie-docker.log
  size: 12K
  lines: 93
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
total 52K
drwxrwxr-x 2 lss lss 4.0K May 21 07:53 .
drwxrwxr-x 8 lss lss 4.0K May 13 03:01 ..
-rw-rw-r-- 1 lss lss 2.9K May 21 07:53 attack_coverage.json
-rw-rw-r-- 1 lss lss 6.9K May 21 08:13 detections.jsonl
-rw-rw-r-- 1 lss lss 2.2K May 21 07:53 engage_coverage.json
-rw-rw-r-- 1 lss lss  13K May 21 08:13 events.jsonl
-rw-rw-r-- 1 lss lss  631 May 21 08:13 events_summary.json
-rw-rw-r-- 1 lss lss 6.1K May 21 07:53 mapping_summary.json

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
  "generated_at": "2026-05-21T00:13:19.362240+00:00",
  "total_detections": 11,
  "total_events": 34
}
[+] Running MITRE mapping analyzer...
[+] Mapping analysis completed.
[+] Mapping summary: /opt/deception-lab/data/events/mapping_summary.json
[+] ATT&CK coverage: /opt/deception-lab/data/events/attack_coverage.json
[+] Engage coverage: /opt/deception-lab/data/events/engage_coverage.json

Defined detection rules: 8
Total detections: 11
ATT&CK mapping coverage: 100.0%
Engage mapping coverage: 100.0%

[+] Generating mapping Markdown report...
[+] Mapping markdown report written to: /opt/deception-lab/reports/mapping_report.md

[+] Mapping output files:
-rw-rw-r-- 1 lss lss 2.9K May 21 08:13 attack_coverage.json
-rw-rw-r-- 1 lss lss 2.2K May 21 08:13 engage_coverage.json
-rw-rw-r-- 1 lss lss 6.1K May 21 08:13 mapping_summary.json

[+] Mapping report:
-rw-rw-r-- 1 lss lss 2.0K May 21 08:13 /opt/deception-lab/reports/mapping_report.md

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
    "SSH_HONEYCREDENTIAL_LOGIN": 1,
    "SSH_HONEYFILE_ACCESS": 1,
    "SSH_RECON_COMMAND": 5,
    "WEB_HONEYCREDENTIAL_USED": 2,
    "WEB_HONEYFILE_ACCESS": 2
  },
  "engage_mapping_coverage": {
    "coverage_percent": 100.0,
    "mapped_defined_rules": 8,
    "total_defined_rules": 8,
    "triggered_without_mapping": [],
    "unmapped_defined_rules": []
  },
  "generated_at": "2026-05-21T00:13:19.439251+00:00",
  "input_detections": "/opt/deception-lab/data/events/detections.jsonl",
  "per_rule": {
    "SSH_HONEYCREDENTIAL_LOGIN": {
      "attack": {
        "rationale": "The attacker used a credential intentionally planted in deception assets. This indicates exposure to unsecured credential material.\n",
        "tactic": "Credential Access",
        "technique": "Unsecured Credentials",
        "technique_id": "T1552"
      },
      "detections": 1,
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
      "detections": 1,
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
      "detections": 5,
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
      "detections": 2,
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
      "detections": 2,
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
      "detections": 0,
      "engage": {
        "activity": "Expose Decoy Service",
        "goal": "Expose",
        "rationale": "The fake web interface exposes probeable paths to observe scanner behavior.\n"
      },
      "has_attack_mapping": true,
      "has_engage_mapping": true
    }
  },
  "total_detections": 11,
  "triggered_detection_rules": [
    "SSH_HONEYCREDENTIAL_LOGIN",
    "SSH_HONEYFILE_ACCESS",
    "SSH_RECON_COMMAND",
    "WEB_HONEYCREDENTIAL_USED",
    "WEB_HONEYFILE_ACCESS"
  ]
}
```

### Step 9.13：建立第九階段完成紀錄
- 如果以上都成功，建立：
```
cat > /opt/deception-lab/PHASE9_READY.md <<'EOF'
# Phase 9 Ready

MITRE ATT&CK and MITRE Engage mapping have been implemented.

Completed items:

- attack_mapping.yml created
- engage_mapping.yml created
- analyze_mapping.py created
- run_mapping.sh created
- show_mapping.sh created
- generate_mapping_markdown.py created
- mapping_summary.json generated
- attack_coverage.json generated
- engage_coverage.json generated
- mapping_report.md generated

Main mapping rule files:

- /opt/deception-lab/parser/rules/attack_mapping.yml
- /opt/deception-lab/parser/rules/engage_mapping.yml

Main mapping outputs:

- /opt/deception-lab/data/events/mapping_summary.json
- /opt/deception-lab/data/events/attack_coverage.json
- /opt/deception-lab/data/events/engage_coverage.json
- /opt/deception-lab/reports/mapping_report.md

Next phase:

Phase 10 - Event timeline and Markdown / JSON report generation.
EOF
```
- 確認：
```
cat /opt/deception-lab/PHASE9_READY.md

# 執行結果：
lss@lss:/opt/deception-lab $ cat /opt/deception-lab/PHASE9_READY.md
# Phase 9 Ready

MITRE ATT&CK and MITRE Engage mapping have been implemented.

Completed items:

- attack_mapping.yml created
- engage_mapping.yml created
- analyze_mapping.py created
- run_mapping.sh created
- show_mapping.sh created
- generate_mapping_markdown.py created
- mapping_summary.json generated
- attack_coverage.json generated
- engage_coverage.json generated
- mapping_report.md generated

Main mapping rule files:

- /opt/deception-lab/parser/rules/attack_mapping.yml
- /opt/deception-lab/parser/rules/engage_mapping.yml

Main mapping outputs:

- /opt/deception-lab/data/events/mapping_summary.json
- /opt/deception-lab/data/events/attack_coverage.json
- /opt/deception-lab/data/events/engage_coverage.json
- /opt/deception-lab/reports/mapping_report.md

Next phase:

Phase 10 - Event timeline and Markdown / JSON report generation.
```

## 第九階段完成檢查
```
/opt/deception-lab/scripts/run_mapping.sh

/opt/deception-lab/scripts/show_mapping.sh

ls -lah /opt/deception-lab/reports

head -n 80 /opt/deception-lab/reports/mapping_report.md

cat /opt/deception-lab/PHASE9_READY.md
```
- 執行結果：
```
lss@lss:/opt/deception-lab $ /opt/deception-lab/scripts/run_mapping.sh
[+] Running parser first to refresh detections...
[+] Collecting latest logs...
[+] Collecting logs at UTC time: 20260521T001703Z
[+] Exporting Cowrie Docker logs...
[+] Collecting Fake Web access log...
[+] Collecting Fake Web auth log...
[+] Creating source manifest...
[+] Creating collection summary...
[+] Archiving collected logs...

[+] Collection completed.
[+] Current collected directory:
total 44K
drwxrwxr-x 2 lss lss 4.0K May 15 07:18 .
drwxrwxr-x 8 lss lss 4.0K May 13 03:01 ..
-rw-rw-r-- 1 lss lss  370 May 21 08:17 collection_summary.txt
-rw-rw-r-- 1 lss lss  12K May 21 08:17 cowrie-docker.log
-rw-rw-r-- 1 lss lss  579 May 15 07:18 README.md
-rw-rw-r-- 1 lss lss  777 May 21 08:17 source_manifest.json
-rwxrwxr-x 1 lss lss 6.5K May 21 08:17 web_access.jsonl
-rwxrwxr-x 1 lss lss 1.7K May 21 08:17 web_auth.jsonl

[+] Summary:
Collection time UTC: 20260521T001703Z

Collected files:

- /opt/deception-lab/data/collected/cowrie-docker.log
  size: 12K
  lines: 93
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
total 52K
drwxrwxr-x 2 lss lss 4.0K May 21 07:53 .
drwxrwxr-x 8 lss lss 4.0K May 13 03:01 ..
-rw-rw-r-- 1 lss lss 2.9K May 21 08:13 attack_coverage.json
-rw-rw-r-- 1 lss lss 6.9K May 21 08:17 detections.jsonl
-rw-rw-r-- 1 lss lss 2.2K May 21 08:13 engage_coverage.json
-rw-rw-r-- 1 lss lss  13K May 21 08:17 events.jsonl
-rw-rw-r-- 1 lss lss  631 May 21 08:17 events_summary.json
-rw-rw-r-- 1 lss lss 6.1K May 21 08:13 mapping_summary.json

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
  "generated_at": "2026-05-21T00:17:03.602337+00:00",
  "total_detections": 11,
  "total_events": 34
}
[+] Running MITRE mapping analyzer...
[+] Mapping analysis completed.
[+] Mapping summary: /opt/deception-lab/data/events/mapping_summary.json
[+] ATT&CK coverage: /opt/deception-lab/data/events/attack_coverage.json
[+] Engage coverage: /opt/deception-lab/data/events/engage_coverage.json

Defined detection rules: 8
Total detections: 11
ATT&CK mapping coverage: 100.0%
Engage mapping coverage: 100.0%

[+] Generating mapping Markdown report...
[+] Mapping markdown report written to: /opt/deception-lab/reports/mapping_report.md

[+] Mapping output files:
-rw-rw-r-- 1 lss lss 2.9K May 21 08:17 attack_coverage.json
-rw-rw-r-- 1 lss lss 2.2K May 21 08:17 engage_coverage.json
-rw-rw-r-- 1 lss lss 6.1K May 21 08:17 mapping_summary.json

[+] Mapping report:
-rw-rw-r-- 1 lss lss 2.0K May 21 08:17 /opt/deception-lab/reports/mapping_report.md

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
    "SSH_HONEYCREDENTIAL_LOGIN": 1,
    "SSH_HONEYFILE_ACCESS": 1,
    "SSH_RECON_COMMAND": 5,
    "WEB_HONEYCREDENTIAL_USED": 2,
    "WEB_HONEYFILE_ACCESS": 2
  },
  "engage_mapping_coverage": {
    "coverage_percent": 100.0,
    "mapped_defined_rules": 8,
    "total_defined_rules": 8,
    "triggered_without_mapping": [],
    "unmapped_defined_rules": []
  },
  "generated_at": "2026-05-21T00:17:03.679497+00:00",
  "input_detections": "/opt/deception-lab/data/events/detections.jsonl",
  "per_rule": {
    "SSH_HONEYCREDENTIAL_LOGIN": {
      "attack": {
        "rationale": "The attacker used a credential intentionally planted in deception assets. This indicates exposure to unsecured credential material.\n",
        "tactic": "Credential Access",
        "technique": "Unsecured Credentials",
        "technique_id": "T1552"
      },
      "detections": 1,
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
      "detections": 1,
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
      "detections": 5,
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
      "detections": 2,
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
      "detections": 2,
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
      "detections": 0,
      "engage": {
        "activity": "Expose Decoy Service",
        "goal": "Expose",
        "rationale": "The fake web interface exposes probeable paths to observe scanner behavior.\n"
      },
      "has_attack_mapping": true,
      "has_engage_mapping": true
    }
  },
  "total_detections": 11,
  "triggered_detection_rules": [
    "SSH_HONEYCREDENTIAL_LOGIN",
    "SSH_HONEYFILE_ACCESS",
    "SSH_RECON_COMMAND",
    "WEB_HONEYCREDENTIAL_USED",
    "WEB_HONEYFILE_ACCESS"
  ]
}
lss@lss:/opt/deception-lab $ /opt/deception-lab/scripts/show_mapping.sh
=== Mapping Files ===
-rw-rw-r-- 1 lss lss 2.9K May 21 08:17 attack_coverage.json
-rw-rw-r-- 1 lss lss 2.2K May 21 08:17 engage_coverage.json
-rw-rw-r-- 1 lss lss 6.1K May 21 08:17 mapping_summary.json

=== Mapping Summary ===
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
    "SSH_HONEYCREDENTIAL_LOGIN": 1,
    "SSH_HONEYFILE_ACCESS": 1,
    "SSH_RECON_COMMAND": 5,
    "WEB_HONEYCREDENTIAL_USED": 2,
    "WEB_HONEYFILE_ACCESS": 2
  },
  "engage_mapping_coverage": {
    "coverage_percent": 100.0,
    "mapped_defined_rules": 8,
    "total_defined_rules": 8,
    "triggered_without_mapping": [],
    "unmapped_defined_rules": []
  },
  "generated_at": "2026-05-21T00:17:03.679497+00:00",
  "input_detections": "/opt/deception-lab/data/events/detections.jsonl",
  "per_rule": {
    "SSH_HONEYCREDENTIAL_LOGIN": {
      "attack": {
        "rationale": "The attacker used a credential intentionally planted in deception assets. This indicates exposure to unsecured credential material.\n",
        "tactic": "Credential Access",
        "technique": "Unsecured Credentials",
        "technique_id": "T1552"
      },
      "detections": 1,
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
      "detections": 1,
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
      "detections": 5,
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
      "detections": 2,
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
      "detections": 2,
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
      "detections": 0,
      "engage": {
        "activity": "Expose Decoy Service",
        "goal": "Expose",
        "rationale": "The fake web interface exposes probeable paths to observe scanner behavior.\n"
      },
      "has_attack_mapping": true,
      "has_engage_mapping": true
    }
  },
  "total_detections": 11,
  "triggered_detection_rules": [
    "SSH_HONEYCREDENTIAL_LOGIN",
    "SSH_HONEYFILE_ACCESS",
    "SSH_RECON_COMMAND",
    "WEB_HONEYCREDENTIAL_USED",
    "WEB_HONEYFILE_ACCESS"
  ]
}

=== ATT&CK Detection Coverage ===
--- Tactics ---
{
  "Collection": 3,
  "Credential Access": 3,
  "Discovery": 5
}

--- Techniques ---
{
  "T1005 Data from Local System": 3,
  "T1082 System Information Discovery": 5,
  "T1552 Unsecured Credentials": 3
}

=== Engage Detection Coverage ===
--- Goals ---
{
  "Elicit": 4,
  "Understand": 7
}

--- Activities ---
{
  "Elicit: Credential Collection": 3,
  "Elicit: Reveal Adversary Intent": 1,
  "Understand: Collect Adversary Behavior": 7
}
lss@lss:/opt/deception-lab $ ls -lah /opt/deception-lab/reports
total 12K
drwxrwxr-x 2 lss lss 4.0K May 21 08:07 .
drwxr-xr-x 8 lss lss 4.0K May 21 08:15 ..
-rw-rw-r-- 1 lss lss 2.0K May 21 08:17 mapping_report.md
lss@lss:/opt/deception-lab $ head -n 80 /opt/deception-lab/reports/mapping_report.md
# MITRE Mapping Report

Generated at: 2026-05-21T00:17:03.716150+00:00

## Mapping Coverage

| Framework | Mapped Rules | Total Rules | Coverage |
| --- | --- | --- | --- |
| MITRE ATT&CK | 8 | 8 | 100.0% |
| MITRE Engage | 8 | 8 | 100.0% |

## Detection Counts by Rule

| Rule ID | Count |
| --- | --- |
| SSH_HONEYCREDENTIAL_LOGIN | 1 |
| SSH_HONEYFILE_ACCESS | 1 |
| SSH_RECON_COMMAND | 5 |
| WEB_HONEYCREDENTIAL_USED | 2 |
| WEB_HONEYFILE_ACCESS | 2 |

## ATT&CK Coverage by Tactic

| Tactic | Detection Count |
| --- | --- |
| Collection | 3 |
| Credential Access | 3 |
| Discovery | 5 |

## ATT&CK Coverage by Technique

| Technique | Detection Count |
| --- | --- |
| T1005 Data from Local System | 3 |
| T1082 System Information Discovery | 5 |
| T1552 Unsecured Credentials | 3 |

## MITRE Engage Coverage by Goal

| Goal | Detection Count |
| --- | --- |
| Elicit | 4 |
| Understand | 7 |

## MITRE Engage Coverage by Activity

| Activity | Detection Count |
| --- | --- |
| Elicit: Credential Collection | 3 |
| Elicit: Reveal Adversary Intent | 1 |
| Understand: Collect Adversary Behavior | 7 |

## Per-rule Mapping

| Rule ID | Detections | ATT&CK ID | ATT&CK Technique | Engage Goal | Engage Activity |
| --- | --- | --- | --- | --- | --- |
| SSH_HONEYCREDENTIAL_LOGIN | 1 | T1552 | Unsecured Credentials | Elicit | Credential Collection |
| SSH_HONEYFILE_ACCESS | 1 | T1005 | Data from Local System | Elicit | Reveal Adversary Intent |
| SSH_LOGIN_FAILED | 0 | T1110 | Brute Force | Expose | Expose Decoy Service |
| SSH_RECON_COMMAND | 5 | T1082 | System Information Discovery | Understand | Collect Adversary Behavior |
| SSH_TOOL_TRANSFER_COMMAND | 0 | T1105 | Ingress Tool Transfer | Affect | Adversary Direction |
| WEB_HONEYCREDENTIAL_USED | 2 | T1552 | Unsecured Credentials | Elicit | Credential Collection |
| WEB_HONEYFILE_ACCESS | 2 | T1005 | Data from Local System | Understand | Collect Adversary Behavior |
| WEB_SCANNER_PROBE | 0 | T1595 | Active Scanning | Expose | Expose Decoy Service |
lss@lss:/opt/deception-lab $ cat /opt/deception-lab/PHASE9_READY.md
# Phase 9 Ready

MITRE ATT&CK and MITRE Engage mapping have been implemented.

Completed items:

- attack_mapping.yml created
- engage_mapping.yml created
- analyze_mapping.py created
- run_mapping.sh created
- show_mapping.sh created
- generate_mapping_markdown.py created
- mapping_summary.json generated
- attack_coverage.json generated
- engage_coverage.json generated
- mapping_report.md generated

Main mapping rule files:

- /opt/deception-lab/parser/rules/attack_mapping.yml
- /opt/deception-lab/parser/rules/engage_mapping.yml

Main mapping outputs:

- /opt/deception-lab/data/events/mapping_summary.json
- /opt/deception-lab/data/events/attack_coverage.json
- /opt/deception-lab/data/events/engage_coverage.json
- /opt/deception-lab/reports/mapping_report.md

Next phase:

Phase 10 - Event timeline and Markdown / JSON report generation.
```

## 確認：第九階段已成功完成。
你的輸出已經完整通過第九階段檢查：
```
| 項目                      |            結果 | 狀態 |
| ----------------------- | ------------: | -- |
| `run_mapping.sh`        |          成功執行 | ✅  |
| parser 事件數              |     34 events | ✅  |
| detection 數             | 11 detections | ✅  |
| detection rules         |       8 rules | ✅  |
| ATT&CK mapping coverage |        100.0% | ✅  |
| Engage mapping coverage |        100.0% | ✅  |
| `mapping_summary.json`  |           已產生 | ✅  |
| `attack_coverage.json`  |           已產生 | ✅  |
| `engage_coverage.json`  |           已產生 | ✅  |
| `mapping_report.md`     |           已產生 | ✅  |
| `PHASE9_READY.md`       |           已建立 | ✅  |

你的 run_mapping.sh 成功刷新 log、執行 parser、產生 mapping summary，並顯示 ATT&CK 與 Engage coverage 都是 100.0%。
```

## 第九階段成果確認
```
# 你目前已產生以下 mapping 輸出：
/opt/deception-lab/data/events/mapping_summary.json
/opt/deception-lab/data/events/attack_coverage.json
/opt/deception-lab/data/events/engage_coverage.json
/opt/deception-lab/reports/mapping_report.md

# ATT&CK 偵測覆蓋目前包含：
Credential Access: 3
Collection: 3
Discovery: 5

# 對應技術包含：
T1552 Unsecured Credentials
T1005 Data from Local System
T1082 System Information Discovery

# Engage 目標目前包含：
Elicit: 4
Understand: 7

而 mapping_report.md 也已經成功列出每條 detection rule 對應的 ATT&CK technique 與 Engage goal，例如 SSH_HONEYCREDENTIAL_LOGIN 對應 T1552 Unsecured Credentials 與 Elicit / Credential Collection。
```







