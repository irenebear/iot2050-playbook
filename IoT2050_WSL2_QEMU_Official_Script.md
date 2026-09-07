# Run the Official IoT2050 Image Without Real Hardware: Win11 + WSL2 + QEMU Official Script

> Published: 2026-08-15
> Audience: Embedded developers who don't own an IoT2050 but want to get the environment running, validate apps, and debug code in advance.
> Environment: Windows 11 + WSL2 (Ubuntu) + QEMU image built with GitHub Actions.
> Official reference: [`siemens/meta-iot2050`](https://github.com/siemens/meta-iot2050) `doc/qemu.md`

## Table of Contents

- [Foreword](#foreword)
- [1. What Is QEMU?](#1-what-is-qemu)
- [2. Prerequisites](#2-prerequisites)
- [3. Step 1: Make GitHub Actions Produce a QEMU Image](#3-step-1-make-github-actions-produce-a-qemu-image)
- [4. Step 2: Install QEMU in WSL2](#4-step-2-install-qemu-in-wsl2)
- [5. Step 3: Copy the Image into WSL2 and Get the Official Launch Script](#5-step-3-copy-the-image-into-wsl2-and-get-the-official-launch-script)
- [6. Step 4: Place Files According to the Official Script Layout (Important)](#6-step-4-place-files-according-to-the-official-script-layout-important)
- [7. Step 5: Verify the System (SSH Login)](#7-step-5-verify-the-system-ssh-login)
- [8. FAQ](#8-faq)
- [References](#references)

---

## Foreword

The Siemens IoT2050 is an ARM64-based industrial edge gateway; Siemens builds its Debian example image with Isar. The problem: **many people don't have the real device**, yet still want to get the environment running, validate applications, and debug code in advance — what to do?

The answer is **QEMU system emulation**: inside WSL2 on Windows 11, use `qemu-system-aarch64` to boot the IoT2050 image as a whole machine. The serial console shows the boot log directly, and SSH gives you the same experience as the real device.

This article covers the whole flow, with the focus on **the correct usage of the official launch script** (there is a big trap: the image files must be placed in the exact directory structure the script expects, otherwise the script fails with `No such file or directory`). This flow is **verified in practice** on WSL2 — follow along and it will run.

---

## 1. What Is QEMU?

**QEMU** (Quick Emulator) is an open-source hardware emulator that lets you run an OS of one CPU architecture on a computer of another architecture. In one sentence: **a software emulator for "wanting to play with another machine/architecture without actually buying one".**

It has two working modes:

| Mode | What it does | Typical command |
| --- | --- | --- |
| **System Emulation** | Emulates a whole machine (CPU + memory + disk + peripherals), can run a **full OS** | `qemu-system-aarch64` |
| **User-mode Emulation** | Only translates instructions of a single program; no standalone OS | `qemu-aarch64` |

An analogy:
- **System emulation = renting a whole office**: you need rooms, desks, and utilities before you can work — costly and slower
- **User-mode emulation = hiring a single person**: the person brings their skills and gets to work — lightweight and fast

This article uses **system emulation**: boot the IoT2050 image built by GitHub Actions (`.wic` disk + kernel + initrd) as a whole machine, then verify the Debian example system over serial and SSH.

> Note: **a QEMU image is not a Docker image.** QEMU is whole-machine virtualization; Docker is process-level isolation — they are different kinds of artifacts. This article uses QEMU throughout; no local Docker is needed.

---

## 2. Prerequisites

| Item | Requirement |
| --- | --- |
| Windows | Win11 with WSL2 (including an Ubuntu distro; `wsl -d Ubuntu` works) |
| GitHub | A fork of `siemens/meta-iot2050` |
| Network | WSL2 can reach the internet (needed to apt-install QEMU) |
| Disk | Image is ~2–5 GB; leave 10 GB+ free on the WSL2 drive |

---

## 3. Step 1: Make GitHub Actions Produce a QEMU Image

### 3.1 Key trap: the official workflow doesn't upload QEMU artifacts!

In your fork of `meta-iot2050`, the `debian-example-qemu-image` job in `.github/workflows/main.yml` **only builds — there's no upload-artifact step** (the example image and swupdate jobs have one; only the two QEMU jobs don't). In other words: **if you trigger a build as-is, the artifacts stay on the cloud runner and can't be downloaded from the Actions page.**

**Fix**: add an upload step to the QEMU job in **your own fork's** main.yml (modify your fork; the official repo stays untouched).

1. Open your fork: `https://github.com/<your-account>/meta-iot2050/blob/master/.github/workflows/main.yml`
   > ⚠️ Note the branch is **`master`**, not `main` (the official repo's default branch is master; many people write `main` and get a 404).
2. Click the pencil (edit) button, find the `debian-example-qemu-image:` job, and append **after** its last step (Build image):

```yaml
      - name: Upload qemu image
        uses: actions/upload-artifact@v4
        with:
          name: iot2050-example-qemu-image
          path: |
            build/tmp/deploy/images/iot2050-qemu/iot2050-image-example-iot2050-debian-iot2050-qemu.wic
            build/tmp/deploy/images/iot2050-qemu/iot2050-image-example-iot2050-debian-iot2050-qemu-*
```

3. Commit (directly to master is fine).

> About the artifact path: the QEMU build switches to `machine: iot2050-qemu` (see `kas-iot2050-qemu.yml`), so artifacts land in `build/tmp/deploy/images/iot2050-qemu/`, not the normal `iot2050/` directory.

### 3.2 Trigger the build

1. Go to your fork → **Actions** page → select **CI** on the left (or click "Run workflow" directly)
2. **Branch / Use workflow from**: select `master`
3. Check **Build Debian example QEMU image** (you can check only this one to speed things up)
4. Click **Run workflow**

> The build takes ~1–2 hours (the kernel is compiled in the cloud; you can walk away — even shut down your PC). When done, you'll see `iot2050-example-qemu-image` under **Summary → Artifacts** of that run.

### 3.3 Download the artifacts

Download the Artifact (a zip). After extracting you should have 3 files:

```
iot2050-image-example-iot2050-debian-iot2050-qemu.wic          ← disk image (core, ~4.9G)
iot2050-image-example-iot2050-debian-iot2050-qemu-vmlinux      ← kernel (note: vmlinux, not vmlinuz!)
iot2050-image-example-iot2050-debian-iot2050-qemu-initrd.img   ← initrd
```

> ⚠️ The kernel file downloaded in practice is named **`-vmlinux`** (spelling!), not the common `vmlinuz`. Always go by the actual filenames from your `ls`.

---

## 4. Step 2: Install QEMU in WSL2

Open a WSL2 Ubuntu terminal (`wsl -d Ubuntu`) and run:

```bash
sudo apt update
sudo apt install -y qemu-system-arm
```

Verify:

```bash
qemu-system-aarch64 --version
# should print something like: QEMU emulator version X.Y.Z
```

> On Ubuntu 22.04/24.04, the `qemu-system-arm` package also provides `qemu-system-aarch64` — one command is enough.

---

## 5. Step 3: Copy the Image into WSL2 and Get the Official Launch Script

```bash
mkdir -p ~/iot2050-qemu && cd ~/iot2050-qemu

# Copy the downloaded/extracted files from the Windows side (adjust the path; D: drive is /mnt/d in WSL)
cp /mnt/d/your-download-directory/* .

# Check the actual filenames
ls -lh
```

Get the launch script from the official repo:

```bash
wget https://raw.githubusercontent.com/siemens/meta-iot2050/master/scripts/host/start-qemu-iot2050.sh
chmod +x start-qemu-iot2050.sh
```

> **Recommended: download on the Windows side with real curl** (avoids all the pitfalls):
>
> ```powershell
> curl.exe -L -o start-qemu-iot2050.sh https://raw.githubusercontent.com/siemens/meta-iot2050/master/scripts/host/start-qemu-iot2050.sh
> ```
>
> Key points: `curl.exe` (bypasses the PowerShell alias) + `-L` (follow redirects) + `-o filename` (explicit output file). Then copy it into WSL with `cp /mnt/d/...`. On Windows there's no need to `chmod +x` — `bash start-qemu-iot2050.sh` works directly.

---

## 6. Step 4: Place Files According to the Official Script Layout (Important)

The official `start-qemu-iot2050.sh` works, but **it's strict about file locations**: the script builds a relative path from its own directory to find the image (lines 58-59: `${BASE_DIR}/build/tmp/deploy/images/iot2050-qemu/...`). So you **must** place the 3 image files in the specified subdirectory, not loose in the root — this is the easiest trap to fall into.

**The directory structure the script requires**:

```
~/iot2050-qemu/                            ← script directory (BASE_DIR)
├── start-qemu-iot2050.sh                  ← launch script
├── .config.yaml                           ← config (the script reads it from the current directory)
└── build/tmp/deploy/images/iot2050-qemu/  ← images MUST be here!
    ├── iot2050-image-example-iot2050-debian-iot2050-qemu.wic
    ├── iot2050-image-example-iot2050-debian-iot2050-qemu-vmlinux
    └── iot2050-image-example-iot2050-debian-iot2050-qemu-initrd.img
```

**Full procedure** (in a WSL2 terminal, done in one go):

```bash
cd ~/iot2050-qemu

# ① Create the directory structure
mkdir -p build/tmp/deploy/images/iot2050-qemu

# ② Move the 3 image files in (use the filenames from your ls)
mv iot2050-image-example-iot2050-debian-iot2050-qemu.wic \
   iot2050-image-example-iot2050-debian-iot2050-qemu-vmlinux \
   iot2050-image-example-iot2050-debian-iot2050-qemu-initrd.img \
   build/tmp/deploy/images/iot2050-qemu/

# ③ Create .config.yaml (the script reads it to decide which image to load)
cat > .config.yaml <<'EOF'
IMAGE_QEMU: true
IMAGE_EXAMPLE: true
IMAGE_SWUPDATE: false
EOF

# ④ Launch the emulator (this is the step that actually runs QEMU)
./start-qemu-iot2050.sh
```

> ✅ **Verified conclusion**: with the layout above, the official script runs without any code changes.
>
> ⚠️ Two must-knows:
> ① The image files **must be** in the `build/tmp/deploy/images/iot2050-qemu/` subdirectory, not the root (otherwise: `No such file or directory`);
> ② `.config.yaml` **must be in the script's directory** (the script uses the relative path `grep .config.yaml`), and must contain `IMAGE_QEMU: true` (it refuses to start if it reads `false`) and `IMAGE_EXAMPLE: true`.

---

## 7. Step 5: Verify the System (SSH Login)

1. **Serial login**: after boot, the terminal shows U-Boot → kernel boot log → Debian login prompt; username `root` (first login asks you to set a password).
2. **SSH login (recommended)**: open **another** WSL2 terminal (the QEMU process occupies the original one):
   ```bash
   ssh root@127.0.0.1 -p 22222
   ```
3. **Verify system info**:
   ```bash
   uname -a            # should show an aarch64 / arm64 kernel
   cat /etc/os-release # Debian trixie or the corresponding release
   ls /opt             # preinstalled components of the example image (Node-RED, etc.)
   ```

**Expected results**:
- ~1–3 minutes to reach the login prompt (software emulation is slower than real hardware — normal)
- After login you can run apt, Node-RED, snap7, etc. — behavior matches the real device (except performance)
- Network has a single virtual interface eth0; outbound access goes through QEMU user-mode NAT (internet works)

---

## 8. FAQ

| Symptom | Cause | Fix |
| --- | --- | --- |
| `qemu-system-aarch64: command not found` | Wrong package installed | `sudo apt install -y qemu-system-arm` |
| Hangs / black screen after boot | Kernel/initrd filenames don't match or are missing | Make sure the filenames from `ls` match what's placed |
| kernel panic: VFS: Unable to mount root fs | Wrong root device or missing initrd | Make sure the script's `.wic`/kernel/initrd are all present |
| Can't download QEMU artifacts from Actions | Official workflow has no upload step | Add the upload-artifact step to your fork per §3.1 |
| Emulation is slow | TCG software translation (no KVM) | Normal; fine for lightweight verification, use real hardware for heavy loads |
| Want to exit the emulator | – | Press **Ctrl+A then X** (serial mon:stdio shortcut) |
| Official script: "Please select IMAGE_QEMU" | `.config.yaml` has `IMAGE_QEMU: false` or is missing | Add .config.yaml (with `IMAGE_QEMU: true`) in the script's directory |
| Official script: "No such file or directory" | Images not in the subdirectory the script expects | Create `build/tmp/deploy/images/iot2050-qemu/` and move the 3 files in |
| `iot2050setup: command not found` | **The QEMU image intentionally omits it** (see below) | Normal — doesn't affect development; the real-hardware image includes it |

### Why is there no `iot2050setup` in the QEMU image? (intentional, by design)

This is **official design**, not an anomaly. `iot2050setup` (provided by the `board-conf-tools` recipe) is a **real-hardware-only configuration tool**; it's useless in QEMU, so the official build excludes it.

**Code evidence** (official repo `meta-example/recipes-core/images/iot2050-image-example.bb`):

```bitbake
IMAGE_INSTALL += " \
    ...
    mraa \
    ${@ 'board-conf-tools' if d.getVar('QEMU_IMAGE') != '1' else '' } \
    ...
"
```

- The QEMU machine config has `QEMU_IMAGE = "1"` → `board-conf-tools` is excluded;
- The same condition also excludes `firmware-update-package` (firmware update — also real-hardware-only).

**Why**: `iot2050setup.py` is entirely about real hardware — identifying the board via `/proc/device-tree/model`, configuring Arduino-header GPIO/UART/PWM pin muxing, driving the real `mraa` hardware library, network and boot-service settings. QEMU is a `machine: virt` virtual board with **no real Arduino headers or GPIO controllers**, so these features either error out or are meaningless in the VM.

**Conclusion**: seeing `command not found` in QEMU is expected and **doesn't affect any development verification** (Node-RED, OPC UA, data acquisition don't depend on it). The real-hardware image (Debian example image) **does include** `iot2050setup`.

---

## References

- Official QEMU guide: https://github.com/siemens/meta-iot2050/blob/master/doc/qemu.md
- Launch script source: https://github.com/siemens/meta-iot2050/blob/master/scripts/host/start-qemu-iot2050.sh
- QEMU build fragment: https://github.com/siemens/meta-iot2050/blob/master/kas-iot2050-qemu.yml
- CI definition: https://github.com/siemens/meta-iot2050/blob/master/.github/workflows/main.yml
- Siemens official forum (building the example image): https://support.industry.siemens.com/forum/ww/en/posts/how-to-build-github-example-image/237116

---

> ⚠️ **Disclaimer**: This article is a personal technical write-up reflecting the author's own experience and opinions, unrelated to any commercial organization. The operations described involve embedded system image building and emulation — proceed with caution. For production or industrial use, have qualified personnel perform and fully test the changes. The author accepts no liability for any direct or indirect loss resulting from the use of this content.
