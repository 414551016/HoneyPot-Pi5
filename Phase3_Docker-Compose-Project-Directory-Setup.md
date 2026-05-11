# 第三階段：Docker Compose 專案目錄建立
這一階段的目標是先把整個專案的「骨架」建立好。還不正式部署 Cowrie，也 還不建立 Fake Web，而是先完成之後會用到的基礎檔案：
```
/opt/deception-lab/
├── docker-compose.yml
├── .env
├── scripts/
│   ├── start_lab.sh
│   ├── stop_lab.sh
│   ├── status_lab.sh
│   ├── logs_lab.sh
│   └── restart_lab.sh
└── README.md
完成後，你會有一個可以管理整個平台的 Docker Compose 專案。
```
## 目標
```
[ ] 建立 .env
[ ] 建立 docker-compose.yml
[ ] 建立 start_lab.sh
[ ] 建立 stop_lab.sh
[ ] 建立 status_lab.sh
[ ] 建立 logs_lab.sh
[ ] 建立 restart_lab.sh
[ ] 建立 README.md
[ ] 測試 docker compose config
[ ] 測試腳本是否可執行
```

### Step 3.1：進入專案資料夾
- 請在 Raspberry Pi 終端機執行：
  ```
  cd /opt/deception-lab
  ```
- 確認目前位置：pwd
  - 你應該看到： /opt/deception-lab

### Step 3.2：確認目前資料夾狀態
```
tree -L 2 /opt/deception-lab
執行結果：
/opt/deception-lab
├── PHASE2_READY.md
├── cowrie
├── data
├── fake-web
├── parser
├── reports
└── scripts
```
