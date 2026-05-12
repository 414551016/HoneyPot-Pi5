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

### Step 6.11：檢查 Docker Compose 設定
- 執行：
    ```
    cd /opt/deception-lab
    docker compose config
    ```
    ```
    lss@lss:/opt/deception-lab $ docker compose config
    name: deception-lab
    services:
      cowrie:
        container_name: deception-cowrie
        environment:
          TZ: Asia/Taipei
        image: cowrie/cowrie:latest
        networks:
          deception_net: null
        ports:
          - mode: ingress
            target: 2222
            published: "2222"
            protocol: tcp
        restart: unless-stopped
        security_opt:
          - no-new-privileges:true
        volumes:
          - type: bind
            source: /opt/deception-lab/cowrie/honeyfs
            target: /cowrie/cowrie-git/src/cowrie/data/honeyfs
            read_only: true
            bind: {}
          - type: bind
            source: /opt/deception-lab/cowrie/etc/userdb.txt
            target: /cowrie/cowrie-git/etc/userdb.txt
            read_only: true
            bind: {}
      fake-web:
        build:
          context: /opt/deception-lab/fake-web
          dockerfile: Dockerfile
        container_name: deception-fake-web
        depends_on:
          cowrie:
            condition: service_started
            required: true
        environment:
          HONEYFILE_DIR: /app/honeyfiles
          TZ: Asia/Taipei
          WEB_LOG_DIR: /app/logs
        networks:
          deception_net: null
        ports:
          - mode: ingress
            target: 8080
            published: "8080"
            protocol: tcp
        restart: unless-stopped
        security_opt:
          - no-new-privileges:true
        volumes:
          - type: bind
            source: /opt/deception-lab/data/logs/web
            target: /app/logs
            bind: {}
          - type: bind
            source: /opt/deception-lab/fake-web/honeyfiles
            target: /app/honeyfiles
            read_only: true
            bind: {}
    networks:
      deception_net:
        name: deception-lab_deception_net
        driver: bridge
    ```
    如果沒有錯誤，就繼續。

### Step 6.12：重新啟動平台
- 執行：
    ```
    docker compose down
    docker compose up -d
    ```
- 確認：
    ```
    docker compose ps
    ```
    ```
    # 你應該看到：
    deception-cowrie     Up
    deception-fake-web   Up
    
    lss@lss:/opt/deception-lab $ docker compose down
    [+] down 3/3
     ✔ Container deception-fake-web        Removed                                                                                        0.5s
     ✔ Container deception-cowrie          Removed                                                                                        0.5s
     ✔ Network deception-lab_deception_net Removed                                                                                        0.2s
    lss@lss:/opt/deception-lab $ docker compose up -d
    [+] up 3/3
     ✔ Network deception-lab_deception_net Created                                                                                        0.0s
     ✔ Container deception-cowrie          Started                                                                                        0.7s
     ✔ Container deception-fake-web        Started                                                                                        0.7s
    lss@lss:/opt/deception-lab $ docker compose ps
    NAME                 IMAGE                    COMMAND                  SERVICE    CREATED          STATUS          PORTS
    deception-cowrie     cowrie/cowrie:latest     "/cowrie/cowrie-env/…"   cowrie     14 seconds ago   Up 13 seconds   0.0.0.0:2222->2222/tcp, [::]:2222->2222/tcp, 2223/tcp
    deception-fake-web   deception-lab-fake-web   "gunicorn --bind 0.0…"   fake-web   14 seconds ago   Up 13 seconds   0.0.0.0:8080->8080/tcp, [::]:8080->8080/tcp
    ```
