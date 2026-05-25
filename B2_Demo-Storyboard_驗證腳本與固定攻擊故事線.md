# Phase B2：建立固定 Demo 攻擊流程
目標是讓你每次展示時，都能產生一組可預期的事件：
```
1. Fake Web Admin 被存取
2. 攻擊者嘗試登入
3. Honeycredential 被提交或使用
4. SSH / Cowrie 收到登入嘗試
5. Honeyfile 被存取
6. Detection rules 觸發
7. ATT&CK coverage 更新
8. Engage coverage 更新
9. Dashboard 顯示完整故事線
```
### 建議建立一個 demo script：
- 執行：
```
cat > /opt/deception-lab/scripts/demo_attack.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

TARGET_IP="${1:-127.0.0.1}"

echo "[1] Access fake web admin"
curl -s "http://${TARGET_IP}:8080/" > /dev/null || true
curl -s "http://${TARGET_IP}:8080/login" > /dev/null || true

echo "[2] Submit fake login attempt"
curl -s -X POST "http://${TARGET_IP}:8080/login" \
  -d "username=admin" \
  -d "password=Spring2026!" > /dev/null || true

echo "[3] Touch honeyfile-like paths"
curl -s "http://${TARGET_IP}:8080/backup.zip" > /dev/null || true
curl -s "http://${TARGET_IP}:8080/secrets.txt" > /dev/null || true
curl -s "http://${TARGET_IP}:8080/.env" > /dev/null || true

echo "[4] Trigger controlled Cowrie SSH login attempt"
sshpass -p "Spring2026!" ssh \
  -o StrictHostKeyChecking=no \
  -o UserKnownHostsFile=/dev/null \
  -p 2222 admin@"${TARGET_IP}" "whoami" || true

echo "[5] Regenerate parser/report outputs"
cd /opt/deception-lab

# Replace these commands with your actual parser/report commands if different.
python3 scripts/parse_events.py || true
python3 scripts/generate_report.py || true

echo "[DONE] Refresh Streamlit Dashboard at http://${TARGET_IP}:8501"

EOF
```
- 








