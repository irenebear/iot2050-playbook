# Remote-Debug Python on the Siemens IoT2050 with VS Code Remote-SSH: from Connecting to Setting Breakpoints (with Full Config)

> Published: 2026-08
> Audience: Developers who want to edit, debug, and inspect Python code on an edge device (Siemens SIMATIC IoT2050, Debian) without leaving VS Code.

## Table of Contents

- [Why Remote-SSH](#why-remote-ssh)
- [Prerequisites](#prerequisites)
- [Install the Remote-SSH Extension](#install-the-remote-ssh-extension)
- [Configure the SSH Target](#configure-the-ssh-target)
- [First Connection](#first-connection)
- [Open the Remote Project](#open-the-remote-project)
- [Install the Python Extension Remotely](#install-the-python-extension-remotely)
- [Select the venv Interpreter (Key)](#select-the-venv-interpreter-key)
- [Create a Debug Configuration](#create-a-debug-configuration)
- [Debugging Example](#debugging-example)
- [FAQ](#faq)
- [Summary](#summary)

---

## Why Remote-SSH

Developing Python on an edge device (e.g. the Siemens SIMATIC IoT2050 on Debian) traditionally means SSH terminal + vim/nano, with poor UX and debugging done by sprinkling `print`. With **VS Code Remote-SSH**, the device becomes "a local project":

- The code runs on the device, but editing, breakpoints, and variable inspection all happen on your PC
- No need to copy code back to the PC to debug (avoids environment mismatch)

## Prerequisites

| Item | Requirement |
| --- | --- |
| IoT2050 | Flashed with a Debian image, SSH reachable |
| IoT2050 IP | `192.168.x.29` in this article (adjust as needed) |
| SSH account | `root` (default password `root`) |
| PC | Windows 11 with VS Code installed |

Verify SSH:

```bash
ssh root@192.168.x.29
exit
```

## Install the Remote-SSH Extension

VS Code → Extensions panel (`Ctrl+Shift+X`) → search and install **Remote - SSH** (by Microsoft).

## Configure the SSH Target

**Method A: command palette (recommended)**

`Ctrl+Shift+P` → `Remote-SSH: Connect to Host...` → `+ Add New SSH Host...` → enter:

```
ssh root@192.168.x.29
```

Save it to `C:\Users\<you>\.ssh\config`.

**Method B: edit config manually**

```ini
# C:\Users\<your-name>\.ssh\config
Host iot2050
    HostName 192.168.x.29
    User root
    Port 22
```

From then on, `Remote-SSH: Connect to Host...` → pick `iot2050`.

## First Connection

1. `Ctrl+Shift+P` → `Remote-SSH: Connect to Host...` → select `iot2050`
2. Enter the password; choose **Linux** for the platform type
3. Wait while VS Code installs its server on the remote (1–2 minutes the first time)

The status bar shows `SSH: iot2050` when connected.

**Passwordless login (optional):**

```bash
ssh-keygen -t ed25519
ssh-copy-id root@192.168.x.29
```

## Open the Remote Project

`File → Open Folder...` (`Ctrl+K Ctrl+O`) → enter the project path (e.g. `/opt/plc_poller/s71200`).

## Install the Python Extension Remotely

In the remote window, install **Python** (Microsoft) — it installs on the remote side and doesn't affect your PC-side setup.

## Select the venv Interpreter (Key)

This step directly decides whether `import` succeeds:

1. Open any `.py` file
2. Click the Python version in the bottom-right status bar → `Enter interpreter path...` → enter the venv path (e.g. `/opt/plc_poller/s71200/bin/python`)

A more robust approach: create `.vscode/settings.json` in the project root:

```json
{
    "python.defaultInterpreterPath": "/opt/plc_poller/s71200/bin/python",
    "python.terminal.activateEnvironment": true
}
```

Apply: `Ctrl+Shift+P` → `Developer: Reload Window`.

> Tip: the venv only fully takes effect after you've run a debug session at least once; the status bar should show the venv path, not the system Python.

## Create a Debug Configuration

`Ctrl+Shift+D` → create `launch.json` → select **Python**:

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Python: hello",
            "type": "python",
            "request": "launch",
            "program": "${workspaceFolder}/hello.py",
            "console": "integratedTerminal",
            "cwd": "${workspaceFolder}",
            "justMyCode": true
        }
    ]
}
```

## Debugging Example

`hello.py`:

```python
import time

for i in range(5):
    print(f"hello {i}")
    time.sleep(1)
```

Debugging steps:

1. Click left of the line number of `print(f"hello {i}")` to set a **breakpoint** (red dot)
2. Press `F5` to start debugging
3. The program stops at the breakpoint: **Variables** to inspect variables, **Watch** to add expressions, `F10` step over, `F5` continue, `Shift+F5` stop

## FAQ

| Symptom | Fix |
| --- | --- |
| `Connection timed out` | `ping` the device IP; check the firewall allows SSH |
| `Permission denied` | Confirm the password; set `PermitRootLogin yes` in `/etc/ssh/sshd_config` |
| Importing third-party libraries fails | You're using the system Python → reselect the venv interpreter (§Select the venv Interpreter) |
| Debug reports `Interpreter not found` | Confirm the venv interpreter path exists: `ls -l` |

## Summary

Remote-SSH turns "remote device debugging" into "local development experience". Three core steps: **connect to the device → pick the right interpreter → set breakpoints**. Selecting the venv interpreter is the easiest step to get wrong — using the system Python makes third-party library `import`s fail.

---

> ⚠️ **Disclaimer**: This article is a personal technical write-up for reference only; it does not represent any vendor's official position. For industrial environments, have qualified personnel evaluate and perform the operations at your own risk.
> IP addresses in this article have been desensitized (192.168.x.x); replace them with your own network configuration in practice.
