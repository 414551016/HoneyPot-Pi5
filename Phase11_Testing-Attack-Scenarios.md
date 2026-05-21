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

























