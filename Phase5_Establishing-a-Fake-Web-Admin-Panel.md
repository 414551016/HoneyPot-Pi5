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

## Step 5.3：建立 Flask requirements
- 執行：
    ```
    cat > /opt/deception-lab/fake-web/requirements.txt <<'EOF'
    flask==3.0.3
    gunicorn==22.0.0
    EOF
    ```
- 確認：
  ```
  cat /opt/deception-lab/fake-web/requirements.txt
  ```

## Step 5.4：建立 Fake Web 主程式 app.py
- 執行：
    ```
    cat > /opt/deception-lab/fake-web/app.py <<'EOF'
    from flask import Flask, request, render_template, redirect, url_for, send_from_directory, jsonify
    from datetime import datetime, timezone
    import json
    import os
    import hashlib
    
    app = Flask(__name__)
    
    LOG_DIR = os.environ.get("WEB_LOG_DIR", "/app/logs")
    HONEYFILE_DIR = os.environ.get("HONEYFILE_DIR", "/app/honeyfiles")
    
    ACCESS_LOG = os.path.join(LOG_DIR, "web_access.jsonl")
    AUTH_LOG = os.path.join(LOG_DIR, "web_auth.jsonl")
    
    HONEYCREDENTIALS = {
        "admin": "Admin@12345",
        "backup": "Backup2026!",
        "iotadmin": "iot_admin_2026",
        "operator": "P@ssw0rd!"
    }
    
    SCANNER_PATHS = [
        "/admin",
        "/wp-admin",
        "/wp-login.php",
        "/phpmyadmin",
        "/.env",
        "/config.php",
        "/server-status",
        "/actuator/env",
        "/api/v1/users"
    ]
    
    
    def now_iso():
        return datetime.now(timezone.utc).isoformat()
    
    
    def client_ip():
        forwarded = request.headers.get("X-Forwarded-For")
        if forwarded:
            return forwarded.split(",")[0].strip()
        return request.remote_addr
    
    
    def write_jsonl(path, obj):
        os.makedirs(os.path.dirname(path), exist_ok=True)
        with open(path, "a", encoding="utf-8") as f:
            f.write(json.dumps(obj, ensure_ascii=False) + "\n")
    
    
    def sha256_text(value):
        return hashlib.sha256(value.encode("utf-8")).hexdigest()
    
    
    @app.before_request
    def log_request():
        path = request.path
        is_scanner_probe = path in SCANNER_PATHS
    
        event = {
            "timestamp": now_iso(),
            "source": "fake-web",
            "event_type": "web_request",
            "src_ip": client_ip(),
            "method": request.method,
            "path": path,
            "query_string": request.query_string.decode("utf-8", errors="ignore"),
            "user_agent": request.headers.get("User-Agent", ""),
            "is_scanner_probe": is_scanner_probe,
            "tags": ["web", "request"] + (["scanner-probe"] if is_scanner_probe else [])
        }
    
        write_jsonl(ACCESS_LOG, event)
    
    
    @app.route("/")
    def index():
        return redirect(url_for("login"))
    
    
    @app.route("/login", methods=["GET", "POST"])
    def login():
        if request.method == "POST":
            username = request.form.get("username", "")
            password = request.form.get("password", "")
    
            honeycredential_used = (
                username in HONEYCREDENTIALS and HONEYCREDENTIALS[username] == password
            )
    
            event = {
                "timestamp": now_iso(),
                "source": "fake-web",
                "event_type": "web_login_attempt",
                "src_ip": client_ip(),
                "username": username,
                "password_sha256": sha256_text(password),
                "password_length": len(password),
                "honeycredential_used": honeycredential_used,
                "user_agent": request.headers.get("User-Agent", ""),
                "severity": "high" if honeycredential_used else "medium",
                "tags": ["web", "login"] + (["honeycredential"] if honeycredential_used else [])
            }
    
            write_jsonl(AUTH_LOG, event)
    
            if honeycredential_used:
                return render_template(
                    "dashboard.html",
                    username=username,
                    alert="Authentication accepted. Some modules are temporarily unavailable."
                )
    
            return render_template(
                "login.html",
                error="Invalid username or password. Please try again."
            )
    
        return render_template("login.html")
    
    
    @app.route("/dashboard")
    def dashboard():
        return render_template(
            "dashboard.html",
            username="guest",
            alert="Read-only maintenance mode is enabled."
        )
    
    
    @app.route("/backup")
    def backup():
        return render_template("backup.html")
    
    
    @app.route("/config")
    def config():
        return render_template("config.html")
    
    
    @app.route("/download/<path:filename>")
    def download_honeyfile(filename):
        event = {
            "timestamp": now_iso(),
            "source": "fake-web",
            "event_type": "web_honeyfile_access",
            "src_ip": client_ip(),
            "filename": filename,
            "path": f"/download/{filename}",
            "user_agent": request.headers.get("User-Agent", ""),
            "severity": "high",
            "tags": ["web", "honeyfile", "download"]
        }
    
        write_jsonl(ACCESS_LOG, event)
    
        return send_from_directory(HONEYFILE_DIR, filename, as_attachment=True)
    
    
    @app.route("/api/status")
    def api_status():
        return jsonify({
            "status": "degraded",
            "device": "edge-gateway-01",
            "backup": "warning",
            "auth": "local",
            "maintenance": True
        })
    
    
    @app.route("/robots.txt")
    def robots():
        content = """User-agent: *
    Disallow: /admin
    Disallow: /backup
    Disallow: /config
    Disallow: /download/
    """
        return content, 200, {"Content-Type": "text/plain; charset=utf-8"}
    
    
    @app.route("/admin")
    def admin():
        return redirect(url_for("login"))
    
    
    @app.errorhandler(404)
    def not_found(error):
        return render_template("404.html"), 404
    
    
    if __name__ == "__main__":
        app.run(host="0.0.0.0", port=8080)
    EOF
    ```
