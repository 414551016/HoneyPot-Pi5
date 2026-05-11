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

























