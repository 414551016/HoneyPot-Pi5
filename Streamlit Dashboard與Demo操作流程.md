# Streamlit Dashboard 與 Demo 操作流程
它會直接讀取你現有的 MVP 輸出，不要求改動目前 /opt/deception-lab pipeline。

### Streamlit Dashboard
- 建議放置路徑：
```
sudo mkdir -p /opt/deception-lab/dashboard
sudo nano /opt/deception-lab/dashboard/app.py
```
- 程式碼 app.py
```
import json
from pathlib import Path
from datetime import datetime
from collections import Counter, defaultdict

import pandas as pd
import streamlit as st

# ============================================================
# Raspberry Pi 5 Deception Lab - Streamlit Dashboard
# ============================================================
# Expected default input directory:
# /opt/deception-lab/data/events/
#   - events.jsonl
#   - detections.jsonl
#   - events_summary.json
#   - mapping_summary.json
#   - attack_coverage.json
#   - engage_coverage.json
# /opt/deception-lab/reports/
#   - report.md
#   - report.json
#   - timeline.md
# ============================================================

DEFAULT_DATA_DIR = Path("/opt/deception-lab/data/events")
DEFAULT_REPORT_DIR = Path("/opt/deception-lab/reports")

st.set_page_config(
    page_title="Deception Lab Dashboard",
    page_icon="🛡️",
    layout="wide",
    initial_sidebar_state="expanded",
)


# -----------------------------
# Loaders
# -----------------------------
@st.cache_data(ttl=5)
def load_json(path: str):
    p = Path(path)
    if not p.exists():
        return None
    try:
        return json.loads(p.read_text(encoding="utf-8"))
    except Exception as exc:
        return {"_load_error": str(exc)}


@st.cache_data(ttl=5)
def load_jsonl(path: str) -> pd.DataFrame:
    p = Path(path)
    rows = []
    if not p.exists():
        return pd.DataFrame()

    with p.open("r", encoding="utf-8", errors="replace") as f:
        for line_no, line in enumerate(f, start=1):
            line = line.strip()
            if not line:
                continue
            try:
                item = json.loads(line)
                if isinstance(item, dict):
                    item["_line_no"] = line_no
                    rows.append(item)
            except json.JSONDecodeError:
                rows.append({"_line_no": line_no, "_parse_error": line[:300]})

    df = pd.json_normalize(rows) if rows else pd.DataFrame()
    df = normalize_time_columns(df)
    return df


@st.cache_data(ttl=5)
def load_text(path: str) -> str:
    p = Path(path)
    if not p.exists():
        return ""
    return p.read_text(encoding="utf-8", errors="replace")


def normalize_time_columns(df: pd.DataFrame) -> pd.DataFrame:
    if df.empty:
        return df

    candidate_cols = [
        "timestamp",
        "time",
        "ts",
        "event_time",
        "@timestamp",
        "log.timestamp",
        "cowrie.timestamp",
    ]
    existing = [c for c in candidate_cols if c in df.columns]
    if not existing:
        df["_time"] = pd.NaT
        return df

    src_col = existing[0]
    df["_time"] = pd.to_datetime(df[src_col], errors="coerce", utc=True)
    return df


def first_existing_column(df: pd.DataFrame, candidates: list[str]) -> str | None:
    for c in candidates:
        if c in df.columns:
            return c
    return None


def value_counts_df(df: pd.DataFrame, col: str, top_n: int = 20) -> pd.DataFrame:
    if df.empty or col not in df.columns:
        return pd.DataFrame(columns=[col, "count"])
    out = df[col].fillna("<missing>").astype(str).value_counts().head(top_n).reset_index()
    out.columns = [col, "count"]
    return out


def filter_by_time(df: pd.DataFrame, start, end) -> pd.DataFrame:
    if df.empty or "_time" not in df.columns or df["_time"].isna().all():
        return df
    mask = pd.Series(True, index=df.index)
    if start:
        mask &= df["_time"] >= pd.Timestamp(start).tz_localize("UTC")
    if end:
        mask &= df["_time"] <= pd.Timestamp(end).tz_localize("UTC")
    return df[mask]


def detect_category(row: pd.Series) -> str:
    text = " ".join(str(v).lower() for v in row.dropna().values)
    if "honeycredential" in text or "honey credential" in text or "credential" in text or "password" in text:
        return "Honeycredential"
    if "honeyfile" in text or "honey file" in text or "secret" in text or "backup" in text:
        return "Honeyfile"
    if "ssh" in text or "cowrie" in text or "login" in text:
        return "SSH / Cowrie"
    if "web" in text or "http" in text or "admin" in text:
        return "Fake Web Admin"
    return "Other"


def extract_coverage_table(obj, framework_name: str) -> pd.DataFrame:
    """Best-effort parser for flexible coverage JSON structures."""
    rows = []

    def walk(x, prefix=""):
        if isinstance(x, dict):
            # Common format: {technique_id: {name, count, covered, ...}}
            for k, v in x.items():
                if isinstance(v, dict):
                    row = {"id": k, "framework": framework_name}
                    row.update(v)
                    rows.append(row)
                    walk(v, prefix=f"{prefix}.{k}" if prefix else str(k))
                elif isinstance(v, (int, float, str, bool)):
                    rows.append({"id": k, "framework": framework_name, "value": v})
                else:
                    walk(v, prefix=f"{prefix}.{k}" if prefix else str(k))
        elif isinstance(x, list):
            for i, item in enumerate(x):
                if isinstance(item, dict):
                    row = {"framework": framework_name}
                    row.update(item)
                    rows.append(row)
                else:
                    rows.append({"framework": framework_name, "id": f"item_{i}", "value": item})

    walk(obj)
    df = pd.DataFrame(rows)
    if df.empty:
        return df

    # Deduplicate noisy recursive rows when possible
    subset = [c for c in ["id", "name", "technique", "activity", "count", "covered"] if c in df.columns]
    if subset:
        df = df.drop_duplicates(subset=subset)
    return df


# -----------------------------
# Sidebar controls
# -----------------------------
st.sidebar.title("🛡️ Deception Lab")
st.sidebar.caption("Raspberry Pi 5 single-node active defense deception MVP")

base_data_dir = Path(st.sidebar.text_input("Data directory", str(DEFAULT_DATA_DIR)))
base_report_dir = Path(st.sidebar.text_input("Report directory", str(DEFAULT_REPORT_DIR)))

refresh = st.sidebar.button("Refresh data")
if refresh:
    st.cache_data.clear()

st.sidebar.divider()
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

# -----------------------------
# Data loading
# -----------------------------
events_df = load_jsonl(str(base_data_dir / "events.jsonl"))
detections_df = load_jsonl(str(base_data_dir / "detections.jsonl"))
events_summary = load_json(str(base_data_dir / "events_summary.json")) or {}
mapping_summary = load_json(str(base_data_dir / "mapping_summary.json")) or {}
attack_coverage = load_json(str(base_data_dir / "attack_coverage.json")) or {}
engage_coverage = load_json(str(base_data_dir / "engage_coverage.json")) or {}
report_md = load_text(str(base_report_dir / "report.md"))
timeline_md = load_text(str(base_report_dir / "timeline.md"))
report_json = load_json(str(base_report_dir / "report.json")) or {}

if not events_df.empty:
    events_df["_category"] = events_df.apply(detect_category, axis=1)
if not detections_df.empty:
    detections_df["_category"] = detections_df.apply(detect_category, axis=1)

# -----------------------------
# Global time filter
# -----------------------------
all_times = []
for df in [events_df, detections_df]:
    if not df.empty and "_time" in df.columns:
        all_times.extend(df["_time"].dropna().tolist())

if all_times:
    min_time = min(all_times).to_pydatetime()
    max_time = max(all_times).to_pydatetime()
    st.sidebar.divider()
    st.sidebar.caption("Global UTC time filter")
    start_date = st.sidebar.date_input("Start date", min_time.date())
    end_date = st.sidebar.date_input("End date", max_time.date())
    events_df = filter_by_time(events_df, start_date, end_date)
    detections_df = filter_by_time(detections_df, start_date, end_date)


# -----------------------------
# Header
# -----------------------------
st.title("🛡️ Deception Lab Dashboard")
st.caption("Streamlit visualization for Cowrie SSH honeypot, fake web admin, honeycredentials, honeyfiles, MITRE ATT&CK, and MITRE Engage mapping.")

missing = []
for p in [
    base_data_dir / "events.jsonl",
    base_data_dir / "detections.jsonl",
    base_data_dir / "events_summary.json",
    base_data_dir / "mapping_summary.json",
    base_data_dir / "attack_coverage.json",
    base_data_dir / "engage_coverage.json",
]:
    if not p.exists():
        missing.append(str(p))

if missing:
    with st.expander("Missing input files", expanded=False):
        st.warning("Some expected files were not found. The dashboard will still render available sections.")
        st.code("\n".join(missing))


# -----------------------------
# Page: Overview
# -----------------------------
if page == "Overview":
    c1, c2, c3, c4 = st.columns(4)
    c1.metric("Total events", len(events_df))
    c2.metric("Detections", len(detections_df))
    c3.metric("Event categories", events_df["_category"].nunique() if "_category" in events_df else 0)
    c4.metric("Report available", "Yes" if report_md or report_json else "No")

    st.subheader("Event category distribution")
    if not events_df.empty and "_category" in events_df.columns:
        cat_df = events_df["_category"].value_counts().reset_index()
        cat_df.columns = ["category", "count"]
        st.bar_chart(cat_df.set_index("category"))
    else:
        st.info("No event data available.")

    st.subheader("Events over time")
    if not events_df.empty and "_time" in events_df.columns and not events_df["_time"].isna().all():
        time_df = events_df.dropna(subset=["_time"]).copy()
        time_df["minute"] = time_df["_time"].dt.floor("min")
        trend = time_df.groupby("minute").size().reset_index(name="events")
        st.line_chart(trend.set_index("minute"))
    else:
        st.info("No timestamp column found in events.jsonl.")

    st.subheader("Current summaries")
    left, right = st.columns(2)
    with left:
        st.caption("events_summary.json")
        st.json(events_summary, expanded=False)
    with right:
        st.caption("mapping_summary.json")
        st.json(mapping_summary, expanded=False)


# -----------------------------
# Page: Event Timeline
# -----------------------------
elif page == "Event Timeline":
    st.subheader("Event timeline")

    if events_df.empty:
        st.info("No events loaded.")
    else:
        search = st.text_input("Search events", "")
        view_df = events_df.copy()
        if search:
            mask = view_df.astype(str).apply(lambda col: col.str.contains(search, case=False, na=False)).any(axis=1)
            view_df = view_df[mask]

        preferred_cols = [
            "_time", "_category", "eventid", "event_type", "src_ip", "source_ip", "username", "password",
            "path", "url", "method", "message", "rule", "severity", "_line_no"
        ]
        cols = [c for c in preferred_cols if c in view_df.columns]
        if not cols:
            cols = list(view_df.columns[:15])
        view_df = view_df.sort_values("_time", ascending=False, na_position="last") if "_time" in view_df.columns else view_df
        st.dataframe(view_df[cols], use_container_width=True, height=560)

        with st.expander("Raw selected event fields"):
            row_no = st.number_input("Row index", min_value=0, max_value=max(len(view_df) - 1, 0), value=0)
            if len(view_df) > 0:
                st.json(view_df.iloc[int(row_no)].dropna().to_dict(), expanded=True)


# -----------------------------
# Page: Detections
# -----------------------------
elif page == "Detections":
    st.subheader("Detection statistics")

    if detections_df.empty:
        st.info("No detections loaded.")
    else:
        c1, c2, c3 = st.columns(3)
        severity_col = first_existing_column(detections_df, ["severity", "level", "risk"])
        rule_col = first_existing_column(detections_df, ["rule", "rule_name", "detection", "name", "type"])
        src_col = first_existing_column(detections_df, ["src_ip", "source_ip", "ip", "remote_host"])

        c1.metric("Total detections", len(detections_df))
        c2.metric("Unique rules", detections_df[rule_col].nunique() if rule_col else 0)
        c3.metric("Unique source IPs", detections_df[src_col].nunique() if src_col else 0)

        left, right = st.columns(2)
        with left:
            st.caption("Detections by category")
            st.bar_chart(detections_df["_category"].value_counts())
        with right:
            if severity_col:
                st.caption("Detections by severity")
                st.bar_chart(detections_df[severity_col].fillna("<missing>").astype(str).value_counts())
            else:
                st.info("No severity column found.")

        if rule_col:
            st.subheader("Top detection rules")
            st.dataframe(value_counts_df(detections_df, rule_col, 30), use_container_width=True)

        st.subheader("Detection table")
        preferred_cols = ["_time", "severity", "level", "rule", "rule_name", "detection", "src_ip", "source_ip", "username", "artifact", "message", "_category", "_line_no"]
        cols = [c for c in preferred_cols if c in detections_df.columns]
        if not cols:
            cols = list(detections_df.columns[:15])
        view_df = detections_df.sort_values("_time", ascending=False, na_position="last") if "_time" in detections_df.columns else detections_df
        st.dataframe(view_df[cols], use_container_width=True, height=520)


# -----------------------------
# Page: ATT&CK Coverage
# -----------------------------
elif page == "ATT&CK Coverage":
    st.subheader("MITRE ATT&CK coverage")
    attack_df = extract_coverage_table(attack_coverage, "ATT&CK")

    if attack_df.empty:
        st.info("No ATT&CK coverage data loaded.")
        st.json(attack_coverage, expanded=False)
    else:
        metric_cols = st.columns(3)
        metric_cols[0].metric("Rows", len(attack_df))
        if "id" in attack_df.columns:
            metric_cols[1].metric("Unique IDs", attack_df["id"].nunique())
        if "covered" in attack_df.columns:
            metric_cols[2].metric("Covered", int(attack_df["covered"].fillna(False).astype(bool).sum()))

        count_col = first_existing_column(attack_df, ["count", "events", "detections", "hits", "value"])
        name_col = first_existing_column(attack_df, ["name", "technique", "tactic", "id"])
        if count_col and name_col:
            plot_df = attack_df[[name_col, count_col]].copy()
            plot_df[count_col] = pd.to_numeric(plot_df[count_col], errors="coerce").fillna(0)
            plot_df = plot_df.groupby(name_col)[count_col].sum().sort_values(ascending=False).head(20)
            st.bar_chart(plot_df)

        st.dataframe(attack_df, use_container_width=True, height=560)
        with st.expander("Raw attack_coverage.json"):
            st.json(attack_coverage, expanded=False)


# -----------------------------
# Page: Engage Coverage
# -----------------------------
elif page == "Engage Coverage":
    st.subheader("MITRE Engage coverage")
    engage_df = extract_coverage_table(engage_coverage, "Engage")

    if engage_df.empty:
        st.info("No Engage coverage data loaded.")
        st.json(engage_coverage, expanded=False)
    else:
        metric_cols = st.columns(3)
        metric_cols[0].metric("Rows", len(engage_df))
        if "id" in engage_df.columns:
            metric_cols[1].metric("Unique IDs", engage_df["id"].nunique())
        if "covered" in engage_df.columns:
            metric_cols[2].metric("Covered", int(engage_df["covered"].fillna(False).astype(bool).sum()))

        count_col = first_existing_column(engage_df, ["count", "events", "detections", "hits", "value"])
        name_col = first_existing_column(engage_df, ["name", "activity", "approach", "goal", "id"])
        if count_col and name_col:
            plot_df = engage_df[[name_col, count_col]].copy()
            plot_df[count_col] = pd.to_numeric(plot_df[count_col], errors="coerce").fillna(0)
            plot_df = plot_df.groupby(name_col)[count_col].sum().sort_values(ascending=False).head(20)
            st.bar_chart(plot_df)

        st.dataframe(engage_df, use_container_width=True, height=560)
        with st.expander("Raw engage_coverage.json"):
            st.json(engage_coverage, expanded=False)


# -----------------------------
# Page: Honey Artifacts
# -----------------------------
elif page == "Honey Artifacts":
    st.subheader("Honeycredential / Honeyfile events")

    source_df = pd.concat([events_df.assign(_source="events"), detections_df.assign(_source="detections")], ignore_index=True, sort=False)
    if source_df.empty:
        st.info("No events or detections loaded.")
    else:
        text_df = source_df.copy()

        # Build a safe per-row text blob. Some parsed JSON fields may be numeric,
        # float, NaN, list, or dict, so avoid pandas string aggregation here.
        def row_to_text(row: pd.Series) -> str:
            values = []
            for value in row.values:
                if pd.isna(value) if not isinstance(value, (list, dict)) else False:
                    continue
                if isinstance(value, (dict, list)):
                    values.append(json.dumps(value, ensure_ascii=False, default=str))
                else:
                    values.append(str(value))
            return " ".join(values).lower()

        text_blob = text_df.apply(row_to_text, axis=1)
        honey_mask = (
            text_blob.str.contains("honeycredential", na=False)
            | text_blob.str.contains("honey credential", na=False)
            | text_blob.str.contains("credential", na=False)
            | text_blob.str.contains("password", na=False)
            | text_blob.str.contains("honeyfile", na=False)
            | text_blob.str.contains("honey file", na=False)
            | text_blob.str.contains("secret", na=False)
            | text_blob.str.contains("backup", na=False)
        )
        honey_df = text_df[honey_mask].copy()

        c1, c2, c3 = st.columns(3)
        c1.metric("Honey artifact events", len(honey_df))
        c2.metric("Honeycredential-like", int(text_blob[honey_mask].str.contains("credential|password", regex=True, na=False).sum()))
        c3.metric("Honeyfile-like", int(text_blob[honey_mask].str.contains("honeyfile|secret|backup", regex=True, na=False).sum()))

        if honey_df.empty:
            st.info("No honeycredential/honeyfile-like records found by keyword scan.")
        else:
            artifact_col = first_existing_column(honey_df, ["artifact", "file", "path", "username", "credential", "message"])
            if artifact_col:
                st.caption("Top artifacts / indicators")
                st.dataframe(value_counts_df(honey_df, artifact_col, 30), use_container_width=True)

            preferred_cols = ["_time", "_source", "_category", "severity", "rule", "src_ip", "source_ip", "username", "password", "path", "artifact", "message", "_line_no"]
            cols = [c for c in preferred_cols if c in honey_df.columns]
            if not cols:
                cols = list(honey_df.columns[:18])
            honey_df = honey_df.sort_values("_time", ascending=False, na_position="last") if "_time" in honey_df.columns else honey_df
            st.dataframe(honey_df[cols], use_container_width=True, height=560)


# -----------------------------
# Page: Demo Runbook
# -----------------------------
elif page == "Demo Runbook":
    st.subheader("Demo operation flow")

    st.markdown(
        """
        This page is intended for controlled, local demonstration only.

        **Recommended demo story:**

        1. Show the architecture: Raspberry Pi 5, Docker Compose, Cowrie SSH honeypot, fake web admin panel, parser, detection rules, and generated reports.
        2. Start with a clean dashboard and show current baseline counts.
        3. Trigger a controlled SSH interaction against Cowrie on port `2222`.
        4. Trigger a controlled fake web admin interaction on port `8080`.
        5. Access or submit honeycredential-like data.
        6. Access honeyfile-like paths or filenames.
        7. Re-run the parser/report generator.
        8. Refresh this dashboard.
        9. Explain the resulting timeline, detections, ATT&CK coverage, and Engage coverage.
        """
    )

    st.code(
        """# On Raspberry Pi 5
cd /opt/deception-lab

# Start services
sudo docker compose up -d

# Watch raw events / detections
sudo tail -f /opt/deception-lab/data/events/events.jsonl
sudo tail -f /opt/deception-lab/data/events/detections.jsonl

# After controlled demo actions, regenerate reports
# Adjust this command to your existing parser/report command.
python3 scripts/parse_events.py
python3 scripts/generate_report.py

# Run dashboard
streamlit run dashboard/app.py --server.address 0.0.0.0 --server.port 8501
""",
        language="bash",
    )

    st.subheader("Presenter checklist")
    st.checkbox("Docker Compose services are running")
    st.checkbox("Cowrie reachable on port 2222 in the controlled lab network")
    st.checkbox("Fake web admin reachable on port 8080 in the controlled lab network")
    st.checkbox("events.jsonl and detections.jsonl are being updated")
    st.checkbox("ATT&CK and Engage mapping JSON files exist")
    st.checkbox("Dashboard refresh shows new evidence")

    st.subheader("Demo narrative")
    st.info(
        "Key message: the platform does not merely log attacks; it turns deceptive interaction into structured evidence, detection signals, ATT&CK mapping, Engage mapping, and an explainable timeline."
    )


# -----------------------------
# Page: Raw Reports
# -----------------------------
elif page == "Raw Reports":
    st.subheader("Generated reports")

    tab1, tab2, tab3 = st.tabs(["report.md", "timeline.md", "report.json"])
    with tab1:
        if report_md:
            st.markdown(report_md)
        else:
            st.info("report.md not found.")
    with tab2:
        if timeline_md:
            st.markdown(timeline_md)
        else:
            st.info("timeline.md not found.")
    with tab3:
        if report_json:
            st.json(report_json, expanded=False)
        else:
            st.info("report.json not found.")


st.sidebar.divider()
st.sidebar.caption("Tip: run Streamlit with --server.address 0.0.0.0 for LAN demo access.")

```
- 安裝依賴：
```
cd /opt/deception-lab
python3 -m venv .venv
source .venv/bin/activate
pip install streamlit pandas

# 啟動 Streamlit
streamlit run dashboard/app.py --server.address 0.0.0.0 --server.port 8501
```
- 啟動 Streamlit
```
cd /opt/deception-lab
source .venv/bin/activate

streamlit run dashboard/app.py \
  --server.address 0.0.0.0 \
  --server.port 8501 \
  --server.headless true
```

