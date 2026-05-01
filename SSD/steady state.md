In SSD testing, steady state is <span style="color: yellow">the long-term, stable performance level an SSD reaches after initial wear-in (FOB) and transition phases, characterized by consistent IOPS/throughput as NAND cells fill and background processes (like garbage collection) balance out write/erase cycles</span>. It reflects real-world, sustained performance, especially for data centers, where drives operate continuously, unlike client drives that idle, allowing TRIM/GC to refresh cells. Testing involves intense, prolonged random I/O to force the drive into this state, measuring performance without significant fluctuation over time. 

In SSD performance testing, **steady state** is:

> **The condition where the SSD’s performance stabilizes and reflects long-term, sustainable behavior after internal processes have fully engaged.**


## 🧠 Why Steady State Exists

Flash-based SSDs don’t behave like HDDs. Internally they rely on:

- **FTL (Flash Translation Layer)**
- **Garbage Collection (GC)**
- **Wear Leveling**
- **Over-provisioning management**

When the drive is **fresh or empty**, performance is artificially high because:

- Writes go to clean (erased) NAND blocks
- Minimal background work is needed

As the drive fills and cycles through writes:

- Valid data must be moved
- Blocks must be erased before reuse
- Internal housekeeping increases

Eventually, the drive reaches a **balance point** → **steady state**

## 🔄 Performance Evolution

