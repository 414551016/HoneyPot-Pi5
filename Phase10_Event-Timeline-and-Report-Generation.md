# 第十階段：事件時間線與報告產生
```
完成後，你會有：
[完成] timeline.md
[完成] report.md
[完成] report.json
[完成] generate_report.py
[完成] generate_timeline.py
[完成] generate_report.sh
[完成] show_report.sh
[完成] PHASE10_READY.md
```

### Step 10.1：進入專案目錄
```
cd /opt/deception-lab
```

### Step 10.2：確認第八、九階段輸出檔案存在
- 先確認必要檔案都存在：
```
ls -lah /opt/deception-lab/data/events

# 你應該看到：
events.jsonl
detections.jsonl
events_summary.json
mapping_summary.json
attack_coverage.json
engage_coverage.json

# 如果少了 mapping 相關檔案，先執行：
/opt/deception-lab/scripts/run_mapping.sh
輸出顯示：
Total events: 34
Total detections: 11
ATT&CK mapping coverage: 100.0%
Engage mapping coverage: 100.0%
這代表第八階段與第九階段的資料都已經刷新並可供第十階段報告產生器使用。

# 執行結果：
確認：Step 10.2 已完成，而且狀態正確。
你執行的結果顯示 /opt/deception-lab/data/events 內已經存在第八、九階段需要的所有輸出檔案：
/opt/deception-lab/data/events
├── attack_coverage.json
├── detections.jsonl
├── engage_coverage.json
├── events.jsonl
├── events_summary.json
└── mapping_summary.json

這正好對應 Step 10.2 要確認的檔案：

檔案	來源階段	狀態
events.jsonl	第八階段 parser	✅ 存在
detections.jsonl	第八階段 detection rules	✅ 存在
events_summary.json	第八階段 summary	✅ 存在
mapping_summary.json	第九階段 mapping analyzer	✅ 存在
attack_coverage.json	第九階段 ATT&CK coverage	✅ 存在
engage_coverage.json	第九階段 Engage coverage	✅ 存在
```

### Step 10.3：建立 timeline 產生器
- 執行：
```
cat > /opt/deception-lab/parser/generate_timeline.py <<'EOF'
#!/usr/bin/env python3
import json
from collections import defaultdict
from datetime import datetime
from pathlib import Path


ROOT = Path("/opt/deception-lab")
EVENT_DIR = ROOT / "data" / "events"
REPORT_DIR = ROOT / "reports"

EVENTS_JSONL = EVENT_DIR / "events.jsonl"
DETECTIONS_JSONL = EVENT_DIR / "detections.jsonl"
OUTPUT = REPORT_DIR / "timeline.md"


def load_jsonl(path):
    records = []
    if not path.exists():
        return records

    with open(path, "r", encoding="utf-8", errors="ignore") as f:
        for line in f:
            line = line.strip()
            if not line:
                continue
            try:
                records.append(json.loads(line))
            except json.JSONDecodeError:
                continue

    return records


def parse_time(value):
    if not value:
        return None
    try:
        return datetime.fromisoformat(value.replace("Z", "+00:00"))
    except ValueError:
        return None


def short_event(event):
    event_type = event.get("event_type", "unknown")
    source = event.get("source", "unknown")
    src_ip = event.get("src_ip") or "-"
    username = event.get("username") or "-"
    command = event.get("command")
    path = event.get("path")
    filename = event.get("filename")

    details = []

    if username != "-":
        details.append(f"username=`{username}`")
    if command:
        details.append(f"command=`{command}`")
    if path:
        details.append(f"path=`{path}`")
    if filename:
        details.append(f"filename=`{filename}`")

    detail_text = "; ".join(details) if details else "no additional details"

    return f"**{event_type}** from `{source}` src_ip=`{src_ip}`; {detail_text}"


def main():
    REPORT_DIR.mkdir(parents=True, exist_ok=True)

    events = load_jsonl(EVENTS_JSONL)
    detections = load_jsonl(DETECTIONS_JSONL)

    detection_by_event_index = defaultdict(list)
    for detection in detections:
        idx = detection.get("event_index")
        if idx is not None:
            detection_by_event_index[idx].append(detection)

    rows = []
    for idx, event in enumerate(events):
        dt = parse_time(event.get("timestamp"))
        if not dt:
            continue
        rows.append((dt, idx, event))

    rows.sort(key=lambda x: x[0])

    lines = []
    lines.append("# Deception Lab Event Timeline")
    lines.append("")
    lines.append(f"Generated at: {datetime.utcnow().isoformat()}Z")
    lines.append("")
    lines.append("## Summary")
    lines.append("")
    lines.append(f"- Total events: {len(events)}")
    lines.append(f"- Total detections: {len(detections)}")
    lines.append("")

    current_date = None

    for dt, idx, event in rows:
        date_str = dt.date().isoformat()
        time_str = dt.time().replace(microsecond=0).isoformat()

        if date_str != current_date:
            current_date = date_str
            lines.append(f"## {date_str}")
            lines.append("")

        lines.append(f"### {time_str} UTC")
        lines.append("")
        lines.append(f"- Event: {short_event(event)}")
        lines.append(f"- Severity: `{event.get('severity', 'info')}`")
        lines.append(f"- Tags: `{', '.join(event.get('tags', []))}`")

        related_detections = detection_by_event_index.get(idx, [])
        if related_detections:
            lines.append("- Detections:")
            for d in related_detections:
                attack = d.get("attack_mapping", {})
                engage = d.get("engage_mapping", {})
                lines.append(
                    f"  - `{d.get('rule_id')}` / {d.get('rule_name')} "
                    f"/ severity=`{d.get('severity')}`"
                )
                if attack:
                    lines.append(
                        f"    - ATT&CK: `{attack.get('technique_id')}` "
                        f"{attack.get('technique')} / tactic=`{attack.get('tactic')}`"
                    )
                if engage:
                    lines.append(
                        f"    - Engage: goal=`{engage.get('goal')}` "
                        f"activity=`{engage.get('activity')}`"
                    )
        else:
            lines.append("- Detections: none")

        lines.append("")

    OUTPUT.write_text("\n".join(lines), encoding="utf-8")
    print(f"[+] Timeline written to: {OUTPUT}")


if __name__ == "__main__":
    main()
EOF
```
- 這支程式會讀取：
```
events.jsonl
detections.jsonl

# 然後產生：
reports/timeline.md
```
- 設定可執行：
```
chmod +x /opt/deception-lab/parser/generate_timeline.py
```

