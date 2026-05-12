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

