![https://images.openai.com/static-rsc-4/RPh2GZEbgRzAWOU9rAtzRl-NA7ScbJRBgiOmse3_Mciu5o4TCAfp_-AseN-OPCZRNNl3_FsmDS5zMPp5hfkR8EseK9FeNTbAgZJRFBa1WhXL3AiyIxDMetV8F7sswv83sLpWNBNYgi5PPgkEdGpaksRI1MPlW4uCPQ6YneEIBPFxinZn9qSsqxIk8n4Z_Z2o?purpose=fullsize|393](https://images.openai.com/static-rsc-4/jPJ5ozQyXUb1neGKMvzFqShV2ufV6xLPmDICYkZFNlKEcq4rxPaEveqE2Seo3Ynf9NUpGt4Cc-8-Yg1aGDNPtjww-xhEM0qck3-ZY5CDnK_qaqOA6Yep3H07a7Ho27n1Ps56EkgTOWYauMKcDOFIqkssedBJ0qeshSz3luwPb8Q?purpose=inline)

![https://images.openai.com/static-rsc-4/dWt8xLTDvCeh0KVRb9CcvDTer5MT7qJJVFs6cm0u9nkP1OT3kysNLE5sN46HfbqfPDYFonoC61g5PBZ8Wdb4TXliromAesQ0rsbAt6X-ViAO8RphrsmtJ2LC0YMWjqjy2WF2qE32BduFv5iYtVbPqmR33ZRgSngRGAeCKKOl7HAOBIL8XYS_tM-lGflEb06V?purpose=fullsize|397](https://images.openai.com/static-rsc-4/dQMuMJL5w8XKnKDwCCimU-uYSp6xaUEYqegj_5De5okw0FY2QsO-61oSkNO_Snkn0DvcSLyc_khvvWpFnluPr7qEFudFCYhdAtzilVTf0QWfd-7gMxWKcnvlpUQGGtwE-wktrX3WGa7Oo8WPXVWSQsny5WleQcc9_IygezyL4OI?purpose=inline)

![https://images.openai.com/static-rsc-4/fhaVXXzs8mLnoleN_MR4ueQmmlc_6da1ocuGK2NX69cGzJQNal9TDSs9tJGHIZJBgZbku9SYJoYZkajMoSQQGEA682KULwuxJDW-O1gyqu16yUOVAzgjk209mrVuLdrvxA9YaeTsOcjtKHTVLGeU3MG36OPkxhgXW9ESVQpoZ4c4Czo_lSOsrHinVbo1JDL6?purpose=fullsize|367](https://images.openai.com/static-rsc-4/IQTAGjlI5BNLfoIIiZDjnSuPkFjZBXW4FHja7VymulHvLCEXnUdfH_enPMT3VMoaeJT9Y71Eid1gxeAWTpA3FrpQYvUHxCc6M70aU7Le6qzPVijPPxATqKvBijJ1JhHmk2SL2ouc6JlbAVIiCJ4NEr792aj90JWvnu5E0dmMifk?purpose=inline)


### Phases:

1. **Fresh / FOB (Fresh-Out-of-Box)**
    - Peak performance
    - No GC overhead
2. **Degradation Phase**
    - Performance drops as GC starts
    - Write amplification increases
3. **Steady State**
    - Performance stabilizes
    - Reflects real-world sustained behavior


## 📊 Formal Definition (Industry Practice)

A device is considered in **steady state** when:

- Performance metrics (IOPS/latency) vary within a **small tolerance band**
- Over a defined observation window (e.g., last 5–10 minutes or test rounds)

Example criterion:
`Variation ≤ ±10% over last N intervals`

(Exact thresholds vary by test methodology)

## 🎯 Why It Matters

### 1) Realistic Benchmarking

- Prevents misleading **“marketing peak numbers”**
- Reflects **production workload conditions**

### 2) Write Workloads Are Most Affected

- Random writes (4K) show the biggest drop from peak → steady state

### 3) Critical for Enterprise SSDs

- Databases, virtualization, logging systems operate in steady state

---
## ⚙️ How to Reach Steady State

### Preconditioning Process:

1. Fill drive to **100% (or target utilization)**
2. Run sustained workload:
    - Typically **random writes (4K, QD high)**
3. Continue until:
    - Performance plateaus

Tools:

- **fio**
- **Iometer**


## 🔍 Example Behavior
|Phase|4K Random Write IOPS|
|---|---|
|Fresh|600K IOPS|
|After 10 min|300K IOPS|
|Steady State|250K IOPS|

## ⚖️ Steady State vs Burst
|Metric|Burst|Steady State|
|---|---|---|
|Duration|Short|Long|
|Performance|High|Lower but stable|
|Reflects|Cache / empty NAND|Real sustained load|
|Reliability|Misleading|Accurate|

## ⚠️ Common Mistakes

- Reporting only **FOB performance**
- Not preconditioning the drive
- Using short-duration tests
- Ignoring thermal throttling effects

---

## 🧩 Key Insight

> **Steady state is where SSD performance becomes honest—everything before it is optimistic.**

---

If you want, I can:

- Show a **full fio script to drive an NVMe SSD into steady state**
- Explain **write amplification and its quantitative impact**
- Break down **JEDEC steady-state test methodology (JESD219/JESD218)**












---
### Why Steady State Matters

- Real-World Performance: Data center SSDs operate 24/7, quickly reaching steady state, making this measurement crucial for enterprise applications.

- Consistent Behavior: It shows how an SSD performs under continuous heavy load, revealing its long-term reliability and sustained speed.

- NAND Management: It reveals the effectiveness of the drive's internal wear-leveling and garbage collection algorithms under stress. 

### How it's Tested

- Preconditioning: The drive is subjected to continuous, heavy writes (often random 4KB) to fill the NAND and force it out of its fresh state.

- Garbage Collection (GC): Background processes like garbage collection work to manage data, causing performance fluctuations.

- Reaching Stability: The drive eventually reaches a point where performance metrics (IOPS, throughput) don't change much over a measurement window, indicating steady state.

- Measurement: The final performance test is run in this stable condition, often after many hours, to get representative, sustained results. 

### Key Takeaway
While client SSDs benefit from idle time for TRIM/GC to maintain high "fresh" performance, enterprise SSDs are measured in steady state because they rarely idle, providing a truer picture of their long-term, sustained capability. 
