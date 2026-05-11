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
lss@lss:/opt/deception-lab $ tree -L 2 /opt/deception-lab/
/opt/deception-lab/
├── cowrie
├── data
│   ├── events
│   ├── logs
│   └── samples
├── fake-web
├── parser
├── PHASE2_READY.md
├── reports
└── scripts
    └── check_env.sh

10 directories, 2 files

```

### Step 3.3：建立 .env
.env 是 Docker Compose 會讀取的環境設定檔。
- 執行：cat /opt/deception-lab/.env
  ```
  cat > /opt/deception-lab/.env <<'EOF'
  # Deception Lab Environment Settings
  
  # Project
  COMPOSE_PROJECT_NAME=deception-lab
  TZ=Asia/Taipei
  
  # Host ports
  HOST_SSH_HONEYPOT_PORT=2222
  HOST_WEB_PORT=8080
  
  # Container ports
  COWRIE_SSH_PORT=2222
  FAKE_WEB_PORT=8080
  
  # Paths on Raspberry Pi host
  PROJECT_ROOT=/opt/deception-lab
  COWRIE_LOG_DIR=/opt/deception-lab/data/logs/cowrie
  WEB_LOG_DIR=/opt/deception-lab/data/logs/web
  EVENT_DIR=/opt/deception-lab/data/events
  REPORT_DIR=/opt/deception-lab/reports
  
  # Lab identity
  LAB_NAME=Raspberry Pi Deception Lab
  LAB_OWNER=lss
  LAB_HOSTNAME=lss
  EOF
  ```

### Step 3.4：建立初版 docker-compose.yml
這一版 docker-compose.yml 先建立專案架構與網路，不急著啟動 Cowrie 和 Fake Web。原因是：
- Cowrie 需要第四階段設定。
- Fake Web 需要第五階段建立程式。
- 第三階段先確定 Docker Compose 專案格式正確。
- 執行：cat /opt/deception-lab/docker-compose.yml
  ```
  cat > /opt/deception-lab/docker-compose.yml <<'EOF'
  services:
    placeholder:
      image: alpine:latest
      container_name: deception-placeholder
      command: ["sh", "-c", "echo 'Deception Lab Docker Compose is ready.' && sleep 5"]
      environment:
        - TZ=${TZ}
      networks:
        - deception_net
      restart: "no"
  
  networks:
    deception_net:
      driver: bridge
  EOF
  ```
#### 為什麼現在只有 placeholder？
- 這是刻意設計的。目前第三階段只是測試：
  ```
  Docker Compose 專案能不能被解析
  network 能不能建立
  script 能不能執行
  ```
- 所以先放一個很小的 Alpine Linux container 當測試服務。
- 之後第四階段會把 Cowrie 加進來。
- 第五階段會把 Fake Web 加進來。
- 到時候這個 placeholder 會被移除。

## Step 3.5：檢查 Docker Compose 設定是否正確
- 執行：
  ```
  cd /opt/deception-lab
  docker compose config
  ```
- 執行結果：
  ```
  如果設定正確，你會看到 Docker Compose 展開後的設定內容。
  應該會看到類似：
  lss@lss:/opt/deception-lab $ cd /opt/deception-lab
  docker compose config
  name: deception-lab
  services:
    placeholder:
      command:
        - sh
        - -c
        - echo 'Deception Lab Docker Compose is ready.' && sleep 5
      container_name: deception-placeholder
      environment:
        TZ: Asia/Taipei
      image: alpine:latest
      networks:
        deception_net: null
      restart: "no"
  networks:
    deception_net:
      name: deception-lab_deception_net
      driver: bridge

  # 如果沒有出現錯誤，代表 docker-compose.yml 格式正確。
  ```

## Step 3.6：建立啟動腳本 start_lab.sh
- 這個腳本之後用來啟動整個平台。請執行：
  ```
  cat > /opt/deception-lab/scripts/start_lab.sh <<'EOF'
  #!/usr/bin/env bash
  set -e
  
  cd /opt/deception-lab
  
  echo "[+] Starting Raspberry Pi Deception Lab..."
  docker compose up -d
  
  echo
  echo "[+] Current service status:"
  docker compose ps
  EOF
  ```
- 設定可執行：
  ```
  chmod +x /opt/deception-lab/scripts/start_lab.sh
  ```

## Step 3.7：建立停止腳本 stop_lab.sh
- 請執行：
  ```
  cat > /opt/deception-lab/scripts/stop_lab.sh <<'EOF'
  #!/usr/bin/env bash
  set -e
  
  cd /opt/deception-lab
  
  echo "[+] Stopping Raspberry Pi Deception Lab..."
  docker compose down
  
  echo
  echo "[+] Lab stopped."
  EOF
  ```
- 設定可執行：
  ```
  chmod +x /opt/deception-lab/scripts/stop_lab.sh
  ```

## Step 3.8：建立狀態檢查腳本 status_lab.sh
- 請執行：
  ```
  cat > /opt/deception-lab/scripts/status_lab.sh <<'EOF'
  #!/usr/bin/env bash
  set -e
  
  cd /opt/deception-lab
  
  echo "=== Docker Compose Services ==="
  docker compose ps
  
  echo
  echo "=== Docker Networks ==="
  docker network ls | grep deception || true
  
  echo
  echo "=== Listening Ports ==="
  sudo ss -tulpn | grep -E ':22|:2222|:8080' || true
  
  echo
  echo "=== Disk Usage ==="
  df -h /
  
  echo
  echo "=== Log Directories ==="
  du -sh /opt/deception-lab/data/logs/* 2>/dev/null || true
  EOF
  ```
- 設定可執行：
  ```
  chmod +x /opt/deception-lab/scripts/status_lab.sh
  ```

## Step 3.9：建立 log 查看腳本 logs_lab.sh
- 請執行：
  ```
  cat > /opt/deception-lab/scripts/logs_lab.sh <<'EOF'
  #!/usr/bin/env bash
  set -e
  
  cd /opt/deception-lab
  
  echo "[+] Showing Docker Compose logs..."
  docker compose logs --tail=100 -f
  EOF
  ```
- 設定可執行：
  ```
  chmod +x /opt/deception-lab/scripts/logs_lab.sh
  ```

## Step 3.10：建立重啟腳本 restart_lab.sh
- 請執行：
  ```
  cat > /opt/deception-lab/scripts/restart_lab.sh <<'EOF'
  #!/usr/bin/env bash
  set -e
  
  cd /opt/deception-lab
  
  echo "[+] Restarting Raspberry Pi Deception Lab..."
  docker compose down
  docker compose up -d
  
  echo
  echo "[+] Current service status:"
  docker compose ps
  EOF
  ```
- 設定可執行：chmod +x /opt/deception-lab/scripts/restart_lab.sh

## Step 3.11：建立 README.md
這個 README 是專案說明檔，記錄目前平台怎麼啟動、停止、查看狀態。
- 請執行：
  ```
  cat > /opt/deception-lab/README.md <<'EOF'
  # Raspberry Pi 5 Deception Lab MVP
  
  This project is a small standalone deception and honeypot lab running on Raspberry Pi 5.
  
  ## Current Stage
  
  Phase 3: Docker Compose project structure created.
  
  ## Host Information
  
  - Hostname: lss
  - Username: lss
  - Project path: /opt/deception-lab
  - Management SSH port: 22
  - Cowrie SSH honeypot port: 2222
  - Fake Web Admin port: 8080
  
  ## Main Components
  
  Planned components:
  
  1. Cowrie SSH Honeypot
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
  EOF
  ```
- Basic Commands：
  - Start lab:/opt/deception-lab/scripts/start_lab.sh
  - Stop lab:/opt/deception-lab/scripts/stop_lab.sh
  - Check status:/opt/deception-lab/scripts/status_lab.sh
  - Show logs:/opt/deception-lab/scripts/logs_lab.sh
  - Restart lab:/opt/deception-lab/scripts/restart_lab.sh
- 安全規則：
  - 在 MVP 測試期間，請勿將此實驗環境直接暴露於網路上。
  - 請勿使用真實憑證。
  - 請勿使用真實 SSH 金鑰。
  - 請勿儲存真實的內部 IP 位址。
  - 請勿將 Docker 套接字掛載到容器中。
  - 請勿使用特權容器。
  - 請勿為 honeypot 服務使用主機網路模式。

## Step 3.13：測試 Docker Compose config
- 請執行：
  ```
  cd /opt/deception-lab
  docker compose config
  ```
- 執行結果：
  ```
  lss@lss:/opt/deception-lab $ docker compose config
  name: deception-lab
  services:
    placeholder:
      command:
        - sh
        - -c
        - echo 'Deception Lab Docker Compose is ready.' && sleep 5
      container_name: deception-placeholder
      environment:
        TZ: Asia/Taipei
      image: alpine:latest
      networks:
        deception_net: null
      restart: "no"
  networks:
    deception_net:
      name: deception-lab_deception_net
      driver: bridge
  ```

## Step 3.14：測試啟動腳本
測試我們剛剛做的 start script。
- 執行：
  ```
  /opt/deception-lab/scripts/start_lab.sh
  ```
- 執行結果：
  ```
  第一次可能會下載 Alpine image，看到類似：
    Unable to find image 'alpine:latest' locally
    latest: Pulling from library/alpine
  成功後應該會看到類似：
    [+] Current service status:
    ...
    這裡 Exited 是正常的。
    因為這個 placeholder 只會印一句話，等 5 秒後就結束。
    
  lss@lss:/opt/deception-lab $ /opt/deception-lab/scripts/start_lab.sh
  [+] Starting Raspberry Pi Deception Lab...
  [+] up 6/6
   ✔ Image alpine:latest                 Pulled                               5.2s
   ✔ Network deception-lab_deception_net Created                              0.0s
   ✔ Container deception-placeholder     Started                              1.0s
  
  [+] Current service status:
  NAME                    IMAGE           COMMAND                  SERVICE       CREATED        STATUS                  PORTS
  deception-placeholder   alpine:latest   "sh -c 'echo 'Decept…"   placeholder   1 second ago   Up Less than a second

  ```

## Step 3.15：查看狀態
- 執行：
  ```
  /opt/deception-lab/scripts/status_lab.sh
  ```
- 執行結果：
  ```
  lss@lss:/opt/deception-lab $ /opt/deception-lab/scripts/status_lab.sh
  === Docker Compose Services ===
  NAME      IMAGE     COMMAND   SERVICE   CREATED   STATUS    PORTS
  
  === Docker Networks ===
  7bc2cde6efdb   deception-lab_deception_net   bridge    local
  
  === Listening Ports ===
  tcp   LISTEN 0      128          0.0.0.0:22         0.0.0.0:*    users:(("sshd",pid=1136,fd=6))
  tcp   LISTEN 0      128             [::]:22            [::]:*    users:(("sshd",pid=1136,fd=7))
  
  === Disk Usage ===
  Filesystem      Size  Used Avail Use% Mounted on
  /dev/mmcblk0p2   58G  4.5G   51G   9% /
  
  === Log Directories ===
  4.0K    /opt/deception-lab/data/logs/cowrie
  4.0K    /opt/deception-lab/data/logs/web

  ```