### Step 10.4：建立正式報告產生器
- 執行：
```
cat > /opt/deception-lab/parser/generate_report.py <<'EOF'
#!/usr/bin/env python3
import json
from collections import Counter
from datetime import datetime, timezone
from pathlib import Path

import yaml


ROOT = Path("/opt/deception-lab")
EVENT_DIR = ROOT / "data" / "events"
REPORT_DIR = ROOT / "reports"

EVENTS_JSONL = EVENT_DIR / "events.jsonl"
DETECTIONS_JSONL = EVENT_DIR / "detections.jsonl"
EVENTS_SUMMARY = EVENT_DIR / "events_summary.json"
MAPPING_SUMMARY = EVENT_DIR / "mapping_summary.json"
ATTACK_COVERAGE = EVENT_DIR / "attack_coverage.json"
ENGAGE_COVERAGE = EVENT_DIR / "engage_coverage.json"
DECEPTION_ASSETS = ROOT / "deception_assets.yml"

REPORT_MD = REPORT_DIR / "report.md"
REPORT_JSON = REPORT_DIR / "report.json"


def load_json(path):
    if not path.exists():
        return {}
    return json.loads(path.read_text(encoding="utf-8"))


def load_yaml(path):
    if not path.exists():
        return {}
    return yaml.safe_load(path.read_text(encoding="utf-8")) or {}


def load_jsonl(path):
    records = []
    if not path.exists():
        return records

    with open(path, "r", encoding="utf-8", errors="ignore") as f:
        for line in f:
            line = line.strip()
            if not line:
                continue
            try:
                records.append(json.loads(line))
            except json.JSONDecodeError:
                continue

    return records


def md_table(headers, rows):
    lines = []
    lines.append("| " + " | ".join(headers) + " |")
    lines.append("| " + " | ".join(["---"] * len(headers)) + " |")
    for row in rows:
        lines.append("| " + " | ".join(str(x) for x in row) + " |")
    return "\n".join(lines)


def top_detections(detections, limit=10):
    severity_rank = {
        "critical": 4,
        "high": 3,
        "medium": 2,
        "low": 1,
        "info": 0
    }

    return sorted(
        detections,
        key=lambda d: (
            severity_rank.get(d.get("severity", "info"), 0),
            d.get("timestamp", "")
        ),
        reverse=True
    )[:limit]


def main():
    REPORT_DIR.mkdir(parents=True, exist_ok=True)

    events = load_jsonl(EVENTS_JSONL)
    detections = load_jsonl(DETECTIONS_JSONL)

    events_summary = load_json(EVENTS_SUMMARY)
    mapping_summary = load_json(MAPPING_SUMMARY)
    attack_coverage = load_json(ATTACK_COVERAGE)
    engage_coverage = load_json(ENGAGE_COVERAGE)
    assets = load_yaml(DECEPTION_ASSETS)

    generated_at = datetime.now(timezone.utc).isoformat()

    detection_by_rule = Counter(d.get("rule_id") for d in detections)
    detection_by_severity = Counter(d.get("severity") for d in detections)
    events_by_source = Counter(e.get("source") for e in events)
    events_by_type = Counter(e.get("event_type") for e in events)

    honeycredential_count = sum(
        1 for d in detections
        if d.get("rule_id") in ["SSH_HONEYCREDENTIAL_LOGIN", "WEB_HONEYCREDENTIAL_USED"]
    )

    honeyfile_count = sum(
        1 for d in detections
        if d.get("rule_id") in ["SSH_HONEYFILE_ACCESS", "WEB_HONEYFILE_ACCESS"]
    )

    report_obj = {
        "generated_at": generated_at,
        "project": {
            "name": "Raspberry Pi Deception Lab MVP",
            "root": str(ROOT),
            "host": assets.get("lab", {}).get("host", "unknown"),
            "owner": assets.get("lab", {}).get("owner", "unknown")
        },
        "summary": {
            "total_events": len(events),
            "total_detections": len(detections),
            "events_by_source": dict(events_by_source),
            "events_by_type": dict(events_by_type),
            "detections_by_rule": dict(detection_by_rule),
            "detections_by_severity": dict(detection_by_severity),
            "honeycredential_detections": honeycredential_count,
            "honeyfile_detections": honeyfile_count
        },
        "mapping": {
            "attack_mapping_coverage": mapping_summary.get("attack_mapping_coverage", {}),
            "engage_mapping_coverage": mapping_summary.get("engage_mapping_coverage", {}),
            "attack_coverage": {
                "detections_by_tactic": attack_coverage.get("detections_by_tactic", {}),
                "detections_by_technique": attack_coverage.get("detections_by_technique", {})
            },
            "engage_coverage": {
                "detections_by_goal": engage_coverage.get("detections_by_goal", {}),
                "detections_by_activity": engage_coverage.get("detections_by_activity", {})
            }
        },
        "top_detections": top_detections(detections, limit=10),
        "deception_assets": {
            "honeycredentials": [
                {
                    "id": x.get("id"),
                    "username": x.get("username"),
                    "role": x.get("role"),
                    "severity": x.get("severity")
                }
                for x in assets.get("honeycredentials", [])
            ],
            "honeyfiles": [
                {
                    "id": x.get("id"),
                    "filename": x.get("filename"),
                    "severity": x.get("severity"),
                    "description": x.get("description")
                }
                for x in assets.get("honeyfiles", [])
            ]
        }
    }

    REPORT_JSON.write_text(
        json.dumps(report_obj, ensure_ascii=False, indent=2, sort_keys=True),
        encoding="utf-8"
    )

    lines = []
    lines.append("# Raspberry Pi Deception Lab MVP Report")
    lines.append("")
    lines.append(f"Generated at: `{generated_at}`")
    lines.append("")
    lines.append("## 1. Executive Summary")
    lines.append("")
    lines.append("This report summarizes events collected from the Raspberry Pi deception lab MVP.")
    lines.append("The lab includes a Cowrie SSH honeypot, a fake web admin panel, honeycredentials, honeyfiles, centralized log collection, detection rules, and MITRE mapping.")
    lines.append("")
    lines.append(md_table(
        ["Metric", "Value"],
        [
            ["Total events", len(events)],
            ["Total detections", len(detections)],
            ["Honeycredential detections", honeycredential_count],
            ["Honeyfile detections", honeyfile_count],
            ["ATT&CK mapping coverage", f"{mapping_summary.get('attack_mapping_coverage', {}).get('coverage_percent', 0)}%"],
            ["Engage mapping coverage", f"{mapping_summary.get('engage_mapping_coverage', {}).get('coverage_percent', 0)}%"],
        ]
    ))
    lines.append("")

    lines.append("## 2. Event Summary")
    lines.append("")
    lines.append("### Events by Source")
    lines.append("")
    lines.append(md_table(["Source", "Count"], sorted(events_by_source.items())))
    lines.append("")
    lines.append("### Events by Type")
    lines.append("")
    lines.append(md_table(["Event Type", "Count"], sorted(events_by_type.items())))
    lines.append("")

    lines.append("## 3. Detection Summary")
    lines.append("")
    lines.append("### Detections by Severity")
    lines.append("")
    lines.append(md_table(["Severity", "Count"], sorted(detection_by_severity.items())))
    lines.append("")
    lines.append("### Detections by Rule")
    lines.append("")
    lines.append(md_table(["Rule ID", "Count"], sorted(detection_by_rule.items())))
    lines.append("")

    lines.append("## 4. Top Detections")
    lines.append("")
    top_rows = []
    for d in top_detections(detections, limit=10):
        attack = d.get("attack_mapping", {})
        engage = d.get("engage_mapping", {})
        top_rows.append([
            d.get("timestamp", ""),
            d.get("severity", ""),
            d.get("rule_id", ""),
            d.get("source", ""),
            d.get("src_ip", ""),
            d.get("username") or "",
            d.get("command") or d.get("path") or d.get("filename") or "",
            f"{attack.get('technique_id', '')} {attack.get('technique', '')}",
            f"{engage.get('goal', '')}: {engage.get('activity', '')}"
        ])

    lines.append(md_table(
        ["Timestamp", "Severity", "Rule", "Source", "Src IP", "Username", "Observed Behavior", "ATT&CK", "Engage"],
        top_rows
    ))
    lines.append("")

    lines.append("## 5. MITRE ATT&CK Coverage")
    lines.append("")
    lines.append("### By Tactic")
    lines.append("")
    lines.append(md_table(
        ["Tactic", "Detection Count"],
        sorted(attack_coverage.get("detections_by_tactic", {}).items())
    ))
    lines.append("")
    lines.append("### By Technique")
    lines.append("")
    lines.append(md_table(
        ["Technique", "Detection Count"],
        sorted(attack_coverage.get("detections_by_technique", {}).items())
    ))
    lines.append("")

    lines.append("## 6. MITRE Engage Coverage")
    lines.append("")
    lines.append("### By Goal")
    lines.append("")
    lines.append(md_table(
        ["Goal", "Detection Count"],
        sorted(engage_coverage.get("detections_by_goal", {}).items())
    ))
    lines.append("")
    lines.append("### By Activity")
    lines.append("")
    lines.append(md_table(
        ["Activity", "Detection Count"],
        sorted(engage_coverage.get("detections_by_activity", {}).items())
    ))
    lines.append("")

    lines.append("## 7. Deception Assets")
    lines.append("")
    lines.append("### Honeycredentials")
    lines.append("")
    credential_rows = []
    for item in assets.get("honeycredentials", []):
        credential_rows.append([
            item.get("id", ""),
            item.get("username", ""),
            item.get("role", ""),
            item.get("severity", "")
        ])
    lines.append(md_table(["ID", "Username", "Role", "Severity"], credential_rows))
    lines.append("")
    lines.append("### Honeyfiles")
    lines.append("")
    file_rows = []
    for item in assets.get("honeyfiles", []):
        file_rows.append([
            item.get("id", ""),
            item.get("filename", ""),
            item.get("severity", ""),
            item.get("description", "")
        ])
    lines.append(md_table(["ID", "Filename", "Severity", "Description"], file_rows))
    lines.append("")

    lines.append("## 8. Generated Files")
    lines.append("")
    lines.append(md_table(
        ["File", "Purpose"],
        [
            ["/opt/deception-lab/data/events/events.jsonl", "Normalized events"],
            ["/opt/deception-lab/data/events/detections.jsonl", "Detection results"],
            ["/opt/deception-lab/data/events/events_summary.json", "Event summary"],
            ["/opt/deception-lab/data/events/mapping_summary.json", "Mapping coverage summary"],
            ["/opt/deception-lab/reports/timeline.md", "Human-readable event timeline"],
            ["/opt/deception-lab/reports/report.md", "Main Markdown report"],
            ["/opt/deception-lab/reports/report.json", "Machine-readable report"]
        ]
    ))
    lines.append("")

    lines.append("## 9. Safety Notes")
    lines.append("")
    lines.append("- This MVP should remain isolated from real internal networks.")
    lines.append("- All credentials and files used by the platform must remain fake.")
    lines.append("- Do not expose the lab directly to the Internet during MVP testing.")
    lines.append("- Do not mount Docker socket into honeypot containers.")
    lines.append("- Do not use privileged containers for honeypot services.")
    lines.append("")

    REPORT_MD.write_text("\n".join(lines), encoding="utf-8")

    print(f"[+] Markdown report written to: {REPORT_MD}")
    print(f"[+] JSON report written to: {REPORT_JSON}")


if __name__ == "__main__":
    main()
EOF
```
- 設定可執行：
```
chmod +x /opt/deception-lab/parser/generate_report.py
```
- 這支程式會產生：
```
report.md
report.json
```