- 測試 Cowrie 是否讀到 userdb.txt
    ```
    # 先清除舊的 SSH key 紀錄：
    ssh-keygen -f '/home/lss/.ssh/known_hosts' -R '[127.0.0.1]:2222'
    
    # 然後測試：
    ssh -p 2222 backup@127.0.0.1
    
    # 密碼輸入：Backup2026!
    如果成功，你會進入 Cowrie 假 shell。
    
    # 進入後請輸入：
    whoami
    pwd
    ls
    ls /home/admin
    cat /home/admin/secrets.txt
    exit
    ```
    ```
    lss@lss:/opt/deception-lab $ ssh-keygen -f '/home/lss/.ssh/known_hosts' -R '[127.0.0.1]:2222'
    # Host [127.0.0.1]:2222 found: line 1
    /home/lss/.ssh/known_hosts updated.
    Original contents retained as /home/lss/.ssh/known_hosts.old
    lss@lss:/opt/deception-lab $ ssh -p 2222 backup@127.0.0.1
    The authenticity of host '[127.0.0.1]:2222 ([127.0.0.1]:2222)' can't be established.
    ED25519 key fingerprint is SHA256:Gx/iT67Oyb1w3bqEXlZzeWi0jpTa4wnbH/Wb2Iuw0QE.
    This key is not known by any other names.
    Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
    Warning: Permanently added '[127.0.0.1]:2222' (ED25519) to the list of known hosts.
    backup@127.0.0.1's password:
    
    The programs included with the Debian GNU/Linux system are free software;
    the exact distribution terms for each program are described in the
    individual files in /usr/share/doc/*/copyright.
    
    Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
    permitted by applicable law.
    backup@svr04:~$ whoami
    backup
    backup@svr04:~$ pwd
    /var/backups
    backup@svr04:~$ is
    -bash: is: command not found
    backup@svr04:~$ ls
    backup@svr04:~$ ls /home/admin
    ls: cannot access /home/admin: No such file or directory
    backup@svr04:~$ ls /home/admin
    ls: cannot access /home/admin: No such file or directory
    backup@svr04:~$ cat /home/admin/secrets.txt
    cat: /home/admin/secrets.txt: No such file or directory
    backup@svr04:~$ exit
    Connection to 127.0.0.1 closed.

    # 執行結果確認：
    1. 確認：Cowrie honeycredential 登入測試成功。
        你已經成功用這組假帳密登入 Cowrie：backup / Backup2026!
        關鍵成功訊號是你進入了假的 shell：backup@svr04:~$
        這代表：
            [成功] Cowrie 已讀到 userdb.txt
            [成功] backup / Backup2026! 已被接受
            [成功] 你進入了 Cowrie 假 SSH shell
            [成功] Cowrie honeycredential 功能已生效
    2. 這代表 Cowrie shell 裡目前看不到我們掛進 honeyfs 的 /home/admin 檔案。但這不影響剛剛的主要成果：honeycredential 登入已成功。你目前已完成第六階段中最重要的一步：讓 Cowrie 使用假帳密。
    ```
- 如果登入失敗
    ```
    docker compose logs --tail=120 cowrie
    ```
    ```
    然後看有沒有這句：
    Could not read etc/userdb.txt
    如果沒有這句，但登入還是失敗，可能是 userdb.txt 格式或 Cowrie 認證規則需要調整。先把 log 貼給我，我再幫你修。
    ```
- 同時測試 Fake Web honeycredential
    - 執行：
        ```
        curl -s -X POST http://127.0.0.1:8080/login \
          -d "username=backup" \
          -d "password=Backup2026!" \
          -o /tmp/fakeweb-login-test.html
        ```
    - 確認：
        ```
        grep -E "Dashboard|Authentication accepted" /tmp/fakeweb-login-test.html
        
        再檢查 Web auth log：
        tail -n 10 /opt/deception-lab/data/logs/web/web_auth.jsonl
        
        你應該看到："honeycredential_used": true
        
        lss@lss:/opt/deception-lab $ curl -s -X POST http://127.0.0.1:8080/login \
          -d "username=backup" \
          -d "password=Backup2026!" \
          -o /tmp/fakeweb-login-test.html
        lss@lss:/opt/deception-lab $ grep -E "Dashboard|Authentication accepted" /tmp/fakeweb-login-test.html
          <title>Dashboard - Internal Device Management Console</title>
            <h1>Device Management Dashboard</h1>
            <div class="warning">Authentication accepted. Some modules are temporarily unavailable.</div>
        lss@lss:/opt/deception-lab $ tail -n 10 /opt/deception-lab/data/logs/web/web_auth.jsonl
        {"timestamp": "2026-05-11T21:46:56.695098+00:00", "source": "fake-web", "event_type": "web_login_attempt", "src_ip": "192.168.1.1", "username": "admin", "password_sha256": "8d969eef6ecad3c29a3a629280e686cf0c3f5d5a86aff3ca12020c923adc6c92", "password_length": 6, "honeycredential_used": false, "user_agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.0.0 Safari/537.36", "severity": "medium", "tags": ["web", "login"]}
        {"timestamp": "2026-05-11T22:18:07.651474+00:00", "source": "fake-web", "event_type": "web_login_attempt", "src_ip": "192.168.1.1", "username": "backup", "password_sha256": "0ecba7213823d57d8b9c6510186aa9ba9e401b2f4508e792f8f3ca4aec6394e1", "password_length": 12, "honeycredential_used": false, "user_agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.0.0 Safari/537.36", "severity": "medium", "tags": ["web", "login"]}
        {"timestamp": "2026-05-12T18:36:26.543366+00:00", "source": "fake-web", "event_type": "web_login_attempt", "src_ip": "172.18.0.1", "username": "backup", "password_sha256": "a597108ff8384b3be204d08bce36fe3e86bffcc699fdac27604db48ae6f27f71", "password_length": 11, "honeycredential_used": true, "user_agent": "curl/8.14.1", "severity": "high", "tags": ["web", "login", "honeycredential"]}
        ```
