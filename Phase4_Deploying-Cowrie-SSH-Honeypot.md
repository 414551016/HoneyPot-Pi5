# 第四階段：部署 Cowrie SSH Honeypot
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