### Step 10.5：建立報告產生腳本
- 執行：
```
cat > /opt/deception-lab/scripts/generate_report.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

PROJECT_ROOT="/opt/deception-lab"
PARSER_VENV="$PROJECT_ROOT/parser/venv"

echo "[+] Refreshing parser and mapping outputs..."
"$PROJECT_ROOT/scripts/run_mapping.sh"

echo
echo "[+] Generating timeline and final reports..."
source "$PARSER_VENV/bin/activate"
python "$PROJECT_ROOT/parser/generate_timeline.py"
python "$PROJECT_ROOT/parser/generate_report.py"
deactivate

echo
echo "[+] Report files:"
ls -lah "$PROJECT_ROOT/reports"

echo
echo "[+] Report generation completed."
echo "Main report: $PROJECT_ROOT/reports/report.md"
echo "Timeline:    $PROJECT_ROOT/reports/timeline.md"
echo "JSON report: $PROJECT_ROOT/reports/report.json"
EOF
```
- 設定可執行：
```
chmod +x /opt/deception-lab/scripts/generate_report.sh
```
- 這個腳本會依序執行：
```
run_mapping.sh
generate_timeline.py
generate_report.py
```

### Step 10.6：執行報告產生
- 執行：
```
/opt/deception-lab/scripts/generate_report.sh
```
- 成功後應該看到：
```
[+] Timeline written to: /opt/deception-lab/reports/timeline.md
[+] Markdown report written to: /opt/deception-lab/reports/report.md
[+] JSON report written to: /opt/deception-lab/reports/report.json

# 執行結果：
lss@lss:/opt/deception-lab $ chmod +x /opt/deception-lab/scripts/generate_report.sh
lss@lss:/opt/deception-lab $ /opt/deception-lab/scripts/generate_report.sh
[+] Refreshing parser and mapping outputs...
[+] Running parser first to refresh detections...
[+] Collecting latest logs...
[+] Collecting logs at UTC time: 20260521T191112Z
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
-rw-rw-r-- 1 lss lss  370 May 22 03:11 collection_summary.txt
-rw-rw-r-- 1 lss lss  12K May 22 03:11 cowrie-docker.log
-rw-rw-r-- 1 lss lss  579 May 15 07:18 README.md
-rw-rw-r-- 1 lss lss  777 May 22 03:11 source_manifest.json
-rwxrwxr-x 1 lss lss 6.5K May 22 03:11 web_access.jsonl
-rwxrwxr-x 1 lss lss 1.7K May 22 03:11 web_auth.jsonl

[+] Summary:
Collection time UTC: 20260521T191112Z

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
-rw-rw-r-- 1 lss lss 2.9K May 21 11:43 attack_coverage.json
-rw-rw-r-- 1 lss lss 6.9K May 22 03:11 detections.jsonl
-rw-rw-r-- 1 lss lss 2.2K May 21 11:43 engage_coverage.json
-rw-rw-r-- 1 lss lss  13K May 22 03:11 events.jsonl
-rw-rw-r-- 1 lss lss  631 May 22 03:11 events_summary.json
-rw-rw-r-- 1 lss lss 6.1K May 21 11:43 mapping_summary.json

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
  "generated_at": "2026-05-21T19:11:12.658610+00:00",
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
-rw-rw-r-- 1 lss lss 2.9K May 22 03:11 attack_coverage.json
-rw-rw-r-- 1 lss lss 2.2K May 22 03:11 engage_coverage.json
-rw-rw-r-- 1 lss lss 6.1K May 22 03:11 mapping_summary.json

[+] Mapping report:
-rw-rw-r-- 1 lss lss 2.0K May 22 03:11 /opt/deception-lab/reports/mapping_report.md

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
  "generated_at": "2026-05-21T19:11:12.734918+00:00",
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

[+] Generating timeline and final reports...
/opt/deception-lab/parser/generate_timeline.py:93: DeprecationWarning: datetime.datetime.utcnow() is deprecated and scheduled for removal in a future version. Use timezone-aware objects to represent datetimes in UTC: datetime.datetime.now(datetime.UTC).
  lines.append(f"Generated at: {datetime.utcnow().isoformat()}Z")
[+] Timeline written to: /opt/deception-lab/reports/timeline.md
[+] Markdown report written to: /opt/deception-lab/reports/report.md
[+] JSON report written to: /opt/deception-lab/reports/report.json

[+] Report files:
total 40K
drwxrwxr-x 2 lss lss 4.0K May 22 03:11 .
drwxr-xr-x 8 lss lss 4.0K May 21 08:15 ..
-rw-rw-r-- 1 lss lss 2.0K May 22 03:11 mapping_report.md
-rw-rw-r-- 1 lss lss  12K May 22 03:11 report.json
-rw-rw-r-- 1 lss lss 5.4K May 22 03:11 report.md
-rw-rw-r-- 1 lss lss 8.0K May 22 03:11 timeline.md

[+] Report generation completed.
Main report: /opt/deception-lab/reports/report.md
Timeline:    /opt/deception-lab/reports/timeline.md
JSON report: /opt/deception-lab/reports/report.json
```

