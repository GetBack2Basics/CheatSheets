# Wherobots & Antigravity Playbook

## Summary

Use this playbook to configure, execute, and troubleshoot geospatial ETL pipelines and Spatial SQL queries on the Wherobots Cloud environment using the Antigravity AI coding assistant and Python SDKs.

---

## What this gives you

- **Serverless Spark Execution**: Offload heavy geospatial computations (such as polygon buffering and unioning) to a remote, distributed Spark cluster.
- **Agentic DB-API Queries**: Direct connection to WherobotsDB, returning results natively as Pandas DataFrames.
- **Robust Pipeline submission**: Run long-running Python scripts as headless Wherobots Spark jobs with automatic file dependency management.
- **Workspace Integration**: Use local editor commands or programmatic scripts to start runtimes, submit jobs, and inspect database catalogs.

---

## Setup & Connection Process

### Step 1: SDK & Driver Installation
Ensure you install both the DB-API driver (for running SQL queries and listing tables) and the job SDK (for submitting scripts):
```bash
python -m pip install wherobots-python-sdk wherobots-python-dbapi
```

### Step 2: Programmatic Job Submission
Use the `WherobotsJob` class from the SDK to package your local ingestion script and its configuration files:
```python
from wherobots import WherobotsJob

# Declare local file dependencies (e.g. JSON configuration files)
config_dep = WherobotsJob.add_file_dependency("config/macquarie.json")

# Initialize and submit the job
job = WherobotsJob(
    script="src/Ingestion/macquarie_spatial_ingest.py",
    name="macquarie-spatial-etl",
    runtime="tiny",
    api_key="<YOUR_WHEROBOTS_API_KEY>",
    dependencies=[config_dep],
)

job.submit()
print(f"Submitted Job ID: {job.run_id}")

# Wait and stream logs
status = job.wait_for_completion(stream_logs=True)
```

### Step 3: Programmatic Spatial SQL Connections
Establish a DB-API connection to run interactive queries and inspect tables. The driver returns Pandas DataFrames natively for all results:
```python
from wherobots.db import connect

conn = connect(api_key="<YOUR_WHEROBOTS_API_KEY>")
cursor = conn.cursor()

# Retrieve catalog tables
cursor.execute("SHOW TABLES IN org_catalog.fgsdb")
df_tables = cursor.fetchall()  # Returns a Pandas DataFrame!
print(df_tables)

# Execute spatial queries
cursor.execute("""
    SELECT precinct_key, ST_Area(net_developable_geom) / 1e4 AS area_ha 
    FROM org_catalog.fgsdb.macquarie_net_developable_zones
""")
df_results = cursor.fetchall()
print(df_results)
```

---

## What not to do (Anti-patterns)

### Anti-pattern 1: Using browser automation (Playwright/Puppeteer) to click the Jupyter HTML runner
- **Why this fails**: The local browser environment can fail due to driver download mismatches or sandboxed network rules (e.g. Playwright downloading driver dependencies failing with 404).
- **Better approach**: Write a Python script using the `wherobots-python-sdk` to submit jobs or `wherobots-python-dbapi` to run SQL statements directly.

### Anti-pattern 2: Double-transforming geometries in queries
- **Why this fails**: If your ingestion loaders already reproject files (e.g. `gdf.to_crs("EPSG:7856")`) before saving them to Wherobots Havasu/Iceberg, applying `ST_Transform` again on the database query (treating them as if they are in EPSG:4326) will distort the coordinates and crash the executors.
- **Better approach**: Track the projection of your database tables and query them directly in their native CRS.

### Anti-pattern 3: Buffering and unioning unfiltered global/state-wide networks
- **Why this fails**: Trying to run buffers or spatial unions (`ST_Union_Aggr`) on a full statewide layer (e.g., the entire NSW railway network) on a `tiny` or `small` Wherobots runtime will cause immediate JVM Out-of-Memory (`OOM`) errors.
- **Better approach**: Join the spatial network with your precinct boundary using `ST_Intersects` *before* applying buffers or union operations:
  ```sql
  SELECT ST_Buffer(g.geometry, 10.0) AS geom 
  FROM org_catalog.fgsdb.macquarie_rail_network g
  JOIN precinct_transform p ON ST_Intersects(g.geometry, p.geom)
  ```

### Anti-pattern 4: Running long-running Python DB-API scripts with stdout buffering
- **Why this fails**: By default, Python buffers stdout when redirected to logs, making the script appear frozen at `"Connecting..."` for several minutes.
- **Better approach**: Force unbuffered output by running the script with the `-u` flag:
  ```bash
  python -u runner/view_results.py
  ```

---

## Lessons Learned & Best Practices
- **Aggregate Function Names**: In Apache Sedona SQL, the aggregate union function is **`ST_Union_Aggr`**, not `ST_Union_Aggregate`.
- **Dynamic Dependency Paths**: When you add a file dependency (like `config/macquarie.json`) to a Wherobots job, Wherobots downloads it to `/opt/wherobots/macquarie.json` in the executor. Make sure your Python scripts check this directory in their candidate config paths.
- **Tolerating Missing Open Data**: Open-data APIs (like NSW SEED Portal) frequently go offline or change endpoints (e.g., returning HTTP 404). Wrap open data ingestion calls in `try-except` blocks and check table existence using `sedona.catalog.tableExists` to ensure the ETL pipeline degrades gracefully instead of failing entirely.
