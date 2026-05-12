# 第六階段：加入 honeycredential / honeyfile
你前面第五階段已經先放了一些假帳密與假檔案。第六階段的目標是把這些誘餌資產正式整理起來，讓 Fake Web 和 Cowrie 都使用同一套 deception assets。

## 第六階段完成的任務：
- [完成] deception_assets.yml
- [完成] Fake Web honeyfiles 整理完成
- [完成] Cowrie honeyfs 內加入 honeyfile
- [完成] Cowrie userdb.txt 加入假帳密
- [完成] docker-compose.yml 掛載 Cowrie userdb.txt
- [完成] 測試 honeycredential 登入
- [完成] 測試 honeyfile 存取
- [完成] 建立 PHASE6_READY.md
Cowrie 的 userdb.txt 是用冒號分隔的格式，第一欄是 username，第二欄目前未使用，第三欄是 password；官方範例也說明 * 可代表任意 username/password，而 password 前面加 ! 代表不允許該密碼通過。

## 本階段完成後 deception assets 會分成三份：
```
/opt/deception-lab/deception_assets.yml
    統一資產清單，給你和後續 parser 使用

/opt/deception-lab/fake-web/honeyfiles/
    Fake Web 可下載的假敏感檔

/opt/deception-lab/cowrie/honeyfs/
    Cowrie 假 Linux 檔案系統內的假檔案

/opt/deception-lab/cowrie/etc/userdb.txt
    Cowrie 可接受或拒絕的假 SSH 帳密規則
```

### Step 6.1：進入專案目錄
```
cd /opt/deception-lab
pwd
# 應該看到：/opt/deception-lab
```

### Step 6.2：建立統一 deception assets 清單
這個檔案是整個平台的「誘餌資產登記表」。
- 執行：
  ```
  cat > /opt/deception-lab/deception_assets.yml <<'EOF'
  lab:
    name: "Raspberry Pi Deception Lab"
    owner: "lss"
    host: "192.168.1.167"
    purpose: "MVP active defense deception and honeypot lab"
  
  honeycredentials:
    - id: HC_ADMIN_001
      username: "admin"
      password: "Admin@12345"
      role: "fake administrator"
      severity: "high"
      exposed_in:
        - "fake-web:/config"
        - "fake-web:honeyfiles/secrets.txt"
        - "cowrie:/home/admin/secrets.txt"
      engage_goal:
        - "Elicit"
        - "Understand"
      attack_mapping:
        tactic: "Credential Access"
        technique_id: "T1552"
        technique: "Unsecured Credentials"
  
    - id: HC_BACKUP_001
      username: "backup"
      password: "Backup2026!"
      role: "fake backup operator"
      severity: "high"
      exposed_in:
        - "fake-web:html-comment"
        - "fake-web:/config"
        - "fake-web:honeyfiles/backup_config.ini"
        - "cowrie:/home/admin/backup_config.ini"
      engage_goal:
        - "Elicit"
        - "Understand"
      attack_mapping:
        tactic: "Credential Access"
        technique_id: "T1552"
        technique: "Unsecured Credentials"
  
    - id: HC_IOT_001
      username: "iotadmin"
      password: "iot_admin_2026"
      role: "fake IoT administrator"
      severity: "high"
      exposed_in:
        - "fake-web:honeyfiles/vpn_users.csv"
        - "cowrie:/home/admin/vpn_users.csv"
      engage_goal:
        - "Elicit"
        - "Understand"
      attack_mapping:
        tactic: "Credential Access"
        technique_id: "T1552"
        technique: "Unsecured Credentials"
  
    - id: HC_OPERATOR_001
      username: "operator"
      password: "P@ssw0rd!"
      role: "fake maintenance operator"
      severity: "high"
      exposed_in:
        - "fake-web:/config"
        - "fake-web:honeyfiles/secrets.txt"
        - "cowrie:/home/operator/maintenance_notes.txt"
      engage_goal:
        - "Elicit"
        - "Understand"
      attack_mapping:
        tactic: "Credential Access"
        technique_id: "T1552"
        technique: "Unsecured Credentials"
  
  honeyfiles:
    - id: HF_SECRET_001
      filename: "secrets.txt"
      severity: "high"
      description: "Fake internal secret note containing honeycredentials."
      locations:
        - "fake-web:/download/secrets.txt"
        - "cowrie:/home/admin/secrets.txt"
      engage_goal:
        - "Elicit"
        - "Understand"
      attack_mapping:
        tactic: "Collection"
        technique_id: "T1005"
        technique: "Data from Local System"
  
    - id: HF_BACKUP_001
      filename: "backup_config.ini"
      severity: "high"
      description: "Fake backup configuration file."
      locations:
        - "fake-web:/download/backup_config.ini"
        - "cowrie:/home/admin/backup_config.ini"
      engage_goal:
        - "Elicit"
        - "Understand"
      attack_mapping:
        tactic: "Credential Access"
        technique_id: "T1552"
        technique: "Unsecured Credentials"
  
    - id: HF_VPN_001
      filename: "vpn_users.csv"
      severity: "high"
      description: "Fake VPN user export."
      locations:
        - "fake-web:/download/vpn_users.csv"
        - "cowrie:/home/admin/vpn_users.csv"
      engage_goal:
        - "Elicit"
        - "Understand"
      attack_mapping:
        tactic: "Discovery"
        technique_id: "T1087"
        technique: "Account Discovery"
  
    - id: HF_SSHKEY_001
      filename: "ssh_keys_backup.txt"
      severity: "high"
      description: "Fake SSH private key backup."
      locations:
        - "fake-web:/download/ssh_keys_backup.txt"
        - "cowrie:/home/admin/ssh_keys_backup.txt"
      engage_goal:
        - "Elicit"
        - "Understand"
      attack_mapping:
        tactic: "Credential Access"
        technique_id: "T1552.004"
        technique: "Private Keys"
  
    - id: HF_DB_001
      filename: "database_passwords.txt"
      severity: "high"
      description: "Fake database credential backup."
      locations:
        - "fake-web:/download/database_passwords.txt"
        - "cowrie:/home/admin/database_passwords.txt"
      engage_goal:
        - "Elicit"
        - "Understand"
      attack_mapping:
        tactic: "Credential Access"
        technique_id: "T1552"
        technique: "Unsecured Credentials"
  EOF
  ```

