# Phase B4: Live Refresh / Auto-refresh / Demo Reliability
目標是讓 Dashboard 在展示時不用一直手動按 Refresh，而是可以自動更新 events.jsonl、detections.jsonl、mapping、report，並讓 demo 展示更穩定。
## Phase B4 目標
```
你目前已有：
  Phase B1: Streamlit Dashboard MVP
  Phase B2: demo_attack.sh controlled demo flow
  Phase B3: Attack Storyline + Demo Mode
B4 要補強成：
  1. Dashboard 可自動刷新
  2. Sidebar 可控制刷新秒數
  3. Demo Mode 顯示最後更新時間
  4. Demo 前後可以快速驗證事件數是否增加
  5. systemd service 保持穩定
  6. 避免重複執行 demo_attack.sh 導致事件累積過快卻無法辨識
```
### B4-1：先備份目前 Dashboard
```
cd /opt/deception-lab

cp dashboard/app.py dashboard/app.py.bak.phase-b3
ls -lh dashboard/app.py*
```

### B4-2：安裝 Streamlit auto-refresh 套件
- 進入 venv：
```
cd /opt/deception-lab
source .venv/bin/activate

# 安裝套件
pip install streamlit-autorefresh
```
- 確認：
```
python -c "from streamlit_autorefresh import st_autorefresh; print('OK')"
執行結果：
(.venv) lss@lss:/opt/deception-lab $ python -c "from streamlit_autorefresh import st_autorefresh; print('OK')"
2026-05-29 03:55:36.762 Thread 'MainThread': missing ScriptRunContext! This warning can be ignored when running in bare mode.
OK
預期：OK
```
- B4-3：修改 dashboard/app.py 加入 auto-refresh
```
nano /opt/deception-lab/dashboard/app.py

找到 import 區塊：
import pandas as pd
import streamlit as st
下面加入：
from streamlit_autorefresh import st_autorefresh
```
- 完整程式碼 app.py 
```
cat > /opt/deception-lab/dashboard/app.py <<'EOF'
import json
from pathlib import Path

import pandas as pd
import streamlit as st

try:
    from streamlit_autorefresh import st_autorefresh
except Exception:
    st_autorefresh = None

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
#   - mapping_report.md
# ============================================================

DEFAULT_DATA_DIR = Path("/opt/deception-lab/data/events")
DEFAULT_REPORT_DIR = Path("/opt/deception-lab/reports")
APP_TZ = "Asia/Taipei"

st.set_page_config(
    page_title="Deception Lab Dashboard",
    page_icon="🛡️",
    layout="wide",
    initial_sidebar_state="expanded",
)


# -----------------------------
# Loaders and time helpers
# -----------------------------
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
        df["_time_tw"] = ""
        return df

    src_col = existing[0]

    # Internal canonical time remains UTC for sorting and filtering.
    df["_time"] = pd.to_datetime(df[src_col], errors="coerce", utc=True)

    # Display-only Taiwan time.
    df["_time_tw"] = ""
    valid_mask = df["_time"].notna()
    if valid_mask.any():
        df.loc[valid_mask, "_time_tw"] = (
            df.loc[valid_mask, "_time"]
            .dt.tz_convert(APP_TZ)
            .dt.strftime("%Y-%m-%d %H:%M:%S")
        )

    return df


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
    return normalize_time_columns(df)


@st.cache_data(ttl=5)
def load_text(path: str) -> str:
    p = Path(path)
    if not p.exists():
        return ""
    return p.read_text(encoding="utf-8", errors="replace")


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
        start_ts = pd.Timestamp(start, tz="UTC")
        mask &= df["_time"] >= start_ts
    if end:
        # Include the full end date.
        end_ts = pd.Timestamp(end, tz="UTC") + pd.Timedelta(days=1) - pd.Timedelta(microseconds=1)
        mask &= df["_time"] <= end_ts
    return df[mask]


def safe_stringify(value) -> str:
    if isinstance(value, (dict, list)):
        return json.dumps(value, ensure_ascii=False, default=str)
    if pd.isna(value) if not isinstance(value, (dict, list)) else False:
        return ""
    return str(value)


def detect_category(row: pd.Series) -> str:
    text = " ".join(safe_stringify(v).lower() for v in row.values)
    event_type = str(row.get("event_type", "")).lower()
    source = str(row.get("source", "")).lower()

    if "honeycredential" in text or "honey credential" in text or "credential" in text or "password" in text:
        return "Honeycredential"
    if "honeyfile" in text or "honey file" in text or "secret" in text or "backup" in text:
        return "Honeyfile"
    if "ssh" in event_type or "cowrie" in source or "cowrie" in text or "login" in event_type:
        return "SSH / Cowrie"
    if "web" in event_type or "fake-web" in source or "http" in text or "admin" in text:
        return "Fake Web Admin"
    return "Other"


def extract_coverage_table(obj, framework_name: str) -> pd.DataFrame:
    """Best-effort parser for flexible coverage JSON structures."""
    rows = []

    def walk(x):
        if isinstance(x, dict):
            for k, v in x.items():
                if isinstance(v, dict):
                    row = {"id": k, "framework": framework_name}
                    row.update(v)
                    rows.append(row)
                    walk(v)
                elif isinstance(v, (int, float, str, bool)):
                    rows.append({"id": k, "framework": framework_name, "value": v})
                else:
                    walk(v)
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

    subset = [c for c in ["id", "name", "technique", "activity", "count", "covered"] if c in df.columns]
    if subset:
        df = df.drop_duplicates(subset=subset)
    return df


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
                if any(x in path.lower() for x in [".env", "wp-admin", "phpmyadmin", "server-status", "actuator", "admin"]):
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
                if command and command.lower() != "nan":
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
    story_df["time_tw"] = ""
    valid_mask = story_df["time"].notna()
    if valid_mask.any():
        story_df.loc[valid_mask, "time_tw"] = (
            story_df.loc[valid_mask, "time"]
            .dt.tz_convert(APP_TZ)
            .dt.strftime("%Y-%m-%d %H:%M:%S")
        )
    story_df = story_df.sort_values("time", ascending=True, na_position="last")
    return story_df


def build_file_freshness(base_data_dir: Path, base_report_dir: Path) -> pd.DataFrame:
    rows = []
    for label, path in [
        ("events.jsonl", base_data_dir / "events.jsonl"),
        ("detections.jsonl", base_data_dir / "detections.jsonl"),
        ("events_summary.json", base_data_dir / "events_summary.json"),
        ("mapping_summary.json", base_data_dir / "mapping_summary.json"),
        ("attack_coverage.json", base_data_dir / "attack_coverage.json"),
        ("engage_coverage.json", base_data_dir / "engage_coverage.json"),
        ("report.md", base_report_dir / "report.md"),
        ("timeline.md", base_report_dir / "timeline.md"),
        ("report.json", base_report_dir / "report.json"),
        ("mapping_report.md", base_report_dir / "mapping_report.md"),
    ]:
        p = Path(path)
        if p.exists():
            mtime = pd.Timestamp.fromtimestamp(p.stat().st_mtime, tz=APP_TZ)
            rows.append({
                "file": label,
                "exists": "yes",
                "last_modified_tw": mtime.strftime("%Y-%m-%d %H:%M:%S"),
                "size_kb": round(p.stat().st_size / 1024, 1),
            })
        else:
            rows.append({
                "file": label,
                "exists": "no",
                "last_modified_tw": "",
                "size_kb": 0,
            })
    return pd.DataFrame(rows)


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
    st.rerun()

st.sidebar.divider()
st.sidebar.caption("Live refresh")
auto_refresh = st.sidebar.checkbox("Enable auto refresh", value=False)
refresh_interval_sec = st.sidebar.selectbox(
    "Refresh interval",
    [5, 10, 15, 30, 60],
    index=1,
)

if auto_refresh:
    if st_autorefresh is not None:
        st_autorefresh(
            interval=refresh_interval_sec * 1000,
            key="deception_dashboard_autorefresh",
        )
    else:
        st.sidebar.warning("streamlit-autorefresh is not installed. Run: pip install streamlit-autorefresh")

st.sidebar.divider()
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
mapping_report_md = load_text(str(base_report_dir / "mapping_report.md"))
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
    min_time = min(all_times).tz_convert(APP_TZ).to_pydatetime()
    max_time = max(all_times).tz_convert(APP_TZ).to_pydatetime()
    st.sidebar.divider()
    st.sidebar.caption("Global date filter, Asia/Taipei")
    start_date = st.sidebar.date_input("Start date", min_time.date())
    end_date = st.sidebar.date_input("End date", max_time.date())
    events_df = filter_by_time(events_df, start_date, end_date)
    detections_df = filter_by_time(detections_df, start_date, end_date)

# -----------------------------
# Header
# -----------------------------
st.title("🛡️ Deception Lab Dashboard")
st.caption("Streamlit visualization for Cowrie SSH honeypot, fake web admin, honeycredentials, honeyfiles, MITRE ATT&CK, and MITRE Engage mapping.")
now_tw = pd.Timestamp.now(tz=APP_TZ).strftime("%Y-%m-%d %H:%M:%S")
st.caption(f"Dashboard last refreshed: {now_tw} Asia/Taipei")

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
        time_df["minute_tw"] = time_df["_time"].dt.tz_convert(APP_TZ).dt.floor("min")
        trend = time_df.groupby("minute_tw").size().reset_index(name="events")
        st.line_chart(trend.set_index("minute_tw"))
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
# Page: Attack Storyline
# -----------------------------
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

        st.subheader("Story phases")
        phase_counts = story_df["phase"].value_counts().sort_index()
        st.bar_chart(phase_counts)

        st.subheader("Timeline narrative")
        for _, item in story_df.tail(80).iterrows():
            severity = str(item.get("severity", "info")).lower()
            icon = "🔴" if severity == "high" else "🟠" if severity == "medium" else "🔵"
            time_text = item.get("time_tw", "") or "unknown time"

            with st.container(border=True):
                st.markdown(f"### {icon} {item.get('title', '')}")
                st.caption(
                    f"{time_text} Asia/Taipei | {item.get('phase', '')} | "
                    f"source={item.get('source', '')} | severity={item.get('severity', '')}"
                )
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
            display_cols = ["time_tw", "phase", "source", "title", "detail", "severity", "attack", "engage"]
            st.dataframe(story_df[[c for c in display_cols if c in story_df.columns]], use_container_width=True, height=520)

# -----------------------------
# Page: Demo Mode
# -----------------------------
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
        attack_cov = mapping_summary.get("attack_mapping_coverage", {}).get("coverage_percent", None)
        engage_cov = mapping_summary.get("engage_mapping_coverage", {}).get("coverage_percent", None)

    c3.metric("ATT&CK coverage", f"{attack_cov}%" if attack_cov is not None else "N/A")
    c4.metric("Engage coverage", f"{engage_cov}%" if engage_cov is not None else "N/A")

    st.subheader("Output file freshness")
    st.dataframe(build_file_freshness(base_data_dir, base_report_dir), use_container_width=True)

    st.subheader("Run controlled demo")
    st.code(
        """cd /opt/deception-lab

echo "[Before]"
wc -l data/events/events.jsonl
wc -l data/events/detections.jsonl

/opt/deception-lab/scripts/demo_attack.sh 127.0.0.1

echo "[After]"
wc -l data/events/events.jsonl
wc -l data/events/detections.jsonl
cat data/events/events_summary.json
""",
        language="bash",
    )

    st.subheader("LAN demo URLs")
    st.code(
        """Dashboard:      http://192.168.1.164:8501
Fake Web Admin: http://192.168.1.164:8080
Cowrie SSH:     192.168.1.164:2222
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
            "_time_tw", "_category", "eventid", "event_type", "src_ip", "source_ip", "username", "password",
            "path", "url", "method", "message", "rule_id", "rule", "severity", "_line_no"
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
        rule_col = first_existing_column(detections_df, ["rule_id", "rule", "rule_name", "detection", "name", "type"])
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
        preferred_cols = [
            "_time_tw", "severity", "level", "rule_id", "rule", "rule_name", "detection",
            "src_ip", "source_ip", "username", "path", "command", "artifact", "message", "_category", "_line_no"
        ]
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

    source_df = pd.concat(
        [events_df.assign(_source="events"), detections_df.assign(_source="detections")],
        ignore_index=True,
        sort=False,
    )

    if source_df.empty:
        st.info("No events or detections loaded.")
    else:
        text_df = source_df.copy()

        def row_to_text(row: pd.Series) -> str:
            values = []
            for value in row.values:
                values.append(safe_stringify(value))
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
            artifact_col = first_existing_column(honey_df, ["artifact", "file", "filename", "path", "username", "credential", "message"])
            if artifact_col:
                st.caption("Top artifacts / indicators")
                st.dataframe(value_counts_df(honey_df, artifact_col, 30), use_container_width=True)

            preferred_cols = [
                "_time_tw", "_source", "_category", "severity", "rule_id", "rule", "rule_name",
                "src_ip", "source_ip", "username", "password", "path", "filename", "artifact", "message", "_line_no"
            ]
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

# Run controlled demo
/opt/deception-lab/scripts/demo_attack.sh 127.0.0.1

# Run dashboard service commands
sudo systemctl status deception-dashboard.service --no-pager
sudo systemctl restart deception-dashboard.service

# Open dashboard from LAN
# http://192.168.1.164:8501
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

    tab1, tab2, tab3, tab4 = st.tabs(["report.md", "timeline.md", "mapping_report.md", "report.json"])
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
        if mapping_report_md:
            st.markdown(mapping_report_md)
        else:
            st.info("mapping_report.md not found.")
    with tab4:
        if report_json:
            st.json(report_json, expanded=False)
        else:
            st.info("report.json not found.")

st.sidebar.divider()
st.sidebar.caption("Tip: run Streamlit with --server.address 0.0.0.0 for LAN demo access.")
st.sidebar.caption(f"Display timezone: {APP_TZ}")
EOF
```
- 檢查PY語法：
```
python -m py_compile dashboard/app.py
```
- 重啟 Dashboard：
```
sudo systemctl restart deception-dashboard.service
sudo systemctl status deception-dashboard.service --no-pager

# 開啟：
http://192.168.1.164:8501

# 驗證重點：
  1. Sidebar 有 Enable auto refresh
  2. Sidebar 可選 Refresh interval
  3. Dashboard title 下方有 Dashboard last refreshed: ... Asia/Taipei
  4. Event Timeline 顯示 _time_tw
  5. Detections 顯示 _time_tw
  6. Attack Storyline 顯示 Asia/Taipei 時間
  7. Demo Mode 有 Output file freshness
  8. Raw Reports 多了 mapping_report.md
```
















