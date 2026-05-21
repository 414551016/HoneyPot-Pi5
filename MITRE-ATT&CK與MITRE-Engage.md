## 何謂"MITRE ATT&CK / MITRE Engage mapping"？
```
我會先確認 MITRE 官方對 ATT&CK 與 Engage 的定義，再用資安脈絡說明「mapping」實際在做什麼。

已思考 4s

MITRE ATT&CK / MITRE Engage mapping 指的是：把攻擊者的行為模式，也就是 MITRE ATT&CK 中的 tactics / techniques，對應到防禦者可以採取的 MITRE Engage 對抗、誘捕、欺敵或干擾活動。

換句話說：

ATT&CK 描述攻擊者會怎麼做；Engage 描述防禦者可以如何主動設計環境，讓攻擊者暴露、誤判、延遲或被引導。

MITRE ATT&CK 是一個公開的攻擊者行為知識庫，用來描述真實世界攻擊中的 tactics、techniques 和 procedures，常用於威脅建模、偵測工程、紅隊演練與防禦覆蓋分析。

MITRE Engage 則是 MITRE 提供的 adversary engagement framework，重點在於規劃 deception、denial、adversary engagement 等防禦操作，也就是主動與攻擊者互動或操控攻擊者認知。

所謂 mapping，就是建立這種對應關係：

ATT&CK 面向	Engage 面向
攻擊者使用什麼 technique	防禦者可放置什麼誘餌、偽裝、阻斷或觀測點
攻擊者在某階段需要什麼資訊	防禦者可操控或污染這些資訊
攻擊行為會暴露什麼弱點	防禦者可利用該弱點設計 engagement activity

MITRE Engage 官方說法的核心概念是：當攻擊者執行某個 ATT&CK 行為時，可能會暴露非預期弱點；防禦者可以根據每個 ATT&CK activity 找出可利用的弱點，再選擇合適的 Engage activities。

例如：

ATT&CK technique	攻擊者行為	可能的 Engage 對應
Credential Dumping	攻擊者嘗試竊取憑證	放置假帳號、假憑證、honeytoken
Network Service Discovery	攻擊者掃描內網服務	佈署假服務、假主機、觀測掃描行為
Lateral Movement	攻擊者橫向移動	建立誘捕路徑、偽造高價值資產
Exfiltration	攻擊者準備外傳資料	放置假資料集、資料浮水印、外傳監控

它的用途通常包括：

防禦規劃：看到某個 ATT&CK technique，就能思考可以用哪些欺敵或誘捕措施。
偵測設計：把攻擊技術對應到可觀測行為與 deception telemetry。
紅隊 / 藍隊演練：紅隊使用 ATT&CK 模擬攻擊，藍隊用 Engage 設計對抗環境。
安全控制覆蓋分析：檢查哪些 ATT&CK techniques 已有 Engage-based controls，哪些還沒有。
主動防禦策略：從被動偵測轉向主動操控攻擊者決策。

簡單比喻：

MITRE ATT&CK mapping 像是在回答：

「攻擊者可能走哪條路？」

MITRE Engage mapping 則進一步問：

「我們要在哪條路上放假門、假鑰匙、假寶箱、監視器，讓攻擊者自己暴露？」
```
