# 第二階段：Raspberry Pi 5 環境建置
---
## 完成目標：
- [完成] 已安裝 Raspberry Pi OS Lite 64-bit.
- [完成] 可以用 SSH 從你的筆電連進 Raspberry Pi
- [完成] 系統已更新
- [完成] 已安裝基本工具
- [完成] 已安裝 Docker
- [完成] 已安裝 Docker Compose plugin
- [完成] 已建立 /opt/deception-lab 專案資料夾
- [完成] 已設定基本防火牆
- [完成] 已確認磁碟空間
- [完成] 已確認 Docker 可以正常執行

Raspberry Pi 官方建議使用 Raspberry Pi Imager 將 Raspberry Pi OS 寫入 microSD 或其他開機媒體；Docker 官方文件也說明，64-bit Raspberry Pi OS 應依照 Debian 安裝方式安裝 Docker Engine，Compose 則以 Docker Compose plugin 方式安裝。

### Step 2.1：用 Raspberry Pi Imager 安裝系統
- 下載 Raspberry Pi Imager，請注意：一定要選：Raspberry Pi OS Lite 64-bit

### Step 2.2：在 Imager 裡先設定 SSH 和帳號
- Hostname：lss
- Username：不要用：pi、root、admin 等名稱，原因是這些名稱太常被攻擊者猜。
- Password：
- 啟用 SSH：
- 配置網路：
- 設定地區：Timezone: Asia/Taipei

### Step 2.4：找到 Raspberry Pi 的 IP
### Step 2.5：從筆電 SSH 進 Raspberry Pi
```
ssh labuser@deception-pi.local
```

### Step 2.6：確認你真的在 Raspberry Pi 裡
- hostname
- whoami
- uname -m

## Step 2.7：更新 Raspberry Pi OS
- sudo apt update
- sudo apt full-upgrade -y
- sudo reboot

## Step 2.8：安裝基本工具
```
sudo apt install -y \
  git \
  curl \
  wget \
  vim \
  nano \
  tree \
  htop \
  jq \
  unzip \
  ca-certificates \
  gnupg \
  lsb-release \
  ufw
```
這些工具用途如下：
| 工具              | 用途          |
| --------------- | ----------- |
| `git`           | 之後抓程式或版本管理  |
| `curl` / `wget` | 下載檔案、測試 Web |
| `vim` / `nano`  | 編輯設定檔       |
| `tree`          | 查看資料夾結構     |
| `htop`          | 查看系統資源      |
| `jq`            | 處理 JSON log |
| `unzip`         | 解壓縮         |
| `ufw`           | 基本防火牆       |

## Step 2.9：檢查狀態
- tree --version
- df -h：Size：53G 
- free -h：Mem:7.9Gi, Swap:512Mi，Raspberry Pi 5 有 4GB 或 8GB RAM 都可以做這個 MVP。

# Step 2.11：安裝 Docker
- 移除可能衝突的舊套件：如果顯示有些套件不存在，是正常的。
  ```
  for pkg in docker.io docker-doc docker-compose podman-docker containerd runc; do
    sudo apt remove -y "$pkg"
  done
  ```
- 加入 Docker 官方 GPG key
```
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```
- 加入 Docker apt repository
```
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
```
- 安裝 Docker Engine 與 Compose plugin
```
sudo apt install -y \
  docker-ce \
  docker-ce-cli \
  containerd.io \
  docker-buildx-plugin \
  docker-compose-plugin
```

## Step 2.12：啟動 Docker 並設定開機自動啟動
- 執行：
```
sudo systemctl enable docker
sudo systemctl start docker
```
- 確認 Docker 狀態：
```
sudo systemctl status docker
```
- q

## Step 2.13：讓 labuser 可以直接使用 docker
- 把 labuser 加入 docker 群組：
```
sudo usermod -aG docker labuser
exit
```
- 重新登入：ssh labuser@deception-pi.local
- 測試：docker version

## Step 2.14：測試 Docker 是否正常
```
docker run hello-world
成功時會看到類似：Hello from Docker!
這代表 Docker 可以正常拉 image 並啟動 container。
```

## Step 2.15：測試 Docker Compose 是否正常
Docker Compose plugin 官方安裝文件說，可以用 docker compose version 測試安裝。
```
docker compose version
成功時會看到類似：Docker Compose version v2.x.x
```

## Step 2.16：建立專案根目錄
```
sudo mkdir -p /opt/deception-lab
sudo chown -R lss:lss /opt/deception-lab
cd /opt/deception-lab
pwd
```