### Step 10.7：確認報告檔案
- 請執行：
```
ls -lah /opt/deception-lab/reports

# 你應該看到：
mapping_report.md
timeline.md
report.md
report.json

## 執行結果：
lss@lss:/opt/deception-lab $ ls -lah /opt/deception-lab/reports
total 40K
drwxrwxr-x 2 lss lss 4.0K May 22 03:11 .
drwxr-xr-x 8 lss lss 4.0K May 21 08:15 ..
-rw-rw-r-- 1 lss lss 2.0K May 22 03:11 mapping_report.md
-rw-rw-r-- 1 lss lss  12K May 22 03:11 report.json
-rw-rw-r-- 1 lss lss 5.4K May 22 03:11 report.md
-rw-rw-r-- 1 lss lss 8.0K May 22 03:11 timeline.md
```

### Step 10.8：查看 timeline.md
- 請執行：
```
head -n 120 /opt/deception-lab/reports/timeline.md

# 你應該會看到：
# Deception Lab Event Timeline
Generated at: ...
## Summary
- Total events: 34
- Total detections: 11
## 2026-...

# 執行結果：
lss@lss:/opt/deception-lab $ head -n 120 /opt/deception-lab/reports/timeline.md
# Deception Lab Event Timeline

Generated at: 2026-05-21T19:11:12.809837Z

## Summary

- Total events: 34
- Total detections: 11

## 2026-05-11

### 21:42:08 UTC

- Event: **web_request** from `fake-web` src_ip=`172.18.0.1`; path=`/login`
- Severity: `info`
- Tags: `web, request`
- Detections: none

### 21:42:34 UTC

- Event: **web_request** from `fake-web` src_ip=`172.18.0.1`; path=`/api/status`
- Severity: `info`
- Tags: `web, request`
- Detections: none

### 21:46:40 UTC

- Event: **web_request** from `fake-web` src_ip=`192.168.1.1`; path=`/`
- Severity: `info`
- Tags: `web, request`
- Detections: none

### 21:46:40 UTC

- Event: **web_request** from `fake-web` src_ip=`192.168.1.1`; path=`/login`
- Severity: `info`
- Tags: `web, request`
- Detections: none

### 21:46:40 UTC

- Event: **web_request** from `fake-web` src_ip=`192.168.1.1`; path=`/static/style.css`
- Severity: `info`
- Tags: `web, request`
- Detections: none

### 21:46:40 UTC

- Event: **web_request** from `fake-web` src_ip=`192.168.1.1`; path=`/favicon.ico`
- Severity: `info`
- Tags: `web, request`
- Detections: none

### 21:46:56 UTC

- Event: **web_request** from `fake-web` src_ip=`192.168.1.1`; path=`/login`
- Severity: `info`
- Tags: `web, request`
- Detections: none

### 21:46:56 UTC

- Event: **web_login_attempt** from `fake-web` src_ip=`192.168.1.1`; username=`admin`
- Severity: `medium`
- Tags: `web, login`
- Detections: none

### 21:46:56 UTC

- Event: **web_request** from `fake-web` src_ip=`192.168.1.1`; path=`/static/style.css`
- Severity: `info`
- Tags: `web, request`
- Detections: none

### 21:57:09 UTC

- Event: **web_request** from `fake-web` src_ip=`192.168.1.1`; path=`/backup`
- Severity: `info`
- Tags: `web, request`
- Detections: none

### 21:57:09 UTC

- Event: **web_request** from `fake-web` src_ip=`192.168.1.1`; path=`/static/style.css`
- Severity: `info`
- Tags: `web, request`
- Detections: none

### 21:57:12 UTC

- Event: **web_request** from `fake-web` src_ip=`192.168.1.1`; path=`/download/secrets.txt`
- Severity: `info`
- Tags: `web, request`
- Detections: none

### 21:57:12 UTC

- Event: **web_honeyfile_access** from `fake-web` src_ip=`192.168.1.1`; path=`/download/secrets.txt`; filename=`secrets.txt`
- Severity: `high`
- Tags: `web, honeyfile, download`
- Detections:
  - `WEB_HONEYFILE_ACCESS` / Web honeyfile accessed / severity=`high`
    - ATT&CK: `T1005` Data from Local System / tactic=`Collection`
    - Engage: goal=`Understand` activity=`Collect Adversary Behavior`

### 21:57:19 UTC

- Event: **web_request** from `fake-web` src_ip=`192.168.1.1`; path=`/download/vpn_users.csv`
- Severity: `info`
- Tags: `web, request`
- Detections: none

### 21:57:19 UTC

- Event: **web_honeyfile_access** from `fake-web` src_ip=`192.168.1.1`; path=`/download/vpn_users.csv`; filename=`vpn_users.csv`
- Severity: `high`
- Tags: `web, honeyfile, download`
- Detections:
  - `WEB_HONEYFILE_ACCESS` / Web honeyfile accessed / severity=`high`
    - ATT&CK: `T1005` Data from Local System / tactic=`Collection`
```

