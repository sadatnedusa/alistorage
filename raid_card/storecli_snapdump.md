# StorCLI snapdump**, containing **low-level controller firmware modules and crash information**.

These files are part of what the controller firmware writes out when something severe (like a controller hang, firmware assert, or PCI error) happens. 
Let’s go through **what each file is**, **what it means**, and **how to interpret it safely without vendor-internal tools**.

```Command
#setenforce 0

#/opt/MegaRAID/storcli/storcli64 /call get snapdump

#ls -lrta
....
....
-rw-------. 1 root      root      1872062 Oct  9 19:33 snapdump_c1_id0_2025_10_09_19_12_23.zip
-rw-------. 1 root      root      1880283 Oct  9 19:33 snapdump_c1_id1_2025_10_09_19_14_51.zip
-rw-------. 1 root      root      1886930 Oct  9 19:33 snapdump_c1_id2_2025_10_09_19_15_14.zip
-rw-------. 1 root      root      1897430 Oct  9 19:33 snapdump_c1_id3_2025_10_09_19_15_45.zip
-rw-------. 1 root      root      1907443 Oct  9 19:33 snapdump_c1_id4_2025_10_09_19_16_17.zip
-rw-------. 1 root      root      1914354 Oct  9 19:33 snapdump_c1_id5_2025_10_09_19_18_49.zip
-rw-------. 1 root      root      1921837 Oct  9 19:33 snapdump_c1_id6_2025_10_09_19_39_46.zip
....
....

Unzip "snapdump_c1_id0_2025_10_09_19_12_23.zip" file into new directory named "extracted"

# unzip snapdump_c1_id0_2025_10_09_19_12_23.zip -d extracted
# cd extracted
# ls -lrta

-rw-------.  1 root root    32768 Oct  9 19:12 tracedump.bin
-rw-------.  1 root root   328366 Oct  9 19:12 tlb_page_pool_dump.txt
-rw-------.  1 root root   340733 Oct  9 19:12 symbol_dump.txt
-rw-------.  1 root root 12582896 Oct  9 19:12 eTTY.txt
-rw-------.  1 root root  3207784 Oct  9 19:12 crashDump.txt
-rw-------.  1 root root    34774 Oct  9 19:12 RMC_MODULE.txt
-rw-------.  1 root root   144935 Oct  9 19:12 NVRAM_MODULE.txt
-rw-------.  1 root root   131781 Oct  9 19:12 DDF_MODULE.txt
-rw-------.  1 root root   186355 Oct  9 19:12 CFGI_MODULE.txt
-rw-------.  1 root root 28636805 Oct  9 19:12 PL_MODULE.txt
-rw-------.  1 root root      126 Oct  9 19:12 ONFI_MODULE.txt
-rw-------.  1 root root   159101 Oct  9 19:12 ENCL_MODULE.txt
-rw-------.  1 root root   111566 Oct  9 19:12 HWREG.txt
-rw-------.  1 root root       97 Oct  9 19:12 crashDumpHost.txt
drwx------.  2 root root      320 Oct  9 19:36 .
drwxrwxrwt. 16 root root      600 Oct  9 19:36 ..


```
---

## 🧩 **1️⃣ Overview**

This type of snapdump is **firmware-level** (below normal StorCLI “show” output).
It’s captured directly from the controller’s NVRAM and trace buffers.

While only Broadcom/Seagate engineering has the full decoders, you can still extract meaningful insight — like whether the crash originated from firmware, hardware, or environment.

---

## 📂 **2️⃣ File-by-File Breakdown**