- 再匯出 Cowrie log
  因為你剛剛已經成功登入 Cowrie，請執行：
    - 執行：
        ```
        docker compose logs --no-color cowrie > /opt/deception-lab/data/logs/cowrie/cowrie-docker.log
        ```
    - 然後確認登入紀錄：
        ```
        grep -E "login attempt|backup|logged in|Command found|CMD" /opt/deception-lab/data/logs/cowrie/cowrie-docker.log | tail -n 40
        ```
        ```
        lss@lss:/opt/deception-lab $ docker compose logs --no-color cowrie > /opt/deception-lab/data/logs/cowrie/cowrie-docker.log
        lss@lss:/opt/deception-lab $ grep -E "login attempt|backup|logged in|Command found|CMD" /opt/deception-lab/data/logs/cowrie/cowrie-docker.log | tail -n 40
        deception-cowrie  | 2026-05-13T02:26:29+0800 [cowrie.ssh.userauth.HoneyPotSSHUserAuthServer#debug] b'backup' trying auth b'none'
        deception-cowrie  | 2026-05-13T02:26:42+0800 [cowrie.ssh.userauth.HoneyPotSSHUserAuthServer#debug] b'backup' trying auth b'password'
        deception-cowrie  | 2026-05-13T02:26:42+0800 [HoneyPotSSHTransport,0,172.18.0.1] login attempt [b'backup'/b'Backup2026!'] succeeded
        deception-cowrie  | 2026-05-13T02:26:42+0800 [cowrie.ssh.userauth.HoneyPotSSHUserAuthServer#debug] b'backup' authenticated with b'password'
        deception-cowrie  | 2026-05-13T02:27:07+0800 [HoneyPotSSHTransport,0,172.18.0.1] CMD: whoami
        deception-cowrie  | 2026-05-13T02:27:07+0800 [HoneyPotSSHTransport,0,172.18.0.1] Command found: whoami
        deception-cowrie  | 2026-05-13T02:27:11+0800 [HoneyPotSSHTransport,0,172.18.0.1] CMD: pwd
        deception-cowrie  | 2026-05-13T02:27:11+0800 [HoneyPotSSHTransport,0,172.18.0.1] Command found: pwd
        deception-cowrie  | 2026-05-13T02:27:14+0800 [HoneyPotSSHTransport,0,172.18.0.1] CMD: is
        deception-cowrie  | 2026-05-13T02:27:16+0800 [HoneyPotSSHTransport,0,172.18.0.1] CMD: ls
        deception-cowrie  | 2026-05-13T02:27:16+0800 [HoneyPotSSHTransport,0,172.18.0.1] Command found: ls
        deception-cowrie  | 2026-05-13T02:27:30+0800 [HoneyPotSSHTransport,0,172.18.0.1] CMD: ls /home/admin
        deception-cowrie  | 2026-05-13T02:27:30+0800 [HoneyPotSSHTransport,0,172.18.0.1] Command found: ls /home/admin
        deception-cowrie  | 2026-05-13T02:27:46+0800 [HoneyPotSSHTransport,0,172.18.0.1] CMD: ls /home/admin
        deception-cowrie  | 2026-05-13T02:27:46+0800 [HoneyPotSSHTransport,0,172.18.0.1] Command found: ls /home/admin
        deception-cowrie  | 2026-05-13T02:27:53+0800 [HoneyPotSSHTransport,0,172.18.0.1] CMD: cat /home/admin/secrets.txt
        deception-cowrie  | 2026-05-13T02:27:53+0800 [HoneyPotSSHTransport,0,172.18.0.1] Command found: cat /home/admin/secrets.txt
        deception-cowrie  | 2026-05-13T02:27:57+0800 [HoneyPotSSHTransport,0,172.18.0.1] CMD: exit
        deception-cowrie  | 2026-05-13T02:27:57+0800 [HoneyPotSSHTransport,0,172.18.0.1] Command found: exit
        deception-cowrie  | 2026-05-13T02:27:57+0800 [HoneyPotSSHTransport,0,172.18.0.1] avatar backup logging out        
        ```