- 這個程式會：
    ```
    1. 顯示假登入頁
    2. 記錄所有 HTTP request
    3. 記錄登入帳密
    4. 偵測 honeycredential
    5. 提供 honeyfile 下載
    6. 產生 JSONL log
    ```

## Step 5.5：建立登入頁模板
- 執行：
    ```
    cat > /opt/deception-lab/fake-web/templates/login.html <<'EOF'
    <!DOCTYPE html>
    <html lang="en">
    <head>
      <meta charset="UTF-8">
      <title>Internal Device Management Console</title>
      <link rel="stylesheet" href="/static/style.css">
    </head>
    <body>
      <div class="login-container">
        <div class="card">
          <h1>Internal Device Management Console</h1>
          <p class="subtitle">Edge Gateway Administration Portal</p>
    
          <!-- TODO: remove temporary backup credential before production -->
          <!-- backup / Backup2026! -->
    
          {% if error %}
          <div class="error">{{ error }}</div>
          {% endif %}
    
          <form method="POST" action="/login">
            <label>Username</label>
            <input type="text" name="username" autocomplete="off" autofocus>
    
            <label>Password</label>
            <input type="password" name="password" autocomplete="off">
    
            <button type="submit">Login</button>
          </form>
    
          <div class="hint">
            System notice: backup synchronization is delayed.
          </div>
        </div>
      </div>
    </body>
    </html>
    EOF
    ```
- 說明：
    ```
    這裡故意放了一個 HTML 註解：
    backup / Backup2026!
    
    這是 honeycredential。
    如果攻擊者看原始碼並拿這組帳密登入，我們就能記錄到。
    ```

## Step 5.6：建立 Dashboard 頁面
- 執行：
    ```
    cat > /opt/deception-lab/fake-web/templates/dashboard.html <<'EOF'
    <!DOCTYPE html>
    <html lang="en">
    <head>
      <meta charset="UTF-8">
      <title>Dashboard - Internal Device Management Console</title>
      <link rel="stylesheet" href="/static/style.css">
    </head>
    <body>
      <div class="page">
        <h1>Device Management Dashboard</h1>
        <p class="subtitle">Logged in as: {{ username }}</p>
    
        {% if alert %}
        <div class="warning">{{ alert }}</div>
        {% endif %}
    
        <div class="grid">
          <div class="panel">
            <h2>System Status</h2>
            <p>Device: edge-gateway-01</p>
            <p>Status: Degraded</p>
            <p>Firmware: 4.2.7</p>
          </div>
    
          <div class="panel">
            <h2>Backup Status</h2>
            <p>Last backup: 2026-05-10 02:00</p>
            <p>Status: Warning</p>
            <a href="/backup">View backup files</a>
          </div>
    
          <div class="panel">
            <h2>Configuration</h2>
            <p>Local configuration mode enabled.</p>
            <a href="/config">View configuration</a>
          </div>
    
          <div class="panel">
            <h2>API</h2>
            <p>Device status endpoint:</p>
            <code>/api/status</code>
          </div>
        </div>
      </div>
    </body>
    </html>
    EOF
    ```

