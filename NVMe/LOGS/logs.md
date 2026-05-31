
In NVMe, “logs” are data structures returned by the **Get Log Page** admin command (opcode `0x02`). Each log page is identified by a **Log Page Identifier (LID)** and exposes controller/namespace telemetry, health, errors, and feature-specific state.

---

## 🧠 How log retrieval works

- Command: **Get Log Page**
- Key fields:
    - **LID** → which log page to fetch
    - **NSID** → namespace scope (if applicable)
    - **NUMD / LPO** → transfer size and offset (for large logs)
- Returned via **Completion Queue** as a data buffer (often 4 KB chunks, but many logs are larger and require pagination).

---
## 📦 Core NVMe log pages (standard)

### 1) SMART / Health Information (LID = 0x02)

![https://images.openai.com/static-rsc-4/KJ_v-dj7oyU2wO7HQIYc63GbsCP1uEKErqbnegucXXrmelJ5uN2xvdnOHecfF7ce0wYgjpasAdEMThWA6B8mNj-g6kxzrDKamDtJvY2RBkpN9Xi8MXFge44N7K06Am1ks2mdtTCccwGSNSNvaiUyffX-Rmcp02Q_Gq4lmT5d2ca1847ZjvC6R40D2ith9omN?purpose=fullsize|403](https://images.openai.com/static-rsc-4/aex-RnEVyVlywwa_0SojvI6Di-x28V0-4rwcd4mENOqmfEM3faELuytPHF2T4Uwyg2QjnbbJljgVbOEx03Voj1rhJmOUZgaFFgjfX4esLSZwS2SIzWCLhmJjRIESDZPtV9N-LhVYjduM4Bylgz3rOMJKX465q1U6dieyNPsDvQY?purpose=inline)

![https://images.openai.com/static-rsc-4/Mp42b4am31fajbXPhlNRqO8Y0qciLeiLyKV2BIm2YGOIsj8FZpCWz9k_GgL9sKocy8iCCf7VMdEycDhER_Pw-OAID4xm4CkJ_7VB5MLzhOVqdsqxzyoE7kTiGHYhR5SXjaRmR11RRYuihDPYDHwULk9y6th202QsjJuYdatrGrmnD1-XDtcAS-JRvo7H1FA9?purpose=fullsize|394](https://images.openai.com/static-rsc-4/soG8CvN1ZdhM6qs1abmIQ9cdaqFCejxzBvc0t9-92Ulnfm9IFo3PkSmdWdnJJwHUSHqgrPz0O9Scs38L3cAS6CSuToIPnpcVaM3s5a2-HOMb0PckYhPN-S3Tz9s_k1NVN7sX8N-t-68ZmEcDw48VpIShkiVDLS3VPY7-gx5RjLc?purpose=inline)

![https://images.openai.com/static-rsc-4/c7RGM7jya-pYP8siWwhK7yWxSwgup8wG4ckXwUWoigAj-HmYvOdKl1q8caH3piYDy5YN8lSwnTTSyQQbfe8-ZWmgZ9Aanggmn_ZzR2K0JAaofju8t37fwx03qimEWZMeG-zrbtSMSGeNDJKF33taPe9dJgpo1coyGj7dmeP9zoDu4W2JEBBU8sGEZ5KAkEwg?purpose=fullsize|385](https://images.openai.com/static-rsc-4/iu8STOiflBvqBV9pRBnfxiltYNKgx649M32eX6QoWUAWxQeZ39tRVhThn7Z6s_K6aONYsMY_TGBq4nIpTLiLQdx6lIvfUGbKYWpz-FVvg2WXdrYP4CEGPu1ELMKgNgjtJ4rV7CY_g2tCW8PQrOU3NNq4uAiQUa5oK1K-wlnIRPU?purpose=inline)

**Purpose:** Device health and wear indicators

**Typical fields:**

- Critical Warning flags
- Temperature (composite + sensors)
- Available Spare / Threshold
- Percentage Used (endurance)
- Data Units Read/Written
- Host Read/Write Commands
- Media Errors, Error Log Entries
- Power Cycles, Power-On Hours

👉 Most frequently polled by OS and monitoring agents.

---
### 2) Error Information Log ([[LID = 0x01]])

![https://images.openai.com/static-rsc-4/KJ_v-dj7oyU2wO7HQIYc63GbsCP1uEKErqbnegucXXrmelJ5uN2xvdnOHecfF7ce0wYgjpasAdEMThWA6B8mNj-g6kxzrDKamDtJvY2RBkpN9Xi8MXFge44N7K06Am1ks2mdtTCccwGSNSNvaiUyffX-Rmcp02Q_Gq4lmT5d2ca1847ZjvC6R40D2ith9omN?purpose=fullsize|414](https://images.openai.com/static-rsc-4/aex-RnEVyVlywwa_0SojvI6Di-x28V0-4rwcd4mENOqmfEM3faELuytPHF2T4Uwyg2QjnbbJljgVbOEx03Voj1rhJmOUZgaFFgjfX4esLSZwS2SIzWCLhmJjRIESDZPtV9N-LhVYjduM4Bylgz3rOMJKX465q1U6dieyNPsDvQY?purpose=inline)

![https://images.openai.com/static-rsc-4/Rb-sMVu-aFxKMY4pVSFV17NFF_1lOkNLTN2eTTrjKxnH3TtSEnpKEGhT5WSxG0OIoB1rlClNd6RTnR_T75wQaPd0_9nygxPbyfH1KXDkYcLphvEbDGpEv1C3N-p-yzMmJup0mYuijMJZ4K0y9vP1qvztYvhp8ndCpnTIf2GZqtLhyXOWBLmuXQs4vovo2oUd?purpose=fullsize|424](https://images.openai.com/static-rsc-4/7sFVZK2Zbc8MHTWlKOugaCDltAKKB990UWx11fwoI3M5ju4bY0wy6PpRrFqTOKahz0n1aklfRDv26WwvTpW6rArSNrUgNzt3-OmKE5vRGtLIIcx3KzTu423eeKF3ptxPnai-7bEKdP1XT8xdknFN5W4HWNhY9iP8BU2N3YwUL94?purpose=inline)

![https://images.openai.com/static-rsc-4/BXteKfNuyFqzQHvQGrdizgU2fNzaCh_OlTLewjVbF798SfbDPIGyBs_9nfx4xBZGQwG_XOcbJn2z6wU3ayuUfpL30wzw7Tq1_mpu9UNy8uTB-y3RSeciDcdbGEp8-B0_lMvhO9gsirMb78g8eaARiy6ZLYTp3-ZuIpukHC7HQZcXiKt35J6B2sIPi27lSFjU?purpose=fullsize|419](https://images.openai.com/static-rsc-4/T4Xt-dfGdmmADw1-pvfzwYnjSVuqZHnVOiEooO7IA4VHM6nIJBLDpOyeccw4b0zrM54SdlKFxKwy57s2XrZdcMMu0dqRE6AM1UvzX1Jm48Ax8pTWuXMp77MCcfnWSLCD4qcvCq-Qnq3JDqZwaLk-YB_K8P0zKXmRua65hqsu2_M?purpose=inline)

**Purpose:** Records failed commands

**Per-entry fields:**

- Error Count
- Submission Queue ID (SQID)
- Command ID (CID)
- Status Code / Status Code Type
- LBA (if relevant)
- Namespace ID

👉 Critical for debugging I/O failures.

---
### 3) Firmware Slot Information ([[LID = 0x03]])

- Active firmware slot
- Next activation slot
- Slot revision strings

👉 Used during firmware update workflows.

---
### 4) Changed Namespace List (LID = 0x04)

- NSIDs that have changed since last read  
    👉 Used for hot-plug / dynamic namespace management.
---
### 5) Commands Supported and Effects (LID = 0x05)

- Which commands are supported
- Side effects (e.g., may change namespace, require reset)

👉 Important for driver capability discovery.

---
### 6) Device Self-test Log (LID = 0x06)

- Results of background or host-initiated self-tests
- Pass/fail status, segment failures
---
### 7) Telemetry Logs (LIDs = 0x07, 0x08)

![https://images.openai.com/static-rsc-4/yEDgD_Z-iN37LHhzxRVXXM-U6BSj2XzQO8m3lkSxnGkgeYnHL4F1x-Nsr9xAzikWbZM2hchMi3teyrXmml5Ig1AOZERjPG6MCTVxlOrR30jOFOYACvBFk8yRX194rmHoOef55UwsNQ1aDPEeNq8a0EHbOFBV-K-1MENT7IKzn7SFv0g3x-ldkO1UBNr-cto9?purpose=fullsize|425](https://images.openai.com/static-rsc-4/wQPzPeUGAxOdtz2jum6L0VxK3pfPjf6cjlvTHWzo7oxjCCvGRlp-5wTR7j8HQAiGJ4hWikaXrjKavbiquXVteQzFQnUFXw295XLIWNDcZSFJewCkNoQzmEbfPvrnemmQytV3GaMMKeknNTB0sZuMnLcdEbiIDr4o1CryNBp2jDM?purpose=inline)

![https://images.openai.com/static-rsc-4/h37lIt9_EhwmG5C_8bWG6TSXc8y7-UiUKka-s7tODy21tfhtonmmXFYANrXA7eHI7tsAXKF0eUaWk0dIMZX81efbIXx0yPxlJUbXylIPHjj9gqEoHFjnlpKZvi3X7OLnIL5pFShsFVib6hDgidLlavQYUCMfu-G_P8U6cHKfua4njyFGxNzauuV6MF7BeYn0?purpose=fullsize|429](https://images.openai.com/static-rsc-4/OV84VYhfqRQYHBw9Z1Tl5tCj_k05oUbaa50HP6dSYh7i-pxrSuQYmdDQ-B2IQQDLL2jzbfMgtClMgpSXqCcW8qdO98RPWLPa89pr2mcs1hItwbpd9BEcjPORkvmmwreHn0SnDIEfvouBm9wY-329nG3_yIAs5dCdR17-jNeZ1_g?purpose=inline)

![https://images.openai.com/static-rsc-4/KF8OEK20zDc9m-4XxXMkOm4VlaPn8YjCpbRYKuEEnnNpRzf2ZpjjpNtwj5hna3HTSA0iVqrZSDdwYeFTwa8WsbNs7f1RQT8SQrjQz9eZjwra1gKB7xYdzszGvoYD4CgWvXlvlH7eTJ9FZJN3twypMV-vuxDSTjK_mLdmyndoVd603AhtseU05g_63CC6UFCI?purpose=fullsize](https://images.openai.com/static-rsc-4/8C3tXNotzf8U8saoQlka6-_hqxcPDCeGtC0hRe3SrvPfZ3DJKYWgyCWlXNDUM6U6sN-u90AE8giyNtqoZSResJ7UDEdEZUAkkoWDaQK0DHgbk5Uo1T9mXUVhNrNTgwRWjiD9tmtrZmSSkVM0JBrRJjE8hUXBYv2Ms0sF_KUdkuw?purpose=inline)

- **0x07**: Host-initiated telemetry
- **0x08**: Controller-initiated telemetry

**Purpose:** Deep diagnostics (often vendor-heavy)

- Internal firmware traces
- Event histories
- Performance anomalies

👉 Typically large and segmented; used for advanced debugging.

---
### 8) Endurance Group Log (LID = 0x09)

- Endurance group metrics (NVMe 1.4+)
- Wear and usage at group level

---

### 9) Predictable Latency / NVM Set Logs (various LIDs)

- Latency monitoring windows
- NVM set statistics

👉 Used in QoS-sensitive deployments.

---
### 10) Sanitize Status (LID = 0x81)

- Progress/status of sanitize operations (block erase, crypto erase)

---

##  Vendor-Specific Logs

Vendors define additional LIDs (typically ≥ `0xC0`), e.g.:

- Detailed NAND statistics
- Wear-level distribution
- Internal error counters
- Thermal throttling history

Accessed via:

- **nvme-cli**
- Vendor tools (Samsung, Intel, Kioxia, etc.)
---
## 🔧 Practical Example
bash
```
# SMART / health
nvme smart-log /dev/nvme0

# Error log
nvme error-log /dev/nvme0

# Firmware slots
nvme fw-log /dev/nvme0

# Telemetry (host-initiated)
nvme telemetry-log /dev/nvme0 --host
```

---
## ⚖️ Log Scope Summary
|Log Type|Scope|Use Case|
|---|---|---|
|SMART / Health|Controller-wide|Monitoring, alerting|
|Error Log|Controller-wide|Debugging failures|
|Namespace logs|Per-NS|Capacity, mapping changes|
|Telemetry|Controller/internal|Deep diagnostics|
|Vendor logs|Vendor-defined|Advanced analysis|

---
##  Key Insight

> **NVMe logs are structured telemetry endpoints accessed via Get Log Page—ranging from lightweight health counters to deep firmware diagnostics.**

---

If you want, I can:

- Break down **bit-level layout of SMART log (byte-by-byte)**
- Show **exact Get Log Page command DW10–DW15 encoding**
- Map logs to **real failure scenarios (media errors, timeout, thermal throttling)**