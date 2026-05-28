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
新增一頁：Demo Mode
這頁專門展示目前 demo 是否成功，包含：
  目前事件數
  目前 detection 數
  ATT&CK coverage
  Engage coverage
  Demo 操作流程
  可複製的 demo_attack.sh 指令
```
- 在 Attack Storyline 區塊後面加入 Demo Mode
```
elif page == "Demo Mode":
    st.subheader("Demo Mode")

    st.markdown(
        """
        This page is designed for live demonstration of the controlled deception workflow.

        **Demo flow:**

        1. Trigger controlled web interaction.
        2. Submit fake web credential.
        3. Probe honeyfile-like and scanner-like paths.
        4. Trigger Cowrie SSH login attempt.
        5. Regenerate parser, mapping, and reports.
        6. Refresh the dashboard and explain the resulting evidence.
        """
    )

    c1, c2, c3, c4 = st.columns(4)
    c1.metric("Total events", len(events_df))
    c2.metric("Total detections", len(detections_df))

    attack_cov = None
    engage_cov = None

    if isinstance(mapping_summary, dict):
        attack_cov = (
            mapping_summary
            .get("attack_mapping_coverage", {})
            .get("coverage_percent", None)
        )
        engage_cov = (
            mapping_summary
            .get("engage_mapping_coverage", {})
            .get("coverage_percent", None)
        )

    c3.metric("ATT&CK coverage", f"{attack_cov}%" if attack_cov is not None else "N/A")
    c4.metric("Engage coverage", f"{engage_cov}%" if engage_cov is not None else "N/A")

    st.subheader("Run controlled demo")

    st.code(
        """cd /opt/deception-lab
/opt/deception-lab/scripts/demo_attack.sh 127.0.0.1
""",
        language="bash",
    )

    st.subheader("LAN demo URL")

    st.code(
        """http://192.168.1.164:8501
""",
        language="text",
    )

    st.subheader("Expected evidence")

    expected = pd.DataFrame([
        {
            "Step": "Web probing",
            "Expected event": "web_request",
            "Expected detection": "WEB_SCANNER_PROBE",
            "ATT&CK": "T1595 Active Scanning",
            "Engage": "Expose / Expose Decoy Service",
        },
        {
            "Step": "Fake web credential submission",
            "Expected event": "web_login_attempt",
            "Expected detection": "WEB_HONEYCREDENTIAL_USED",
            "ATT&CK": "T1552 Unsecured Credentials",
            "Engage": "Elicit / Credential Collection",
        },
        {
            "Step": "Honeyfile-like path access",
            "Expected event": "web_honeyfile_access",
            "Expected detection": "WEB_HONEYFILE_ACCESS",
            "ATT&CK": "T1005 Data from Local System",
            "Engage": "Understand / Collect Adversary Behavior",
        },
        {
            "Step": "SSH login attempt",
            "Expected event": "ssh_login_failed",
            "Expected detection": "SSH_LOGIN_FAILED",
            "ATT&CK": "T1110 Brute Force",
            "Engage": "Expose / Expose Decoy Service",
        },
        {
            "Step": "SSH honeycredential login",
            "Expected event": "ssh_login_success",
            "Expected detection": "SSH_HONEYCREDENTIAL_LOGIN",
            "ATT&CK": "T1552 Unsecured Credentials",
            "Engage": "Elicit / Credential Collection",
        },
        {
            "Step": "SSH command behavior",
            "Expected event": "ssh_command",
            "Expected detection": "SSH_RECON_COMMAND / SSH_HONEYFILE_ACCESS / SSH_TOOL_TRANSFER_COMMAND",
            "ATT&CK": "T1082 / T1005 / T1105",
            "Engage": "Understand / Elicit / Affect",
        },
    ])

    st.dataframe(expected, use_container_width=True, height=300)

    st.subheader("Current detection rules")

    if not detections_df.empty:
        rule_col = first_existing_column(detections_df, ["rule_id", "rule", "rule_name", "detection"])
        if rule_col:
            demo_rules = detections_df[rule_col].fillna("<missing>").astype(str).value_counts().reset_index()
            demo_rules.columns = ["rule", "count"]
            st.bar_chart(demo_rules.set_index("rule"))
            st.dataframe(demo_rules, use_container_width=True)
        else:
            st.info("No detection rule column found.")
    else:
        st.info("No detections loaded.")
```
- B3-4：檢查 Python 語法：修改完後先不要急著重啟，先檢查語法：
```
cd /opt/deception-lab

source .venv/bin/activate
python -m py_compile dashboard/app.py

-------------------------------------------
執行結果：如果沒有輸出，代表語法通過。
```
- B3-5：重啟 Dashboard
```
因為你是 systemd 管理 Streamlit，所以執行：
sudo systemctl restart deception-dashboard.service

---------------------------------------------
確認狀態：
sudo systemctl status deception-dashboard.service --no-pager
執行結果：
(.venv) lss@lss:/opt/deception-lab $ sudo systemctl status deception-dashboard.service --no-pager
● deception-dashboard.service - Deception Lab Streamlit Dashboard
     Loaded: loaded (/etc/systemd/system/deception-dashboard.service; enabled; preset: enabled)
     Active: active (running) since Fri 2026-05-29 03:11:38 CST; 1min 58s ago
 Invocation: e451d6bc783541049727f0322333f906
   Main PID: 1967 (streamlit)
      Tasks: 8 (limit: 9621)
        CPU: 750ms
     CGroup: /system.slice/deception-dashboard.service
             └─1967 /opt/deception-lab/.venv/bin/python3 /opt/deception-lab/.venv/bin/streamlit run /opt/deception-lab/dashboa…

May 29 03:11:38 lss systemd[1]: Started deception-dashboard.service - Deception Lab Streamlit Dashboard.
May 29 03:11:39 lss streamlit[1967]: Collecting usage statistics. To deactivate, set browser.gatherUsageStats to false.
May 29 03:11:39 lss streamlit[1967]: 2026-05-29 03:11:39.269 Uvicorn server started on 0.0.0.0:8501
May 29 03:11:39 lss streamlit[1967]:   You can now view your Streamlit app in your browser.
May 29 03:11:39 lss streamlit[1967]:   Local URL: http://localhost:8501
May 29 03:11:39 lss streamlit[1967]:   Network URL: http://192.168.1.164:8501
May 29 03:11:39 lss streamlit[1967]:   External URL: http://125.229.21.105:8501

# 再開瀏覽器：http://192.168.1.164:8501
```
- 左側應該看到新增：
  ![B3 Deception Lab Dashboard](image/B3_Deception-Lab-Dashboard.png)






















