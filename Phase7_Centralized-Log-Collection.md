# 第七階段：集中式 log 收集
```
# 目前你的 log 來源有三種：
1. Cowrie Docker logs
2. Fake Web access log
3. Fake Web auth log

#目前位置大致是：
Cowrie:
  docker compose logs cowrie
  /opt/deception-lab/data/logs/cowrie/cowrie-docker.log

Fake Web:
  /opt/deception-lab/data/logs/web/web_access.jsonl
  /opt/deception-lab/data/logs/web/web_auth.jsonl

# 第七階段會整理成：
/opt/deception-lab/data/collected/
├── cowrie-docker.log
├── web_access.jsonl
├── web_auth.jsonl
├── source_manifest.json
└── collection_summary.txt
```

### Step 7.1：進入專案目錄
```
cd /opt/deception-lab
```

### Step 7.2：建立集中 log 目錄
- 執行：
  ```
  mkdir -p \
    /opt/deception-lab/data/collected \
    /opt/deception-lab/data/archive \
    /opt/deception-lab/data/raw
  ```
  你應該看到類似：
  ```
  lss@lss:/opt/deception-lab $ tree -L 2 /opt/deception-lab/data
  /opt/deception-lab/data
  ├── archive
  ├── collected
  ├── events
  ├── logs
  │   ├── cowrie
  │   └── web
  ├── raw
  └── samples
      └── uploads
  
  10 directories, 0 files
  ```



















