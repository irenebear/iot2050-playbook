# S7-1200 Data to Cloud: IoT2050 + Python — from 1000-Point Acquisition, SQLite Archiving to MQTT Reporting (with Full Source Code & 4 Real-World Findings)

> Published: 2026-08
> Audience: IIoT developers collecting PLC data at the edge and reporting it to the cloud.
> Environment: S7-1200 (CPU1214 DC/DC/DC) → IoT2050 (Python + python-snap7) → SQLite archive → MQTT broker.

## Table of Contents

- [Background and Goal](#background-and-goal)
- [Test Topology](#test-topology)
- [Device Configuration Essentials](#device-configuration-essentials)
- [The Acquisition Script (plc_mqtt_loop.py)](#the-acquisition-script-plc_mqtt_looppy)
- [4 Real-World Findings (the meat)](#4-real-world-findings-the-meat)
- [Deployment Steps](#deployment-steps)
- [Summary](#summary)

---

## Background and Goal

In IIoT scenarios, you often need to collect PLC data at an edge gateway and report it to the cloud. This article documents a complete pipeline that has been **verified end-to-end in practice**:

- **Data source**: S7-1200 (CPU1214 DC/DC/DC), two `Array[0..999] of INT` in DB1
- **Acquisition**: IoT2050 edge gateway, Python + python-snap7
- **Local archiving**: SQLite, 1000 rows/sec, 700MB cap, FIFO auto-cleanup
- **Reporting**: MQTT publishing JSON (1Hz), with a subscribe loop-back for verification

## Test Topology

| Device | IP | Role |
| --- | --- | --- |
| CPU1214 (G1) | 192.168.x.12 | PLC, data source |
| IoT2050 | 192.168.x.29 | Edge gateway, Python acquisition + MQTT reporting |
| Win11 (Docker Mosquitto) | 192.168.x.176 | Local test MQTT broker, anonymous access |

## Device Configuration Essentials

**CPU1214:**
1. Activate **S7 PUT/GET** in Device configuration
2. Create a **non-optimized-access** DB1 with two `Array[0..999] of INT`, each `Array[0]` increments by 1 every second

**IoT2050:**
- Python venv: `/opt/plc_poller/s71200`
- Dependencies: `python-snap7`, `paho-mqtt`

## The Acquisition Script (plc_mqtt_loop.py)

```python
#!/opt/plc_poller/s71200/bin/python
"""
plc_mqtt_loop.py - 1Hz S7-1200 → MQTT test script on IoT2050

Features:
  1. Every second, read 1000 INTs (2000 bytes) starting at DB1.DBW0 of the CPU1214
     - Exceeds the S7-1200 PDU limit of 240B; Snap7 auto-splits at the transport level (verified)
  2. Local archiving: 1000 rows/sec into SQLite (with timestamps), 700MB cap, FIFO cleanup
  3. Pack as JSON and publish to an MQTT broker
  4. Subscribe to the same topic and print the loop-back (only the 1st INT)

Deploy:
  scp plc_mqtt_loop.py root@192.168.x.29:/opt/plc_poller/s71200/
  ssh root@192.168.x.29 "/opt/plc_poller/s71200/bin/python /opt/plc_poller/s71200/plc_mqtt_loop.py"

Dependencies (already in the venv):
  pip install python-snap7 paho-mqtt
"""
import os
import json
import time
import signal
import sqlite3
import logging

import snap7
from snap7.util import get_int
import paho.mqtt.client as mqtt

# ---------- Config ----------
PLC_IP    = "192.168.x.12"     # CPU1214 IP
DB_NUM    = 1                   # DB1
START_B   = 0                   # DBW0
INT_N     = 1000                # read 1000 consecutive INTs (= 2000 bytes)
PERIOD_S  = 1.0                 # 1Hz sampling

MQTT_HOST = "192.168.x.176"    # docker mosquitto on Win11
MQTT_PORT = 1883
TOPIC     = "Node-red"

# ---------- SQLite archiving config ----------
DB_PATH     = "/opt/plc_poller/s71200/plc_data.db"
MAX_DB_MB   = 700                 # archive file cap: 700MB
ROW_BYTES   = 60                  # estimated bytes per row (incl. page/index overhead)
MAX_ROWS    = int(MAX_DB_MB * 1024 * 1024 / ROW_BYTES)   # ~12.22M rows
FIFO_EVERY  = 100                 # check FIFO every 100 rounds (100s), avoiding a full COUNT(*) each round

# ---------- Logging ----------
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(message)s",
    datefmt="%H:%M:%S",
)
log = logging.getLogger("plc-mqtt")

# ---------- SQLite ----------
_db = None
_fifo_ticks = 0                  # FIFO check counter (downsampled)

def init_db(path: str):
    """Create the table + enable WAL. Schema: raw_data(ts, tag, value, quality)."""
    global _db
    os.makedirs(os.path.dirname(path), exist_ok=True)
    _db = sqlite3.connect(path, check_same_thread=False)
    _db.execute("PRAGMA journal_mode=WAL")
    _db.execute("PRAGMA synchronous=NORMAL")
    _db.execute("""
        CREATE TABLE IF NOT EXISTS raw_data (
            id      INTEGER PRIMARY KEY AUTOINCREMENT,
            ts      TEXT    NOT NULL,        -- timestamp (ISO8601)
            tag     TEXT    NOT NULL,        -- DB1.INT0 .. DB1.INT999
            value   INTEGER NOT NULL,
            quality INTEGER NOT NULL
        )
    """)
    _db.commit()
    log.info("SQLite ready: %s (max %.0fMB, ~%d rows)", path, MAX_DB_MB, MAX_ROWS)

def fifo_cleanup():
    """FIFO cleanup (called at a reduced rate):
    estimate the row count from the id delta (id is AUTOINCREMENT, MAX-MIN≈row count),
    which uses the primary-key index O(log n) and avoids a full COUNT(*) per round."""
    global _fifo_ticks
    _fifo_ticks += 1
    if _fifo_ticks % FIFO_EVERY != 0:        # check only every FIFO_EVERY rounds
        return
    try:
        (mn,) = _db.execute("SELECT MIN(id) FROM raw_data").fetchone()
        (mx,) = _db.execute("SELECT MAX(id) FROM raw_data").fetchone()
        rows_now = mx - mn + 1               # estimated current row count
        if rows_now > MAX_ROWS:
            del_until = mx - MAX_ROWS        # keep only the newest MAX_ROWS rows
            _db.execute("DELETE FROM raw_data WHERE id < ?", (del_until,))
            _db.commit()
            log.warning("FIFO: deleted rows id<%d, now ~%d rows", del_until, rows_now - (del_until - mn))
    except Exception as e:
        log.error("SQLite FIFO failed: %s", e)

def archive(ts: str, values: list):
    """Batch-write 1000 rows; FIFO cleanup is handled by fifo_cleanup() (called at a reduced rate)."""
    if _db is None:
        return
    rows = [(ts, f"DB1.INT{i}", v, 0) for i, v in enumerate(values)]
    try:
        _db.executemany(
            "INSERT INTO raw_data(ts,tag,value,quality) VALUES(?,?,?,?)",
            rows,
        )
        _db.commit()
    except Exception as e:
        log.error("SQLite write failed: %s", e)

# ---------- S7 ----------
def s7_connect(ip: str) -> snap7.client.Client:
    c = snap7.client.Client()
    c.connect(ip, 0, 1, tcp_port=102)  # rack 0 / slot 1 (S7-1200 default)
    log.info("S7 connected: %s (rack0/slot1)", ip)
    return c

def read_ints(plc: snap7.client.Client):
    """Read 1000 INTs (2000B) in one call: beyond the PDU 240B cap, Snap7 auto-splits underneath."""
    raw = plc.db_read(DB_NUM, START_B, INT_N * 2)            # Snap7 auto-splits
    return [get_int(raw, i * 2) for i in range(INT_N)]       # S7 big-endian

# ---------- MQTT callbacks ----------
def on_connect(client, userdata, flags, rc):
    if rc == 0:
        log.info("MQTT connected rc=0  client=%s", client._client_id.decode())
    else:
        log.error("MQTT connect failed rc=%s  client=%s", rc, client._client_id.decode())

def on_message(client, userdata, msg):
    """Loop-back print: only print the 1st INT in the payload."""
    try:
        d = json.loads(msg.payload.decode("utf-8", errors="replace"))
        vals = d.get("values", [])
        first = vals[0] if vals else "N/A"
        print(f"[recv] {msg.topic}  INT0={first}", flush=True)
    except Exception as e:
        print(f"[recv] {msg.topic}  (parse fail: {e})", flush=True)

# ---------- Main loop ----------
_stop = False
def _term(signum, frame):
    global _stop
    _stop = True
signal.signal(signal.SIGINT, _term)
signal.signal(signal.SIGTERM, _term)

def main():
    plc = s7_connect(PLC_IP)
    init_db(DB_PATH)

    pub = mqtt.Client(client_id="iot2050-pub")
    pub.on_connect = on_connect
    pub.connect(MQTT_HOST, MQTT_PORT, keepalive=60)
    pub.loop_start()                                              # auto-reconnect

    sub = mqtt.Client(client_id="iot2050-sub")
    sub.on_connect = on_connect
    sub.on_message = on_message
    sub.connect(MQTT_HOST, MQTT_PORT, keepalive=60)
    sub.subscribe(TOPIC, qos=0)
    sub.loop_start()

    log.info(
        "1Hz loop started: DB%d.DBW%d..%d (x INT) -> mqtt://%s:%d/%s",
        DB_NUM, START_B, INT_N, MQTT_HOST, MQTT_PORT, TOPIC,
    )

    try:
        _next = time.monotonic()          # first-round anchor
        while not _stop:
            _next += PERIOD_S             # target time: strictly +PERIOD_S per round
            # ---- read PLC ----
            try:
                values = read_ints(plc)
            except Exception as e:
                log.error("S7 read failed: %s; reconnecting in 2s", e)
                try:
                    plc.disconnect()
                except Exception:
                    pass
                time.sleep(2)
                plc = s7_connect(PLC_IP)
                continue

            # ---- archive (local SQLite, with timestamp) ----
            ts = time.strftime("%Y-%m-%dT%H:%M:%S")
            archive(ts, values)
            fifo_cleanup()                            # FIFO cleanup (internally downsampled)

            # ---- publish ----
            payload = {
                "ts":     ts,
                "device": "iot2050-01",
                "src":    "CPU1214 DC/DC/DC",
                "db":     DB_NUM,
                "offset": START_B,
                "values": values,
            }
            pub.publish(TOPIC, json.dumps(payload), qos=0, retain=False)
            print(f"[send] {len(values)} INTs, INT0={values[0]}", flush=True)

            # ---- absolute-time alignment: sleep to the whole-second boundary (absorbs read/write/publish cost) ----
            delay = _next - time.monotonic()
            if delay > 0:
                time.sleep(delay)
            else:
                # overrun: log the real per-round overrun and re-anchor to now (prevents cumulative drift)
                log.warning("loop overrun by %.3fs", -delay)
                _next = time.monotonic()      # re-anchor, avoid unbounded overrun accumulation
    finally:
        log.info("shutting down...")
        sub.loop_stop(); sub.disconnect()
        pub.loop_stop(); pub.disconnect()
        try:
            plc.disconnect()
        except Exception:
            pass
        if _db is not None:
            _db.close()
        log.info("bye")

if __name__ == "__main__":
    main()
```

## 4 Real-World Findings (the meat)

### Finding 1: S7 communication continuity and data integrity

Analyzed from the SQLite archive data (94 hours, 3.44M rows):

- Steady-state 10 rows/sec for 94.1 hours
- Drop rate 0.12%, no major interruptions
- ⚠️ Lesson: a WAL-mode database **cannot be copied with `cp`** — it truncates/corrupts the file; use the `.backup` command instead

### Finding 2: CPU1214 PDU = 240 bytes, Snap7 auto-splits

- `plc.get_pdu_length()` returns **240**
- Reading 2000 bytes in one call: Snap7 splits it into multiple PDUs underneath — **no manual segmentation needed**

### Finding 3: 50ms stress test — serial PDU round-trips are the hard limit

Measured: **2849 bytes cannot be done every 50ms; the practical limit is ~6–7 times per second**:

- 2849B ÷ 240B/PDU ≈ 12 PDUs; Snap7 sends/receives them serially
- 12 × ~13ms ≈ 156ms per call
- Ways to speed up: read less data / async concurrent PDUs / use a CPU with a larger PDU (e.g. S7-1500, 960B) / lower the sampling rate to match

### Finding 4: fixed sleep causes sampling-period drift

The PLC increments by 1 each second, but readings skip numbers (+2)? Root cause:

- `sleep(1.0)` is a fixed sleep, but each round's read/write/publish overhead (~0.5s) isn't accounted for → the actual period becomes 1.5s
- **Fix: absolute-time alignment** — use the target time `_next += PERIOD_S` instead of a fixed sleep; the overhead is absorbed by the sleep, and the period stays strictly at 1s

## Deployment Steps

```bash
# 1. Create the venv on the IoT2050
sudo apt install -y python3-venv python3-pip
sudo python3 -m venv /opt/plc_poller/s71200

# 2. Install dependencies
/opt/plc_poller/s71200/bin/pip install python-snap7 paho-mqtt

# 3. Push the script and run
scp plc_mqtt_loop.py root@192.168.x.29:/opt/plc_poller/s71200/
ssh root@192.168.x.29 "/opt/plc_poller/s71200/bin/python /opt/plc_poller/s71200/plc_mqtt_loop.py"
```

## Summary

This solution closes the full "PLC data → edge archiving → cloud reporting" loop. Key takeaways:

1. **240B PDU is a hard limit of the CPU1214** — large reads are auto-split by Snap7, but high-frequency large reads have a performance ceiling
2. **Use SQLite + WAL + FIFO for local archiving** — 700MB cap with automatic cycling
3. **Align the sampling period with absolute time** — otherwise a fixed sleep causes drift and skipped readings
4. **Back up WAL databases with `.backup`** — a plain `cp` corrupts the file

---

> ⚠️ **Disclaimer**: This article is a personal technical write-up for reference only; it does not represent any vendor's official position. For industrial environments, have qualified personnel evaluate and perform the operations at your own risk.
> IP addresses in this article have been desensitized (192.168.x.x); replace them with your own network configuration in practice.