### Step 10.9：查看 report.md
- 請執行：
```
head -n 160 /opt/deception-lab/reports/report.md

#你應該會看到：
# Raspberry Pi Deception Lab MVP Report

## 1. Executive Summary
...
Total events
Total detections
Honeycredential detections
Honeyfile detections
ATT&CK mapping coverage
Engage mapping coverage

# 執行結果：
lss@lss:/opt/deception-lab $ head -n 160 /opt/deception-lab/reports/report.md
# Raspberry Pi Deception Lab MVP Report

Generated at: `2026-05-21T19:11:12.864557+00:00`

## 1. Executive Summary

This report summarizes events collected from the Raspberry Pi deception lab MVP.
The lab includes a Cowrie SSH honeypot, a fake web admin panel, honeycredentials, honeyfiles, centralized log collection, detection rules, and MITRE mapping.

| Metric | Value |
| --- | --- |
| Total events | 34 |
| Total detections | 11 |
| Honeycredential detections | 3 |
| Honeyfile detections | 3 |
| ATT&CK mapping coverage | 100.0% |
| Engage mapping coverage | 100.0% |

## 2. Event Summary

### Events by Source

| Source | Count |
| --- | --- |
| cowrie | 11 |
| fake-web | 23 |

### Events by Type

| Event Type | Count |
| --- | --- |
| ssh_command | 8 |
| ssh_connection | 1 |
| ssh_login_success | 1 |
| ssh_logout | 1 |
| web_honeyfile_access | 2 |
| web_login_attempt | 4 |
| web_request | 17 |

## 3. Detection Summary

### Detections by Severity

| Severity | Count |
| --- | --- |
| high | 6 |
| medium | 5 |

### Detections by Rule

| Rule ID | Count |
| --- | --- |
| SSH_HONEYCREDENTIAL_LOGIN | 1 |
| SSH_HONEYFILE_ACCESS | 1 |
| SSH_RECON_COMMAND | 5 |
| WEB_HONEYCREDENTIAL_USED | 2 |
| WEB_HONEYFILE_ACCESS | 2 |

## 4. Top Detections

| Timestamp | Severity | Rule | Source | Src IP | Username | Observed Behavior | ATT&CK | Engage |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 2026-05-12T18:41:52.679019+00:00 | high | WEB_HONEYCREDENTIAL_USED | fake-web | 172.18.0.1 | backup |  | T1552 Unsecured Credentials | Elicit: Credential Collection |
| 2026-05-12T18:36:26.543366+00:00 | high | WEB_HONEYCREDENTIAL_USED | fake-web | 172.18.0.1 | backup |  | T1552 Unsecured Credentials | Elicit: Credential Collection |
| 2026-05-12T18:27:53+00:00 | high | SSH_HONEYFILE_ACCESS | cowrie | 172.18.0.1 |  | cat /home/admin/secrets.txt | T1005 Data from Local System | Elicit: Reveal Adversary Intent |
| 2026-05-12T18:26:42+00:00 | high | SSH_HONEYCREDENTIAL_LOGIN | cowrie | 172.18.0.1 | backup |  | T1552 Unsecured Credentials | Elicit: Credential Collection |
| 2026-05-11T21:57:19.331247+00:00 | high | WEB_HONEYFILE_ACCESS | fake-web | 192.168.1.1 |  | /download/vpn_users.csv | T1005 Data from Local System | Understand: Collect Adversary Behavior |
| 2026-05-11T21:57:12.514844+00:00 | high | WEB_HONEYFILE_ACCESS | fake-web | 192.168.1.1 |  | /download/secrets.txt | T1005 Data from Local System | Understand: Collect Adversary Behavior |
| 2026-05-12T18:27:46+00:00 | medium | SSH_RECON_COMMAND | cowrie | 172.18.0.1 |  | ls /home/admin | T1082 System Information Discovery | Understand: Collect Adversary Behavior |
| 2026-05-12T18:27:30+00:00 | medium | SSH_RECON_COMMAND | cowrie | 172.18.0.1 |  | ls /home/admin | T1082 System Information Discovery | Understand: Collect Adversary Behavior |
| 2026-05-12T18:27:16+00:00 | medium | SSH_RECON_COMMAND | cowrie | 172.18.0.1 |  | ls | T1082 System Information Discovery | Understand: Collect Adversary Behavior |
| 2026-05-12T18:27:11+00:00 | medium | SSH_RECON_COMMAND | cowrie | 172.18.0.1 |  | pwd | T1082 System Information Discovery | Understand: Collect Adversary Behavior |

## 5. MITRE ATT&CK Coverage

### By Tactic

| Tactic | Detection Count |
| --- | --- |
| Collection | 3 |
| Credential Access | 3 |
| Discovery | 5 |

### By Technique

| Technique | Detection Count |
| --- | --- |
| T1005 Data from Local System | 3 |
| T1082 System Information Discovery | 5 |
| T1552 Unsecured Credentials | 3 |

## 6. MITRE Engage Coverage

### By Goal

| Goal | Detection Count |
| --- | --- |
| Elicit | 4 |
| Understand | 7 |

### By Activity

| Activity | Detection Count |
| --- | --- |
| Elicit: Credential Collection | 3 |
| Elicit: Reveal Adversary Intent | 1 |
| Understand: Collect Adversary Behavior | 7 |

## 7. Deception Assets

### Honeycredentials

| ID | Username | Role | Severity |
| --- | --- | --- | --- |
| HC_ADMIN_001 | admin | fake administrator | high |
| HC_BACKUP_001 | backup | fake backup operator | high |
| HC_IOT_001 | iotadmin | fake IoT administrator | high |
| HC_OPERATOR_001 | operator | fake maintenance operator | high |

### Honeyfiles

| ID | Filename | Severity | Description |
| --- | --- | --- | --- |
| HF_SECRET_001 | secrets.txt | high | Fake internal secret note containing honeycredentials. |
| HF_BACKUP_001 | backup_config.ini | high | Fake backup configuration file. |
| HF_VPN_001 | vpn_users.csv | high | Fake VPN user export. |
| HF_SSHKEY_001 | ssh_keys_backup.txt | high | Fake SSH private key backup. |
| HF_DB_001 | database_passwords.txt | high | Fake database credential backup. |

## 8. Generated Files

| File | Purpose |
| --- | --- |
| /opt/deception-lab/data/events/events.jsonl | Normalized events |
| /opt/deception-lab/data/events/detections.jsonl | Detection results |
| /opt/deception-lab/data/events/events_summary.json | Event summary |
| /opt/deception-lab/data/events/mapping_summary.json | Mapping coverage summary |
| /opt/deception-lab/reports/timeline.md | Human-readable event timeline |
| /opt/deception-lab/reports/report.md | Main Markdown report |
| /opt/deception-lab/reports/report.json | Machine-readable report |

## 9. Safety Notes

- This MVP should remain isolated from real internal networks.
- All credentials and files used by the platform must remain fake.
- Do not expose the lab directly to the Internet during MVP testing.
- Do not mount Docker socket into honeypot containers.
- Do not use privileged containers for honeypot services.
```

