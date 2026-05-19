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




