### Step 6.3：建立 Cowrie 使用者資料夾
- 在前章己完成有：
```
/opt/deception-lab/cowrie/honeyfs/home/admin
/opt/deception-lab/cowrie/honeyfs/home/root
```
- 執行：現在再補幾個假使用者家目錄。
```
mkdir -p \
  /opt/deception-lab/cowrie/etc \
  /opt/deception-lab/cowrie/honeyfs/home/backup \
  /opt/deception-lab/cowrie/honeyfs/home/iotadmin \
  /opt/deception-lab/cowrie/honeyfs/home/operator \
  /opt/deception-lab/cowrie/honeyfs/var/backups \
  /opt/deception-lab/cowrie/honeyfs/opt/edge-gateway/config
```

### Step 6.4：建立 Cowrie userdb.txt
- 執行：
    ```
    cat > /opt/deception-lab/cowrie/etc/userdb.txt <<'EOF'
    # Cowrie fake SSH credential database
    # Format:
    # username:x:password
    #
    # These are deception-only credentials.
    # Do not use real credentials here.
    
    admin:x:Admin@12345
    backup:x:Backup2026!
    iotadmin:x:iot_admin_2026
    operator:x:P@ssw0rd!
    
    # Common brute-force attempts intentionally denied.
    root:x:!root
    root:x:!123456
    admin:x:!admin
    test:x:!test
    user:x:!password
    EOF
    ```
- 說明
    ```
    admin / Admin@12345       允許登入 Cowrie 假 shell
    backup / Backup2026!      允許登入 Cowrie 假 shell
    iotadmin / iot_admin_2026 允許登入 Cowrie 假 shell
    operator / P@ssw0rd!      允許登入 Cowrie 假 shell
    root / root               拒絕
    root / 123456             拒絕
    ```

### Step 6.5：把 honeyfile 複製到 Cowrie honeyfs
- 執行：
    ```
    cp /opt/deception-lab/fake-web/honeyfiles/secrets.txt \
      /opt/deception-lab/cowrie/honeyfs/home/admin/secrets.txt
    
    cp /opt/deception-lab/fake-web/honeyfiles/backup_config.ini \
      /opt/deception-lab/cowrie/honeyfs/home/admin/backup_config.ini
    
    cp /opt/deception-lab/fake-web/honeyfiles/vpn_users.csv \
      /opt/deception-lab/cowrie/honeyfs/home/admin/vpn_users.csv
    
    cp /opt/deception-lab/fake-web/honeyfiles/ssh_keys_backup.txt \
      /opt/deception-lab/cowrie/honeyfs/home/admin/ssh_keys_backup.txt
    
    cp /opt/deception-lab/fake-web/honeyfiles/database_passwords.txt \
      /opt/deception-lab/cowrie/honeyfs/home/admin/database_passwords.txt
    ```