### Step 10.10：查看 report.json
- 請執行：
```
cat /opt/deception-lab/reports/report.json | jq '.summary'

# 你應該會看到：
{
  "total_events": 34,
  "total_detections": 11,
  ...
}

# 執行結果：
lss@lss:/opt/deception-lab $ cat /opt/deception-lab/reports/report.json | jq '.summary'
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
  "honeycredential_detections": 3,
  "honeyfile_detections": 3,
  "total_detections": 11,
  "total_events": 34
}
```
- 再查看 mapping：
```
cat /opt/deception-lab/reports/report.json | jq '.mapping'

# 執行結果：
lss@lss:/opt/deception-lab $ cat /opt/deception-lab/reports/report.json | jq '.mapping'
{
  "attack_coverage": {
    "detections_by_tactic": {
      "Collection": 3,
      "Credential Access": 3,
      "Discovery": 5
    },
    "detections_by_technique": {
      "T1005 Data from Local System": 3,
      "T1082 System Information Discovery": 5,
      "T1552 Unsecured Credentials": 3
    }
  },
  "attack_mapping_coverage": {
    "coverage_percent": 100.0,
    "mapped_defined_rules": 8,
    "total_defined_rules": 8,
    "triggered_without_mapping": [],
    "unmapped_defined_rules": []
  },
  "engage_coverage": {
    "detections_by_activity": {
      "Elicit: Credential Collection": 3,
      "Elicit: Reveal Adversary Intent": 1,
      "Understand: Collect Adversary Behavior": 7
    },
    "detections_by_goal": {
      "Elicit": 4,
      "Understand": 7
    }
  },
  "engage_mapping_coverage": {
    "coverage_percent": 100.0,
    "mapped_defined_rules": 8,
    "total_defined_rules": 8,
    "triggered_without_mapping": [],
    "unmapped_defined_rules": []
  }
}
```

### Step 10.11：建立報告快速查看腳本
- 請執行：
```
cat > /opt/deception-lab/scripts/show_report.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

REPORT_DIR="/opt/deception-lab/reports"

echo "=== Report Files ==="
ls -lah "$REPORT_DIR"

echo
echo "=== Main Report Preview ==="
if [ -f "$REPORT_DIR/report.md" ]; then
  head -n 80 "$REPORT_DIR/report.md"
else
  echo "report.md not found."
fi

echo
echo "=== Timeline Preview ==="
if [ -f "$REPORT_DIR/timeline.md" ]; then
  head -n 80 "$REPORT_DIR/timeline.md"
else
  echo "timeline.md not found."
fi

echo
echo "=== JSON Report Summary ==="
if [ -f "$REPORT_DIR/report.json" ]; then
  cat "$REPORT_DIR/report.json" | jq '.summary'
else
  echo "report.json not found."
fi
EOF
```
- 設定可執行：
```
chmod +x /opt/deception-lab/scripts/show_report.sh
```
- 執行：
```
/opt/deception-lab/scripts/show_report.sh

# 執行結果：
lss@lss:/opt/deception-lab $ chmod +x /opt/deception-lab/scripts/show_report.sh
lss@lss:/opt/deception-lab $ /opt/deception-lab/scripts/show_report.sh
=== Report Files ===
total 40K
drwxrwxr-x 2 lss lss 4.0K May 22 03:11 .
drwxr-xr-x 8 lss lss 4.0K May 21 08:15 ..
-rw-rw-r-- 1 lss lss 2.0K May 22 03:11 mapping_report.md
-rw-rw-r-- 1 lss lss  12K May 22 03:11 report.json
-rw-rw-r-- 1 lss lss 5.4K May 22 03:11 report.md
-rw-rw-r-- 1 lss lss 8.0K May 22 03:11 timeline.md

=== Main Report Preview ===
# Raspberry Pi Deception Lab MVP Report

Generated at: `2026-05-21T19:11:12.864557+00:00`

## 1. Executive Summary

This report summarizes events collected from the Raspberry Pi deception lab MVP.
The lab includes a Cowrie SSH honeypot, a fake web admin panel, honeycredentials, honeyfiles, centralized log collection, detection rules, and MITRE mapping.

| Metric | Value |
| --- | --- |
| Total events | 34 |
| Total detections | 11 |
| Honeycredential detections | 3 |
| Honeyfile detections | 3 |
| ATT&CK mapping coverage | 100.0% |
| Engage mapping coverage | 100.0% |

## 2. Event Summary

### Events by Source

| Source | Count |
| --- | --- |
| cowrie | 11 |
| fake-web | 23 |

### Events by Type

| Event Type | Count |
| --- | --- |
| ssh_command | 8 |
| ssh_connection | 1 |
| ssh_login_success | 1 |
| ssh_logout | 1 |
| web_honeyfile_access | 2 |
| web_login_attempt | 4 |
| web_request | 17 |

## 3. Detection Summary

### Detections by Severity

| Severity | Count |
| --- | --- |
| high | 6 |
| medium | 5 |

### Detections by Rule

| Rule ID | Count |
| --- | --- |
| SSH_HONEYCREDENTIAL_LOGIN | 1 |
| SSH_HONEYFILE_ACCESS | 1 |
| SSH_RECON_COMMAND | 5 |
| WEB_HONEYCREDENTIAL_USED | 2 |
| WEB_HONEYFILE_ACCESS | 2 |

## 4. Top Detections

| Timestamp | Severity | Rule | Source | Src IP | Username | Observed Behavior | ATT&CK | Engage |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 2026-05-12T18:41:52.679019+00:00 | high | WEB_HONEYCREDENTIAL_USED | fake-web | 172.18.0.1 | backup |  | T1552 Unsecured Credentials | Elicit: Credential Collection |
| 2026-05-12T18:36:26.543366+00:00 | high | WEB_HONEYCREDENTIAL_USED | fake-web | 172.18.0.1 | backup |  | T1552 Unsecured Credentials | Elicit: Credential Collection |
| 2026-05-12T18:27:53+00:00 | high | SSH_HONEYFILE_ACCESS | cowrie | 172.18.0.1 |  | cat /home/admin/secrets.txt | T1005 Data from Local System | Elicit: Reveal Adversary Intent |
| 2026-05-12T18:26:42+00:00 | high | SSH_HONEYCREDENTIAL_LOGIN | cowrie | 172.18.0.1 | backup |  | T1552 Unsecured Credentials | Elicit: Credential Collection |
| 2026-05-11T21:57:19.331247+00:00 | high | WEB_HONEYFILE_ACCESS | fake-web | 192.168.1.1 |  | /download/vpn_users.csv | T1005 Data from Local System | Understand: Collect Adversary Behavior |
| 2026-05-11T21:57:12.514844+00:00 | high | WEB_HONEYFILE_ACCESS | fake-web | 192.168.1.1 |  | /download/secrets.txt | T1005 Data from Local System | Understand: Collect Adversary Behavior |
| 2026-05-12T18:27:46+00:00 | medium | SSH_RECON_COMMAND | cowrie | 172.18.0.1 |  | ls /home/admin | T1082 System Information Discovery | Understand: Collect Adversary Behavior |
| 2026-05-12T18:27:30+00:00 | medium | SSH_RECON_COMMAND | cowrie | 172.18.0.1 |  | ls /home/admin | T1082 System Information Discovery | Understand: Collect Adversary Behavior |
| 2026-05-12T18:27:16+00:00 | medium | SSH_RECON_COMMAND | cowrie | 172.18.0.1 |  | ls | T1082 System Information Discovery | Understand: Collect Adversary Behavior |
| 2026-05-12T18:27:11+00:00 | medium | SSH_RECON_COMMAND | cowrie | 172.18.0.1 |  | pwd | T1082 System Information Discovery | Understand: Collect Adversary Behavior |

## 5. MITRE ATT&CK Coverage

### By Tactic

| Tactic | Detection Count |
| --- | --- |
| Collection | 3 |

=== Timeline Preview ===
# Deception Lab Event Timeline

Generated at: 2026-05-21T19:11:12.809837Z

## Summary

- Total events: 34
- Total detections: 11

## 2026-05-11

### 21:42:08 UTC

- Event: **web_request** from `fake-web` src_ip=`172.18.0.1`; path=`/login`
- Severity: `info`
- Tags: `web, request`
- Detections: none

### 21:42:34 UTC

- Event: **web_request** from `fake-web` src_ip=`172.18.0.1`; path=`/api/status`
- Severity: `info`
- Tags: `web, request`
- Detections: none

### 21:46:40 UTC

- Event: **web_request** from `fake-web` src_ip=`192.168.1.1`; path=`/`
- Severity: `info`
- Tags: `web, request`
- Detections: none

### 21:46:40 UTC

- Event: **web_request** from `fake-web` src_ip=`192.168.1.1`; path=`/login`
- Severity: `info`
- Tags: `web, request`
- Detections: none

### 21:46:40 UTC

- Event: **web_request** from `fake-web` src_ip=`192.168.1.1`; path=`/static/style.css`
- Severity: `info`
- Tags: `web, request`
- Detections: none

### 21:46:40 UTC

- Event: **web_request** from `fake-web` src_ip=`192.168.1.1`; path=`/favicon.ico`
- Severity: `info`
- Tags: `web, request`
- Detections: none

### 21:46:56 UTC

- Event: **web_request** from `fake-web` src_ip=`192.168.1.1`; path=`/login`
- Severity: `info`
- Tags: `web, request`
- Detections: none

### 21:46:56 UTC

- Event: **web_login_attempt** from `fake-web` src_ip=`192.168.1.1`; username=`admin`
- Severity: `medium`
- Tags: `web, login`
- Detections: none

### 21:46:56 UTC

- Event: **web_request** from `fake-web` src_ip=`192.168.1.1`; path=`/static/style.css`
- Severity: `info`
- Tags: `web, request`
- Detections: none

### 21:57:09 UTC

- Event: **web_request** from `fake-web` src_ip=`192.168.1.1`; path=`/backup`
- Severity: `info`
- Tags: `web, request`
- Detections: none

=== JSON Report Summary ===
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
  "honeycredential_detections": 3,
  "honeyfile_detections": 3,
  "total_detections": 11,
  "total_events": 34
}
```

