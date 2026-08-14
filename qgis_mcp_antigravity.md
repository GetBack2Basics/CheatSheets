# Antigravity QGIS MCP Ingestion & Control Cheat Sheet

This cheat sheet explains how to configure and run the QGIS MCP (Model Context Protocol) TCP socket server, perform generalized spatial data ingestion, and control QGIS programmatically.

---

## 1. QGIS MCP Architecture on Antigravity

Antigravity communicates with QGIS Desktop using a length-prefixed TCP socket protocol over port `9876`. A Python plugin running inside QGIS evaluates commands in QGIS's main application thread.

```
+--------------------------+                 +--------------------------+
|  Antigravity / Client    | --(TCP: 9876)-->|   Running QGIS Session   |
|   (run_pyqgis_code.py)   |                 | (qgis_mcp_plugin Active) |
+--------------------------+                 +--------------------------+
```

---

## 2. Launching QGIS with the Correct Environment

On Windows, launching QGIS from standard command sessions can fail with `0xC0000135` (DLL Not Found) or load incorrect Python libraries. 

Always launch QGIS using a detached batch launcher that sets up the OSGeo4W environment and adds QGIS's bin directory to the `%PATH%`:

```bat
@echo off
REM launcher.bat
set PROJECT_PATH=C:\path\to\your\project.qgz

REM 1. Detect OSGeo4W Root Directory
if exist "C:\OSGeo4W\bin\o4w_env.bat" (
    set OSGEO4W_ROOT=C:\OSGeo4W
    set QGIS_EXE=qgis-dev-bin.exe
    set APP_DIR=qgis-dev
    set QT_DIR=qt6
) else (
    set OSGEO4W_ROOT=C:\Users\%USERNAME%\AppData\Local\Programs\OSGeo4W
    set QGIS_EXE=qgis-ltr-bin.exe
    set APP_DIR=qgis-ltr
    set QT_DIR=qt5
)

REM 2. Set QGIS/OSGeo4W Environment Variables
call "%OSGEO4W_ROOT%\bin\o4w_env.bat"
if exist "%OSGEO4W_ROOT%\bin\qt6_env.bat" call "%OSGEO4W_ROOT%\bin\qt6_env.bat"
if exist "%OSGEO4W_ROOT%\bin\gdal-dev-py-env.bat" call "%OSGEO4W_ROOT%\bin\gdal-dev-py-env.bat"

path %OSGEO4W_ROOT%\apps\%APP_DIR%\bin;%PATH%
set QGIS_PREFIX_PATH=%OSGEO4W_ROOT:\=/%/apps/%APP_DIR%
set QT_PLUGIN_PATH=%OSGEO4W_ROOT%\apps\%APP_DIR%\qtplugins;%OSGEO4W_ROOT%\apps\%QT_DIR%\plugins

REM 3. Launch QGIS Detached
cd /d "%OSGEO4W_ROOT%\bin"
start "QGIS" "%OSGEO4W_ROOT%\bin\%QGIS_EXE%" "%PROJECT_PATH%"
exit /b 0
```

---

## 3. Standalone Data Ingestion (Generic Runner)

For ingesting spatial data, use a single Python CLI script that runs under the OSGeo4W Python wrapper (`python-qgis-dev.bat` or `python-qgis-ltr.bat`). This gives the script access to the standalone `qgis.core` library to write directly into GeoPackage database tables without launching the desktop GUI.

