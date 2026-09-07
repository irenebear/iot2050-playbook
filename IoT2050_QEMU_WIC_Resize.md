# Enlarging the IoT2050 QEMU Disk Image (.wic) — Two Approaches (4.9G → 8G / 20G)

> Published: 2026-08-17
> Audience: Embedded developers running Siemens IoT2050 in WSL2 + QEMU who find the system disk `/dev/vda1` too small.
> Environment: Windows 11 + WSL2 (Ubuntu) + QEMU image built with GitHub Actions.
> Official reference: [`siemens/meta-iot2050`](https://github.com/siemens/meta-iot2050) `doc/qemu.md`

## Table of Contents

- [The Problem](#the-problem)
- [Option A (Recommended): Change the Build Config — Build an 8G Partition Directly](#option-a-recommended-change-the-build-config)
- [Option B: Resize a Downloaded Image Manually](#option-b-resize-a-downloaded-image-manually)
- [Notes](#notes)
- [FAQ](#faq)
- [References](#references)

---

## The Problem

When emulating IoT2050 with WSL2 + QEMU, many people hit an awkward wall: **the system disk `/dev/vda1` is only 4.9G**, and after installing a few tools and Node-RED nodes it is already 90% full. A quick `df -h` shows:

```
/dev/vda1  4.6G  3.9G  467M  90%  /
```

You cannot install what you want, and cleaning up only treats the symptom — what to do?

**The answer: enlarge the .wic image.** There are two paths:

| | Option A: Change build config (recommended) | Option B: Resize the local image manually |
| --- | --- | --- |
| How | Edit the `wks` file in your fork so GitHub Actions **builds an 8G partition directly** | After downloading, resize manually with `qemu-img` + `parted` + `resize2fs` |
| Effect | **One change, permanent** — every build produces a large partition | One-shot — **you must redo it after every fresh image download** |
| Difficulty | Edit 1 line of config + rebuild | ~10 minutes of CLI work |
| Best for | Long-term emulator development | Occasional use, don't want to touch the repo |

Both paths are **verified in practice**. Full steps for each follow below.

---

## Option A (Recommended): Change the Build Config

### A.1 Why 8G instead of 20G?

The `.wic` partition size defaults to an estimate based on the rootfs content at build time (that is where 4.9G comes from). Adding `--size` to the wks file fixes the partition size.

**8G, not 20G**, because of two hard limits:

| Limit | Value | Consequence |
| --- | --- | --- |
| GitHub Actions runner disk | Standard `ubuntu-latest` has only **~14G available** | A 20G raw image + build cache will **fill the disk and fail** |
| Artifact upload limit | **Single artifact ≤ 10GB** | A 20G `.wic` **fails to upload** (4.9G works because it is <10G) |

An 8G partition is plenty for IoT2050 development (even the current 4.9G is only 90% used), and build/upload/download all stay safe.

### A.2 Which file to edit?

In your fork, locate:

```
meta/wic/iot2050.wks.in
```

Add `--size 8G` to the rootfs partition line (around line 11):

```diff
- part / --source rootfs-u-boot --sourceparams "no_initrd=yes" --fstype ext4 --label rootfs --align 1024 --use-uuid
+ part / --source rootfs-u-boot --sourceparams "no_initrd=yes" --fstype ext4 --label rootfs --align 1024 --use-uuid --size 8G
```

> Mechanism: when wks has no `--size`, wic estimates the partition size from the rootfs content; with `--size 8G` the partition is fixed at 8G.

### A.3 Commit and push

```bash
cd <your-local-fork-directory>

git add meta/wic/iot2050.wks.in
git commit -m "wic: enlarge rootfs partition to 8G for QEMU development"
git push origin <your-build-branch>
```

### A.4 Trigger a rebuild on GitHub Actions

1. Open your fork: `https://github.com/<your-account>/meta-iot2050/actions`
2. **Branch**: select the branch you just pushed
3. Check **Build Debian example QEMU image** (you can enable only the QEMU job to speed things up)
4. Click **Run workflow** and wait for the build to finish (~1–2 hours)

### A.5 Verify the artifact

```bash
# In WSL2, check the image size (should be ~8G, not 4.9G)
ls -lh <your-image-name>.wic

# After booting QEMU, check the partition (should be ~7.8G)
df -h /
```

### A.6 ⚠️ Scope of impact (important)

`meta/conf/machine/iot2050.conf` defines `WKS_FILE ?= "iot2050.wks.in"` — **all non-SWUpdate images share this wks file**. The 5 build targets in the fork's Actions:

| Workflow option | Image | Affected? |
| --- | --- | --- |
| **Debian example image** | `iot2050-image-example` (real hardware) | ✅ becomes 8G |
| **Debian example QEMU image** ← this article's scenario | QEMU variant (`machine: iot2050-qemu`) | ✅ becomes 8G |
| **Debian secure boot SWUpdate image** | SWUpdate A/B real hardware | ❌ unaffected |
| **Debian SWUpdate QEMU image** | SWUpdate QEMU variant | ❌ unaffected |
| **Bootloaders** | bootloader only | ❌ irrelevant |

> SWUpdate images use their own `iot2050-swu.wks.in` (A/B partitions hardcoded with `--fixed-size 4G`), so they are completely unaffected.
> **Real hardware note**: when flashing a normal image to an SD card, the **card must be ≥ 8G** (a 4G card cannot fit it).

### A.7 Optional variant: 8G for QEMU only, real hardware unchanged

If you do not want to affect real-hardware images, make it QEMU-only:

1. Create `meta/wic/iot2050.wks.qemu.in` (copy of the original, with `--size 8G` on the rootfs line)
2. Append to `meta/conf/machine/iot2050-qemu.conf`:
   ```bitbake
   WKS_FILE:iot2050-qemu = "iot2050.wks.qemu.in"
   ```
3. Revert the change to `iot2050.wks.in`

---

## Option B: Resize a Downloaded Image Manually

> For when the image is already downloaded and you don't want to rebuild.

### B.1 Understand the principle first: why is vda1 only 4.9G?

The `.wic` is a **raw disk image file** generated at build time; QEMU maps it straight to a virtual disk via `-drive file=xxx.wic,format=raw`. **The partition size is fixed when the image is built** — QEMU run-time parameters cannot change it.

Manual resizing therefore has three steps:

```
① Grow the raw file (qemu-img resize)
        ↓
② Grow the partition table (parted resizepart)
        ↓
③ Grow the filesystem (resize2fs)
```

> The image in this article has a single rootfs partition (GPT), which is the simplest case. If your image has partitions *after* rootfs, you must move partitions first (much more complex) — out of scope here.

### B.2 Step 0: Check the partition layout (determines difficulty)

```bash
cd <your-image-directory>
fdisk -l <your-image-name>.wic
```

- ✅ **Single partition** (Type = Linux filesystem) → follow this article, it's simple
- ⚠️ Multiple partitions and rootfs is not the last one → you must move partitions first; this article doesn't apply

### B.3 Step 1: Back up (mandatory!)

```bash
cp <your-image-name>.wic <your-image-name>.wic.bak
```

> Resizing rewrites the partition table; one mistake and the image is dead. Backing up is the safety floor — don't skip it.

### B.4 Step 2: Unlock the file (easiest trap to fall into!)

If `resize` reports `Failed to get "write" lock`, the **.wic file is in use**. Two causes:

**Cause 1: a loop device is still attached** → detach it:

```bash
sudo losetup -d /dev/loop0
```

**Cause 2: the QEMU VM is still running** (it holds the .wic file) → shut it down:

```bash
sudo pkill -f qemu-system-aarch64    # force kill (unsaved data will be lost)
```

> ⚠️ **Order matters: unlock first, then resize.** A mounted loop device keeps the file locked, and resize will keep failing until you detach it.

### B.5 Step 3: Grow the raw file

```bash
# Note: you must add -f raw, otherwise you get warnings and writes may be restricted
qemu-img resize -f raw <your-image-name>.wic 20G
```

A success prints `Image resized.` The target size 20G is adjustable (30G, 50G are fine).

> If `qemu-img` is missing: `sudo apt install -y qemu-utils`.

### B.6 Step 4: Re-attach as a loop device

```bash
sudo losetup -Pf <your-image-name>.wic
losetup -a    # confirm /dev/loop0 is attached
```

> The `-P` flag makes each partition appear as `/dev/loop0p1` etc., which is needed later for growing the filesystem.

### B.7 Step 5: Fix the GPT partition table (trap #2!)

If you run `resizepart` directly, you will see an error like:

```
Warning: Not all of the space available to /dev/loop0 appears to be used,
you can fix the GPT to use all of the space (an extra 31806970 blocks)...
Error: Unable to satisfy all constraints on the partition.
```

**Cause**: `qemu-img resize` only "stretched" the file, but **the backup GPT header still sits at the old end of the disk** — the partition table does not know about the new space at all.

**Fix: repair GPT first, then resize the partition**:

```bash
# parted is required (install it if missing)
sudo apt install -y parted

# Repair GPT: type Fix (or F) when prompted
sudo parted /dev/loop0 print fix
```

> Non-interactive: `sudo parted -s /dev/loop0 print fix 2>/dev/null || sudo parted /dev/loop0 print fix`
> Or use sgdisk for a cleaner fix: `sudo apt install -y gdisk && sudo sgdisk -e /dev/loop0`

### B.8 Step 6: Grow the partition + filesystem

```bash
# Grow the partition to the end of the disk (-s skips interactive confirmation)
sudo parted -s /dev/loop0 resizepart 1 100%

# Grow the filesystem (order matters: check first, then grow)
sudo e2fsck -f /dev/loop0p1
sudo resize2fs /dev/loop0p1
```

> `resize2fs` supports online ext4 resizing. If it reports filesystem errors, fix them with `e2fsck -f` first and retry.

### B.9 Step 7: Detach + verify

```bash
sudo losetup -d /dev/loop0

# ① File-level check: virtual size should now be 20G
qemu-img info <your-image-name>.wic | head -4
#   virtual size: 20 GiB (21474836480 bytes)  ← success

# ② Partition-level check (optional)
sudo losetup -Pf <your-image-name>.wic
sudo fdisk -l /dev/loop0 | grep loop0p1    # Size should be ~19.6G
sudo losetup -d /dev/loop0

# ③ Final check: boot QEMU normally, then inside the system
df -h /    # should show ~19.6G
```

### B.10 One-shot script (copy & paste)

```bash
cd <your-image-directory>
IMG=<your-image-name>.wic

# Unlock
sudo losetup -d /dev/loop0 2>/dev/null
sudo pkill -f qemu-system-aarch64 2>/dev/null

# Back up + grow the file
cp $IMG $IMG.bak
qemu-img resize -f raw $IMG 20G

# Fix GPT + grow partition + grow filesystem
sudo apt install -y parted
sudo losetup -Pf $IMG
sudo parted -s /dev/loop0 print fix 2>/dev/null || sudo parted /dev/loop0 print fix
sudo parted -s /dev/loop0 resizepart 1 100%
sudo e2fsck -f /dev/loop0p1
sudo resize2fs /dev/loop0p1
sudo losetup -d /dev/loop0

# Verify
qemu-img info $IMG | head -4
```

---

## Notes

| Item | Explanation |
| --- | --- |
| **Option A is permanent, Option B is one-shot** | A: change the build config once, every build gets a large partition; B: redo it after every fresh download |
| **No fstab changes needed** | Resizing doesn't change the partition UUID or mount point; it mounts normally after reboot |
| **rootfs is not the last partition** | Option B doesn't apply; you must move partitions first (complex) |
| **Backup file consumes disk** | `.wic.bak` is ~5G; you can delete it once you confirm everything works |
| **`--size 8G` is a floor, not a ceiling** | If the rootfs content ever exceeds 8G (too many packages), the build fails; change 8G to 10G and rebuild |

---

## FAQ

| Question | Answer |
| --- | --- |
| Getting `Failed to get "write" lock`? | The .wic is in use: `losetup -d /dev/loop0` + shut down QEMU, then retry |
| Getting `Unable to satisfy all constraints`? | You forgot to fix GPT: run `parted /dev/loop0 print fix` before `resizepart` |
| `qemu-img resize` shows warnings? | Add `-f raw` to specify the format; the warnings disappear |
| After resizing, `df -h` still shows 4.6G? | The partition table or filesystem wasn't grown; check whether all of B.7 / B.8 ran |
| Why doesn't Option A use 20G? | GitHub runner disk ~14G, artifact upload ≤10G — 20G would fail; 8G is safe |
| Does Option A affect real-hardware images? | Yes (normal images share the wks); use A.7 for QEMU-only if you don't want that |
| Don't want to touch the image at all? | Attach an extra vdb data disk to QEMU (`qemu-img create -f qcow2 data.qcow2 20G`) and store data there |

---

## References

- Official QEMU guide: https://github.com/siemens/meta-iot2050/blob/master/doc/qemu.md
- qemu-img manual: https://www.qemu.org/docs/master/tools/qemu-img.html
- GPT repair (sgdisk): https://www.rodsbooks.com/gdisk/
- Isar user manual (wks partition definitions): https://github.com/ilbers/isar/blob/master/doc/user_manual.md

---

> ⚠️ **Disclaimer**: This article is a personal technical write-up reflecting the author's own experience and opinions, unrelated to any commercial organization. The operations described involve modifying embedded system images — proceed with caution and always back up the image before modifying it. For production or industrial use, have qualified personnel perform and fully test the changes. The author accepts no liability for any direct or indirect loss resulting from the use of this content.
