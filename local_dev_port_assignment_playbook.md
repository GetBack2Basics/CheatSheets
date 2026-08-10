# Local Dev Port Assignment Playbook

> **Problem:** When running multiple Vite/Node projects simultaneously, hardcoded ports collide (e.g. two projects both want `:3000`). This playbook documents the `findFreePort` pattern and conventions for keeping projects conflict-free.

---

## Port Registry

Maintain a simple table in your team wiki or repo README listing each project and its assigned port range. Spacing projects **5+ ports apart** leaves room for fallback increments.

```
| Project       | Dev start port | Backend/API port |
|---------------|---------------|------------------|
| project-alpha | 3000          | 8080             |
| project-beta  | 3005          | 3005             |
| project-gamma | 3010          | 3010             |
```

> **Rule of thumb:** when onboarding a new project, claim the next unclaimed block in the registry before writing any config.

---

## The Pattern — `findFreePort` Helper

Paste this into any project's `vite.config.js` or `vite.config.ts`.

### JavaScript (`vite.config.js`)

```js
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import net from 'net';

// Find a free TCP port starting from `start`
function findFreePort(start = 3000) {
  return new Promise((resolve) => {
    let port = start;
    const tryPort = () => {
      const srv = net.createServer();
      srv.once('error', () => { port++; tryPort(); });
      srv.once('listening', () => { srv.close(() => resolve(port)); });
      srv.listen(port, '127.0.0.1');
    };
    tryPort();
  });
}

// <- set to this project's registered start port
const port = await findFreePort(3000);

export default defineConfig({
  plugins: [react()],
  server: {
    port,
    strictPort: true,   // already confirmed free — let Vite trust it
    host: '0.0.0.0',
    proxy: {
      '/api': { target: 'http://localhost:8080', changeOrigin: true },
      '/ws':  { target: 'ws://localhost:8080',   ws: true, changeOrigin: true },
    },
  },
});
```

### TypeScript (`vite.config.ts`)

```ts
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import fs from 'fs'
import path from 'path'
import net from 'net'

// Find a free TCP port starting from `start`
function findFreePort(start = 3005): Promise<number> {
  return new Promise((resolve) => {
    let port = start;
    const tryPort = () => {
      const srv = net.createServer();
      srv.once('error', () => { port++; tryPort(); });
      srv.once('listening', () => { srv.close(() => resolve(port)); });
      srv.listen(port, '127.0.0.1');
    };
    tryPort();
  });
}

// Optional: read backend port from a .server-port file written by the Node server at startup
function getBackendPort(): number {
  try {
    const portFile = path.resolve((import.meta as any).dirname || '.', '.server-port');
    if (fs.existsSync(portFile)) {
      const p = parseInt(fs.readFileSync(portFile, 'utf-8').trim(), 10);
      if (!isNaN(p)) return p;
    }
  } catch { /* fallback */ }
  return 3005; // <- match your .env PORT
}

// <- set to this project's registered start port
const devPort = await findFreePort(3005);

export default defineConfig({
  plugins: [react()],
  server: {
    port: devPort,
    strictPort: true,
    host: '0.0.0.0',
    proxy: {
      '/api': {
        target: `http://localhost:${getBackendPort()}`,
        changeOrigin: true,
        secure: false,
        configure: (proxy) => {
          proxy.on('proxyReq', (_proxyReq, req) => {
            req.headers['host'] = `localhost:${getBackendPort()}`;
          });
        }
      }
    }
  }
})
```

---

## Why `strictPort: true` after `findFreePort`?

By the time Vite starts, `findFreePort` has **already confirmed** the port is free via a real TCP probe. If Vite is told `strictPort: false` it might re-check and pick a *different* port, defeating the purpose. Setting it to `true` tells Vite: *"trust the port I gave you, fail loudly if something grabbed it in the meantime."*

---

## The `.server-port` Pattern (Backend → Frontend handshake)

When your Node/Express server starts, write its resolved port to a `.server-port` file. The Vite config reads this file to know where to proxy `/api` calls — useful when the backend also uses dynamic port resolution.

```ts
// server/index.ts (Node/Express)
import fs from 'fs';
import path from 'path';

const PORT = parseInt(process.env.PORT || '3005', 10);

app.listen(PORT, () => {
  // Write resolved port so vite.config knows where to proxy
  fs.writeFileSync(path.resolve('.server-port'), String(PORT));
  console.log(`Server running on http://localhost:${PORT}`);
});
```

Add `.server-port` to `.gitignore` — it is a runtime artefact, not source code:

```
# .gitignore
.server-port
```

---

## Diagnosing Port Conflicts

### Windows (PowerShell)
```powershell
# See what is using a specific port
Get-NetTCPConnection -LocalPort 3005 | Select-Object LocalAddress, LocalPort, OwningProcess

# Find the process name from the PID
Get-Process -Id <PID>

# Kill it
Stop-Process -Id <PID> -Force
```

### macOS / Linux
```bash
lsof -i :3005
kill -9 <PID>
```

### Windows (cmd)
```cmd
netstat -ano | findstr :3005
taskkill /PID <PID> /F
```

---

## Quick Start Checklist for a New Project

- [ ] Claim the next available port block in your team's port registry
- [ ] Add `findFreePort` to `vite.config.js/ts`, passing your registered start port
- [ ] Set `strictPort: true` in the Vite `server` block
- [ ] Set `PORT=<your start port>` in `.env`
- [ ] Have the Node server write `.server-port` on startup (if using backend proxy)
- [ ] Add `.server-port` to `.gitignore`
- [ ] Update the port registry with your new project entry