### 3.1. Ingestion CLI Template (`ingest_spatial_data.py`)
```python
# Save as ingest_spatial_data.py
import argparse
import urllib.request
import urllib.parse
import json
from qgis.core import QgsApplication, QgsVectorLayer, QgsVectorFileWriter, QgsCoordinateTransformContext

def fetch_arcgis_data(url, bbox, fields_str="*", batch_size=1000):
    # Query Object IDs
    params = {'where': '1=1', 'geometry': bbox, 'geometryType': 'esriGeometryEnvelope', 'inSR': '4326', 'returnIdsOnly': 'true', 'f': 'json'}
    req = urllib.request.Request(f"{url}/query?{urllib.parse.urlencode(params)}", headers={'User-Agent': 'Mozilla/5.0'})
    with urllib.request.urlopen(req) as resp:
        id_data = json.loads(resp.read().decode('utf-8'))
        object_ids = id_data.get('objectIds', [])
        oid_field = id_data.get('objectIdFieldName', 'objectid')

    # Paginate features in batches
    features = []
    for idx in range(0, len(object_ids), batch_size):
        batch = object_ids[idx:idx+batch_size]
        where = f"{oid_field} IN ({','.join(map(str, batch))})"
        batch_params = {'where': where, 'outSR': '4326', 'outFields': fields_str, 'f': 'geojson'}
        batch_req = urllib.request.Request(f"{url}/query?{urllib.parse.urlencode(batch_params)}", headers={'User-Agent': 'Mozilla/5.0'})
        with urllib.request.urlopen(batch_req) as b_resp:
            features.extend(json.loads(b_resp.read().decode('utf-8')).get('features', []))
            
    return {"type": "FeatureCollection", "features": features}

# Ingest and write directly to GPKG using PyQGIS standalone
def main():
    # Parse arguments (--type, --url, --bbox, --gpkg, --layer-name, --fields)
    # Fetch data to temporary GeoJSON, then load and write to GeoPackage:
    qgs = QgsApplication([], False)
    qgs.initQgis()
    layer = QgsVectorLayer("temp.geojson", "my_layer", "ogr")
    options = QgsVectorFileWriter.SaveVectorOptions()
    options.driverName = "GPKG"
    options.layerName = "my_layer"
    options.actionOnExistingFile = QgsVectorFileWriter.CreateOrOverwriteLayer
    QgsVectorFileWriter.writeAsVectorFormatV3(layer, "data.gpkg", QgsCoordinateTransformContext(), options)
    qgs.exitQgis()
```

### 3.2. Example Command: ArcGIS REST Ingestion
```powershell
C:\OSGeo4W\bin\python-qgis-dev.bat ingest_spatial_data.py `
  --type arcgis `
  --url "https://spatial-gis.information.qld.gov.au/arcgis/rest/services/PlanningCadastre/LandParcelPropertyFramework/MapServer/8" `
  --bbox "145.65,-17.15,145.85,-16.75" `
  --fields "objectid,lot,plan,tenure,lot_area" `
  --gpkg "C:\projects\my_project\data.gpkg" `
  --layer-name "cadastre_base"
```

---

## 4. Socket Execution Client

To execute PyQGIS scripts inside a running QGIS session, use a lightweight, zero-dependency TCP socket client that handles length-prefixed framing (`>I` big-endian uint32 header):

```python
# run_pyqgis_code.py
import sys, socket, struct, json

HEADER_STRUCT = struct.Struct(">I")

def run_in_qgis(code, host="127.0.0.1", port=9876):
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.setsockopt(socket.IPPROTO_TCP, socket.TCP_NODELAY, 1)
    s.connect((host, port))
    
    # Send
    payload = json.dumps({"type": "execute_code", "params": {"code": code}}).encode("utf-8")
    s.sendall(HEADER_STRUCT.pack(len(payload)) + payload)
    
    # Recv Length
    header = s.recv(4)
    resp_len = HEADER_STRUCT.unpack(header)[0]
    
    # Recv Payload
    data = b""
    while len(data) < resp_len:
        data += s.recv(resp_len - len(data))
    s.close()
    return json.loads(data.decode("utf-8"))

if __name__ == "__main__":
    with open(sys.argv[1], "r", encoding="utf-8") as f:
        print(run_in_qgis(f.read()))
```

---

## 5. Performance and Styling Best Practices

### 5.1. Scale-Dependent Visibility
Always set scale-dependent visibility on large vector polygon datasets (e.g. cadastre base parcels) to prevent startup crashes or rendering lags when opening QGIS:
```python
# visible only when zoomed in closer than 1:25,000 (denominator <= 25000)
cad_layer.setScaleBasedVisibility(True)
cad_layer.setMinimumScale(25000.0)
cad_layer.setMaximumScale(0.0)
```

### 5.2. Text Halos (Buffers) on Rule-Based Labels
To make labels legible over complex background geometries, add a 1mm white text buffer:
```python
from qgis.core import QgsPalLayerSettings, QgsVectorLayerSimpleLabeling, QgsTextFormat, QgsTextBufferSettings
from qgis.PyQt.QtGui import QColor, QFont

label_settings = QgsPalLayerSettings()
label_settings.fieldName = "road_name_full"
label_settings.filterExpression = "class IN ('Highway', 'Secondary')"
label_settings.enabled = True

text_format = QgsTextFormat()
text_format.setFont(QFont("Arial", 8))
text_format.setColor(QColor("#222222"))

# White Halo Buffer
buffer_settings = QgsTextBufferSettings()
buffer_settings.setEnabled(True)
buffer_settings.setSize(1.0)
buffer_settings.setColor(QColor("#ffffff"))
text_format.setBuffer(buffer_settings)

label_settings.setFormat(text_format)
layer.setLabeling(QgsVectorLayerSimpleLabeling(label_settings))
layer.setLabelsEnabled(True)
```