## Step 5.7：建立 Backup 頁面
- 執行：
    ```
    cat > /opt/deception-lab/fake-web/templates/backup.html <<'EOF'
    <!DOCTYPE html>
    <html lang="en">
    <head>
      <meta charset="UTF-8">
      <title>Backup Files</title>
      <link rel="stylesheet" href="/static/style.css">
    </head>
    <body>
      <div class="page">
        <h1>Backup Files</h1>
        <p class="subtitle">Restricted backup archive area</p>
    
        <div class="panel">
          <h2>Available Files</h2>
          <ul>
            <li><a href="/download/secrets.txt">secrets.txt</a></li>
            <li><a href="/download/backup_config.ini">backup_config.ini</a></li>
            <li><a href="/download/vpn_users.csv">vpn_users.csv</a></li>
            <li><a href="/download/ssh_keys_backup.txt">ssh_keys_backup.txt</a></li>
            <li><a href="/download/database_passwords.txt">database_passwords.txt</a></li>
          </ul>
        </div>
    
        <p><a href="/dashboard">Back to dashboard</a></p>
      </div>
    </body>
    </html>
    EOF
    ```

## Step 5.8：建立 Config 頁面
- 執行：
    ```
    cat > /opt/deception-lab/fake-web/templates/config.html <<'EOF'
    <!DOCTYPE html>
    <html lang="en">
    <head>
      <meta charset="UTF-8">
      <title>Configuration</title>
      <link rel="stylesheet" href="/static/style.css">
    </head>
    <body>
      <div class="page">
        <h1>Configuration</h1>
        <p class="subtitle">Read-only configuration snapshot</p>
    
        <div class="panel">
          <h2>Backup Server</h2>
          <pre>
    host=192.0.2.20
    username=backup
    password=Backup2026!
    schedule=02:00
          </pre>
        </div>
    
        <div class="panel">
          <h2>Local Admin</h2>
          <pre>
    username=operator
    password=P@ssw0rd!
    role=maintenance
          </pre>
        </div>
    
        <p><a href="/dashboard">Back to dashboard</a></p>
      </div>
    </body>
    </html>
    EOF
    ```
- 說明：
    ```
    注意：
    這裡的 192.0.2.20 是文件範例用 IP，不是真實內網 IP。
    ```

## Step 5.9：建立 404 頁面
- 執行：
    ```
    cat > /opt/deception-lab/fake-web/templates/404.html <<'EOF'
    <!DOCTYPE html>
    <html lang="en">
    <head>
      <meta charset="UTF-8">
      <title>404 Not Found</title>
      <link rel="stylesheet" href="/static/style.css">
    </head>
    <body>
      <div class="page">
        <h1>404 Not Found</h1>
        <p>The requested management resource was not found.</p>
        <p><a href="/login">Return to login</a></p>
      </div>
    </body>
    </html>
    EOF
    ```

