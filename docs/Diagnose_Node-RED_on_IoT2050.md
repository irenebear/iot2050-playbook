# Diagnosing Node-RED on the IoT2050: A Troubleshooting Handbook (with a Real Crash Case)

> Published: 2026-08
> Audience: Embedded developers running Node-RED on a Siemens IoT2050 (Debian) who hit service crashes, node loading failures, or stuck flows.
> Environment: IoT2050 + Debian, Node-RED running as a systemd service (service name `node-red`, default port 1880, working directory `/root/.node-red`).
> Official references: Node-RED docs https://nodered.org/docs/ , Siemens [`siemens/meta-iot2050`](https://github.com/siemens/meta-iot2050)

## Table of Contents

- [Foreword](#foreword)
- [1. Manage the Node-RED Service with systemctl (most used)](#1-manage-the-node-red-service-with-systemctl-most-used)
- [2. View Debug Logs (most common)](#2-view-debug-logs-most-common)
- [3. Adjust Runtime Log Level](#3-adjust-runtime-log-level)
- [4. Check Node Execution](#4-check-node-execution)
- [5. Trace Message Flow](#5-trace-message-flow)
- [6. Check Dependencies and Modules](#6-check-dependencies-and-modules)
- [7. Common Error Troubleshooting](#7-common-error-troubleshooting)
- [8. Network / External Services](#8-network--external-services)
- [9. System Resources](#9-system-resources)
- [10. Real Case: better-sqlite3 version incompatibility crashes Node-RED repeatedly](#10-real-case-better-sqlite3-version-incompatibility-crashes-node-red-repeatedly)
- [11. Quick-Start Mnemonic](#11-quick-start-mnemonic)
- [References](#references)

---

## Foreword

On industrial edge gateways like the IoT2050, Node-RED usually runs as a **systemd service**. When the service crashes, a node fails to load, or a flow hangs, it's easy to end up searching for a needle in a haystack if you don't know where to start.

This article compiles the common Node-RED diagnostic methods on the IoT2050 into a **handbook**: from service management, log viewing, and version checks, to per-layer troubleshooting ideas for nodes/dependencies/network/resources — and ends with a **real crash case** (better-sqlite3 version incompatibility causing repeated segfault crashes), walking you through a complete troubleshooting session.

> Prerequisite: Node-RED runs on the IoT2050 (Debian) as a systemd service (service name `node-red`, default port 1880, working directory `/root/.node-red`).

---

## 1. Manage the Node-RED Service with systemctl (most used)

The official IoT2050 example image runs Node-RED as a systemd service. After logging in over SSH/serial, manage everything with systemctl:

```bash
# Service status (running? PID? recent logs)
systemctl status node-red

# Only whether it's running (active = running, inactive = stopped)
systemctl is-active node-red

# Restart the service (most used after editing flows / installing nodes)
sudo systemctl restart node-red

# Stop / start
sudo systemctl stop node-red
sudo systemctl start node-red

# Auto-start on boot: check / enable / disable
systemctl is-enabled node-red
sudo systemctl enable node-red
sudo systemctl disable node-red
```

### 1.1 View Node-RED logs (journalctl)

```bash
# Follow the log live (Ctrl+C to exit)
sudo journalctl -u node-red -f

# Last 100 lines
sudo journalctl -u node-red -n 100

# Today / last 1 hour
sudo journalctl -u node-red --since today
sudo journalctl -u node-red --since "1 hour ago"

# Error level only
sudo journalctl -u node-red -p err

# Or check syslog
grep node-red /var/log/syslog | tail -50
```

### 1.2 Confirm the listening port

```bash
ss -tlnp | grep 1880    # Node-RED default port 1880; should show LISTEN
```

### 1.3 View the service definition file

```bash
systemctl cat node-red        # ExecStart, User, working directory, etc.
# Definition is usually at /usr/lib/systemd/system/node-red.service
# Typical ExecStart: /usr/bin/node-red -u /root/.node-red
```

### 1.4 Check the Node-RED and Node.js versions (check before diagnosing version issues)

```bash
# ① Most direct: node-red's own version flag
node-red -v
# e.g. 4.1.0

# ② If that fails, check from the install directory (npm global install)
npm list -g node-red 2>/dev/null

# ③ Or from the working directory (/root/.node-red)
cd /root/.node-red && npm list node-red 2>/dev/null

# ④ Also confirm the Node.js version (for native-module compatibility judgment)
node -v
# e.g. v20.19.2
```

> 💡 **Why check versions**: a common root cause of Node-RED crashes is **node/native-module incompatibility with the Node.js major version** (e.g. better-sqlite3@13 requires Node ≥ 22; installed on Node 20 it segfaults). Before troubleshooting, confirm `node-red -v` and `node -v`, then compare against the node's `engines` requirement.
> 📌 The version also appears in the startup log: `journalctl -u node-red | grep -i version` or the startup banner.

> ⚠️ Node-RED doesn't always auto-restart after a crash — just run `sudo systemctl restart node-red`. If it crashes repeatedly, focus on `journalctl -u node-red` to find which node/flow causes it.

---

## 2. View Debug Logs (most common)

- **Debug panel**: the Debug tab in the right sidebar shows `node.warn()`, `node.log()` output and data passed to debug nodes.
- **Debug node**: drag a debug node into the flow; checking "complete msg object" shows all msg properties.
- **Console output**: the terminal that started Node-RED (`npm start` or `node-red`) prints logs live.

---

## 3. Adjust Runtime Log Level

Edit `settings.js`:

```js
logging: {
  console: {
    level: "debug",   // options: fatal / error / warn / info / debug / trace
    metrics: false,
    audit: false
  }
}
```

Restart Node-RED to apply; `debug` level outputs much more internal info.

---

## 4. Check Node Execution

- **Node status colors**:
  - Green = running normally
  - Red dot = error (hover to see the message)
  - Grey / none = not connected or not deployed
- **Connection points**: if the wires between nodes are disconnected or ports aren't plugged, data won't flow.

---

## 5. Trace Message Flow

- Add a **catch node**: catches errors thrown anywhere in the flow.

```json
{ "type": "catch", "scope": ["flow-id"] }
```

- Add a **status node**: watch node status changes.
- Add debug nodes at A and B before/after a step, and use "binary search" to find where data is lost or broken.

---

## 6. Check Dependencies and Modules

```bash
# List installed nodes
npm list --depth=0
# or check via Manage palette

# Reinstall / update a broken custom node
npm install -g node-red-contrib-xxx
```

> ⚠️ **IoT2050 (systemd service) note**: after installing/updating custom nodes, you must `sudo systemctl restart node-red` for the new nodes to load. If you `npm install -g` global nodes, make sure Node-RED's `NODE_PATH` or the `editorTheme/package` config in settings.js can reference them (the official example image installs nodes under `/usr/lib/node_modules` or `~/.node-red/node_modules`; installing to the wrong place gives "module not found").

If a custom node reports "module not found", it's usually not installed correctly or is version-incompatible.

---

## 7. Common Error Troubleshooting

| Symptom | Possible cause |
| --- | --- |
| Node shows red error | Code threw an exception; look at the error stack |
| Data doesn't reach a node | Wire not connected, msg property empty, previous node never called `node.send` |
| Messages duplicated / stuck | Loop without `msg.loop` or missing a termination condition |
| Chinese characters garbled | Encoding issue; check `Buffer` usage or charset conversion |
| Timer never fires | Check the `cron` expression or the system timezone |

---

## 8. Network / External Services

- Test whether the target API is reachable with an HTTP request node first.
- Check firewalls and proxies (Node-RED's `httpProxy` setting).
- Check the browser dev-tools Network panel (when using Dashboard + HTTP nodes).

---

## 9. System Resources

When a flow hangs, check:

- Memory: whether the `node-red` process is out of memory (tune with `--max-old-space-size`).
- File descriptors / ports: whether `context` storage, MongoDB, or other external stores connect normally.
- Background task pileup: some nodes (e.g. an infinite-loop `function`) can block the event loop.

---

## 10. Real Case: better-sqlite3 version incompatibility crashes Node-RED repeatedly

> Scenario: the Node-RED service on an IoT2050 **crashes repeatedly, dies on startup**; `journalctl` shows a segfault. This is a reconstruction of a real troubleshooting session (resolved).

### 10.1 Symptoms

```
journalctl -u node-red
# ... service restarts repeatedly, core error:
# status=11/SEGV     ← Segmentation Fault
```

Node-RED crashes on every startup and can't work.

### 10.2 Environment

| Item | Value |
| --- | --- |
| Node.js | v20.x |
| Node-RED | v4.x |
| Architecture | Linux arm64 |
| User directory | `/root/.node-red` |
| Service name | `node-red.service` |

### 10.3 Root cause analysis

**`better-sqlite3@13` is incompatible with Node 20**:

- better-sqlite3 v13's `package.json` declares `engines: node >= 22`
- The environment is **Node 20**, yet v13 is installed → the C++ native module's **NODE_MODULE_VERSION (ABI) mismatch** → loading immediately triggers `status=11/SEGV`
- Native modules (C++ addons) are sensitive to the Node version; installing across major versions crashes

> Rule of thumb: **Node-RED repeatedly segfaults → first suspect a recently installed/upgraded native module whose version doesn't match the Node major version.**

### 10.4 Solution (downgrade to a compatible version)

```bash
# ① Run under /root/.node-red (Node-RED only loads from here)
cd /root/.node-red
npm install better-sqlite3@11
```

**Key: verify the module no longer SEGVs standalone before restarting the service**:

```bash
# ② Standalone load test
cd /root/.node-red && node -e "const Database=require('better-sqlite3'); const db=new Database(':memory:'); db.exec('CREATE TABLE t(a)'); console.log('OK')"
```

- Prints `OK` with no segfault → module is fine
- Then restart the service:

```bash
systemctl restart node-red
```

### 10.5 Troubleshooting cheat-sheet

| Step | Command | Purpose |
| --- | --- | --- |
| See the crash log | `journalctl -u node-red -n 50` | Confirm whether it's SEGV / which module |
| Check the module version | `cat /root/.node-red/node_modules/better-sqlite3/package.json \| grep -E 'version\|engines'` | Confirm whether the version matches the Node major |
| Standalone module test | `cd /root/.node-red && node -e "require('better-sqlite3'); console.log('OK')"` | Test without Node-RED to quickly isolate native-module issues |
| Confirm Node version | `node -v` | Compare against the module's engines requirement |

---

## 11. Quick-Start Mnemonic

> **"Check Debug first, bisect to locate; watch status colors, check the console; check dependencies, watch the data flow."**

When you hit a specific error (a node stuck red, or a clear error message), paste the error text to an AI or a community forum along with the `journalctl -u node-red` output — it locates the root cause much faster.

---

## References

- Node-RED official docs: https://nodered.org/docs/
- Node-RED logging & configuration: https://nodered.org/docs/user-guide/runtime/logging
- Siemens meta-iot2050: https://github.com/siemens/meta-iot2050
- better-sqlite3 (npm): https://www.npmjs.com/package/better-sqlite3

---

> ⚠️ **Disclaimer**: This article is a personal technical write-up reflecting the author's own experience and opinions, unrelated to any commercial organization. The operations described involve embedded system operations — proceed with caution. For production or industrial use, have qualified personnel perform and fully test the changes. The author accepts no liability for any direct or indirect loss resulting from the use of this content.