| File                       | Description                                                  | How to Interpret / Check                                                                                      |
| -------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------- |
| **symbol_dump.txt**        | Symbol table of firmware function names and offsets.         | Used to map trace addresses in crash dumps to function names. Mostly for vendor debugging.                    |
| **eTTY.txt**               | Embedded firmware console output (like a kernel serial log). | Look for lines like `ASSERT`, `panic`, `Timeout`, or `Error`. Shows last firmware activity before crash.      |
| **tracedump.bin**          | Binary trace buffer from controller firmware.                | Needs internal Broadcom “trace decoder”. Without it, you can only note its size and timestamp.                |
| **tlb_page_pool_dump.txt** | Memory management trace (Translation Lookaside Buffer info). | Look for anomalies like “allocation failure” or “corrupted entry.”                                            |
| **RMC_MODULE.txt**         | RAID Management Core data.                                   | Lists logical/physical config tables in memory at dump time. Can show missing drives or config mismatch.      |
| **NVRAM_MODULE.txt**       | Controller NVRAM snapshot.                                   | Contains cached metadata. You can check timestamps or corruption markers (`CRC`, `Invalid`).                  |
| **DDF_MODULE.txt**         | DDF (Disk Data Format) metadata from RAID config.            | Lists arrays, VDs, and PDs — can correlate with `/cX show` output.                                            |
| **crashDump.txt**          | Main controller firmware crash log.                          | **Start here.** Look for timestamps, error codes (e.g., `Exception`, `FW panic`, `ASSERT`), and module names. |
| **CFGI_MODULE.txt**        | Controller configuration state (volumes, policies).          | Often useful to confirm RAID topology.                                                                        |
| **PL_MODULE.txt**          | Physical Layer module (SAS/SATA link data).                  | Look for “Link Down”, “Phy Reset”, or “CRC error” messages — these hint at cabling/backplane issues.          |
| **ONFI_MODULE.txt**        | ONFI (flash interface) state.                                | Relevant if controller has NAND flash for cache or boot. Usually stable unless firmware bug.                  |
| **ENCL_MODULE.txt**        | Enclosure management (SES / SGPIO).                          | Shows connected JBOD enclosures, sensors, fan/temperature status.                                             |
| **HWREG.txt**              | Raw hardware registers dump.                                 | Low-level register values — not human readable, but confirms the controller hardware was alive.               |
| **crashDumpHost.txt**      | Host-side dump.                                              | May show PCIe transaction or driver-level fault detected by the OS (useful if host OS logged a PCI error).    |

---

## 🧠 **3️⃣ Practical Reading Strategy**

Even without vendor tools, you can do some triage:

### ✅ **Step 1 – Start with crashDump.txt**

```bash
grep -iE "error|fail|assert|panic|exception" crashDump.txt
```

If you see something like:

```
ASSERT: scsi_cmd.c:1200
FW panic: Null pointer
```

→ It’s a firmware crash; needs controller firmware update.

---

### ✅ **Step 2 – Check eTTY.txt**

This file often shows **live console traces** up to the crash.

```bash
grep -A3 -B3 "ASSERT" eTTY.txt
```

This can tell you *what operation the controller was performing* when it crashed (e.g., rebuilding, background patrol read, etc.).

---

### ✅ **Step 3 – Review DDF_MODULE.txt / CFGI_MODULE.txt**

Look for:

```bash
grep -i "degraded" DDF_MODULE.txt
grep -i "failed" DDF_MODULE.txt
```

If you find failed PDs or missing arrays, the crash may relate to degraded RAID volumes or bad media.

---

### ✅ **Step 4 – Check PL_MODULE.txt for SAS issues**

```bash
grep -i "link" PL_MODULE.txt | head
grep -i "error" PL_MODULE.txt | head
```

If you see repeated link resets or CRC errors, that’s usually a **backplane or cable issue**, not firmware.

---

### ✅ **Step 5 – Check NVRAM_MODULE.txt**

```bash
grep -i "CRC" NVRAM_MODULE.txt
grep -i "Invalid" NVRAM_MODULE.txt
```

Corrupted NVRAM → potential controller fault or firmware bug.

---

### ✅ **Step 6 – Match timestamps**

The top of each file often includes a timestamp like:

```
Timestamp: 2025-10-10 00:05:43
```

Use it to correlate with:

* OS syslog or dmesg (`megaraid_sas` driver logs)
* Controller reset events
* System power cycles

---

## ⚙️ **4️⃣ Common Root Causes Identified via Snapdump**

| Symptom                          | Typical Log Indicator                   | Possible Root Cause                          |
| -------------------------------- | --------------------------------------- | -------------------------------------------- |
| Repeated firmware panics         | “FW panic” or “ASSERT” in crashDump.txt | Firmware bug → upgrade controller firmware   |
| Link resets in PL_MODULE.txt     | “Link Down” or “Phy Reset”              | Bad cable, expander, or enclosure port       |
| Missing drives in DDF_MODULE.txt | “Drive not present”                     | Drive failure, or enclosure disconnect       |
| NVRAM corruption                 | “Invalid checksum”                      | Power loss during write, or NVRAM chip fault |
| PCIe errors                      | Lines in crashDumpHost.txt              | Host/driver PCIe reset issue                 |

---

## 🧾 **5️⃣ Next Steps (Best Practice)**

1. **Collect controller info:**

   ```bash
   /opt/MegaRAID/storcli/storcli64 /cX show all
   ```
2. **Attach the entire snapdump `.zip`** when opening a case with Broadcom or Seagate.
3. **Note the OS syslog or `dmesg` entries** around the same timestamp.
4. **Plan firmware upgrade** if your version is older than vendor’s latest “Critical” or “Recommended” release.

---