### Step 10.12：更新 README，加入報告指令
- 執行：
```
cat >> /opt/deception-lab/README.md <<'EOF'

## Phase 10 Report Generation

Generate final reports:

```bash
/opt/deception-lab/scripts/generate_report.sh
EOF
```
- Show report preview:
```
/opt/deception-lab/scripts/show_report.sh

# 執行結果：
lss@lss:/opt/deception-lab $ /opt/deception-lab/scripts/show_report.sh
=== Report Files ===
total 40K
drwxrwxr-x 2 lss lss 4.0K May 22 03:11 .
drwxr-xr-x 8 lss lss 4.0K May 21 08:15 ..
-rw-rw-r-- 1 lss lss 2.0K May 22 03:11 mapping_report.md
-rw-rw-r-- 1 lss lss  12K May 22 03:11 report.json
-rw-rw-r-- 1 lss lss 5.4K May 22 03:11 report.md
-rw-rw-r-- 1 lss lss 8.0K May 22 03:11 timeline.md

=== Main Report Preview ===
# Raspberry Pi Deception Lab MVP Report

Generated at: `2026-05-21T19:11:12.864557+00:00`

## 1. Executive Summary

This report summarizes events collected from the Raspberry Pi deception lab MVP.
The lab includes a Cowrie SSH honeypot, a fake web admin panel, honeycredentials, honeyfiles, centralized log collection, detection rules, and MITRE mapping.

| Metric | Value |
| --- | --- |
| Total events | 34 |
| Total detections | 11 |
| Honeycredential detections | 3 |
| Honeyfile detections | 3 |
| ATT&CK mapping coverage | 100.0% |
| Engage mapping coverage | 100.0% |

## 2. Event Summary

### Events by Source

| Source | Count |
| --- | --- |
| cowrie | 11 |
| fake-web | 23 |

### Events by Type

| Event Type | Count |
| --- | --- |
| ssh_command | 8 |
| ssh_connection | 1 |
| ssh_login_success | 1 |
| ssh_logout | 1 |
| web_honeyfile_access | 2 |
| web_login_attempt | 4 |
| web_request | 17 |

## 3. Detection Summary

### Detections by Severity

| Severity | Count |
| --- | --- |
| high | 6 |
| medium | 5 |

### Detections by Rule

| Rule ID | Count |
| --- | --- |
| SSH_HONEYCREDENTIAL_LOGIN | 1 |
| SSH_HONEYFILE_ACCESS | 1 |
| SSH_RECON_COMMAND | 5 |
| WEB_HONEYCREDENTIAL_USED | 2 |
| WEB_HONEYFILE_ACCESS | 2 |

## 4. Top Detections

