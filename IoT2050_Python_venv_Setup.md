# Creating a Python venv on the Siemens IoT2050: Debian 12's pip Ban (PEP 668) and the Right Way Forward

> Published: 2026-08
> Audience: Developers who want to install Python dependencies (e.g. `python-snap7`, `paho-mqtt`) on the Siemens IoT2050 (Debian) without breaking the system Python.
> Environment: IoT2050 Debian example image (Debian 12, Python 3.11+).

## Table of Contents

- [Why a venv is mandatory on IoT2050](#why-a-venv-is-mandatory-on-iot2050)
- [First, check: does a Debian package exist for the library you need?](#first-check-does-a-debian-package-exist-for-the-library-you-need)
- [Install venv and pip support](#install-venv-and-pip-support)
- [Create the project directory and virtual environment](#create-the-project-directory-and-virtual-environment)
- [Install dependencies inside the venv](#install-dependencies-inside-the-venv)
- [Using the venv](#using-the-venv)
- [Optional: systemd auto-start](#optional-systemd-auto-start)
- [FAQ](#faq)
- [Summary](#summary)

---

## Why a venv is mandatory on IoT2050

The IoT2050 example image is based on **Debian Linux**. Since **Debian 12 (Python 3.11+)**, the system Python is treated as a **critical system component**:

- If `pip` writes freely into `/usr/lib/python3.x`, it can break `apt`, system tools, and the upgrade path
- Therefore Debian **directly forbids** `pip install xxx` / `pip3 install xxx`

Running it directly gives:

```
error: externally-managed-environment
hint: See PEP 668
```

**Only two installation methods are allowed:**

| Method | Description |
| --- | --- |
| `apt install python3-xxx` | Install Debian's officially maintained package |
| **`python3 -m venv`** (recommended) | Create an isolated virtual environment with isolated dependencies |

The IoT2050 fully follows this Debian policy.

**Source**: [Why does the IoT2050 example image prohibit pip install? (Siemens official FAQ)](https://github.com/siemens/meta-iot2050/wiki/FAQs#why-does-the-iot2050-example-image-prohibit-pip-install)

---

## First, check: does a Debian package exist for the library you need?

Before choosing between `apt` and pip, check on the device itself:

```bash
# ① Update the sources first (otherwise the search index may be stale)
sudo apt update

# ② Fuzzy search: find packages by name/description
apt-cache search python3-snap7
apt-cache search python3- | grep -i snap7

# ③ Check the exact version/status of a package in the repo
apt-cache policy python3-snap7 python3-paho-mqtt

# ④ See what other packages a package depends on
apt-cache depends python3-paho-mqtt

# ⑤ List installed python3-related packages
dpkg -l | grep python3
```

**Reading the results:**

| Output | Meaning | Action |
| --- | --- | --- |
| `python3-snap7` appears in the results | The official repo has this Python package | You can `apt install python3-snap7` as a fallback |
| Only `libsnap7-1` / `libsnap7-dev` | The official repo only has the C library, no Python bindings | Must use venv + pip |
| No output at all | The official repo doesn't have this library | Use venv + pip |

> Practical experience: the Debian official package for `python-snap7` is frequently missing — apt only has the C library (`libsnap7-1`), so the Python bindings must be installed with pip. This is exactly the typical use case for the venv approach.

---

## Install venv and pip support

The IoT2050 Debian image ships with python3, but **the venv module isn't installed by default**. Install it first:

```bash
sudo apt update
sudo apt install -y python3-venv python3-pip
```

Verify the venv module works:

```bash
python3 -m venv --help >/dev/null && echo "venv OK"
```

---

## Create the project directory and virtual environment

```bash
# Create the project directory (/opt requires sudo)
sudo mkdir -p /opt/plc_poller

# Create the venv as root (avoids permission mess)
sudo python3 -m venv /opt/plc_poller/s71200

# (Optional) change ownership to the current user for day-to-day convenience
sudo chown -R $USER:$USER /opt/plc_poller
```

Confirm the interpreter exists afterwards:

```bash
ls -l /opt/plc_poller/s71200/bin/python
/opt/plc_poller/s71200/bin/python --version
```

---

## Install dependencies inside the venv

**Key: always use the venv's pip, never the system pip!**

```bash
# Upgrade pip (inside the venv)
/opt/plc_poller/s71200/bin/python -m pip install --upgrade pip

# Install the S7 communication library + MQTT client
/opt/plc_poller/s71200/bin/pip install python-snap7 paho-mqtt

# (Optional) local archiving with SQLite needs no install — it's in the stdlib
/opt/plc_poller/s71200/bin/python -c "import sqlite3; print(sqlite3.sqlite_version)"
```

Verify snap7 imports:

```bash
/opt/plc_poller/s71200/bin/python -c "import snap7; print(snap7.__version__)"
```

---

## Using the venv

**Method A: call it directly by absolute path (recommended for scripts/services)**

```bash
/opt/plc_poller/s71200/bin/python /opt/plc_poller/s71200/plc_poller.py
```

**Method B: activate for interactive use**

```bash
source /opt/plc_poller/s71200/bin/activate
python --version        # points to the venv's python
deactivate
```

---

## Optional: systemd auto-start

Create the service unit file `/etc/systemd/system/plc-poller.service`:

```ini
[Unit]
Description=PLC data poller (snap7 -> MQTT)
After=network-online.target

[Service]
ExecStart=/opt/plc_poller/s71200/bin/python /opt/plc_poller/s71200/plc_poller.py
WorkingDirectory=/opt/plc_poller/s71200
Restart=always
RestartSec=5
User=root

[Install]
WantedBy=multi-user.target
```

Enable and start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now plc-poller
sudo systemctl status plc-poller
```

---

## FAQ

| Symptom | Cause | Fix |
| --- | --- | --- |
| `ensurepip is not available` / can't create venv | Missing `python3-venv` package | `sudo apt install python3-venv` |
| `ModuleNotFoundError: No module named 'snap7'` | Not installed inside the venv | Use `/opt/plc_poller/s71200/bin/pip install python-snap7` (not the system pip) |
| `pip` reports `externally-managed-environment` | PEP 668 restricts system-level installs | Always install inside the venv |
| Forgot `sudo` when creating the /opt directory | /opt belongs to root | `sudo mkdir -p /opt/plc_poller` |
| `python` inside the venv still shows the system path | Not activated, or not using the absolute path | Use `/opt/plc_poller/s71200/bin/python` absolute path — most reliable |

---

## Summary

1. **System-level `pip install` has been forbidden since Debian 12 (PEP 668)** — venv is the officially recommended, correct approach
2. **Check with `apt-cache search` first** whether an official package exists; if not, use pip
3. **Always use the venv's pip** (`/path/to/venv/bin/pip`), never touch the system pip
4. For scripts/services, **call the interpreter by absolute path** — most reliable, no dependency on `activate`

---

> ⚠️ **Disclaimer**: This article is a personal technical write-up for reference only; it does not represent any vendor's official position. For industrial environments, have qualified personnel evaluate and perform the operations at your own risk.