- 小結

### Step 6 補做：建立 check_assets.sh
因為執行：/opt/deception-lab/scripts/check_assets.sh  出現 No such file or directory 這代表你還沒有建立 check_assets.sh 腳本。
- 執行：
    ```
    cat > /opt/deception-lab/scripts/check_assets.sh <<'EOF'
    #!/usr/bin/env bash
    set -e
    
    echo "=== Deception Assets File ==="
    ls -lah /opt/deception-lab/deception_assets.yml
    
    echo
    echo "=== Cowrie userdb ==="
    ls -lah /opt/deception-lab/cowrie/etc/userdb.txt
    echo
    cat /opt/deception-lab/cowrie/etc/userdb.txt
    
    echo
    echo "=== Fake Web Honeyfiles ==="
    ls -lah /opt/deception-lab/fake-web/honeyfiles
    
    echo
    echo "=== Cowrie Honeyfs Key Files ==="
    for f in \
      /opt/deception-lab/cowrie/honeyfs/home/admin/secrets.txt \
      /opt/deception-lab/cowrie/honeyfs/home/admin/backup_config.ini \
      /opt/deception-lab/cowrie/honeyfs/home/admin/vpn_users.csv \
      /opt/deception-lab/cowrie/honeyfs/home/admin/ssh_keys_backup.txt \
      /opt/deception-lab/cowrie/honeyfs/home/admin/database_passwords.txt \
      /opt/deception-lab/cowrie/honeyfs/home/backup/backup_jobs.txt \
      /opt/deception-lab/cowrie/honeyfs/home/operator/maintenance_notes.txt \
      /opt/deception-lab/cowrie/honeyfs/etc/edge-gateway/app.conf
    do
      if [ -f "$f" ]; then
        echo "[OK] $f"
      else
        echo "[MISSING] $f"
      fi
    done
    EOF
    ```
    設定可執行權限：
    ```
    chmod +x /opt/deception-lab/scripts/check_assets.sh
    ```  
    執行檢查：
    ```
    /opt/deception-lab/scripts/check_assets.sh
    
    lss@lss:/opt/deception-lab $ chmod +x /opt/deception-lab/scripts/check_assets.sh
    lss@lss:/opt/deception-lab $ /opt/deception-lab/scripts/check_assets.sh
    === Deception Assets File ===
    -rw-rw-r-- 1 lss lss 3.9K May 13 01:47 /opt/deception-lab/deception_assets.yml
    
    === Cowrie userdb ===
    -rw-rw-r-- 1 lss lss 363 May 13 01:51 /opt/deception-lab/cowrie/etc/userdb.txt
    
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
    
    === Fake Web Honeyfiles ===
    total 28K
    drwxrwxr-x 2 lss lss 4.0K May 12 05:30 .
    drwxrwxr-x 5 lss lss 4.0K May 12 05:32 ..
    -rw-rw-r-- 1 lss lss  233 May 12 05:27 backup_config.ini
    -rw-rw-r-- 1 lss lss  160 May 12 05:30 database_passwords.txt
    -rw-rw-r-- 1 lss lss  172 May 12 05:26 secrets.txt
    -rw-rw-r-- 1 lss lss  268 May 12 05:30 ssh_keys_backup.txt
    -rw-rw-r-- 1 lss lss  192 May 12 05:29 vpn_users.csv
    
    === Cowrie Honeyfs Key Files ===
    [OK] /opt/deception-lab/cowrie/honeyfs/home/admin/secrets.txt
    [OK] /opt/deception-lab/cowrie/honeyfs/home/admin/backup_config.ini
    [OK] /opt/deception-lab/cowrie/honeyfs/home/admin/vpn_users.csv
    [OK] /opt/deception-lab/cowrie/honeyfs/home/admin/ssh_keys_backup.txt
    [OK] /opt/deception-lab/cowrie/honeyfs/home/admin/database_passwords.txt
    [OK] /opt/deception-lab/cowrie/honeyfs/home/backup/backup_jobs.txt
    [OK] /opt/deception-lab/cowrie/honeyfs/home/operator/maintenance_notes.txt
    [OK] /opt/deception-lab/cowrie/honeyfs/etc/edge-gateway/app.conf
    
    ```




