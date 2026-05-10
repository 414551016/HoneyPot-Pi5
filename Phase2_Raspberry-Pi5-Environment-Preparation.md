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

## Step 2.1：用 Raspberry Pi Imager 安裝系統
- 下載 Raspberry Pi Imager，請注意：一定要選：Raspberry Pi OS Lite 64-bit

## Step 2.2：在 Imager 裡先設定 SSH 和帳號
- Hostname：lss
- Username：不要用：pi、root、admin 等名稱，原因是這些名稱太常被攻擊者猜。
- Password：
- 啟用 SSH：
- 配置網路：
- 設定地區：Timezone: Asia/Taipei

## Step 2.4：找到 Raspberry Pi 的 IP
## Step 2.5：從筆電 SSH 進 Raspberry Pi
```
ssh labuser@deception-pi.local
```

## Step 2.6：確認你真的在 Raspberry Pi 裡
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
