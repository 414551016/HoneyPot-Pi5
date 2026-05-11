# 第四階段：部署 Cowrie SSH Honeypot
何謂"Cowrie SSH Honeypot"？
Cowrie SSH Honeypot 是一種開源的資安誘捕系統（Honeypot），專門用來模擬 SSH（以及 Telnet）服務，誘使攻擊者嘗試入侵，並全程記錄其行為，以利資安分析與研究。
- 觀察與記錄 SSH 暴力破解（brute force）
- 分析攻擊者登入後的指令、操作流程
- 蒐集惡意檔案（malware）與攻擊工具
- 作為威脅情報（Threat Intelligence）與教學研究資料

Cowrie 是如何運作的？
1. 偽裝成真實的 SSH 伺服器
   - 對外開放 SSH 連線（常用 22 或轉接後的 2222）
   - 顯示正常的 SSH banner 與 login prompt
   - 接受「錯誤或假帳密」讓攻擊者登入成功（依設定）
3. 提供「假的 Linux 系統」
   - 模擬 UNIX/Linux shell
   - 內建假的檔案系統（如 /etc/passwd、/bin/ls）
   - 攻擊者執行的指令看似成功，實際都在沙盒中
5. 全程記錄行為
   - 嘗試的帳號與密碼
   - 所有輸入指令（如 wget、curl、uname -a）
   - 上傳與下載的檔案（SCP / SFTP）
   - 完整 Session Replay（可重播整個攻擊過程）
   - JSON 格式日誌，方便送到 SIEM 分析

這一階段會真正把 Cowrie SSH honeypot 放到你的 Raspberry Pi 5 上執行。
Cowrie 官方文件說，Cowrie 可以用 Docker 執行，快速測試方式是把 host port 映射到 container 的 2222，例如 docker run -p 2222:2222 cowrie/cowrie:latest；Cowrie 也支援 JSON logging，適合後續做事件分析。

## 第四階段目標
- [完成] Cowrie SSH honeypot container
- [完成] Raspberry Pi port 2222 對應到 Cowrie
- [完成] Cowrie log 掛載到 /opt/deception-lab/data/logs/cowrie
- [完成] 可以用 ssh -p 2222 測試 honeypot
- [完成] 可以看到 Cowrie 記錄登入嘗試
- [完成] docker compose 可以啟動 / 停止 Cowrie

重要提醒：真實 SSH 與假 SSH
- 真實 SSH：192.168.1.167:22，管理 Raspberry Pi
- 假的 SSH：192.168.1.167:2222，給攻擊測試(SSH honeypot)

## Step 4.1：進入專案目錄
- 在 Raspberry Pi 裡執行：cd /opt/deception-lab
- 確認位置：

## Step 4.2：先確認目前沒有服務在跑
- 執行：
  ```
  docker compose ps
  或
  docker ps
  ```
- 執行結果：
  ```
  lss@lss:/opt/deception-lab $ docker compose ps
  NAME      IMAGE     COMMAND   SERVICE   CREATED   STATUS    PORTS
  
  lss@lss:/opt/deception-lab $ docker ps
  CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES

  # 如果沒有正在執行的 container，是正常的。
  ```

## Step 4.3：建立 Cowrie 用資料夾
我們要讓 Cowrie 的資料可以保存到 Raspberry Pi 本機。
- 執行：
  ```
  mkdir -p \
  /opt/deception-lab/cowrie/etc \
  /opt/deception-lab/cowrie/var \
  /opt/deception-lab/cowrie/honeyfs \
  /opt/deception-lab/data/logs/cowrie \
  /opt/deception-lab/data/samples/uploads
  ```
- 確認：
  ```
  tree -L 3 /opt/deception-lab/cowrie
  執行結果：
  lss@lss:/opt/deception-lab $ tree -L 3 /opt/deception-lab/cowrie
  /opt/deception-lab/cowrie
  ├── etc
  ├── honeyfs
  └── var
  ```








