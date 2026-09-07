# Install Node-RED Custom Nodes on IoT2050 WITHOUT Internet — Complete Offline Tutorial (with Common Pitfalls)

> Published: 2026-08
> Audience: Developers who need to install custom Node-RED nodes on an IoT2050 that has no internet access (or is firewalled) at industrial sites.

## Table of Contents

- [Foreword](#foreword)
- [1. Understand first: the core principle of offline installation](#1-understand-first-the-core-principle-of-offline-installation)
- [2. Prerequisites](#2-prerequisites)
- [3. Step 1: Prepare the node offline package on an online machine (⚠️ must match IoT2050's architecture)](#3-step-1-prepare-the-node-offline-package-on-an-online-machine)
- [4. Step 2: SSH into the IoT2050](#4-step-2-ssh-into-the-iot2050)
- [5. Step 3: Upload the node folder via SCP](#5-step-3-upload-the-node-folder-via-scp)
- [6. Step 4: Edit package.json (recommended, optional)](#6-step-4-edit-packagejson-recommended-optional)
- [7. Step 5: Restart Node-RED](#7-step-5-restart-node-red)
- [8. Step 6: Verify the node is loaded](#8-step-6-verify-the-node-is-loaded)
- [9. Troubleshooting](#9-troubleshooting)
- [10. Quick checklist](#10-quick-checklist)
- [References](#references)

---

## Foreword

IoT2050s at industrial sites are often **offline** (or the network is heavily firewalled). When you want to install a custom Node-RED node in that situation, `npm install` simply can't run. What to do?

This article shares a **fully offline** installation method: package the node (with all its dependencies) on an online computer, transfer it to the IoT2050 via SCP, and restart — it takes effect immediately. Every step is detailed with a verification method, so beginners can do it in one go.

The example node is `node-red-contrib-mcprotocol` (Mitsubishi PLC MC protocol); the method applies to any custom node.

---

## 1. Understand first: the core principle of offline installation

Node-RED **automatically scans** `/root/.node-red/node_modules` at startup and auto-discovers/loads any node packages inside.

So offline installation boils down to one sentence:

> **Run `npm install` on an online machine to install the node (with all dependencies) → copy the whole node folder into IoT2050's `node_modules` → restart Node-RED and it works.**

Two key facts (the ones beginners misunderstand most):

1. **Editing `package.json` is not required** — as long as the node folder is copied correctly, the palette picks it up. Manually adding a line to `dependencies` is "icing on the cake": it lets `npm install` reinstall these nodes automatically when you're online later. **Recommended, but not required.**
2. **Don't just download the ZIP source package** — source packages often **lack sub-dependencies** (the other npm packages the node depends on), so startup fails with `module not found`. The right way is in §3.

---

## 2. Prerequisites

| Item | Description |
| --- | --- |
| IoT2050 | Node-RED installed and working; working directory `/root/.node-red` (nodes, flows.json, settings.js all live here) |
| Windows tools | For **SSH / SCP transfer only** (Xshell / FinalShell / built-in PowerShell ssh + WinSCP) — **cannot** be used for npm packaging (see §3 warning) |
| Online packaging machine | **Must match IoT2050's architecture (Linux + arm64)**: no arm64 device? Use Docker emulation on your PC, see §3 |

---

## 3. Step 1: Prepare the node offline package on an online machine (⚠️ must match IoT2050's architecture)

> ❗ **The two easiest traps to fall into**:
>
> 1. **Don't use a Win11 / x86 PC** to `npm install` the node and transfer it to the IoT2050! The native modules you get on Windows are **win32-x64** binaries, but the IoT2050 is **Linux-arm64** — it will inevitably fail with `ERR_DLOPEN_FAILED` / `module not found`. Pure-JS nodes aren't affected by architecture, but you can't guarantee the dependency tree has no native modules, so **always package on arm64 to be safe**.
> 2. **Don't just download the ZIP source package** — source packages often lack sub-dependencies and fail with `module not found` at startup. **Use the npm packaging method below** — one step, dependencies complete.

### Package with npm on an arm64 online machine (✅ the only recommended way, all dependencies included)

Run on an online machine with **the same architecture as the IoT2050 (Linux + arm64)**:

```bash
# Create a temp directory and install the node there (local install, no -g)
mkdir ~/offline-node && cd ~/offline-node
npm install node-red-contrib-mcprotocol
```

After installation, **copy out the whole `node-red-contrib-mcprotocol` folder from `node_modules`** (it already contains all dependencies) for upload.

**No arm64 machine? Emulate arm64 with Docker on your PC (free):**

First install Docker (Docker Desktop + WSL2 backend on Windows; docker engine on Linux) and confirm `docker --version` prints a version. Then start the container **in the directory where you want the artifacts** — note `$PWD` only works well in bash, two terminal styles:

```bash
# ✅ Recommended: WSL2 / Linux terminal ($PWD is naturally a Linux path, no traps)
docker run --rm -it --platform linux/arm64 -v "$PWD":/work -w /work node:20-bookworm-slim bash
```

```powershell
# Windows PowerShell: don't use "$PWD" (it expands to a backslash path; docker reports invalid reference format)
# Use a hardcoded forward-slash path instead (replace D:/offline-node with your own directory):
docker run --rm -it --platform linux/arm64 -v D:/offline-node:/work -w /work node:20-bookworm-slim bash
```

> ⚠️ Running `-v "$PWD":/work` in PowerShell gives `docker: invalid reference format` — PowerShell's `$PWD` expands to `D:\offline-node` (backslashes), and docker's volume parsing breaks. **Use the forward-slash hardcoded path above**, or **switch to WSL2**.

**Inside the container, in sequence: install the node → verify the artifact architecture → exit** (all inside the container, one pipeline):

```bash
npm install node-red-contrib-mcprotocol

# Verify the artifacts really are arm64 (the slim image has no `file` command by default — install it first):
apt update && apt install -y file
find node_modules -name "*.node" -exec file {} \;
# Expected output: ELF 64-bit LSB shared object, ARM aarch64, ...
# No .node files → pure JS, architecture doesn't matter ✅
# x86-64 → wrong environment, repackage ❌

exit   # exit the container
```

**After exiting the container, where do I find the artifacts?**

Go back to **the folder where you ran `docker run`** (that's what `$PWD` was on the command line, and it's the mount point). Artifact path = `that-folder\node_modules\node-red-contrib-mcprotocol`.

Example: if on Windows you `cd D:\offline-node` first and then run docker run, the artifacts are at:

```
D:\offline-node\node_modules\node-red-contrib-mcprotocol
```

Confirm with File Explorer, or (in a WSL2 / Git Bash terminal):

```bash
ls node_modules/node-red-contrib-mcprotocol
```

> 📌 **Notes on the Docker route**:
>
> 1. **Docker must be installed first**: Docker Desktop on Windows (needs the WSL2 backend, reboot after install); docker engine on Linux. `docker --version` should print a version.
> 2. **How it works**: Docker emulates arm64 with QEMU; node-gyp calls the arm64 gcc *inside the container*, so the artifacts are genuine aarch64 binaries.
> 3. **There is no Node-RED in the container** — `node:20-bookworm-slim` is the official Node.js image, only node/npm. **That's fine, no error**: packaging a node only needs node/npm; **Node-RED itself isn't needed** (Node-RED is just the host that *runs* nodes — it's not used during packaging). `npm install node-red-contrib-mcprotocol` puts the node plus all dependencies into the container's `/work/node_modules`; after exiting, they're directly available in the host's current directory.
> 4. **The verify command must run inside the container** (or a WSL2 / Linux terminal) — Windows PowerShell **has no `file` command** and `find` isn't native either; and the slim image has no `file` by default, so install it first as shown above.

> 💡 **workflows (flows.json) are cross-platform, but nodes are not**: flows.json is plain JSON, so a flow debugged on Windows can be copied to the IoT2050 directly; but the **nodes it references must be repackaged for arm64**. In other words — in the "debug on Windows + move everything" approach, **the flow file can move; the node packages must be rebuilt**.

---

## 4. Step 2: SSH into the IoT2050

### Method 1: PowerShell (built into Windows 10/11)

```powershell
ssh root@<IoT2050_IP>
```

Enter the login password to reach the device shell.

### Method 2: WinSCP GUI (beginner-friendly)

| Item | Value |
| --- | --- |
| Protocol | SFTP (or SCP) |
| Hostname | IoT2050's IP |
| Username | root |
| Port | 22 |

---

## 5. Step 3: Upload the node folder via SCP

> Target path: `/root/.node-red/node_modules/`

### Option A: WinSCP (graphical, simplest)

1. Locate the `node-red-contrib-mcprotocol` folder packaged in §3
2. Drag the whole folder to the remote path: `/root/.node-red/node_modules/`

### Option B: PowerShell scp command

```powershell
# Upload the whole folder
scp -r D:\offline-package\node-red-contrib-mcprotocol root@<IoT2050_IP>:/root/.node-red/node_modules/
```

### ✅ Verify after upload (run in the IoT2050 shell)

```bash
ls /root/.node-red/node_modules/node-red-contrib-mcprotocol
```

If you see `package.json` → upload succeeded.

> ⚠️ If it says the directory doesn't exist, create it first:
>
> ```bash
> mkdir -p /root/.node-red/node_modules
> ```

---

## 6. Step 4: Edit package.json (recommended, optional)

> Note: this step is **not required** (Node-RED discovers nodes by scanning node_modules), but adding it lets `npm install` reinstall these nodes automatically when you're online later — recommended.

```bash
# ① Enter the working directory
cd /root/.node-red

# ② Back up the original file (don't skip this as a beginner)
cp package.json package.json.bak

# ③ Edit
nano package.json
```

Find the `dependencies` field and add one line (**use the version you actually downloaded**):

```json
"node-red-contrib-mcprotocol": "^2.4.0"
```

Reference after editing (version is illustrative — use the actual one):

```json
{
    "name": "node-red-project",
    "description": "A Node-RED Project",
    "version": "0.0.1",
    "private": true,
    "dependencies": {
        "node-red-contrib-mcprotocol": "^2.4.0"
    }
}
```

Save in nano: `Ctrl+O` → Enter → `Ctrl+X`.

> 📌 **How to find the real version?** In the IoT2050 shell:
>
> ```bash
> cat /root/.node-red/node_modules/node-red-contrib-mcprotocol/package.json | grep '"version"'
> ```

---

## 7. Step 5: Restart Node-RED

### Method 1: systemd management (IoT2050's standard deployment, ✅ recommended)

```bash
sudo systemctl restart node-red
```

Check the status:

```bash
sudo systemctl status node-red
```

- `Active: active (running)` → started normally
- `failed` → troubleshoot per §9

Watch the startup log live:

```bash
sudo journalctl -u node-red -f
```

### Method 2: process-based start/stop (fallback, non-systemd setups)

```bash
pkill -f node-red
node-red
```

> ⚠️ If the IoT2050 uses the systemd service (Method 1), **don't** start it manually with Method 2 — it conflicts with the service. Pick one.

---

## 8. Step 6: Verify the node is loaded

1. **Open the Node-RED editor**: browser at `http://<IoT2050_IP>:1880`
2. In the left node palette, search for `MC` (or the node-name keyword)
3. If you see `MC Protocol` nodes → install succeeded ✅

Command-line verification (optional):

```bash
# Or just check the log for errors
sudo journalctl -u node-red | grep -i mcprotocol
```

---

## 9. Troubleshooting

### 9.1 New node not visible in the palette

- Make sure the folder name is exactly right: `node-red-contrib-mcprotocol` (case and hyphens must match)
- Make sure `package.json` exists inside: `ls /root/.node-red/node_modules/node-red-contrib-mcprotocol/`
- Check the log for load errors: `sudo journalctl -u node-red -f`
- Check the folder owner: `ls -ld /root/.node-red/node_modules/node-red-contrib-mcprotocol` (should be root)

### 9.2 `module not found` at startup

> ❗ **The most common trap**: you only uploaded the node source ZIP, missing sub-dependencies.
> ✅ **Correct**: you must **copy the complete folder after `npm install` on an online machine** (§3), so all dependencies are present.
> ✅ If dependencies are still missing: make sure the packaging machine is **arm64 + a close Node.js version**, then repackage.

### 9.3 `ERR_DLOPEN_FAILED` / "wrong ELF class" / "cannot open shared object"

> ❗ **Architecture mismatch**: most likely packaged with `npm install` on a **Win11 / x86 PC** — you got win32-x64 native binaries, but the IoT2050 is Linux-arm64, so loading fails.
> ✅ **Fix**: go back to §3, **repackage with Docker-emulated arm64**, and run `find node_modules -name "*.node" -exec file {} \;` to confirm the output is `ARM aarch64` before uploading.

### 9.4 Permission errors (EACCES / can't write)

```bash
chown -R root:root /root/.node-red
```

### 9.5 Remember these path rules

- `/usr/lib/node_modules` is **managed by Debian packages** (global modules installed via apt), not an npm project directory — **don't touch it**
- From now on, run `npm install` for custom nodes under `/root/.node-red`

---

## 10. Quick checklist

```
□ Node folder uploaded to /root/.node-red/node_modules/ and contains package.json
□ (optional) package.json dependencies has a line added with the actual version
□ sudo systemctl restart node-red has no errors
□ sudo systemctl status node-red shows active (running)
□ Browser http://<IoT2050_IP>:1880 palette can find the MC Protocol node
```

---

## References

- Node-RED official docs: https://nodered.org/docs/
- Node library (flows): https://flows.nodered.org/
- Example node `node-red-contrib-mcprotocol`: https://flows.nodered.org/node/node-red-contrib-mcprotocol
- Node-RED directory structure (user data directory): https://nodered.org/docs/user-guide/runtime/

---

> ⚠️ **Disclaimer**: This article is a personal technical write-up reflecting the author's own experience and opinions, unrelated to any commercial organization. The operations described involve software deployment on embedded devices — proceed with caution. For production or industrial use, have qualified personnel perform and fully test the changes. The author accepts no liability for any direct or indirect loss resulting from the use of this content.