### 讓 Raspberry Pi 開機後自動啟動 Streamlit Dashboard
- Deception Lab IP
```
http://192.168.1.164:8501
```
- 假設你的專案與 venv 是：
```
/opt/deception-lab
/opt/deception-lab/.venv
/opt/deception-lab/dashboard/app.py

# 1. 建立 systemd service
sudo nano /etc/systemd/system/deception-dashboard.service
## 貼上：
[Unit]
Description=Deception Lab Streamlit Dashboard
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=lss
Group=lss
WorkingDirectory=/opt/deception-lab
Environment="PATH=/opt/deception-lab/.venv/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
ExecStart=/opt/deception-lab/.venv/bin/streamlit run /opt/deception-lab/dashboard/app.py --server.address 0.0.0.0 --server.port 8501 --server.headless true
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target

## 存檔離開。
如果你的 Linux 使用者不是 lss，請先確認：whoami
然後把 service 裡的這兩行改成你的帳號：
User=lss
Group=lss

# 2. 重新載入 systemd
sudo systemctl daemon-reload

# 3. 設定開機自動啟動
sudo systemctl enable deception-dashboard.service

# 4. 立即啟動
sudo systemctl start deception-dashboard.service

# 5. 檢查狀態
sudo systemctl status deception-dashboard.service

## 正常會看到類似：Active: active (running)

# 6. 然後從你的電腦開：
http://192.168.1.164:8501
```
- 常用維護指令
```
# 重啟 Dashboard：
sudo systemctl restart deception-dashboard.service

# 停止 Dashboard：
sudo systemctl stop deception-dashboard.service

# 看即時 log：
journalctl -u deception-dashboard.service -f

# 看最近 100 行 log：
journalctl -u deception-dashboard.service -n 100 --no-pager
-------
執行結果：
lss@lss:~ $ journalctl -u deception-dashboard.service -n 100 --no-pager
May 26 02:17:48 lss systemd[1]: Started deception-dashboard.service - Deception Lab Streamlit Dashboard.
May 26 02:17:50 lss streamlit[1200]: Collecting usage statistics. To deactivate, set browser.gatherUsageStats to false.
May 26 02:17:51 lss streamlit[1200]: 2026-05-26 02:17:51.012 Uvicorn server started on 0.0.0.0:8501
May 26 02:17:51 lss streamlit[1200]:   You can now view your Streamlit app in your browser.
May 26 02:17:51 lss streamlit[1200]:   Local URL: http://localhost:8501
May 26 02:17:51 lss streamlit[1200]:   Network URL: http://192.168.1.164:8501
May 26 02:17:51 lss streamlit[1200]:   External URL: http://125.229.21.105:8501

```
### 建議再確認 Docker 服務也會開機啟動
- 如果你的 deception lab 是 Docker Compose 啟動，建議也確認 containers 有 restart policy。查看：
```
cd /opt/deception-lab
sudo docker compose ps

--------------------------
# 執行結果：
lss@lss:~ $ sudo docker compose ps
no configuration file provided: not found
這是正常的：你現在在家目錄 ~ 執行，所以 Docker Compose 找不到 docker-compose.yml。

lss@lss:/opt/deception-lab $ sudo docker compose ps
NAME                 IMAGE                    COMMAND                  SERVICE    CREATED      STATUS          PORTS
deception-cowrie     cowrie/cowrie:latest     "/cowrie/cowrie-env/…"   cowrie     2 days ago   Up 14 minutes   0.0.0.0:2222->2222/tcp, [::]:2222->2222/tcp, 2223/tcp
deception-fake-web   deception-lab-fake-web   "gunicorn --bind 0.0…"   fake-web   2 days ago   Up 14 minutes   0.0.0.0:8080->8080/tcp, [::]:8080->8080/tcp
確認：你的 Docker Compose 服務是正常的。
目前狀態：
deception-cowrie      Up 17 minutes   0.0.0.0:2222->2222/tcp
deception-fake-web    Up 17 minutes   0.0.0.0:8080->8080/tcp
這代表：
Cowrie SSH honeypot 正在跑
Fake Web Admin 正在跑
Docker Compose 專案位置正確：/opt/deception-lab


```
- 檢查 docker-compose.yml 裡有沒有設定自動重啟：
```
cd /opt/deception-lab
grep -n "restart:" docker-compose.yml

-------
# 執行結果：
lss@lss:/opt/deception-lab $ grep -n "restart:" docker-compose.yml
5:    restart: unless-stopped
28:    restart: unless-stopped
很好，這代表 docker-compose.yml 已經有自動重啟設定：
也就是你的兩個 Docker services：
cowrie
fake-web
都會在 Docker daemon 啟動後自動恢復。這部分已完成。
```
- 確認 Streamlit Dashboard systemd service 是否也完成
```
sudo systemctl status deception-dashboard.service

---------------------------
# 執行結果：
lss@lss:/opt/deception-lab $ sudo systemctl status deception-dashboard.service
● deception-dashboard.service - Deception Lab Streamlit Dashboard
     Loaded: loaded (/etc/systemd/system/deception-dashboard.service; enabled; preset: enabled)
     Active: active (running) since Tue 2026-05-26 02:17:48 CST; 21min ago
 Invocation: 9d81822ddcd04641adfd1f8299bccabd
   Main PID: 1200 (streamlit)
      Tasks: 12 (limit: 9626)
        CPU: 4.088s
     CGroup: /system.slice/deception-dashboard.service
             └─1200 /opt/deception-lab/.venv/bin/python3 /opt/deception-lab/.venv/bin/streamlit run /opt/deception-la>

May 26 02:17:48 lss systemd[1]: Started deception-dashboard.service - Deception Lab Streamlit Dashboard.
May 26 02:17:50 lss streamlit[1200]: Collecting usage statistics. To deactivate, set browser.gatherUsageStats to fals>
May 26 02:17:51 lss streamlit[1200]: 2026-05-26 02:17:51.012 Uvicorn server started on 0.0.0.0:8501
May 26 02:17:51 lss streamlit[1200]:   You can now view your Streamlit app in your browser.
May 26 02:17:51 lss streamlit[1200]:   Local URL: http://localhost:8501
May 26 02:17:51 lss streamlit[1200]:   Network URL: http://192.168.1.164:8501
May 26 02:17:51 lss streamlit[1200]:   External URL: http://125.229.21.105:8501
lines 1-17/17 (END)
確認完成：Streamlit Dashboard 已經設定成開機自動啟動，而且目前正在正常執行。
狀態重點：
Loaded: loaded (/etc/systemd/system/deception-dashboard.service; enabled)
Active: active (running)
Uvicorn server started on 0.0.0.0:8501
Network URL: http://192.168.1.164:8501
這代表：
PASS: systemd service 已建立
PASS: service 已 enabled，開機會自動啟動
PASS: Streamlit 目前正在執行
PASS: Dashboard 監聽 0.0.0.0:8501
PASS: LAN 可用網址為 http://192.168.1.164:8501


現在你的平台開機後應該會自動恢復：
Raspberry Pi boot
→ Docker daemon starts
→ Cowrie starts on 2222
→ Fake Web Admin starts on 8080
→ Streamlit Dashboard starts on 8501
```
- 最後做一次 reboot 驗證：
```
sudo reboot

重開後等 1 分鐘，再執行：
cd /opt/deception-lab
sudo docker compose ps
sudo systemctl status deception-dashboard.service --no-pager
ss -lntp | grep -E "2222|8080|8501"

----------------------------------------
# 執行結果：
Using username "lss".
lss@192.168.1.167's password:
Linux lss 6.18.29+rpt-rpi-2712 #1 SMP PREEMPT Debian 1:6.18.29-1+rpt1 (2026-05-1                                     2) aarch64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Tue May 26 02:27:06 2026 from 192.168.1.1
lss@lss:~ $ cd /opt/deception-lab
lss@lss:/opt/deception-lab $ sudo docker compose ps
[sudo] password for lss:
NAME                 IMAGE                    COMMAND                  SERVICE                                         CREATED      STATUS          PORTS
deception-cowrie     cowrie/cowrie:latest     "/cowrie/cowrie-env/…"   cowrie                                          2 days ago   Up 27 seconds   0.0.0.0:2222->2222/tcp, [::]:2222->2222/tcp, 2223                                     /tcp
deception-fake-web   deception-lab-fake-web   "gunicorn --bind 0.0…"   fake-web                                        2 days ago   Up 28 seconds   0.0.0.0:8080->8080/tcp, [::]:8080->8080/tcp
lss@lss:/opt/deception-lab $ sudo systemctl status deception-dashboard.service -                                     -no-pager
● deception-dashboard.service - Deception Lab Streamlit Dashboard
     Loaded: loaded (/etc/systemd/system/deception-dashboard.service; enabled; p                                     reset: enabled)
     Active: active (running) since Tue 2026-05-26 02:43:07 CST; 36s ago
 Invocation: 0869fe85669241ac8789c9ac600eb06d
   Main PID: 1216 (streamlit)
      Tasks: 8 (limit: 9621)
        CPU: 800ms
     CGroup: /system.slice/deception-dashboard.service
             └─1216 /opt/deception-lab/.venv/bin/python3 /opt/deception-lab/.ve…

May 26 02:43:07 lss systemd[1]: Started deception-dashboard.service - Dece…oard.
May 26 02:43:09 lss streamlit[1216]: Collecting usage statistics. To deacti…lse.
May 26 02:43:09 lss streamlit[1216]: 2026-05-26 02:43:09.987 Uvicorn server…8501
May 26 02:43:10 lss streamlit[1216]:   You can now view your Streamlit app …ser.
May 26 02:43:10 lss streamlit[1216]:   Local URL: http://localhost:8501
May 26 02:43:10 lss streamlit[1216]:   Network URL: http://192.168.1.164:8501
May 26 02:43:10 lss streamlit[1216]:   External URL: http://125.229.21.105:8501
Hint: Some lines were ellipsized, use -l to show in full.
lss@lss:/opt/deception-lab $ ss -lntp | grep -E "2222|8080|8501"
LISTEN 0      2048         0.0.0.0:8501      0.0.0.0:*    users:(("streamlit",pi                                     d=1216,fd=6))
LISTEN 0      4096         0.0.0.0:8080      0.0.0.0:*                                                               
LISTEN 0      4096         0.0.0.0:2222      0.0.0.0:*                                                               
LISTEN 0      4096            [::]:8080         [::]:*                                                               
LISTEN 0      4096            [::]:2222         [::]:*  
```





