# Phase B3: Dashboard 視覺化強化
目標：把 Phase B2 的 controlled demo flow 變成可展示的攻擊故事線與 Demo Mode。
```
你現在已經有資料、parser、mapping、report、dashboard。Phase B3 的重點是讓展示時不用一直切 terminal，而是在 Dashboard 上直接看出：
攻擊者做了什麼
系統偵測到什麼
對應到哪些 ATT&CK 技術
對應到哪些 Engage 欺敵目標

Phase B3 操作總覽：建議分成 4 個小階段：
  B3-1: 備份目前 Dashboard
  B3-2: 新增 Attack Storyline 頁面
  B3-3: 新增 Demo Mode 頁面
  B3-4: 執行 demo_attack.sh 驗證 Dashboard 是否正確顯示
```
### B3-1：先備份目前 Dashboard
- 在 Raspberry Pi 上執行：先備份目前
```
cd /opt/deception-lab

cp dashboard/app.py dashboard/app.py.bak.phase-b2

----------------------------------------------------------------------
ls -lh dashboard/app.py*
執行結果：
lss@lss:/opt/deception-lab $ ls -lh dashboard/app.py*
-rwxrwxrwx 1 root root 23K May 26 02:02 dashboard/app.py
-rwxrwxr-x 1 lss  lss  23K May 29 02:39 dashboard/app.py.bak.phase-b2 (備份檔)

預期看到：
  dashboard/app.py
  dashboard/app.py.bak.phase-b2
```

### B3-2：新增 Attack Storyline 頁面
```
在 Dashboard 左側選單新增：
Attack Storyline
這頁要把 events.jsonl 和 detections.jsonl 整合成一條 demo 故事線，例如：
  1. Web probing observed
  2. Fake login / honeycredential submitted
  3. Honeyfile-like path accessed
  4. SSH login attempt observed
  5. Detection rules triggered
  6. ATT&CK / Engage mapping generated
```
- 修改位置 1：Sidebar 頁面清單
```
page = st.sidebar.radio(
    "Dashboard section",
    [
        "Overview",
        "Event Timeline",
        "Detections",
        "ATT&CK Coverage",
        "Engage Coverage",
        "Honey Artifacts",
        "Demo Runbook",
        "Raw Reports",
    ],
)

----------------------------------------
修正為：
page = st.sidebar.radio(
    "Dashboard section",
    [
        "Overview",
        "Attack Storyline",
        "Demo Mode",
        "Event Timeline",
        "Detections",
        "ATT&CK Coverage",
        "Engage Coverage",
        "Honey Artifacts",
        "Demo Runbook",
        "Raw Reports",
    ],
)
```
- 修改位置 2：新增 helper function
```
def build_storyline(events_df: pd.DataFrame, detections_df: pd.DataFrame) -> pd.DataFrame:
    rows = []

    def add_story_item(ts, phase, source, title, detail, severity="info", attack="", engage=""):
        rows.append({
            "time": ts,
            "phase": phase,
            "source": source,
            "title": title,
            "detail": detail,
            "severity": severity,
            "attack": attack,
            "engage": engage,
        })

    if not events_df.empty:
        for _, row in events_df.iterrows():
            event_type = str(row.get("event_type", "")).lower()
            source = str(row.get("source", "unknown"))
            ts = row.get("_time", pd.NaT)
            path = str(row.get("path", ""))
            username = str(row.get("username", ""))
            command = str(row.get("command", ""))
            severity = str(row.get("severity", "info"))

            if event_type == "web_request":
                if any(x in path.lower() for x in [".env", "wp-admin", "phpmyadmin", "server-status", "actuator"]):
                    add_story_item(
                        ts,
                        "1. Web probing",
                        source,
                        "Scanner-like web probing observed",
                        f"Requested suspicious path: {path}",
                        severity,
                    )

            elif event_type == "web_login_attempt":
                add_story_item(
                    ts,
                    "2. Web credential interaction",
                    source,
                    "Fake web login attempt observed",
                    f"Username: {username}",
                    severity,
                )

            elif event_type == "web_honeyfile_access":
                add_story_item(
                    ts,
                    "3. Web honeyfile access",
                    source,
                    "Honeyfile-like web path accessed",
                    f"Requested path: {path}",
                    severity,
                )

            elif event_type == "ssh_connection":
                add_story_item(
                    ts,
                    "4. SSH connection",
                    source,
                    "SSH honeypot connection observed",
                    f"Source IP: {row.get('src_ip', '')}",
                    severity,
                )

            elif event_type == "ssh_login_failed":
                add_story_item(
                    ts,
                    "5. SSH login attempt",
                    source,
                    "Failed SSH login attempt observed",
                    f"Username: {username}",
                    severity,
                )

            elif event_type == "ssh_login_success":
                add_story_item(
                    ts,
                    "6. SSH honeycredential used",
                    source,
                    "SSH honeycredential login succeeded",
                    f"Username: {username}",
                    severity,
                )

            elif event_type == "ssh_command":
                if command and command != "nan":
                    add_story_item(
                        ts,
                        "7. Post-login command behavior",
                        source,
                        "Command executed in Cowrie",
                        command,
                        severity,
                    )

    if not detections_df.empty:
        for _, row in detections_df.iterrows():
            ts = row.get("_time", pd.NaT)
            rule_id = str(row.get("rule_id", row.get("rule", "")))
            rule_name = str(row.get("rule_name", ""))
            severity = str(row.get("severity", "medium"))
            attack_mapping = row.get("attack_mapping", {})
            engage_mapping = row.get("engage_mapping", {})

            if isinstance(attack_mapping, dict):
                attack = f"{attack_mapping.get('technique_id', '')} {attack_mapping.get('technique', '')}".strip()
            else:
                attack = ""

            if isinstance(engage_mapping, dict):
                engage = f"{engage_mapping.get('goal', '')} / {engage_mapping.get('activity', '')}".strip()
            else:
                engage = ""

            add_story_item(
                ts,
                "8. Detection and mapping",
                str(row.get("source", "detection")),
                f"Detection triggered: {rule_id}",
                rule_name,
                severity,
                attack,
                engage,
            )

    story_df = pd.DataFrame(rows)
    if story_df.empty:
        return story_df

    story_df["time"] = pd.to_datetime(story_df["time"], errors="coerce", utc=True)
    story_df = story_df.sort_values("time", ascending=True, na_position="last")
    return story_df
```
- 修改位置 3：新增 Attack Storyline page
  在 Overview 區塊後面，也就是這段結束後：
