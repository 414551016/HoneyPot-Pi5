# 第五階段：建立 Fake Web Admin Panel
這一階段會在 Raspberry Pi 上新增一個假的 Web 管理介面，對外使用：
```
http://192.168.1.167:8080
```
```
[完成] 顯示假登入頁
[完成] 記錄所有 Web request
[完成] 記錄登入嘗試
[完成] 放置 honeycredential
[完成] 放置 honeyfile 下載點
[完成] 將 Web log 寫入 /opt/deception-lab/data/logs/web
[完成] 透過 Docker Compose 啟動 fake-web container
```
## 第五階段架構
完成後你的平台會變成：
```
Raspberry Pi 5
├── 真實 SSH 管理入口
│   └── port 22
│
├── Cowrie SSH Honeypot
│   └── port 2222
│
└── Fake Web Admin Panel
    └── port 8080

log 會放在：
/opt/deception-lab/data/logs/web/web_access.jsonl
/opt/deception-lab/data/logs/web/web_auth.jsonl
```

## Step 5.1：進入專案資料夾
```
在 Raspberry Pi 終端機執行：
cd /opt/deception-lab
pwd
```

## Step 5.2：建立 Fake Web 目錄結構
- 執行：
    ```
    mkdir -p \
      /opt/deception-lab/fake-web/templates \
      /opt/deception-lab/fake-web/static \
      /opt/deception-lab/fake-web/honeyfiles \
      /opt/deception-lab/data/logs/web
    ```
- 執行結果：
  ```
  # Fake Web 目錄結構
  lss@lss:/opt/deception-lab $ tree -L 3 /opt/deception-lab/fake-web
  /opt/deception-lab/fake-web
  ├── honeyfiles
  ├── static
  └── templates

  # 專案全部目錄結構
  lss@lss:/opt/deception-lab $ tree -L 3 /opt/deception-lab
  /opt/deception-lab
  ├── cowrie
  │   ├── etc
  │   ├── honeyfs
  │   │   ├── etc
  │   │   ├── home
  │   │   ├── tmp
  │   │   └── var
  │   └── var
  │       ├── lib
  │       └── log
  ├── data
  │   ├── events
  │   ├── logs
  │   │   ├── cowrie
  │   │   └── web
  │   └── samples
  │       └── uploads
  ├── docker-compose.phase3.yml
  ├── docker-compose.phase4.broken.yml
  ├── docker-compose.phase4.permission-error.yml
  ├── docker-compose.yml
  ├── fake-web
  │   ├── honeyfiles
  │   ├── static
  │   └── templates
  ├── parser
  ├── PHASE2_READY.md
  ├── PHASE3_READY.md
  ├── PHASE4_READY.md
  ├── README.md
  ├── reports
  └── scripts
      ├── check_env.sh
      ├── logs_lab.sh
      ├── restart_lab.sh
      ├── start_lab.sh
      ├── status_lab.sh
      └── stop_lab.sh
  
  25 directories, 14 files

  ```






























