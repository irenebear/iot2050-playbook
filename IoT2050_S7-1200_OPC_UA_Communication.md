# IoT2050 ↔ S7-1200 OPC UA Communication: Full Flow with an open62541 Client

> Published: 2026-08
> Audience: Embedded developers who have integrated OPC UA with open62541 on IoT2050 and need to verify real communication between the edge gateway and a Siemens S7-1200 (OPC UA server).
> Environment: IoT2050 (Debian, arm64) + open62541 1.4 + S7-1200 (CPU with built-in OPC UA server).
> Official references: [`siemens/meta-iot2050`](https://github.com/siemens/meta-iot2050), Siemens [S7-1200 G2 OPC UA official document](https://www.ad.siemens.com.cn/download/materialaggregation_3999.html)

## Table of Contents

- [Foreword](#foreword)
- [1. Goal and Principle](#1-goal-and-principle)
- [2. Prerequisites](#2-prerequisites)
- [3. TIA Portal Configuration (Key Steps)](#3-tia-portal-configuration-key-steps)
- [4. Verification on IoT2050 (open62541 Client)](#4-verification-on-iot2050-open62541-client)
- [5. Common Pitfalls Quick Reference](#5-common-pitfalls-quick-reference)
- [6. Appendix: S7-1200 G2 as OPC UA Server with UaExpert](#6-appendix-s7-1200-g2-as-opc-ua-server-with-uaexpert)
- [References](#references)

---

## Foreword

Using **IoT2050 (edge gateway) as an OPC UA client** to connect to the **OPC UA server built into a Siemens S7-1200** and read PLC data is the most common industrial use case. In practice, though, people get stuck in a few places: **why can't I browse the variables on the PLC side**, **why does the open62541 client segfault**, and **where is the interface entry in different TIA versions**.

This article distills the whole flow after running it end-to-end (**verified in practice**), covering:

- The **key step on the TIA Portal side** (server interface modeling, unique to S7-1200)
- Two ready-to-compile open62541 C clients (browse + read/poll)
- **3 open62541 coding pitfalls** hit in practice (SEGV / double free)
- The official reference path for S7-1200 G2 + UaExpert

---

## 1. Goal and Principle

```
IoT2050 (OPC UA client, open62541)  ──opc.tcp://<PLC_IP>:4840──▶  S7-1200 (OPC UA server)
```

IoT2050 acts as an edge gateway, collecting variables from the PLC data blocks via OPC UA.

---

## 2. Prerequisites (verify all of these first)

| Condition | Requirement | Notes |
| --- | --- | --- |
| CPU firmware | **Classic ≥ V4.4 / G2 ≥ V4.1** | Classic S7-1200 has a built-in OPC UA server from FW V4.4; S7-1200 G2 from V4.1 |
| Engineering software | **TIA Portal ≥ V16** (V21 for G2) | V16 is the minimum for configuring V4.4 S7-1200 |
| **Runtime license** | **SIMATIC OPC UA S7-1200 Basic** (order no. `6ES7823-0BA00-2BA0` paper / `6ES7823-0BE00-2BA0` download) | One license per PLC. Without it the OPC UA server won't work (or use the Basic license tied to the CPU) |
| Network | IoT2050 and PLC on the same subnet, pingable | Client address is fixed format `opc.tcp://<PLC_IP>:4840` |

---

## 3. TIA Portal Configuration (Key Steps)

### 3.1 Activate the OPC UA server

Device view → select the CPU → Properties → `OPC UA → Server` → check **"Activate OPC UA server"**.

After activation the server address is fixed: `opc.tcp://<PLC's IP>:4840`.

> ⚠️ **If you can't see the "OPC UA communication" folder in the project tree**: first make sure the OPC UA server is activated (in some versions the folder appears only after activating and compiling), then confirm the firmware version meets the requirement.

### 3.2 Model a "server interface" (unique to S7-1200 — mandatory!)

**Key difference between S7-1200 and S7-1500**:

| | S7-1500 | **S7-1200 (this article's scenario)** |
| --- | --- | --- |
| Checking "OPC UA accessible" on the DB | ✅ variables appear automatically after download | ❌ **not enough!** |
| Server interface | Optional | **must be modeled manually!** |

S7-1200 **does not support** the standard SIMATIC server interface — **just checking "OPC UA accessible" on the DB does nothing**. You must manually model an interface, drag variables into it, compile, and download before they become visible to clients. This is the trap beginners hit most often.

**Creating a server interface (entry differs by TIA version)**:

- **Method A (TIA V16~V20, generic)**: project tree → expand the CPU → `OPC UA communication > Server interfaces` → double-click **"Add new server interface"**
- **Method B (TIA V21, possible)**: project tree → CPU → **"Program blocks" → double-click "Add new block"** → choose object type **"OPC UA server interface"**

After naming and confirming, in the interface table's **"Add"** row, drag in the **DB or individual variables** to expose from the left side.

> ⚠️ **Arrays**: in S7-1200 OPC UA, every array element counts as a variable, with a limit of 2000. If your DB has two `Array[0..999]` blocks (2000 elements total) you'll hit the ceiling — it's better to **create a small optimized-access DB** with a few variables for testing, or drag in only the first few array elements.
>
> ⚠️ **Non-optimized DB compatibility**: OPC UA only supports **symbolic access**. Non-optimized DBs (e.g. ones used with snap7) **work if they have symbol names** — they can be dragged into the server interface too; but to be safe, use a small optimized-access DB during verification.

### 3.3 Security policy + runtime license + download

- **Security policy**: during debugging, CPU Properties → `OPC UA → Security`, check **"No security" (None)** for simplicity; for production use `Basic256Sha256` + certificates and disable other policies.
- **Runtime license**: CPU Properties → `Runtime licenses → OPC UA` → set the purchased license type (SIMATIC OPC UA S7-1200 Basic).
- **Trusted clients** (optional): during debugging you can check "automatically accept client certificates at runtime" to skip manual trusted-list setup.
- Compile the project → **download to the PLC (both hardware and software)**.

---

## 4. Verification on IoT2050 (open62541 Client)

### 4.1 Step 1: Browse the address space to confirm variables are exposed

Write a C client with open62541 that **recursively browses** the address space (skipping the Server system subtree, looking only at PLC data under DeviceSet):

```bash
cat > /tmp/ua_client_test.c <<'EOF'
#include <open62541/client.h>
#include <open62541/client_highlevel.h>
#include <open62541/client_config_default.h>
#include <stdio.h>

static UA_Boolean isInstance(UA_NodeClass nc) {
    return nc == UA_NODECLASS_OBJECT || nc == UA_NODECLASS_VARIABLE;
}

static void browseRecursive(UA_Client *client, UA_NodeId node, int depth) {
    if (depth > 6) return;
    UA_BrowseRequest bReq; UA_BrowseRequest_init(&bReq);
    bReq.requestedMaxReferencesPerNode = 100;
    bReq.nodesToBrowse = UA_BrowseDescription_new();
    bReq.nodesToBrowseSize = 1;
    /* Key: deep-copy into the request so bReq owns its memory (shallow copy of a STRING NodeId double-frees!) */
    UA_NodeId_copy(&node, &bReq.nodesToBrowse[0].nodeId);
    bReq.nodesToBrowse[0].resultMask = UA_BROWSERESULTMASK_ALL;
    UA_BrowseResponse bResp = UA_Client_Service_browse(client, bReq);
    if (bResp.responseHeader.serviceResult != UA_STATUSCODE_GOOD) {
        UA_BrowseResponse_clear(&bResp); UA_BrowseRequest_clear(&bReq);
        return;
    }
    for (size_t i = 0; i < bResp.resultsSize; ++i) {
        for (size_t j = 0; j < bResp.results[i].referencesSize; ++j) {
            UA_ReferenceDescription *ref = &bResp.results[i].references[j];
            if (!isInstance(ref->nodeClass)) continue;
            if (UA_NodeId_isNull(&ref->nodeId.nodeId)) continue;
            printf("%*s%.*s  [%s]  ns=%u", depth*2, "",
                   (int)ref->browseName.name.length, ref->browseName.name.data,
                   ref->nodeClass == UA_NODECLASS_VARIABLE ? "VAR" : "OBJ",
                   ref->nodeId.nodeId.namespaceIndex);
            if (ref->nodeId.nodeId.identifierType == UA_NODEIDTYPE_NUMERIC)
                printf(" i=%u", ref->nodeId.nodeId.identifier.numeric);
            else if (ref->nodeId.nodeId.identifierType == UA_NODEIDTYPE_STRING)
                printf(" s=%.*s", (int)ref->nodeId.nodeId.identifier.string.length,
                       ref->nodeId.nodeId.identifier.string.data);
            printf("\n");
            /* Skip the Server system subtree; only look at PLC data under DeviceSet */
            if (ref->nodeId.nodeId.namespaceIndex == 0 &&
                ref->nodeId.nodeId.identifier.numeric == 2253) continue; /* Server */
            UA_NodeId child; UA_NodeId_init(&child);
            if (UA_NodeId_copy(&ref->nodeId.nodeId, &child) == UA_STATUSCODE_GOOD) {
                browseRecursive(client, child, depth + 1);
                UA_NodeId_clear(&child);
            }
        }
    }
    UA_BrowseResponse_clear(&bResp); UA_BrowseRequest_clear(&bReq);
}

int main(void) {
    UA_Client *client = UA_Client_new();
    UA_ClientConfig_setDefault(UA_Client_getConfig(client));
    UA_StatusCode ret = UA_Client_connect(client, "opc.tcp://<PLC_IP>:4840");
    if (ret != UA_STATUSCODE_GOOD) {
        printf("Connection failed: %s (0x%08X)\n", UA_StatusCode_name(ret), ret);
        UA_Client_delete(client); return 1;
    }
    printf("=== Browsing PLC instance nodes (skipping Server subtree) ===\n");
    browseRecursive(client, UA_NODEID_NUMERIC(0, UA_NS0ID_OBJECTSFOLDER), 0);
    UA_Client_disconnect(client); UA_Client_delete(client);
    return 0;
}
EOF
gcc -o /tmp/ua_client_test /tmp/ua_client_test.c -lopen62541 && /tmp/ua_client_test
```

**Expected output (after correct configuration)**:

```
=== Browsing PLC instance nodes (skipping Server subtree) ===
Server  [OBJ]  ns=0 i=2253
DeviceSet  [OBJ]  ns=2 i=5001
  PLC_2  [OBJ]  ns=3 s=PLC
    OrderNumber  [VAR]  ns=3 s=OrderNumber
    SerialNumber  [VAR]  ns=3 s=SerialNumber
    ...
  ServerInterfaces  [OBJ]  ns=3 s=ServerInterfaces
    server_interface_1  [OBJ]  ns=4 i=1        ← the server interface you created
      aa  [VAR]  ns=4 i=12                      ← the variable you dragged in
      bb  [VAR]  ns=4 i=13
```

**Two normal phenomena (ignore them)**:
1. **Seeing two PLC_2 entries**: their NodeIds are identical (`ns=3 s=PLC`) — the same PLC object is browsed twice via multiple references under DeviceSet (HasComponent/HasOrderedComponent). **It's not two PLCs.**
2. **`CreatedAt` timestamp warning**: a hint that the PLC clock differs from IoT2050; it doesn't affect the connection.

**If `ServerInterfaces` is empty** → the server interface wasn't built or variables weren't dragged in on the PLC side; go back to §3.2.

### 4.2 Step 2: Read CPU info + poll variables every second (final verification)

Once the variable NodeIds are confirmed (from the browse output), read the order number / serial number and poll every second:

```bash
cat > /tmp/ua_client_test.c <<'EOF'
#include <open62541/client.h>
#include <open62541/client_highlevel.h>
#include <open62541/client_config_default.h>
#include <stdio.h>
#include <unistd.h>   /* sleep */

static void readValue(UA_Client *client, UA_NodeId nodeId, const char *name) {
    UA_Variant val; UA_Variant_init(&val);
    UA_StatusCode st = UA_Client_readValueAttribute(client, nodeId, &val);
    if (st != UA_STATUSCODE_GOOD) {
        printf("%s: read failed %s (0x%08X)\n", name, UA_StatusCode_name(st), st);
        return;
    }
    if (UA_Variant_hasScalarType(&val, &UA_TYPES[UA_TYPES_BOOLEAN]))
        printf("%s = %s\n", name, *(UA_Boolean*)val.data ? "true" : "false");
    else if (UA_Variant_hasScalarType(&val, &UA_TYPES[UA_TYPES_INT16]))
        printf("%s = %d\n", name, *(UA_Int16*)val.data);
    else if (UA_Variant_hasScalarType(&val, &UA_TYPES[UA_TYPES_INT32]))
        printf("%s = %d\n", name, *(UA_Int32*)val.data);
    else if (UA_Variant_hasScalarType(&val, &UA_TYPES[UA_TYPES_UINT16]))
        printf("%s = %u\n", name, *(UA_UInt16*)val.data);
    else if (UA_Variant_hasScalarType(&val, &UA_TYPES[UA_TYPES_UINT32]))
        printf("%s = %u\n", name, *(UA_UInt32*)val.data);
    else if (UA_Variant_hasScalarType(&val, &UA_TYPES[UA_TYPES_FLOAT]))
        printf("%s = %f\n", name, *(UA_Float*)val.data);
    else if (UA_Variant_hasScalarType(&val, &UA_TYPES[UA_TYPES_DOUBLE]))
        printf("%s = %f\n", name, *(UA_Double*)val.data);
    else if (UA_Variant_hasScalarType(&val, &UA_TYPES[UA_TYPES_STRING]))
        printf("%s = %.*s\n", name, (int)((UA_String*)val.data)->length, ((UA_String*)val.data)->data);
    else
        printf("%s = (type: %s not printed)\n", name, val.type ? val.type->typeName : "unknown");
    UA_Variant_clear(&val);
}

int main(void) {
    UA_Client *client = UA_Client_new();
    UA_ClientConfig_setDefault(UA_Client_getConfig(client));
    UA_StatusCode ret = UA_Client_connect(client, "opc.tcp://<PLC_IP>:4840");
    if (ret != UA_STATUSCODE_GOOD) {
        printf("Connection failed: %s (0x%08X)\n", UA_StatusCode_name(ret), ret);
        UA_Client_delete(client); return 1;
    }
    printf("Connected! (Ctrl+C to exit)\n");

    /* ===== One-shot: read CPU order number / serial number (ns=3 string nodes) ===== */
    UA_NodeId orderNo  = UA_NODEID_STRING_ALLOC(3, "OrderNumber");
    UA_NodeId serialNo = UA_NODEID_STRING_ALLOC(3, "SerialNumber");
    printf("--- CPU info ---\n");
    readValue(client, orderNo,  "OrderNumber");
    readValue(client, serialNo, "SerialNumber");
    UA_NodeId_clear(&orderNo);
    UA_NodeId_clear(&serialNo);

    /* ===== Poll every second: variables aa / bb (NodeIds from browse output; adjust as needed) ===== */
    UA_NodeId aa = UA_NODEID_NUMERIC(4, 12);
    UA_NodeId bb = UA_NODEID_NUMERIC(4, 13);
    printf("--- Polling aa / bb every second (Ctrl+C to exit) ---\n");
    int count = 0;
    while (1) {
        printf("[t=%d] ", count++);
        readValue(client, aa, "aa");
        readValue(client, bb, "bb");
        fflush(stdout);
        sleep(1);
    }
    UA_Client_disconnect(client); UA_Client_delete(client);
    return 0;
}
EOF
gcc -o /tmp/ua_client_test /tmp/ua_client_test.c -lopen62541 && /tmp/ua_client_test
```

**Expected output**:

```
Connected! (Ctrl+C to exit)
--- CPU info ---
OrderNumber = 6ES7214-1AG40-0XB0          ← CPU order number
SerialNumber = SVP-XXXXXXXXXX             ← serial number
--- Polling aa / bb every second (Ctrl+C to exit) ---
[t=0] aa = 1
[t=0] bb = 0
[t=1] aa = 1
...
```

**Verification points**:
- Order number / serial number print correctly → device info read OK (string parsing works)
- aa/bb refresh every second → dynamic polling works
- Change aa/bb online in TIA → the next output follows → **OPC UA communication verified** ✅

> 💡 To test "aa increments by 1 every second": add an OB1 timer in the PLC program to increment aa, and IoT2050's output will go up by 1 per second — the typical "PLC → OPC UA → edge gateway" data flow.

---

## 5. Common Pitfalls Quick Reference

| Symptom | Cause | Fix |
| --- | --- | --- |
| Connection refused / `BadSecurityModeInsufficient` | Server-side security policy isn't None; client has no certificate | During debugging set the PLC's OPC UA security policy to None; use certificates in production |
| Connection timeout | Wrong subnet / firewall | First `ping <PLC_IP>`, confirm same subnet |
| Server no response, diagnostics say "no license" | **Missing OPC UA runtime license** | Buy SIMATIC OPC UA S7-1200 Basic (6ES7823-0BA00-2BA0) and activate it |
| **Can't browse variables / ServerInterfaces is empty** | **Server interface not modeled** (checking "OPC UA accessible" on the DB isn't enough!) | Per §3.2 add a server interface and drag variables in, then compile and download |
| Can't find "Add new server interface" | Different TIA UI / OPC UA not activated | V16~V20: `OPC UA communication > Server interfaces`; V21: try "Program blocks > Add new block" and pick OPC UA server interface; activate the OPC UA server first |
| Firmware < V4.4 (classic) / < V4.1 (G2) | CPU too old | Upgrade the CPU firmware in TIA (needs a matching TIA version) |
| Client SEGV / double free | open62541 code memory-management error | See "open62541 coding pitfalls" below |

### open62541 coding pitfalls (hit in practice)

| Pitfall | Symptom | Correct approach |
| --- | --- | --- |
| Wrong `UA_NodeId_print(id, &s)` signature | **SEGV** (in open62541 1.4 this function is the **single-argument version returning UA_String**) | Don't use `UA_NodeId_print`; read the `namespaceIndex`/`identifier` fields and printf directly |
| Shallow-copying a STRING NodeId into the browse request | **double free / corruption** | Deep-copy with `UA_NodeId_copy` for `bReq.nodesToBrowse[0].nodeId`; deep-copy recursively passed parameters too |
| Recursive browse with no depth limit | Type nodes reference each other → explosion / crash | Limit depth to 5–6 and browse only instance nodes (OBJECT/VARIABLE), skipping type definitions |

---

## 6. Appendix: S7-1200 G2 as OPC UA Server with UaExpert

> Reference: Siemens official document (updated 2026-03): https://www.ad.siemens.com.cn/download/materialaggregation_3999.html
> Applies to: **S7-1200 G2** (new hardware) + **TIA V21**, using **UaExpert** as the test client.

### 6.1 Differences from the classic S7-1200

| Item | S7-1200 (classic) | **S7-1200 G2** |
| --- | --- | --- |
| Firmware requirement for OPC UA server | ≥ V4.4 | **≥ V4.1** |
| TIA version | ≥ V16 | **V21** |
| Server interface | must be modeled manually | also must be modeled manually (G2 doesn't support the standard SIMATIC interface) |
| Test client | UaExpert / open62541 etc. | UaExpert (G2 doesn't support OPC UA *client* functionality) |

### 6.2 TIA V21 configuration steps (G2)

1. **Create the PLC station**: create an S7-1200 G2 CPU (V4.1+) in TIA V21, set the subnet and IP.
2. **Activate the OPC UA server**: Device view → CPU Properties → `OPC UA` → check "Activate OPC UA server". Server address: `opc.tcp://<IP>:4840` (port 4840 by default, range 1024–49151; max sessions default 10).
3. **Security settings** (`OPC UA → Security`):
   - Security policy: check **"No security"** (None) during debugging; use `Basic256Sha256` in production and disable others.
   - Server certificate: a self-signed certificate is auto-generated after activation (manageable under `Protection & Security → Certificate manager`).
   - Trusted clients: during debugging you can check **"automatically accept client certificates at runtime"** to simplify.
4. **Set the runtime license**: CPU Properties → `Runtime licenses → OPC UA` → set the purchased license type (SIMATIC OPC UA S7-1200 Basic).
5. **Create a communication DB + server interface**:
   - Create a DB (e.g. `opc ua data`), check the property **"Data accessible from OPC UA"**, and enable read/write access on the variables as needed;
   - Project tree → PLC → `OPC UA communication → Server interfaces` → **Add new server interface** → select "Server interface" → OK;
   - Double-click `server_interface_1`, **drag** the OPC UA elements from the right side into the empty rows below the interface.
6. **Compile and download** to the PLC.

### 6.3 UaExpert connection steps

1. **Download UaExpert** (free): https://www.unified-automation.com/downloads/opc-ua-clients.html
2. Right-click **"Servers"** in the project tree on the left → Add Server → under `Custom Discovery` click `Double click to Add Server` → enter `opc.tcp://<PLC IP>:4840`.
3. Select security policy **None-None (uatcp-uasc-uabinary)** + **Anonymous** login → OK.
4. On first connect a certificate dialog appears → click **TRUST SERVER CERTIFICATE** to trust the server certificate.
5. After connecting, browse the variables under **server interface** in the address space window, drag them into the **Data Access View** for live reading; select a node to view its attributes in the Attributes window.

### 6.4 Notes

- G2's OPC UA server interface mechanism is the same as the classic S7-1200 (**no standard SIMATIC server interface** — you must model the interface manually and drag variables in).
- Use "no security" during debugging; enable `Basic256Sha256` + certificates in production.

---

## References

- open62541 website: https://open62541.org
- Siemens application example (S7-1200 OPC UA Server modeling): https://cache.industry.siemens.com/dl/files/701/109781701/att_1038809/v3/109781701_S7_1200_OPC_UA_Server_DOCU_V10_en.pdf
- Siemens FAQ (why the S7-1200 OPC UA server doesn't show interface content): https://support.industry.siemens.com/cs/document/109781442
- TIA Portal Information System (other OPC UA server settings): https://docs.tia.siemens.cloud/r/en-us/v21/configuring-automation-systems/using-opc-ua-communication-s7-1200-s7-1500-s7-1500t-s7-1200-g2/using-the-s7-1200-cpu-as-opc-ua-server-s7-1200/configuring-the-opc-ua-server-s7-1200/other-opc-ua-server-settings-s7-1200
- S7-1200 G2 OPC UA & UaExpert official document: https://www.ad.siemens.com.cn/download/materialaggregation_3999.html

---

> ⚠️ **Disclaimer**: This article is a personal technical write-up reflecting the author's own experience and opinions, unrelated to any commercial organization. The operations described involve configuring communication between PLCs and edge gateways — proceed with caution. For production or industrial use, have qualified personnel perform and fully test the changes. The author accepts no liability for any direct or indirect loss resulting from the use of this content.
