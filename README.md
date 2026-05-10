# 使用 Raspberry Pi 5 實作一個最小可行的單機版／小型實驗版主動式資安防禦誘捕欺敵平台

## 摘要：
我的計畫目的為發展主動式防禦之欺敵、阻斷、分析技術，並對應 MITRE Engage 中 Prepare、Expose、Affect、Elicit、Understand 五個戰略目標。系統希望能自動部署誘捕欺敵環境，提供可攻擊目標，偵測攻擊者行為，誘導或干擾攻擊者，蒐集攻擊歷程，並產生防禦知識。

## 目標：
1. 單機版部署於 Raspberry Pi 5
2. 使用 Docker Compose
3. 包含 SSH honeypot，例如 Cowrie
4. 包含假 Web 管理介面
5. 包含 honeycredential / honeyfile
6. 收集 honeypot 與 Web log
7. 實作簡單攻擊行為偵測
8. 將事件對應到 MITRE ATT&CK 與 MITRE Engage
9. 產生事件時間線與 Markdown / JSON 報告
10. 注意實驗安全隔離，不影響真實內網