## Step 2.17：建立基本資料夾
- 執行：
```
mkdir -p \
  cowrie \
  fake-web \
  parser \
  data/logs/cowrie \
  data/logs/web \
  data/events \
  data/samples/uploads \
  reports \
  scripts
```
- 查看資料夾：
```
tree -L 3 /opt/deception-lab
```

## Step 2.18：建立第二階段檢查檔
- 執行：
  ```
  cat > /opt/deception-lab/PHASE2_READY.md <<'EOF'
  # Phase 2 Ready
  
  This Raspberry Pi 5 is prepared for the deception lab MVP.
  
  Completed items:
  
  - Raspberry Pi OS installed
  - SSH enabled
  - System updated
  - Basic tools installed
  - Docker installed
  - Docker Compose plugin installed
  - Project directory created
  - Basic log/report folders created
  
  Project path:
  
  /opt/deception-lab
  EOF
  ```
- 確認檔案存在：
  ```
  cat /opt/deception-lab/PHASE2_READY.md
  ```

## Step 2.19：設定基本防火牆 UFW
這一步要小心，因為如果你設定錯，可能會把自己 SSH 擋掉。
- 先允許 SSH 管理 port 22
  ```
  sudo ufw allow 22/tcp
  ```
- 先允許之後會用到的 honeypot port
  ```
  sudo ufw allow 2222/tcp
  sudo ufw allow 8080/tcp
  ```
- 啟用 UFW
  ```
  sudo ufw enable
  ```
- 查看防火牆狀態
  ```
  sudo ufw status verbose
  結果：應該看到類似
  Status: active
  22/tcp    ALLOW IN
  2222/tcp  ALLOW IN
  8080/tcp  ALLOW IN
  ```
- 重要提醒：Docker 與 UFW
  - Docker 官方文件提醒，如果使用 UFW 或 firewalld 管理防火牆，Docker 發布 container port 時可能繞過某些防火牆規則；較嚴格的規則應放在 Docker 的 DOCKER-USER chain。
  - 第二階段先做基本防火牆即可。
  - 到了第十二階段，我們會再補上更完整的隔離與 egress 限制。

## Step 2.20：確認目前開放 port
```
sudo ss -tulpn
目前你應該至少會看到 SSH port 22。
2222 和 8080 現在可能還不會出現，因為 Cowrie 和 Fake Web 還沒部署。這是正常的。
```

## Step 2.21：確認系統時間
```
timedatectl
請確認有看到類似：
Time zone: Asia/Taipei
System clock synchronized: yes

如果時區不是 Asia/Taipei，執行：
sudo timedatectl set-timezone Asia/Taipei
```

## Step 2.22：建立常用操作腳本
先建立一個檢查環境的腳本，之後每次都可以快速檢查 Raspberry Pi 狀態。
- 執行：
  ```
  cat > /opt/deception-lab/scripts/check_env.sh <<'EOF'
  #!/usr/bin/env bash
  set -e
  
  echo "=== Hostname ==="
  hostname
  
  echo
  echo "=== User ==="
  whoami
  
  echo
  echo "=== Architecture ==="
  uname -m
  
  echo
  echo "=== OS ==="
  cat /etc/os-release | grep -E 'PRETTY_NAME|VERSION_CODENAME'
  
  echo
  echo "=== Disk ==="
  df -h /
  
  echo
  echo "=== Memory ==="
  free -h
  
  echo
  echo "=== Docker ==="
  docker version --format 'Client: {{.Client.Version}} | Server: {{.Server.Version}}'
  
  echo
  echo "=== Docker Compose ==="
  docker compose version
  
  echo
  echo "=== UFW ==="
  sudo ufw status verbose
  EOF
  ```
- 設定可執行：
  ```
  chmod +x /opt/deception-lab/scripts/check_env.sh
  ```
- 執行：
  ```
  /opt/deception-lab/scripts/check_env.sh
  ```

## 第二階段完成後，你的 Raspberry Pi 狀態
完成後，你的 Raspberry Pi 應該像這樣：
```
Raspberry Pi 5
├── Raspberry Pi OS Lite 64-bit
├── SSH 管理入口
│   └── port 22
├── Docker
├── Docker Compose plugin
├── UFW firewall
│   ├── allow 22/tcp
│   ├── allow 2222/tcp
│   └── allow 8080/tcp
└── /opt/deception-lab
    ├── cowrie
    ├── fake-web
    ├── parser
    ├── data
    │   ├── logs
    │   ├── events
    │   └── samples
    ├── reports
    └── scripts
```

### 現在先不要做的事
- 不要啟動 Cowrie
- 不要寫 docker-compose.yml
- 不要開放到 Internet
- 不要設定 port forwarding
- 不要放真實帳密
- 不要接到正式內網做攻擊測試
