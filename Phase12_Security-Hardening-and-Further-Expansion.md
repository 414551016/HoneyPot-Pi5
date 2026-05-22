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

# 你應該確認：
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










































