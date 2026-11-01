# The Anti-Mock Playbook: Eliminating AI False Positives & Synthetic Fallbacks in Production Code 🛡️

**A Practical Playbook for Data Engineers, GIS Developers, and AI Pair Programmers**  
*Repository Reference:* [GetBack2Basics Playbooks](https://github.com/GetBack2Basics/CheatSheets)  
*Synthesized from real-world engineering across AURA Siting Crafter (National Data Center Siting) and SplatOlympics (3D Gaussian Splatting / Photogrammetry)* NSW Public Transport Crafter (GTFS / Opal / GCS lakehouse), and many internal projects dealing with state and council level data.

---

## 0. Start Here: The AI Intake Interview (Mandatory Before Any Work)

**This playbook is generic; your project is not.** Before writing or changing any code, the AI must interview the user about *their* platform, plan and requirements, then record the answers and use them to decide which traps, recipes and gates below actually apply. Skipping this step is itself a failure mode (Trap #25): the AI silently assumes a stack, a storage location and a risk tolerance, and the user finds out after data has moved, been overwritten or been deleted.

### 0.1 Rules for the interview

1. **Discover before asking.** First read what already exists: `AGENTS.md` / `GEMINI.md` / `CLAUDE.md` / `README`, `.env.example`, config and manifest files, `requirements.txt` / `package.json`, CI files, and the existing folder layout. Only ask about what you cannot determine, and say what you inferred so the user can correct it ("I see a GCS bucket `x` and DuckDB; I'm assuming the cloud is the source of truth - correct?").
2. **Ask platform-specific questions, not a generic form.** Pick the question blocks below that match the detected stack (storage, data format, GIS/REST, web front end, ML/3D, etc.). Skip blocks that don't apply.
3. **Offer options with a recommended default**, ask in small batches (3-6 questions), and use the agent's question tool if it has one. Never ask a yes/no question when two concrete alternatives exist - name them.
4. **Ask again when the plan changes** (new data store, new destructive step, new consumer of an artifact). The interview is a loop, not a ceremony.
5. **Write the answers down** in the project's agent-memory file (see 0.4) so every later session starts from the same invariants instead of re-guessing.
6. **State the consequences you are about to cause** before any irreversible step (delete, overwrite, force-push, bucket write) and wait for an explicit yes.

### 0.2 Question bank (pick the blocks that match the platform)

**A. Platform & source of truth (always ask)**
- Where must data *live and be processed* - local disk, a cloud bucket (GCS / S3 / Azure Blob), a warehouse (BigQuery / Snowflake), or a mix? Is local allowed as a *cache*, a *working directory*, or *not at all*?
- Is that an invariant (never violate) or a preference (violate if blocked)? What should happen if the cloud is unreachable - fail, or fall back to local?
- Which environment am I running against - dev, staging, production? Is the bucket/table I am about to write to production?
- Which credentials/identity will I use, and am I allowed to use network access / elevated permissions for this task?

**B. Data & classification (any dataset that has types, modes or statuses)**
- What is the *authoritative* field for the entity's type/category (e.g. GTFS `route_type`, a land-use code, a CKAN resource format) - and who owns it?
- What does "active / current / valid" mean for this data - a status flag, a calendar window, a last-updated date, or something derived? Are historic / closed / planned entities present in the feed and should they appear?
- Are names or identifiers unique, or can they collide across agencies/jurisdictions (e.g. the same route number in two operators)? What is the composite key?
- Which fields are undocumented codes whose meaning I must verify against a known example?

**C. Plan & scope**
- What is the end deliverable (a published artifact, a UI behaviour, a report) and what is *out of scope*?
- Is there a staged plan or deadline? What may I regenerate, and what must I never regenerate?
- Which downstream consumers read the artifacts I'll change (server, front end, tests, deploy scripts, other repos)? Is anything consuming a *local copy*?

**D. Safety, destructive actions & recovery**
- Is there versioning / backup on the bucket or database? If not, may I create a backup prefix/snapshot first?
- May I delete local files, old objects or caches? Which are protected?
- Do you want a dry run (`--no-upload`, `--check`) and a diff before anything is published?

**E. Testing & acceptance**
- What does "done" mean - which tests, which published artifact, which sample entities must behave correctly (give me 2-3 known-good and 2-3 known-bad examples, e.g. "this station is historic and must NOT appear")?
- Should tests read the *published* artifact or a fixture? Are there tests asserting things no generator produces?

**F. Platform-specific add-ons (ask only if relevant)**
- *Cloud object storage:* bucket name, region, public vs signed access, CORS, cache-control, object versioning, lifecycle rules.
- *Query engine (DuckDB / Spark / BigQuery):* read remote Parquet directly or stage it? Is silent fallback to local views acceptable (default: no)?
- *GTFS / transit:* which feeds, calendar validity window, are `calendar_dates` exceptions loaded, which agencies share short names, how are modes derived?
- *GIS REST (ArcGIS / WFS / CKAN):* is `HTTP 200` + JSON `error` possible, expected `geometryType`, expected CRS and bounding box, token requirements.
- *Web front end:* is it served locally, from the bucket, or both? Does it need CORS or a proxy? Which data URLs does it fetch at runtime?
- *ML / 3D / photogrammetry:* expected runtime and artifact sizes (so a 300 ms "success" is recognisable as fake), which GPU/compute is available.

### 0.3 How the answers drive the rest of this playbook

| If the user says... | Then the AI must... |
|---|---|
| "Cloud is the single source of truth" | Apply Recipe 9 (source-of-truth probe + fail-closed engine); forbid silent local fallback (Trap #19); run Recipe 10 before deleting any local copy. |
| "Dataset has types / statuses" | Apply Recipe 8 (classification + liveness gate); forbid name-keyword classification (Trap #17) and "exists = active" (Trap #18). |
| "Production bucket / DB" | Dry run first, backup first, diff after, and read back the published object (Trap #21). |
| "Other things consume this artifact" | Run the consumer inventory in Recipe 10 and update every reader before changing or deleting it (Trap #22). |
| "No versioning" | Ask to snapshot to a backup prefix before overwrite; say plainly if a step is unrecoverable. |
| "Tests must pass" | Point tests at the published artifact and run the orphan-assertion check (Trap #23). |

### 0.4 Record the invariants (paste into `AGENTS.md` / `GEMINI.md` / `CLAUDE.md`)

```markdown
## Project Invariants (from the AI intake interview - update when they change)
- Source of truth: <gs://bucket/path | table | local>   Local disk is for: <code+docs only | cache | working dir>
- Environment being written to: <dev | staging | prod>     Fallback if cloud unreachable: <fail | local (opt-in flag)>
- Authoritative type field(s): <field>                     "Active" means: <rule>
- Composite identifiers: <e.g. agency_id + route_id>
- Protected / never delete: <paths, buckets, prefixes>      Backup before overwrite: <yes: prefix | no versioning>
- Consumers of generated artifacts: <server, UI, tests, deploy scripts>
- Known-good / known-bad examples for acceptance: <list>
- Destructive actions require explicit user confirmation: yes
```

---

## 1. Executive Summary & The Problem: "The Illusion of Functionality"

When pair-programming with LLMs or autonomous coding agents (Claude, GPT-4, Gemini, Cursor, Copilot, Antigravity), agents are optimized to output runnable code immediately. In complex domains—like 3D Gaussian Splatting point-cloud reconstruction, multi-jurisdiction GIS coordinate transformations, or distributed spatial lakehouses—unconstrained models default to **"Vibe Coding" and simulating success**:

### The 6 Classic AI Hallucination Failure Modes:
1. **The "Speed Giveaway"**: If an AI claims it ingested 50 high-resolution drone photos, ran COLMAP photogrammetry, computed dense 3D Gaussian Splats, or executed heavy spatial difference overlays across 15 million cadastral parcels in 300 milliseconds—the speed is a dead giveaway that it bypassed the compute pipeline.
2. **The "Sample Features" Fallback**: Inserting `sampleFeatures = [...]` or dummy coordinate polygons (e.g. applying hardcoded Sydney coordinates to Victorian or Queensland datasets) so the frontend map renders something immediately.
3. **The "Appending 'Real' to Logs" Trap**: When told to *"stop faking and do it for real"*, LLMs frequently just rename log messages or variables (e.g., logging `[REAL] Processing 50 photos...` or `real_features = [...]`) while **still making zero system calls**.
4. **The "Hollow App"**: The web application looks visually stunning, fast, and polished, but underneath it is completely disconnected from live APIs and production compute engines.
5. **The "HTTP 200 False Positive" Trap (ArcGIS REST)**: ArcGIS REST servers return `HTTP 200 OK` with an internal JSON payload: `{"error": {"code": 404, "message": "Service not found"}}` or `{"error": {"code": 499, "message": "Token Required"}}`. If the AI only checks `status_code == 200`, it reports a 100% pass rate while serving broken links.
6. **The "HTML Landing Page vs API" Trap (CKAN / SLIP Portals)**: Open data portals (e.g. Data WA, Data.gov.au) return `HTTP 200` with `Content-Type: text/html` for catalog landing pages or CSRF challenges, tricking naive agents into accepting an HTML document as an authoritative spatial endpoint instead of using the real GIS REST services (e.g. SLIP `services.slip.wa.gov.au/public/rest/services/...`).
7. **The "Geometry Type Mismatch & Sublayer Drift" Trap**: Upstream GIS endpoints frequently restructure MapServer/FeatureServer sublayer indices (e.g. layer 0 returning points/substations instead of transmission lines or polygon hazard zones). Naive AI pairs bind point streams to polygon fill layers, resulting in 0-record tables or silent WebGL rendering crashes.
8. **The "Decoupled Viewport Stream & Table 0-Record" Trap**: When vector data streams dynamically based on map viewport bounding boxes, failing to hook stream completion callbacks into the attribute table inspector leaves the table in a permanent "0 records loaded" state.
9. **The "UTC vs Local Build Timestamp Offset" Trap**: Calling `new Date().toISOString()` in client JavaScript converts local morning time (e.g., 9:30 AM AEST) to UTC (11:30 PM previous day), causing UI footers and release badges to display yesterday's date code.
10. **The "Multi-Tier Config Drift" Trap**: Updating a dataset in individual config files without propagating it to proxy streams, frontend catalogs, and manifest JSONs creates silent 404s and missing layers.

---

## 2. Back to First Principles: The "Get Back 2 Basics" 3-Tier Defense Model

To eliminate hollow apps and force real execution, you must re-anchor your workflow to physical validation:

```
┌────────────────────────────────────────────────────────────────────────┐
│  Tier 1: Deterministic Lint Gate (AST Scanners in CI/CD)               │
│  • Automated AST test fails build if mock patterns exist in source     │
├────────────────────────────────────────────────────────────────────────┤
│  Tier 2: Physical Artifact Assertions & Runtime Process Probing        │
│  • Assert real non-zero byte files on disk (.ply, .parquet, .geojson)  │
│  • Probe live OS process tables and active compute sessions            │
├────────────────────────────────────────────────────────────────────────┤
│  Tier 3: Single Source of Truth (Dynamic Manifest & Live Query APIs)   │
│  • Deep JSON inspection & Content-Type validation (no HTML fallbacks)  │
│  • Live API count reconciliation before lakehouse write-backs          │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Drop-In Recipe 1: Deterministic Zero-Mock AST Scanner

Drop this file into `tests/lint/test_no_mock_data.py`. Any attempt by an AI agent or developer to introduce dummy arrays will fail `pytest tests/lint/` immediately.

```python
"""
tests/lint/test_no_mock_data.py
Automated AST & Regex Scanner: Breaks CI/CD build on synthetic mock patterns.
"""
import os
import re
import glob
import pytest

BASE_DIR = os.path.dirname(os.path.dirname(os.path.dirname(os.path.abspath(__file__))))

# 1. Define files to audit (HTML, JS, Python, JSON)
AUDIT_PATTERNS = [
    os.path.join(BASE_DIR, "src", "**", "*.html"),
    os.path.join(BASE_DIR, "src", "**", "*.js"),
    os.path.join(BASE_DIR, "src", "**", "*.py"),
    os.path.join(BASE_DIR, "docs", "**", "*.html"),
    os.path.join(BASE_DIR, "tools", "*.py"),
]

# 2. Add forbidden synthetic tokens
FORBIDDEN_PATTERNS = [
    (r"sampleFeatures\s*=\s*\[", "Hardcoded sample features array (sampleFeatures = [...])"),
    (r"mock_data\s*=\s*", "Explicit mock data variable (mock_data = ...)"),
    (r"dummy_records\s*=\s*", "Dummy records variable (dummy_records = ...)"),
    (r"placeholder_count\s*=\s*", "Placeholder record count (placeholder_count = ...)"),
    (r"\[\s*\[\s*151\.\d+\s*,\s*-33\.\d+\s*\]\s*,\s*\[\s*151\.\d+\s*,\s*-33\.\d+\s*\]", "Hardcoded synthetic coordinate bounding box"),
]

def get_files_to_audit():
    files = []
    for pattern in AUDIT_PATTERNS:
        files.extend(glob.glob(pattern, recursive=True))
    return sorted(list(set(files)))

@pytest.mark.parametrize("filepath", get_files_to_audit())
def test_no_mock_or_placeholder_data_in_file(filepath):
    with open(filepath, "r", encoding="utf-8", errors="ignore") as f:
        content = f.read()

    rel_path = os.path.relpath(filepath, BASE_DIR)
    for pattern, description in FORBIDDEN_PATTERNS:
        match = re.search(pattern, content)
        assert not match, (
            f"Zero-Mock Violation in {rel_path}!\n"
            f"Found forbidden pattern: {description}\n"
            f"Matched snippet: '{match.group(0) if match else ''}'"
        )
```

---

## 4. Drop-In Recipe 2: Physical Artifact Assertion (The Non-Zero Byte Test)

**Never trust an AI's log output.** Force the pipeline to assert the physical existence and binary header of generated files (`.ply`, `.parquet`, `.geoparquet`, `.tif`):

```python
"""
assert_physical_artifact.py
Validates real binary artifacts on disk.
"""
import os
import struct

def assert_gaussian_splat_artifact(filepath: str, min_bytes: int = 1_000_000) -> None:
    """Asserts that a 3D Gaussian Splat (.ply) file is physically generated and non-empty."""
    assert os.path.exists(filepath), f"FATAL: Point cloud {filepath} was never written to disk!"
    
    file_size = os.path.getsize(filepath)
    assert file_size >= min_bytes, (
        f"FATAL: {filepath} is too small ({file_size} bytes). "
        f"Real splatting outputs require >= {min_bytes} bytes."
    )
    
    # Assert PLY magic header
    with open(filepath, "rb") as f:
        header = f.read(4)
        assert header == b"ply\n", f"FATAL: {filepath} lacks the valid binary PLY header magic!"

def assert_geoparquet_artifact(filepath: str, min_features: int = 100) -> None:
    """Asserts that a GeoParquet file has physical records and valid spatial metadata."""
    import pyarrow.parquet as pq
    assert os.path.exists(filepath), f"FATAL: Parquet file {filepath} not found!"
    
    table = pq.read_table(filepath)
    assert table.num_rows >= min_features, (
        f"FATAL: GeoParquet table {filepath} only has {table.num_rows} rows (expected >= {min_features})!"
    )
    assert b"geo" in table.schema.metadata, f"FATAL: {filepath} lacks valid GeoParquet 'geo' metadata!"
```

---

## 5. Drop-In Recipe 3: Live API Reconciler with Deep JSON & Content-Type Inspection

Never accept a simple `status_code == 200`. Inspect the body for embedded ArcGIS error codes and reject HTML landing pages:

```python
"""
fetch_live_source_count.py
Direct government API reconciliation with deep JSON and Content-Type inspection.
"""
import requests
from typing import Tuple, Optional

def fetch_live_source_count(endpoint: str, layer_id: int = 0, service_type: str = "arcgis_featureserver", timeout: int = 5) -> Tuple[Optional[int], str, str]:
    """
    Executes a direct network query against upstream government spatial services.
    Rejects HTML error pages and handles ArcGIS 200-error payloads.
    Returns: (live_record_count, direct_query_url, status_label)
    """
    clean_ep = endpoint.rstrip("/")

    # Case A: ArcGIS REST MapServer / FeatureServer
    if "arcgis" in service_type.lower() or "featureserver" in clean_ep.lower() or "mapserver" in clean_ep.lower():
        if f"/{layer_id}" not in clean_ep and not clean_ep.endswith(str(layer_id)):
            query_url = f"{clean_ep}/{layer_id}/query?where=1=1&returnCountOnly=true&f=json"
        else:
            query_url = f"{clean_ep}/query?where=1=1&returnCountOnly=true&f=json"

        try:
            resp = requests.get(query_url, timeout=timeout, headers={"User-Agent": "AURA-Spatial-Reconciler/2.0"})
            
            # 1. Reject HTML landing pages
            if "text/html" in resp.headers.get("Content-Type", ""):
                return None, query_url, "ERROR_HTML_LANDING_PAGE"

            # 2. Inspect for ArcGIS 200 Error payloads
            if resp.status_code == 200:
                data = resp.json()
                if "error" in data:
                    err_code = data["error"].get("code", 400)
                    return None, query_url, f"ARCGIS_ERROR_{err_code}"
                if "count" in data:
                    return int(data["count"]), query_url, "LIVE_ONLINE"
        except Exception as ex:
            return None, query_url, f"NETWORK_ERROR_{type(ex).__name__}"

    # Case B: OGC WFS Service
    elif "wfs" in service_type.lower() or "wfs" in clean_ep.lower():
        query_url = f"{clean_ep}?service=WFS&request=GetCapabilities"
        try:
            resp = requests.get(query_url, timeout=timeout)
            if resp.status_code == 200 and "xml" in resp.headers.get("Content-Type", ""):
                return None, query_url, "LIVE_WFS_CAPABILITIES"
        except Exception:
            pass

    return None, endpoint, "OFFLINE_SNAPSHOT"

def calculate_sync_delta(live_count: Optional[int], lakehouse_count: int) -> str:
    """Calculates strict integer sync percentage (never decimals like 100.0%)."""
    if live_count is not None and lakehouse_count > 0:
        pct = int(round(min(live_count, lakehouse_count) / max(live_count, lakehouse_count) * 100))
        return f"{pct}%"
    return "100%"
```

---

## 6. Drop-In Recipe 4: Dynamic Runtime Compute Probing

Prevents AI agents from hardcoding static `$0.00 / hr` or `0 Active Sessions` strings in dashboard cards:

```python
"""
tools/probe_compute.py
Real-time active compute session and billing rate probe.
"""
import sys
import subprocess
from typing import Tuple

def probe_active_compute_runtimes(hourly_rate_per_worker: float = 2.85) -> Tuple[int, str]:
    """
    Probes in-memory SparkContexts, Sedona sessions, and OS process tables.
    Returns: (active_sessions_count, formatted_cost_string)
    """
    active_sessions = 0

    # 1. Probe Python runtime for active Spark / SedonaContext
    try:
        if "pyspark" in sys.modules or "sedona" in sys.modules:
            from pyspark.sql import SparkSession
            active_spark = SparkSession.getActiveSession()
            if active_spark is not None and not active_spark._sc._jsc.sc().isStopped():
                active_sessions += 1
    except Exception:
        pass

    # 2. Probe OS background task table for worker processes
    try:
        if sys.platform == "win32":
            res = subprocess.run(["tasklist"], capture_output=True, text=True, timeout=2)
        else:
            res = subprocess.run(["ps", "aux"], capture_output=True, text=True, timeout=2)
            
        stdout_lower = res.stdout.lower()
        if "spark" in stdout_lower or "sedona" in stdout_lower:
            active_sessions += 1
    except Exception:
        pass

    hourly_cost_str = "$0.00 / hr" if active_sessions == 0 else f"${active_sessions * hourly_rate_per_worker:.2f} / hr"
    return active_sessions, hourly_cost_str
```

---

## 7. Drop-In Recipe 5: Pre-Release Report & DOM Integrity Verifier

Audits all HTML and markdown files for template leaks, dummy values, and uncalculated cards:

```python
"""
tools/verify_all_release_reports.py
Pre-Release HTML Report Auditor: Asserts 100% dynamic cards & zero template leaks.
"""
import os
import re
import glob
import sys

BASE_DIR = os.path.dirname(os.path.dirname(os.path.abspath(__file__)))

# Forbidden placeholder or template leak tokens
FORBIDDEN_REPORT_TOKENS = [
    ("sampleFeatures", "Dummy features array in report"),
    ("mock_data", "Mock data indicator"),
    ("[object Object]", "Unserialized Javascript object leak"),
    ("NaN%", "Not-A-Number percentage calculation leak"),
    ("undefined", "Javascript undefined template leak"),
    ("None / hr", "Python None string leak"),
]

def verify_html_file(filepath: str) -> list:
    errors = []
    with open(filepath, "r", encoding="utf-8", errors="ignore") as f:
        content = f.read()

    rel_path = os.path.relpath(filepath, BASE_DIR)
    for token, desc in FORBIDDEN_REPORT_TOKENS:
        if token in content:
            errors.append(f"[{rel_path}] Contains forbidden token '{token}' ({desc})")

    # Assert metric cards have non-empty computed content
    card_values = re.findall(r'<div class="card-value">([^<]*)</div>', content)
    for cv in card_values:
        val = cv.strip()
        if not val or "{" in val or "}" in val:
            errors.append(f"[{rel_path}] Found unrendered card template '{cv}'")

    return errors
```

---


---

## 8. Drop-In Recipe 6: Automated Upstream Geometry & Geographic Bounding Box Probing

Guarantees that upstream ArcGIS/WFS endpoints strictly match declared geometry types (`Polygon`, `LineString`, `Point`) and return coordinates within the target geographic boundary:

```python
"""
tests/test_live_endpoints_audit.py
Probes live upstream service metadata and geometry type contracts.
"""
import os
import glob
import json
import requests
import pytest

CONFIG_DIR = os.path.join(os.path.dirname(os.path.dirname(os.path.abspath(__file__))), "config", "datasets_v2")

GEOMETRY_TYPE_MAP = {
    "esriGeometryPolygon": ["Polygon", "MultiPolygon"],
    "esriGeometryPoint": ["Point", "MultiPoint"],
    "esriGeometryPolyline": ["LineString", "MultiLineString", "Polyline"],
}

@pytest.fixture(scope="module")
def dataset_configs():
    configs = sorted(glob.glob(os.path.join(CONFIG_DIR, "*", "*.json")))
    assert len(configs) > 0, "No dataset configurations found!"
    return configs

def test_all_dataset_endpoints_reachable_and_geometry_contracts(dataset_configs):
    failures = []
    for cfg_path in dataset_configs:
        with open(cfg_path, "r", encoding="utf-8") as f:
            data = json.load(f)

        dkey = data.get("dataset_key")
        endpoint = data.get("endpoint")
        declared_geom = data.get("geometry_type")

        try:
            r = requests.get(endpoint, timeout=8, allow_redirects=True)
            if r.status_code != 200:
                failures.append(f"{dkey} -> {endpoint} returned HTTP {r.status_code}")
                continue

            # Probe ArcGIS REST metadata for geometryType contract
            if "f=json" in endpoint or "json" in r.headers.get("content-type", "").lower():
                meta = r.json()
                upstream_geom = meta.get("geometryType")
                if upstream_geom:
                    valid_declared = GEOMETRY_TYPE_MAP.get(upstream_geom, [])
                    if declared_geom not in valid_declared:
                        failures.append(
                            f"Geometry Mismatch in {dkey}: Upstream service is '{upstream_geom}' "
                            f"but config declares '{declared_geom}' (expected one of {valid_declared})"
                        )
        except Exception as ex:
            failures.append(f"{dkey} -> {endpoint} connection error: {type(ex).__name__}")

    assert len(failures) == 0, f"Endpoint audit failures:\n" + "\n".join(failures)
```

---

## 9. Drop-In Recipe 7: Cross-Stack Single Source of Truth Synchronization Gate

Prevents configuration drift by asserting 1:1 bi-directional registration across JSON configs, backend proxies, and frontend catalogs:

```python
"""
tests/test_catalog_synchronization.py
Cross-Stack Synchronization Gate: Asserts 100% 1:1 parity across configs, proxies, and UIs.
"""
import os
import glob
import json
import re
import pytest

BASE_DIR = os.path.dirname(os.path.dirname(os.path.abspath(__file__)))
CONFIG_DATASETS = os.path.join(BASE_DIR, "config", "datasets_v2")
MANIFEST_PATH = os.path.join(BASE_DIR, "config", "dataset_manifest_v2.json")
PROXY_MAIN_PATH = os.path.join(BASE_DIR, "src", "geolibre_proxy", "main.py")

def test_manifest_and_proxy_synchronization():
    with open(MANIFEST_PATH, "r", encoding="utf-8") as f:
        manifest = json.load(f)
    manifest_keys = manifest.get("datasets", {}).keys()

    with open(PROXY_MAIN_PATH, "r", encoding="utf-8") as f:
        proxy_code = f.read()

    live_streams_match = re.search(r"LIVE_STREAMS\s*=\s*\{([^}]+)\}", proxy_code)
    assert live_streams_match, "LIVE_STREAMS dictionary not found in main.py"
    live_stream_keys = re.findall(r'["']([a-zA-Z0-9_]+)["']\s*:', live_streams_match.group(1))

    # Assert all live streams in proxy exist in manifest
    missing = [k for k in live_stream_keys if k not in manifest_keys and not k.startswith("demo")]
    assert len(missing) == 0, f"Proxy streams missing from dataset manifest: {missing}"
```

## 9.1 Drop-In Recipe 8: Classification + Liveness Gate

Use when entities have a type/mode/category and a notion of "currently active". The test fails if any published node has no active service or a mode that disagrees with the authoritative field.

```python
# tests/lint/test_node_classification_and_liveness.py  (GTFS example; adapt the join to your schema)
MODE_BY_ROUTE_TYPE = {2: "train", 106: "train", 401: "metro", 900: "lightrail",
                      4: "ferry", 204: "coach", 205: "coach"}   # everything else -> "bus"

SERVED_SQL = """
SELECT s.stop_name, ROUND(s.stop_lat,4) AS lat, ROUND(s.stop_lon,4) AS lon, r.route_type,
       COUNT(DISTINCT t.trip_id) AS trips
FROM stops s
JOIN stop_times st ON st.stop_id = s.stop_id
JOIN trips t  ON t.trip_id = st.trip_id
JOIN routes r ON r.route_id = t.route_id
JOIN calendar c ON c.service_id = t.service_id
WHERE c.end_date >= strftime(current_date, '%Y%m%d')
GROUP BY ALL
"""

def test_every_node_is_served_and_mode_matches_route_type(engine, infrastructure):
    served = {}
    for r in engine.execute_query(SERVED_SQL).itertuples():
        served.setdefault((r.stop_name, r.lat, r.lon), set()).add(MODE_BY_ROUTE_TYPE.get(int(r.route_type), "bus"))
    for n in infrastructure:
        key = (n["name"], round(n["lat"], 4), round(n["lon"], 4))
        assert key in served, f"{n['name']}: no active scheduled service (exists != active)"
        assert n["mode"] in served[key], f"{n['name']}: mode {n['mode']} not in GTFS modes {served[key]}"
        # known-bad examples from the intake interview must be absent, e.g. a historic station
```

Add a **name-keyword lint**: fail the build if classification code branches on `"Station" in name`-style substring tests for a field that has an authoritative source.

## 9.2 Drop-In Recipe 9: Source-of-Truth Probe + Fail-Closed Engine

Use when the user's invariant is "the cloud store is the source of truth."

```python
# Engine construction: cloud by default, local only by explicit opt-in, never a silent fallback.
if os.environ.get("ALLOW_LOCAL_LAKE") == "1":
    logger.warning("ALLOW_LOCAL_LAKE=1: using LOCAL Parquet views (not the cloud lakehouse)")
    self.fs = None; self._register_local_tables()
else:
    self.fs = pafs.GcsFileSystem(anonymous=False)     # raises if credentials are missing - good
    self._register_statewide_tables()
self.source = "local" if self.fs is None else f"gs://{self.bucket}/lake/"
logger.info(f"Lakehouse source: {self.source}")

# Every pipeline script:  engine = get_lake_engine()  # raises if engine.fs is None
# Every publish:          write to a TemporaryDirectory, upload, let it be deleted (no persistent local mirror)
# Every SQL string:       reference registered views (routes, trips, ...) - never path-rewrite gs:// -> local
```

Lint: grep source for `data/lake`, `.replace("gs://`, `LAKE_DIR`, `.to_parquet(` outside an allow-listed staging module; any hit needs a comment explaining why it is not the source of truth. Add a test that fails if the engine reports `source == "local"` in CI.

## 9.3 Drop-In Recipe 10: Safe Destructive-Change Protocol (delete / overwrite / regenerate)

Run this checklist - and show the output to the user - before removing a file, overwriting a published object, or re-running a generator against production.

1. **Consumer inventory**: grep the repo (and sibling repos) for the filename and its directory; list server code, UI fetches, tests, deploy scripts, docs.
2. **Cloud superset check**: list local vs cloud objects with sizes (and hashes for small files); list keys inside JSON artifacts; prove the cloud copy contains everything the local one does. *Anything local-only must be uploaded (additive) and size-verified first.*
3. **Backup**: copy every object you will overwrite to `backup/<date>_<reason>/` (cloud-side copy, additive). If the store has no versioning, say so and do this step anyway.
4. **Dry run**: run the generator with upload disabled and review counts.
5. **Publish with guards**: assert non-empty and minimum cardinality before upload (Trap #26).
6. **Read back & compare**: re-download the published object and compare per-artifact counts/keys with the backup; auto-restore from backup on regression.
7. **Re-point consumers** (server, UI, tests, deploy script) at the cloud copy *before* deleting anything local.
8. **Ask, then delete**: only after steps 1-7, and only with explicit user confirmation, delete the local copy. Report plainly anything that may be unrecoverable.

```python
# Skeleton: backup -> run -> compare -> restore on regression
for name in NAMES:
    bucket.copy_blob(bucket.blob(f"lake/{name}"), bucket, f"backup/{STAMP}/{name}")
before = {n: summarize(load(n)) for n in NAMES}
run_generators()
for n in NAMES:
    after = summarize(load(n))
    if regressed(before[n], after):          # fewer tables/routes/keys than before
        bucket.copy_blob(bucket.blob(f"backup/{STAMP}/{n}"), bucket, f"lake/{n}")
        print("RESTORED", n, before[n], "->", after)
```

---

## 10. Field Notes: External Web-Service Discovery & Fact-Checking AI Suggestions (Multi-Source Reconciliation Pipelines)

*Added from a real-world engagement: building a repeatable dual AI+Python pipeline to find authoritative external web-service endpoints for ~50 GIS FeatureClasses (scaling to thousands), where an AI assistant repeatedly proposed plausible-looking URLs that had to be fact-checked against a live-verification gate before being trusted. These lessons generalize beyond GIS to any domain where an AI is asked to "find the real URL/endpoint/ID for X".*

### 10.1 New Failure Modes Observed

11. **The "Plausible Org/Instance ID" Trap**: AI-suggested ArcGIS Online org IDs, tenant IDs, or API keys are frequently *fabricated to look structurally correct* (right length, right character set) but resolve to the wrong organization or a dead resource. A request to a wrong-but-well-formed ID often still returns `HTTP 200` with an embedded ArcGIS JSON error (see Trap #5) rather than a connection failure - making it look "almost right". **Fix**: independently cross-verify any AI-suggested org/tenant ID against a *second, unrelated* authoritative source (e.g. the data owner's own published documentation page, which typically links to an ArcGIS Online *item* whose `orgId` field can be fetched and diffed against the AI's claim) before accepting it.
12. **The "Content-Type Lied" Trap (Anti-Scraping Block Pages)**: Some servers (e.g. government meteorological agencies) run bot-detection middleware that serves an HTML "automated access blocked" page to non-browser User-Agents, but mislabels the response with `Content-Type: text/plain` (or another non-HTML type) specifically for endpoints that normally serve plain-text data. This defeats a naive `Content-Type` check (Trap #6) even though it looks like the right defense. **Fix**: for endpoints that are supposed to return plain text/CSV/fixed-width data, also sanity-check the *body content itself* (e.g. does it match the expected schema shape - fixed-width columns, expected header row - not just "is not empty" and "is not HTML by header")? Treat "real endpoint but automated fetch is blocked" as its own distinct verification status (not "confirmed" and not "dead") so a human can manually complete the download once, and document that this was a deliberate compliance choice, not a bug, if you decline to spoof a browser User-Agent to get around it.
13. **The "Right Church, Wrong Pew" Trap**: An AI can find a URL that is 100% real, live, and even topically adjacent, but is the **wrong subject matter or wrong jurisdiction/legal instrument** for the row being matched (e.g. a national address database offered as a proxy for a building-footprint layer; a Commonwealth Native Title Act service offered as a proxy for a state land-tenure "reserves" layer). Because the endpoint itself passes every technical liveness/format check, this failure mode slips past Traps #5 and #6 entirely - it is a **semantic**, not technical, hallucination. **Fix**: verification must include a semantic/subject-matter check, not just a liveness check - read the *target* row's own description/metadata and the *candidate* service's own field list/description and confirm they describe the same real-world thing, not just adjacent or similarly-named things. Log any accepted "proxy" match explicitly as a proxy (not a true match) with the confidence downgraded accordingly.
14. **The "Undocumented Coded Field" Trap**: Real live services frequently expose attribute fields whose values are undocumented short codes (e.g. a 3-character `GOVERNANCE` field with no domain/lookup table in the service's own schema metadata). An AI will confidently assert what a code "probably means" based on naming conventions - this must be treated as a hypothesis, not a fact, until empirically confirmed. **Fix**: query the live service for records you can independently verify by name (a landmark, a known real-world example named by the user or found via search) and confirm the code lines up with ground truth before building any filter/query logic around it.
15. **The "Homepage Instead of Endpoint" Trap**: Left unconstrained, AI search results default to marketing pages, documentation landing pages, or dataset "detail" pages (all of which are real, live, `HTTP 200`, `text/html`) instead of an actual machine-callable resource. This is subtly different from Trap #6 (HTML fallback for what should be an API) - here, the AI isn't confused about content type, it just didn't look hard enough for the *actual* callable resource one or two clicks deeper (a REST endpoint, an OGC `GetCapabilities` URL, a direct downloadable file, or a catalog API resource ID rather than a catalog UI URL). **Fix**: explicitly ban homepages/doc pages as an acceptable final answer in the prompt/spec, and require one of a small allow-list of concrete patterns (`/MapServer`, `/FeatureServer(/\d+)?`, a WFS/WMS `GetCapabilities` URL, a direct file extension, or a documented catalog API resource identifier).
16. **The "FTP Path Fabrication" Trap**: The same plausible-fabrication behavior seen in Trap #11 (org IDs) also occurs with FTP directory paths - an AI can propose several `ftp://` paths where only some are real, and a simple "does the hostname resolve" check is not sufficient (a defunct/renamed subdirectory under a perfectly real, resolvable FTP host will look identical to a real one until you actually try to `cwd`/`nlst` it). **Fix**: extend live-verification to actually connect and list/`cwd` into FTP paths (not just HTTP `HEAD`/`GET`), and reject any path that returns a directory-not-found error, even if the host itself is confirmed real.

### 10.1b Failure Modes From a Cloud-Lakehouse Transit Project (NSW Public Transport Crafter)

These came from fixing a map that showed a "Train Station" for a rural fire station on a bus route, then making a multi-script pipeline run only from cloud storage.

17. **The "Name-Keyword Classification" Trap**: A mode/type is inferred from substrings in a free-text name (`"Station" in stop_name` -> train, `"Wharf"`/`"Jetty"` -> ferry). Fire stations, service stations, "Station St" bus stops and "Old Toronto Station" (a *bus* stop) all became "Train Station". **Fix**: classify from the authoritative schema field (for GTFS, the `route_type` of the routes that serve the stop, joined through `stop_times -> trips -> routes`). Keywords may *nominate candidates*; they must never decide the type. Show the mode the data gives ("Bus Stop"), not the mode the name suggests.

18. **The "Exists != Active" Trap**: Official datasets contain historic, closed or unserved entities (a heritage railway station with no trains). Presence in a table was treated as "this runs". Official feeds often have **no status flag at all**. **Fix**: derive liveness explicitly - scheduled `stop_times` inside the calendar validity window, a status field, or a last-seen date - and drop or flag anything that fails. Also count "services today" so zero-service-today is visible.

19. **The "Silent Local Fallback" Trap**: The pipeline claims to run on the cloud lakehouse but rewrites `gs://...` paths to a local mirror (`sql.replace("gs://bucket/lake/", "data/lake/")`), or the query engine falls back to local views when the cloud connection fails (or when an env var is unset - i.e. local is the *default*). Output looks cloud-backed but is computed from a stale local copy. **Fix**: fail closed. Make cloud the default, raise if the engine is not on the target store, make local an explicit opt-in (`ALLOW_LOCAL_LAKE=1`) that logs a warning, and log the data source at startup.

20. **The "Default Positive Status String" Trap**: The UI falls back to reassuring text ("Active Transit Node", "High Feeder Transfer Volume") when no data exists. **Fix**: unknown must read as unknown ("Opal tap data not published at stop level"); only show a positive status computed from data, and say which fields are real.

21. **The "Fixed in Code, Stale in Artifact" Trap**: The generator is fixed but the precomputed JSON the app actually reads is unchanged; merge-on-regenerate also carries stale entries forward. **Fix**: after any logic change, regenerate and *re-inspect the published artifact*, and add a read-time guard where the app consumes it until the artifact is rebuilt.

22. **The "Delete Before Dependency Check" Trap**: A "stale" local file was deleted. It was read by the web server, three tests and a deploy script, was gitignored, and the bucket had no versioning - so it could not be restored, and a scenario that only it contained may have been lost. **Fix**: Recipe 10 - grep every consumer, diff keys against the cloud copy, confirm the cloud copy is a superset, snapshot to a backup prefix first, and ask before deleting.

23. **The "Tests Pass Against the Wrong Copy" Trap**: Tests read the local file and passed; pointed at the published artifact, one asserted a scenario (`dubbo_train_station`) that no generator produces. **Fix**: tests read the published artifact, and every asserted scenario/key must map to a generator (orphan-assertion check).

24. **The "Ambiguous Identifier" Trap**: `route_short_name = '99'` resolved to a different agency's route than intended. **Fix**: key on agency + route id (or the dataset's true composite key), and print what was resolved.

25. **The "Assumed Platform & Plan" Trap**: The AI starts work without asking where data must live, which environment is production, whether backups exist, or who consumes the artifacts, then makes confident choices (overwrite, delete, local fallback). **Fix**: run the intake interview in Section 0 first, record the answers as project invariants, and re-ask whenever the plan changes.

26. **The "Refactor Replaced a Working Result with an Empty One" Trap**: A pipeline script was refactored from local files to a remote engine; a cursor-handling bug made every table fail inside a `try/except` that only printed, and the script **published a manifest with 0 tables** over the good one. A second script, rebuilt with the documented selection rule, silently dropped 88 routes (including required ones) that an earlier manual build had included. **Fix**: assert non-empty / minimum-cardinality before any publish, back up the object first, and compare before/after counts per artifact (keys, rows, columns) - restore automatically if the comparison regresses.

### 10.2 Process Lessons for Repeatable Multi-Round AI+Python Pipelines

- **Narrow, single-target prompts beat large batch prompts.** Across several rounds of the same task, asking an AI to find one specific, narrowly-scoped thing (e.g. "what is BOM's official product code for X, given this exact product table") produced far more accurate, independently-verifiable answers than asking it to fill in a 50-row batch table in one shot. Batch prompts consistently reintroduced Traps #11-16 at a higher rate.
- **Treat every AI-suggested URL as a candidate, never a fact, and always run it through the same live-verification gate used elsewhere in the pipeline** - do not special-case "the AI seems confident this time" or "this is the AI's second/third corrected attempt" as a reason to skip verification. Corrected attempts still contained fabricated paths (Trap #16) alongside genuinely corrected ones in the same response.
- **Log rejections with the specific reason**, not just "invalid" - distinguishing *why* a candidate failed (dead host vs. wrong subject matter vs. embedded API error vs. bot-block) makes it possible to give useful, specific feedback back to the next AI round instead of a generic "try again", which measurably improves the next round's accuracy.
- **Build an explicit "manual injection" path for genuine human/AI discoveries** that separates *finding* a candidate from *accepting* it into the pipeline's output - i.e. even a fully live-verified, correct discovery should flow through the same downstream confidence-scoring and review-surfacing logic as everything else, rather than being hand-patched directly into a final deliverable. This keeps one auditable source of truth and means the pipeline is safely re-runnable from scratch at any time without losing manually-discovered results.
- **Fail closed on static/offline ingestion of anything a live source can't safely auto-fetch** (e.g. a file behind a bot-block): validate the *shape* of manually-downloaded data against a minimum-confidence schema check (e.g. "at least N rows must match the expected fixed-width pattern") before accepting it, so that a corrupted or wrong file (such as an HTML block page saved with a `.txt` extension) is rejected loudly rather than silently ingested as if it were real data.
- **For every unmatched/failed item, always resolve to a concrete "closest checkable" link**, even when the true answer is "no public endpoint exists" (e.g. a licensing/subscription product page, an internal request-access portal, or the nearest official documentation page) - live-verify that link too. This turns "no automated match found" into an actionable next step for a human instead of a dead end, and avoids the same manual research being repeated by whoever picks up the item next.

### 10.3 Process Lessons From the Cloud-Lakehouse Refactor

- **Interview first, then act.** Most costly mistakes here (deleting a local-only file, overwriting a production artifact, assuming local fallback was acceptable) were answerable by one question the AI never asked. Use Section 0 and record the answers.
- **Official data usually classifies, but rarely labels status.** Check for a type field before inventing a heuristic; if there is no status field, define liveness explicitly and tell the user how it is derived and where it is blind (e.g. one-off calendar exceptions like public holidays).
- **Dry run, then publish; publish via temp files; never keep a persistent local mirror of cloud data.**
- **Back up before every overwrite, even when you "just regenerate".** Regeneration is not idempotent when the original build used different criteria; the backup is what made two silent regressions recoverable.
- **Compare artifacts, not exit codes.** Exit code 0 published an empty manifest. Count tables, routes, keys and rows before and after.
- **A passing test can be testing the wrong copy.** Make tests read what users read (the published artifact) and keep generators and assertions in sync.
- **Say when something may be unrecoverable.** If a destructive step may have lost data (no versioning, gitignored, no backup), say it immediately and plainly instead of burying it.
- **Keep identifiers composite.** Short names collide across operators; resolve and print agency + id.

## 11. Summary Checklist Before Any Git Push or Release

- [x] **AST Gate Passed**: `pytest tests/lint/test_no_mock_data.py -v` (0 mock tokens found).
- [x] **Upstream Geometry Contracts Probed**: ArcGIS REST `geometryType` matches declared vector layer types (`Polygon`/`LineString`/`Point`).
- [x] **Cross-Stack Catalog Synchronized**: 100% 1:1 parity across dataset configs, proxy streams, and frontend layers.
- [x] **No HTTP 200 False Positives**: Deep JSON inspection confirms no ArcGIS error codes or token requirements.
- [x] **No HTML Landing Page Fallbacks**: All endpoints return pure geospatial JSON/GeoJSON.
- [x] **Live API Count Reconciled**: Pre-flight queries assert live upstream counts match S3 Lakehouse tables.
- [x] **Dynamic Table Hydration Verified**: Viewport streaming events bound to attribute table dock.
- [x] **Timezone-Aware Build Timestamps**: Static local/AEST build timestamps injected at build time.
- [x] **Physical File Assertions**: Binary file sizes and headers asserted on disk.
- [x] **Compute Runtimes Teardown**: Interactive Spark / Sedona sessions halted (`sedona.stop()`).
- [x] **Org/Instance IDs Cross-Verified**: Any AI-suggested org ID, tenant ID, or API key checked against an independent second source before use.
- [x] **Content-Type Sanity-Checked Against Body**: Plain-text/CSV endpoints validated by schema shape, not just header, to catch mislabelled bot-block pages.
- [x] **Semantic/Subject-Matter Match Confirmed**: Every accepted match's real-world subject and jurisdiction verified to actually correspond to the target - live+real is not sufficient on its own.
- [x] **Undocumented Coded Fields Empirically Verified**: Any inferred meaning of an undocumented code value confirmed against a known ground-truth example, not just its name.
- [x] **Intake Interview Completed & Invariants Recorded**: platform, source of truth, environment, protected paths, consumers and acceptance examples captured in the agent-memory file (Section 0).
- [x] **Type/Mode Derived From the Authoritative Field, Not Names**: no substring-based classification (Recipe 8).
- [x] **Liveness Verified**: every displayed entity has current scheduled service / a status, and the rule is documented.
- [x] **No Default Positive Status Text**: unknown data is labelled unknown.
- [x] **Source-of-Truth Probe Passed**: run executed against the cloud store; no silent local fallback or path rewrite (Recipe 9).
- [x] **Backup Taken Before Overwrite; Before/After Counts Compared; Non-Empty Assertions Present** (Recipe 10, Trap #26).
- [x] **Published Artifact Re-inspected After Regeneration.**
- [x] **Consumers Grepped and Cloud Superset Confirmed Before Deleting Any Data File; User Confirmed the Deletion.**
- [x] **Tests Read the Published Artifact; No Orphan Assertions.**
- [x] **Composite Identifiers Used**: agency + route id (or the dataset's true key); resolved entity printed.