| Timestamp | Severity | Rule | Source | Src IP | Username | Observed Behavior | ATT&CK | Engage |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 2026-05-12T18:41:52.679019+00:00 | high | WEB_HONEYCREDENTIAL_USED | fake-web | 172.18.0.1 | backup |  | T1552 Unsecured Credentials | Elicit: Credential Collection |
| 2026-05-12T18:36:26.543366+00:00 | high | WEB_HONEYCREDENTIAL_USED | fake-web | 172.18.0.1 | backup |  | T1552 Unsecured Credentials | Elicit: Credential Collection |
| 2026-05-12T18:27:53+00:00 | high | SSH_HONEYFILE_ACCESS | cowrie | 172.18.0.1 |  | cat /home/admin/secrets.txt | T1005 Data from Local System | Elicit: Reveal Adversary Intent |
| 2026-05-12T18:26:42+00:00 | high | SSH_HONEYCREDENTIAL_LOGIN | cowrie | 172.18.0.1 | backup |  | T1552 Unsecured Credentials | Elicit: Credential Collection |
| 2026-05-11T21:57:19.331247+00:00 | high | WEB_HONEYFILE_ACCESS | fake-web | 192.168.1.1 |  | /download/vpn_users.csv | T1005 Data from Local System | Understand: Collect Adversary Behavior |
| 2026-05-11T21:57:12.514844+00:00 | high | WEB_HONEYFILE_ACCESS | fake-web | 192.168.1.1 |  | /download/secrets.txt | T1005 Data from Local System | Understand: Collect Adversary Behavior |
| 2026-05-12T18:27:46+00:00 | medium | SSH_RECON_COMMAND | cowrie | 172.18.0.1 |  | ls /home/admin | T1082 System Information Discovery | Understand: Collect Adversary Behavior |
| 2026-05-12T18:27:30+00:00 | medium | SSH_RECON_COMMAND | cowrie | 172.18.0.1 |  | ls /home/admin | T1082 System Information Discovery | Understand: Collect Adversary Behavior |
| 2026-05-12T18:27:16+00:00 | medium | SSH_RECON_COMMAND | cowrie | 172.18.0.1 |  | ls | T1082 System Information Discovery | Understand: Collect Adversary Behavior |
| 2026-05-12T18:27:11+00:00 | medium | SSH_RECON_COMMAND | cowrie | 172.18.0.1 |  | pwd | T1082 System Information Discovery | Understand: Collect Adversary Behavior |

## 5. MITRE ATT&CK Coverage

### By Tactic

| Tactic | Detection Count |
| --- | --- |
| Collection | 3 |

=== Timeline Preview ===
# Deception Lab Event Timeline

Generated at: 2026-05-21T19:11:12.809837Z

## Summary

- Total events: 34
- Total detections: 11

## 2026-05-11

### 21:42:08 UTC

- Event: **web_request** from `fake-web` src_ip=`172.18.0.1`; path=`/login`
- Severity: `info`
- Tags: `web, request`
- Detections: none

### 21:42:34 UTC

- Event: **web_request** from `fake-web` src_ip=`172.18.0.1`; path=`/api/status`
- Severity: `info`
- Tags: `web, request`
- Detections: none

### 21:46:40 UTC

- Event: **web_request** from `fake-web` src_ip=`192.168.1.1`; path=`/`
- Severity: `info`
- Tags: `web, request`
- Detections: none

### 21:46:40 UTC

- Event: **web_request** from `fake-web` src_ip=`192.168.1.1`; path=`/login`
- Severity: `info`
- Tags: `web, request`
- Detections: none

### 21:46:40 UTC

- Event: **web_request** from `fake-web` src_ip=`192.168.1.1`; path=`/static/style.css`
- Severity: `info`
- Tags: `web, request`
- Detections: none

### 21:46:40 UTC

- Event: **web_request** from `fake-web` src_ip=`192.168.1.1`; path=`/favicon.ico`
- Severity: `info`
- Tags: `web, request`
- Detections: none

### 21:46:56 UTC

- Event: **web_request** from `fake-web` src_ip=`192.168.1.1`; path=`/login`
- Severity: `info`
- Tags: `web, request`
- Detections: none

### 21:46:56 UTC

- Event: **web_login_attempt** from `fake-web` src_ip=`192.168.1.1`; username=`admin`
- Severity: `medium`
- Tags: `web, login`
- Detections: none

### 21:46:56 UTC

- Event: **web_request** from `fake-web` src_ip=`192.168.1.1`; path=`/static/style.css`
- Severity: `info`
- Tags: `web, request`
- Detections: none

### 21:57:09 UTC

- Event: **web_request** from `fake-web` src_ip=`192.168.1.1`; path=`/backup`
- Severity: `info`
- Tags: `web, request`
- Detections: none

=== JSON Report Summary ===
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
  "honeycredential_detections": 3,
  "honeyfile_detections": 3,
  "total_detections": 11,
  "total_events": 34
}
```
- Generated report files:
```
/opt/deception-lab/reports/timeline.md
/opt/deception-lab/reports/report.md
/opt/deception-lab/reports/report.json
```
- 執行完成後，用這個確認 README 最後內容：
```
tail -n 40 /opt/deception-lab/README.md

# 你應該會看到：
## Phase 10 Report Generation
Generate final reports:
```bash
/opt/deception-lab/scripts/generate_report.sh

# 執行結果：
lss@lss:/opt/deception-lab $ tail -n 40 /opt/deception-lab/README.md
2. Fake Web Admin Panel
3. Honeycredentials
4. Honeyfiles
5. Log Collector
6. Python Event Parser
7. MITRE ATT&CK / MITRE Engage Mapping
8. Markdown / JSON Report Generator

## Directory Structure

```text
/opt/deception-lab
├── docker-compose.yml
├── .env
├── cowrie
├── fake-web
├── parser
├── data
│   ├── logs
│   │   ├── cowrie
│   │   └── web
│   ├── events
│   └── samples
│       └── uploads
├── reports
└── scripts

## Phase 10 Report Generation

Generate final reports:

```bash
/opt/deception-lab/scripts/generate_report.sh

## Phase 10 Report Generation

Generate final reports:

```bash
/opt/deception-lab/scripts/generate_report.sh
```

## 第十階段完成檢查
- 最後請執行以下指令，把輸出貼給我確認：
```
/opt/deception-lab/scripts/generate_report.sh
ls -lah /opt/deception-lab/reports
head -n 120 /opt/deception-lab/reports/report.md
head -n 120 /opt/deception-lab/reports/timeline.md
cat /opt/deception-lab/reports/report.json | jq '.summary'
cat /opt/deception-lab/PHASE10_READY.md
```
- 確認：第十階段主要功能已經成功完成，目前只差建立 PHASE10_READY.md 完成紀錄檔。
```
你這次輸出已經證明以下項目都正常：

項目	結果	狀態
generate_report.sh	成功執行	✅
timeline.md	已產生，8.0K	✅
report.md	已產生，5.4K	✅
report.json	已產生，12K	✅
mapping_report.md	已存在	✅
Total events	34	✅
Total detections	11	✅
Honeycredential detections	3	✅
Honeyfile detections	3	✅
ATT&CK coverage	100.0%	✅
Engage coverage	100.0%	✅
你貼出的報告內容也已經包含 Executive Summary、事件統計、偵測統計、Top Detections、MITRE ATT&CK coverage、MITRE Engage coverage、Deception Assets，以及 timeline 中的 honeyfile access 偵測事件。這代表第十階段的報告產生流程已經正常。
```