- 再建立一些不同目錄的誘餌檔：
    ```
    cat > /opt/deception-lab/cowrie/honeyfs/home/backup/backup_jobs.txt <<'EOF'
    Backup Jobs
    
    daily-config-backup:
      target=/opt/edge-gateway/config
      account=backup
      password=Backup2026!
    
    weekly-db-export:
      target=192.0.2.30
      account=dbadmin
      password=ChangeMe_2026!
    EOF
    ```
    ```
    cat > /opt/deception-lab/cowrie/honeyfs/home/operator/maintenance_notes.txt <<'EOF'
    Maintenance Notes
    
    Temporary operator account:
    username=operator
    password=P@ssw0rd!
    
    Check gateway status:
    curl http://127.0.0.1:8080/api/status
    EOF
    ```
    ```
    cat > /opt/deception-lab/cowrie/honeyfs/var/backups/device_inventory.txt <<'EOF'
    Device Inventory
    
    edge-gateway-01,192.0.2.10,linux,active
    backup-server-01,192.0.2.20,linux,active
    db-main-01,192.0.2.30,linux,maintenance
    iot-controller-01,192.0.2.40,linux,active
    EOF
    ```
    ```
    cat > /opt/deception-lab/cowrie/honeyfs/opt/edge-gateway/config/local.conf <<'EOF'
    # Edge Gateway Local Configuration
    
    device_id=edge-gateway-01
    mgmt_user=admin
    mgmt_password=Admin@12345
    backup_user=backup
    backup_password=Backup2026!
    api_token=fake-local-token-2026
    EOF
    ```

### Step 6.6：更新 Cowrie fake passwd 與 group
你之前已經建立過 /etc/passwd 與 /etc/group。現在重新覆蓋成比較完整的版本。
- 執行：
    ```
    cat > /opt/deception-lab/cowrie/honeyfs/etc/passwd <<'EOF'
    root:x:0:0:root:/root:/bin/bash
    admin:x:1000:1000:admin:/home/admin:/bin/bash
    backup:x:1001:1001:backup:/home/backup:/bin/bash
    iotadmin:x:1002:1002:iotadmin:/home/iotadmin:/bin/bash
    operator:x:1003:1003:operator:/home/operator:/bin/bash
    www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
    mysql:x:110:115:MySQL Server:/nonexistent:/bin/false
    EOF
    ```
    ```
    cat > /opt/deception-lab/cowrie/honeyfs/etc/group <<'EOF'
    root:x:0:
    admin:x:1000:
    backup:x:1001:
    iotadmin:x:1002:
    operator:x:1003:
    sudo:x:27:admin,operator
    www-data:x:33:
    mysql:x:115:
    users:x:100:
    EOF
    ```

### Step 6.7：建立 Cowrie 內常見敏感假路徑
- 執行：
    ```
    mkdir -p \
      /opt/deception-lab/cowrie/honeyfs/etc/edge-gateway \
      /opt/deception-lab/cowrie/honeyfs/var/www/html \
      /opt/deception-lab/cowrie/honeyfs/root/.ssh \
      /opt/deception-lab/cowrie/honeyfs/home/admin/.ssh
    ```
- 建立假設定：
    ```
    cat > /opt/deception-lab/cowrie/honeyfs/etc/edge-gateway/app.conf <<'EOF'
    [web-admin]
    url=http://192.0.2.10:8080
    username=admin
    password=Admin@12345
    
    [backup]
    server=192.0.2.20
    username=backup
    password=Backup2026!
    EOF
    ```
- 建立假 SSH key：
    ```
    cat > /opt/deception-lab/cowrie/honeyfs/home/admin/.ssh/id_rsa <<'EOF'
    -----BEGIN OPENSSH PRIVATE KEY-----
    fake-fake-fake-fake-fake-fake-fake-fake
    this-is-not-a-real-private-key
    fake-fake-fake-fake-fake-fake-fake-fake
    -----END OPENSSH PRIVATE KEY-----
    EOF
    ```
    ```
    cat > /opt/deception-lab/cowrie/honeyfs/root/.ssh/authorized_keys <<'EOF'
    ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIFakeFakeFakeFakeFakeFakeFakeFakeFakeFake admin@edge-gateway
    EOF
    ```
- 建立假 Web 設定備份：
    ```
    cat > /opt/deception-lab/cowrie/honeyfs/var/www/html/config_backup.txt <<'EOF'
    Fake Web Admin Backup
    
    username=backup
    password=Backup2026!
    login_url=http://192.0.2.10:8080/login
    EOF
    ```

### Step 6.8：檢查 Cowrie honeyfs 結構
- 執行：
    ```
    tree -L 4 /opt/deception-lab/cowrie/honeyfs | head -n 120
    ```
