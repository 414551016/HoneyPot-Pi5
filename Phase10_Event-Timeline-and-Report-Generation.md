# 第十階段：事件時間線與報告產生
```
完成後，你會有：
[完成] timeline.md
[完成] report.md
[完成] report.json
[完成] generate_report.py
[完成] generate_timeline.py
[完成] generate_report.sh
[完成] show_report.sh
[完成] PHASE10_READY.md
```

### Step 10.1：進入專案目錄
```
cd /opt/deception-lab
```

### Step 10.2：確認第八、九階段輸出檔案存在
- 先確認必要檔案都存在：
```
ls -lah /opt/deception-lab/data/events

# 你應該看到：
events.jsonl
detections.jsonl
events_summary.json
mapping_summary.json
attack_coverage.json
engage_coverage.json

# 如果少了 mapping 相關檔案，先執行：
/opt/deception-lab/scripts/run_mapping.sh
輸出顯示：
Total events: 34
Total detections: 11
ATT&CK mapping coverage: 100.0%
Engage mapping coverage: 100.0%
這代表第八階段與第九階段的資料都已經刷新並可供第十階段報告產生器使用。

# 執行結果：
確認：Step 10.2 已完成，而且狀態正確。
你執行的結果顯示 /opt/deception-lab/data/events 內已經存在第八、九階段需要的所有輸出檔案：
/opt/deception-lab/data/events
├── attack_coverage.json
├── detections.jsonl
├── engage_coverage.json
├── events.jsonl
├── events_summary.json
└── mapping_summary.json

這正好對應 Step 10.2 要確認的檔案：

檔案	來源階段	狀態
events.jsonl	第八階段 parser	✅ 存在
detections.jsonl	第八階段 detection rules	✅ 存在
events_summary.json	第八階段 summary	✅ 存在
mapping_summary.json	第九階段 mapping analyzer	✅ 存在
attack_coverage.json	第九階段 ATT&CK coverage	✅ 存在
engage_coverage.json	第九階段 Engage coverage	✅ 存在



```





