## Step 5.10：建立 CSS
- 執行：
    ```
    cat > /opt/deception-lab/fake-web/static/style.css <<'EOF'
    body {
      margin: 0;
      font-family: Arial, sans-serif;
      background: #101820;
      color: #f4f4f4;
    }
    
    a {
      color: #7dd3fc;
    }
    
    .login-container {
      min-height: 100vh;
      display: flex;
      align-items: center;
      justify-content: center;
    }
    
    .card {
      width: 420px;
      background: #182635;
      padding: 32px;
      border-radius: 12px;
      box-shadow: 0 8px 24px rgba(0,0,0,0.35);
    }
    
    h1 {
      margin-top: 0;
      font-size: 26px;
    }
    
    .subtitle {
      color: #a8b3c7;
    }
    
    label {
      display: block;
      margin-top: 18px;
      margin-bottom: 6px;
    }
    
    input {
      width: 100%;
      padding: 10px;
      border: 1px solid #334155;
      border-radius: 6px;
      background: #0f172a;
      color: #f8fafc;
      box-sizing: border-box;
    }
    
    button {
      width: 100%;
      margin-top: 22px;
      padding: 12px;
      border: 0;
      border-radius: 6px;
      background: #2563eb;
      color: white;
      font-weight: bold;
      cursor: pointer;
    }
    
    button:hover {
      background: #1d4ed8;
    }
    
    .error {
      background: #7f1d1d;
      padding: 10px;
      border-radius: 6px;
      margin: 14px 0;
    }
    
    .warning {
      background: #78350f;
      padding: 10px;
      border-radius: 6px;
      margin: 14px 0;
    }
    
    .hint {
      margin-top: 18px;
      font-size: 13px;
      color: #94a3b8;
    }
    
    .page {
      padding: 32px;
    }
    
    .grid {
      display: grid;
      grid-template-columns: repeat(auto-fit, minmax(240px, 1fr));
      gap: 18px;
    }
    
    .panel {
      background: #182635;
      padding: 20px;
      border-radius: 12px;
      margin-bottom: 18px;
    }
    
    pre {
      background: #0f172a;
      padding: 16px;
      border-radius: 8px;
      overflow-x: auto;
    }
    EOF
    ```

## Step 5.11：建立 honeyfiles
- 執行：
    ```
    cat > /opt/deception-lab/fake-web/honeyfiles/secrets.txt <<'EOF'
    Internal Secret Notes
    
    Do not distribute.
    
    Temporary accounts:
    admin / Admin@12345
    backup / Backup2026!
    operator / P@ssw0rd!
    
    Legacy VPN account:
    iotadmin / iot_admin_2026
    EOF
    ```
    ```
    cat > /opt/deception-lab/fake-web/honeyfiles/backup_config.ini <<'EOF'
    [backup-server]
    host=192.0.2.20
    username=backup
    password=Backup2026!
    schedule=02:00
    
    [database]
    host=192.0.2.30
    username=dbadmin
    password=ChangeMe_2026!
    
    [api]
    endpoint=https://api.example.invalid/internal
    token=fake-token-123456789
    EOF
    ```
    ```
    cat > /opt/deception-lab/fake-web/honeyfiles/vpn_users.csv <<'EOF'
    username,role,last_login,status
    admin,administrator,2026-05-01,active
    backup,backup-operator,2026-05-02,active
    iotadmin,device-admin,2026-04-29,active
    operator,maintenance,2026-05-03,disabled
    EOF
    ```
    ```
    cat > /opt/deception-lab/fake-web/honeyfiles/vpn_users.csv <<'EOF'
    username,role,last_login,status
    admin,administrator,2026-05-01,active
    backup,backup-operator,2026-05-02,active
    iotadmin,device-admin,2026-04-29,active
    operator,maintenance,2026-05-03,disabled
    EOF
    ```
    ```
    cat > /opt/deception-lab/fake-web/honeyfiles/ssh_keys_backup.txt <<'EOF'
    This is a fake SSH key backup file for deception testing.
    
    DO NOT USE REAL KEYS HERE.
    
    -----BEGIN OPENSSH PRIVATE KEY-----
    fake-fake-fake-fake-fake-fake-fake-fake
    this-is-not-a-real-private-key
    fake-fake-fake-fake-fake-fake-fake-fake
    -----END OPENSSH PRIVATE KEY-----
    EOF
    ```
    ```
    cat > /opt/deception-lab/fake-web/honeyfiles/database_passwords.txt <<'EOF'
    Database Credential Backup
    
    db-main:
    host=192.0.2.30
    username=dbadmin
    password=ChangeMe_2026!
    
    db-report:
    host=192.0.2.31
    username=report
    password=Report@2026!
    EOF
    ```
- 說明：
    ```
    這些檔案全部是假資料，不能放真實密碼。

    ls -lah /opt/deception-lab/fake-web/honeyfiles
    ```