```
elif page == "Attack Storyline":
    st.subheader("Attack Storyline")

    story_df = build_storyline(events_df, detections_df)

    if story_df.empty:
        st.info("No storyline data available.")
    else:
        c1, c2, c3, c4 = st.columns(4)
        c1.metric("Story items", len(story_df))
        c2.metric("High severity", int((story_df["severity"].astype(str).str.lower() == "high").sum()))
        c3.metric("Sources", story_df["source"].nunique())
        c4.metric("Phases", story_df["phase"].nunique())

        phase_counts = story_df["phase"].value_counts().sort_index()
        st.subheader("Story phases")
        st.bar_chart(phase_counts)

        st.subheader("Timeline narrative")

        for _, item in story_df.tail(80).iterrows():
            severity = str(item.get("severity", "info")).lower()
            icon = "🔴" if severity == "high" else "🟠" if severity == "medium" else "🔵"

            time_value = item.get("time", "")
            if pd.notna(time_value):
                time_text = str(time_value)
            else:
                time_text = "unknown time"

            with st.container(border=True):
                st.markdown(f"### {icon} {item.get('title', '')}")
                st.caption(f"{time_text} | {item.get('phase', '')} | source={item.get('source', '')} | severity={item.get('severity', '')}")
                st.write(item.get("detail", ""))

                col_a, col_b = st.columns(2)
                with col_a:
                    attack = item.get("attack", "")
                    if attack:
                        st.markdown(f"**ATT&CK:** `{attack}`")
                with col_b:
                    engage = item.get("engage", "")
                    if engage:
                        st.markdown(f"**Engage:** `{engage}`")

        with st.expander("Storyline table"):
            st.dataframe(story_df, use_container_width=True, height=520)
```
- B3-3：新增 Demo Mode 頁面
```
```




