- 執行結果：你應該會看到類似：
    ```
    lss@lss:/opt/deception-lab $ tree -L 4 /opt/deception-lab/cowrie/honeyfs | head -n 120
    /opt/deception-lab/cowrie/honeyfs
    ├── etc
    │   ├── edge-gateway
    │   │   └── app.conf
    │   ├── group
    │   ├── issue
    │   └── passwd
    ├── home
    │   ├── admin
    │   │   ├── backup_config.ini
    │   │   ├── database_passwords.txt
    │   │   ├── notes.txt
    │   │   ├── secrets.txt
    │   │   ├── ssh_keys_backup.txt
    │   │   └── vpn_users.csv
    │   ├── backup
    │   │   └── backup_jobs.txt
    │   ├── iotadmin
    │   ├── operator
    │   │   └── maintenance_notes.txt
    │   └── root
    ├── opt
    │   └── edge-gateway
    │       └── config
    │           └── local.conf
    ├── root
    ├── tmp
    └── var
        ├── backups
        │   └── device_inventory.txt
        ├── tmp
        └── www
            └── html
                └── config_backup.txt
    
    19 directories, 15 files
    
    ```

### Step 6.9：確認 Fake Web honeyfiles
- 執行：
    ```
    ls -lah /opt/deception-lab/fake-web/honeyfiles
    ```
- 執行結果：你應該看到：
    ```
    lss@lss:/opt/deception-lab $ ls -lah /opt/deception-lab/fake-web/honeyfiles
    total 28K
    drwxrwxr-x 2 lss lss 4.0K May 12 05:30 .
    drwxrwxr-x 5 lss lss 4.0K May 12 05:32 ..
    -rw-rw-r-- 1 lss lss  233 May 12 05:27 backup_config.ini
    -rw-rw-r-- 1 lss lss  160 May 12 05:30 database_passwords.txt
    -rw-rw-r-- 1 lss lss  172 May 12 05:26 secrets.txt
    -rw-rw-r-- 1 lss lss  268 May 12 05:30 ssh_keys_backup.txt
    -rw-rw-r-- 1 lss lss  192 May 12 05:29 vpn_users.csv
    ```

### Step 6.10：更新 docker-compose.yml，掛載 Cowrie userdb.txt
現在要讓 Cowrie 使用我們建立的 userdb.txt。
- 先備份目前 compose：
    ```
    cp /opt/deception-lab/docker-compose.yml /opt/deception-lab/docker-compose.phase5.yml
    ```
    - 用 Python 自動修改，比手動安全：
    ```
    python3 - <<'PY'
    from pathlib import Path
    
    path = Path("/opt/deception-lab/docker-compose.yml")
    text = path.read_text()
    
    old = """    volumes:
          - ./cowrie/honeyfs:/cowrie/cowrie-git/src/cowrie/data/honeyfs:ro
    """
    
    new = """    volumes:
          - ./cowrie/honeyfs:/cowrie/cowrie-git/src/cowrie/data/honeyfs:ro
          - ./cowrie/etc/userdb.txt:/cowrie/cowrie-git/etc/userdb.txt:ro
    """
    
    if old not in text:
        raise SystemExit("Expected Cowrie volumes block not found. Please show docker-compose.yml.")
    path.write_text(text.replace(old, new))
    PY
    ```
- 執行確認：
    ```
    grep -A18 -n "cowrie:" /opt/deception-lab/docker-compose.yml
    執行結果：
    lss@lss:/opt/deception-lab $ grep -A18 -n "cowrie:" /opt/deception-lab/docker-compose.yml
    2:  cowrie:
    3:    image: cowrie/cowrie:latest
    4-    container_name: deception-cowrie
    5-    restart: unless-stopped
    6-    ports:
    7-      - "${HOST_SSH_HONEYPOT_PORT}:2222"
    8-    environment:
    9-      - TZ=${TZ}
    10-    volumes:
    11-      - ./cowrie/honeyfs:/cowrie/cowrie-git/src/cowrie/data/honeyfs:ro
    12-      - ./cowrie/etc/userdb.txt:/cowrie/cowrie-git/etc/userdb.txt:ro
    13-    networks:
    14-      - deception_net
    15-    security_opt:
    16-      - no-new-privileges:true
    17-
    18-  fake-web:
    19-    build:
    20-      context: ./fake-web
    21-      dockerfile: Dockerfile

    grep -n "userdb" /opt/deception-lab/docker-compose.yml
    執行結果：
    lss@lss:/opt/deception-lab $ grep -n "userdb" /opt/deception-lab/docker-compose.yml
    12:      - ./cowrie/etc/userdb.txt:/cowrie/cowrie-git/etc/userdb.txt:ro
    
    docker compose config | grep -A5 -n "userdb"
    執行結果：
    lss@lss:/opt/deception-lab $ docker compose config | grep -A5 -n "userdb"
    25:        source: /opt/deception-lab/cowrie/etc/userdb.txt
    26:        target: /cowrie/cowrie-git/etc/userdb.txt
    27-        read_only: true
    28-        bind: {}
    29-  fake-web:
    30-    build:
    31-      context: /opt/deception-lab/fake-web
    
    ```

















