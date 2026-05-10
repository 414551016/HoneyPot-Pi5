# 第一階段：系統架構設計

## 1.1 MVP 系統定位

本次挑戰的是一個 **單機版／小型實驗版主動式資安防禦誘捕欺敵平台**，部署在 **Raspberry Pi 5** 上，使用 **Docker Compose** 管理多個服務。

本 MVP 的核心目的如下：

1. 建立可被攻擊者互動的假目標。
2. 收集攻擊行為與登入嘗試。
3. 放置 honeycredential 與 honeyfile 誘導攻擊者互動。
4. 將攻擊行為轉換成事件。
5. 使用簡單 detection rules 判斷攻擊意圖。
6. 將事件對應到 MITRE ATT&CK 與 MITRE Engage。
7. 產生 Markdown / JSON 報告。
8. 嚴格限制實驗環境，不影響真實內網

本 MVP 不追求高互動完整企業環境，而是先做出一個 **可展示、可測試、可擴充的 deception lab**。

---

## 1.2 技術基礎

本設計採用以下基礎元件：
| 類別 | 技術 |
|------|------|
| 硬體 | Raspberry Pi 5 |
| 作業系統 | Raspberry Pi OS Lite 64-bit |
| 容器管理 | Docker + Docker Compose |
| SSH honeypot | Cowrie |
| 假 Web 管理介面 | Flask |
| Web server | Python Flask development server，MVP 階段可用 |
| Log 格式 | JSONL + plaintext log |
| Event parser | Python |
| Detection rules | YAML |
| Report generator | Python |
| 報告格式 | Markdown + JSON |
| MITRE ATT&CK | Enterprise Matrix |
| MITRE Engage | Prepare / Expose / Affect / Elicit / Understand |

- Cowrie 官方文件指出，Cowrie 可透過 Docker 執行，並支援 SSH / Telnet honeypot、JSON logging、SSH exec command、SFTP/SCP upload 等功能，適合本 MVP 作為 SSH honeypot 基礎。
- MITRE ATT&CK Enterprise Matrix 提供跨 Windows、macOS、Linux、Cloud、Network Devices、Containers 等平台的攻擊技術分類；你的平台後續會使用它來標註攻擊事件。
- MITRE Engage 則提供 deception / denial / adversary engagement 的設計語言，包含 Prepare、Expose、Affect、Elicit、Understand 等戰略目標，正好對應你的主動式防禦誘捕欺敵平台。

---

## 1.3 整體架構圖
MVP 架構如下：
```
                攻擊者 / 測試端
                      |
                      |
        +-------------+-------------+
        |                           |
   TCP 2222                     TCP 8080
   Fake SSH                    Fake Web Admin
        |                           |
        v                           v
+----------------+          +---------------------+
| Cowrie SSH     |          | Fake Web Admin       |
| Honeypot       |          | Flask App            |
| container      |          | container            |
+----------------+          +---------------------+
        |                           |
        | cowrie.json               | web_access.jsonl
        | cowrie.log                | auth_events.jsonl
        v                           v
+--------------------------------------------------+
| Shared Log Directory on Raspberry Pi             |
| /opt/deception-lab/data/logs                     |
+--------------------------------------------------+
                      |
                      v
+--------------------------------------------------+
| Python Event Parser                              |
| - parse Cowrie log                               |
| - parse Web log                                  |
| - normalize events                               |
| - apply detection rules                          |
| - map ATT&CK / Engage                            |
+--------------------------------------------------+
                      |
                      v
+--------------------------------------------------+
| Output                                           |
| /opt/deception-lab/reports                       |
| - timeline.md                                    |
| - report.md                                      |
| - events.json                                    |
| - detections.json                                |
+--------------------------------------------------+
```
