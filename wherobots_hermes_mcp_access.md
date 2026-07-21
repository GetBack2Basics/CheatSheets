# wherobots_hermes_mcp_access

Step-by-step cheat sheet for connecting **Hermes** to the **Wherobots MCP Server** on Windows.

---

## Prerequisites

- Hermes installed locally (desktop app).
- Wherobots Cloud account org with **MCP access enabled**.
  - If you see `MCP access is not enabled for your organization`, you need a **Professional or Enterprise** org plan, or you must contact **support@wherobots.com**.
- A Wherobots **API key**:
  - Create at `cloud.wherobots.com/settings#api-keys`
  - Copy and store it — Wherobots never shows it again.
- Python `mcp` package available to Hermes:
  ```bash
  pip install mcp
  ```

---

## 1. Find the active Hermes config

On Windows, Hermes may use either:

- `C:\Users\corea\AppData\Local\hermes\config.yaml`
- `~/.hermes/config.yaml`

Verify actual path:
```bash
hermes config path
```

---

## 2. Add the Wherobots MCP server

Edit the active `config.yaml` and add under `mcp_servers:`:

```yaml
mcp_servers:
  qgis:
    command: C:\Users\corea\AppData\Local\hermes\qgis-mcp-launch.bat
    enabled: true

  wherobots:
    url: "https://api.cloud.wherobots.com/mcp/"
    headers:
      X-API-KEY: YOUR_API_KEY_HERE    # NOT Authorization: Bearer
    timeout: 180
    connect_timeout: 30
    enabled: true
```

**Critical:** Use header `X-API-KEY`, not `Authorization: Bearer`.

---

## 3. Restart Hermes

MCP servers are discovered at startup. You must fully restart the Hermes desktop app after editing config.

---

## 4. Verify

```bash
hermes mcp list
```

Expected:
```
Name             Transport                      Tools        Status
──────────────────────────────────────────────────  ────────────────  ────────────  ────
qgis             ...                            all          ✓ enabled
wherobots        https://api.cloud.wherobots...  all          ✓ enabled
```

Test connection:
```bash
hermes mcp test wherobots
```

---

## 5. Use it

MCP tools are prefixed `mcp_wherobots_*` and available in every conversation:

- `mcp_wherobots_list_catalogs_tool`
- `mcp_wherobots_list_databases_tool`
- `mcp_wherobots_list_tables_tool`
- `mcp_wherobots_describe_table_tool`
- `mcp_wherobots_generate_spatial_query_tool`
- `mcp_wherobots_execute_query_tool`
- `mcp_wherobots_search_documentation_tool`

Example:
```
List the catalogs available in my Wherobots organization
```

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `hermes mcp list` shows no `wherobots` | Config not under `mcp_servers:` or wrong file edited | Use `hermes config path`, ensure correct YAML path |
| `401 Unauthorized` | Wrong header or expired key | Use header `X-API-KEY` with a fresh key |
| `MCP access is not enabled for your organization` | Org plan gate | Upgrade to Professional/Enterprise, or email **support@wherobots.com** |
| `MCP SDK not available` | `mcp` package missing | `pip install mcp` |
| Still failing after restart | Cached session | Close + reopen Hermes completely, or run `/reset` |

---

## References

- MCP Server Overview: https://docs.wherobots.com/develop/mcp/mcp-server-overview
- Usage best practices: https://docs.wherobots.com/develop/mcp/mcp-server-usage
- Create API key: https://docs.wherobots.com/get-started/wherobots-cloud/api-keys
- Agentic setup: https://docs.wherobots.com/develop/agentic-tools
