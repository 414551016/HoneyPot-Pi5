# 第十一階段：測試攻擊情境
```
這一階段的目標是用「受控、低風險、只打自己的 Raspberry Pi 實驗環境」的方式，產生一組測試攻擊行為，確認你的平台可以完整做到：
攻擊互動
    ↓
Cowrie / Fake Web 記錄 log
    ↓
collect_logs.sh 收集
    ↓
run_parser.sh 解析
    ↓
run_mapping.sh 對應 MITRE
    ↓
generate_report.sh 產生報告

#第十一階段安全前提：本階段只允許測試你自己的實驗環境：
Raspberry Pi IP: 192.168.1.167
真實 SSH 管理 port: 22
Cowrie SSH honeypot port: 2222
Fake Web Admin port: 8080
不要對學校網段、真實內網、公開網站、別人的主機執行掃描或攻擊測試。
```
## 第十一階段測試目標
```
完成後，你會驗證以下情境：
[完成] SSH 登入失敗測試
[完成] SSH honeycredential 成功登入測試
[完成] SSH 偵察指令測試
[完成] SSH honeyfile 存取測試
[完成] SSH 工具下載行為測試
[完成] Web 登入失敗測試
[完成] Web honeycredential 使用測試
[完成] Web honeyfile 下載測試
[完成] Web scanner path 探測測試
[完成] 重新收集 log
[完成] 重新執行 parser
[完成] 重新執行 mapping
[完成] 重新產生 report
[完成] 驗證 detections 增加
```
### Step 11.1：確認服務狀態
- 請先在 Raspberry Pi 上執行：
```
cd /opt/deception-lab
/opt/deception-lab/scripts/status_lab.sh
```































