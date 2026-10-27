
Specification baseline: NVMe Base Specification revisions before 2.3, primarily NVMe 2.1 and applicable FDP technical proposals

## 1. Validation objectives and scope

Flexible Data Placement (FDP) is an NVMe feature that allows the host to influence the physical placement of written data by associating write operations with Placement Identifiers (PIDs).

The primary purpose of FDP is to reduce Write Amplification Factor (WAF) by allowing the host to separate data according to characteristics such as lifetime, update frequency, or workload.

An exhaustive FDP validation plan must verify more than the ability to enable FDP and issue writes with a PID. It must validate the entire feature lifecycle, including configuration discovery, namespace association, directive handling, PID management, Reclaim Unit Handle (RUH) management, garbage collection behavior, event reporting, error handling, persistence, and interaction with other NVMe features.

### Validation coverage matrix

|Validation category|Priority|Primary objective|
|---|---|---|
|Capability discovery|P0|Verify FDP support and advertised capabilities|
|FDP configuration|P0|Validate configuration descriptors and fields|
|Enable/disable|P0|Verify state transitions and reset requirements|
|Namespace integration|P0|Verify FDP association and namespace behavior|
|Data Placement Directive|P0|Validate directive enablement and DTYPE/DSPEC|
|PID validation|P0|Validate PID encoding, ranges, and mapping|
|RUH management|P0|Validate RUH selection, status, and updates|
|FDP log pages|P0|Validate all supported FDP logs|
|FDP events|P1|Validate event generation, filtering, and reporting|
|I/O operations|P0|Validate data placement across write operations|
|Data integrity|P0|Verify no corruption, loss, or cross-namespace leakage|
|Reset and power|P0|Validate controller reset, shutdown, and power loss|
|Error injection|P0|Validate invalid inputs and recovery behavior|
|Garbage collection|P1|Validate reclaim and RUH isolation|
|Performance/WAF|P1|Verify workload behavior and placement effectiveness|
|Endurance/stress|P1|Validate long-duration stability|
|Feature interaction|P1|Validate interoperability with other NVMe features|

P0 means release-blocking functionality or integrity; P1 means important extended coverage. A production qualification program should include both.

Important distinction: FDP is a placement mechanism, not a guarantee that each PID maps permanently to a dedicated physical NAND block. The controller retains responsibility for internal physical allocation and garbage collection. Validation must therefore distinguish between externally observable NVMe compliance and internal firmware behavior that requires diagnostic instrumentation.

## 2. Establish the exact specification baseline

For a pre-2.3 implementation, use the ratified NVMe 2.2 specification as the primary reference, with NVMe 2.1 and TP4146 as historical references.

Official specifications:

- [NVMe Base Specification 2.2](https://nvmexpress.org/wp-content/uploads/NVM-Express-Base-Specification-Revision-2.2-2025.03.11-Ratified.pdf?utm_source=chatgpt.com)
- [NVMe Base Specification 2.1](https://nvmexpress.org/wp-content/uploads/NVM-Express-Base-Specification-Revision-2.1-2024.08.05-Ratified.pdf?utm_source=chatgpt.com)
- [NVM Express specification archive and technical proposals](https://nvmexpress.org/specifications/?utm_source=chatgpt.com)

The specification defines mandatory FDP dependencies, including I/O Management Send/Receive, FDP features, FDP logs, and the Data Placement Directive.

Use the matching revision of the NVM Command Set specification for I/O-specific behavior, especially Write, Copy, Write Zeroes, and related command semantics.

For every test, record:
- Applicable specification section and figure
- DUT firmware version and hardware revision
- Advertised FDP configuration
- Initial FDP state
- NVMe commands and exact DWORD values
- Expected completion status and returned data
- Actual result and firmware diagnostic evidence
    
Avoid treating behavior introduced or changed in revision 2.3 as a mandatory requirement for a 2.2 device.

## 3. FDP discovery and capability validation

Objective: Ensure the host can correctly discover whether the controller supports FDP and identify all available configurations.

### 3.1 Identify Controller

|Test ID|Test procedure|Expected result|
|---|---|---|
|FDP-DISC-001|Issue Identify Controller|Command succeeds|
|FDP-DISC-002|Read FDP support indication in CTRATT|Correct support state advertised|
|FDP-DISC-003|Compare CTRATT with firmware configuration|No inconsistency|
|FDP-DISC-004|Verify I/O Management Send support|Supported when required by FDP|
|FDP-DISC-005|Verify I/O Management Receive support|Supported when required by FDP|
|FDP-DISC-006|Inspect supported log pages|Required FDP logs supported|
|FDP-DISC-007|Inspect supported features|FDP and FDP Events features supported|
|FDP-DISC-008|Inspect command effects|Advertised effects agree with behavior|
|FDP-DISC-009|Test FDP-disabled device|No unsupported capability advertised|
|FDP-DISC-010|Repeat after controller reset|Capability reporting remains consistent|

Also validate that capability discovery is consistent across controllers in the same subsystem where applicable.

### 3.2 FDP Configurations Log Page — LID 20h
This log page is central to FDP discovery. It describes available FDP configurations for an Endurance Group.

Validate at least the following fields:

|Field|Validation requirement|
|---|---|
|NUMFDPC|Number of configurations is correctly encoded|
|VERSION|Supported log page format version|
|SIZE|Total returned log size is consistent|
|DS|Each descriptor size is valid|
|FDPA|FDP configuration attributes are valid|
|NRG|Number of Reclaim Groups|
|NRUH|Number of Reclaim Unit Handles|
|MAXPIDS|Maximum supported Placement Identifier index|
|NNSS|Namespace-related capability limit|
|RUNS|Reclaim Unit nominal size|
|ERUTL|Estimated Reclaim Unit Time Limit|
|RUH descriptors|Count and types match configuration|

Field interpretations and zero-based encodings must follow the exact revision's figures. Do not assume all reported counts are raw counts.

### Detailed test cases

##### FDP-CONF-001: Retrieve complete configuration log

1. Identify the target Endurance Group.
2. Issue Get Log Page with LID 20h and the correct Endurance Group identifier.
3. Read the fixed header.
4. Determine the required complete log length.
5. Retrieve the complete log.
6. Parse every FDP configuration descriptor.
7. Verify descriptor boundaries and the reported total size.
    
Expected result: The log is structurally valid, complete, and internally consistent.

##### FDP-CONF-002: Multiple configurations

- Test firmware advertising one, two, and the maximum supported number of configurations.

Expected result: Every configuration is individually accessible and correctly described.

##### FDP-CONF-003: Invalid Endurance Group

- Request the log for a nonexistent Endurance Group.

Expected result: The controller rejects the request using the specification-defined status, without modifying any FDP state.

##### FDP-CONF-004: Partial buffer retrieval

- Request a buffer shorter than the complete log.

Expected result: The returned data and completion status follow Get Log Page transfer rules, with no memory overrun or controller failure.

##### FDP-CONF-005: Configuration consistency

- Verify relationships among NRG, NRUH, MAXPIDS, RGIF, RUNS, and the RUH descriptors.

Expected result: The advertised geometry can produce a valid set of placement identifiers, and the controller accepts only the supported range.

### Configuration boundary coverage

Run the configuration tests with:

- One Reclaim Group and multiple Reclaim Groups
- One RUH and multiple RUHs
- Minimum and maximum supported RUH counts
- Minimum and maximum supported RUNS
- One and multiple Endurance Groups
- Different RUH types
- Multiple namespace configurations
    
Some of these configurations require firmware-controlled test modes rather than host commands.

## 4. FDP enable and disable validation

Objective: Validate the complete FDP state machine and required Controller Level Reset behavior.

FDP is controlled at the Endurance Group level. The host selects an advertised configuration through the FDP feature, and the required activation procedure includes Controller Level Reset.

### 4.1 FDP feature — FID 1Dh

|Test ID|Procedure|Expected result|
|---|---|---|
|FDP-EN-001|Get Features before enabling|Current state correctly reported|
|FDP-EN-002|Set Features with valid configuration|Accepted according to feature rules|
|FDP-EN-003|Perform required Controller Level Reset|Requested configuration becomes active|
|FDP-EN-004|Get Features after reset|Active state matches request|
|FDP-EN-005|Disable FDP using valid procedure|FDP becomes disabled after required transition|
|FDP-EN-006|Select another valid configuration|New configuration activated correctly|
|FDP-EN-007|Specify invalid configuration index|Command rejected|
|FDP-EN-008|Specify invalid Endurance Group|Command rejected|
|FDP-EN-009|Set reserved bits|Appropriate field validation|
|FDP-EN-010|Repeat enable with current configuration|Behavior follows specification|
|FDP-EN-011|Repeat disable while disabled|No corruption or invalid state|
|FDP-EN-012|Attempt FDP operations while disabled|Appropriate feature-disabled behavior|
|FDP-EN-013|Verify state after warm reset|Matches persistence and activation rules|
|FDP-EN-014|Verify state after power cycle|Matches persistence and activation rules|

### 4.2 State transition diagram

Conceptual FDP activation path. The precise namespace prerequisites and feature transition rules must be checked against the target revision.

### 4.3 Reset during activation

Inject reset or power interruption at each stage:
1. Before Set Features
2. During Set Features processing
3. After successful Set Features completion but before the required reset
4. During Controller Level Reset
5. Immediately after the controller becomes ready
6. During namespace initialization after activation
    
Expected result: The controller must recover to a valid, discoverable configuration. It must not expose partially initialized placement metadata or lose previously acknowledged user data.

## 5. Namespace management and FDP integration

Objective: Validate that namespace creation, attachment, and placement-handle provisioning correctly interact with the Endurance Group's FDP configuration.

FDP configuration belongs to an Endurance Group, while a namespace uses its associated placement-handle list to access the relevant RUHs.

### Namespace test matrix

|Test ID|Scenario|Validation|
|---|---|---|
|FDP-NS-001|Create namespace with FDP enabled|Successful valid provisioning|
|FDP-NS-002|Create namespace with FDP disabled|Correct non-FDP behavior|
|FDP-NS-003|Create namespace with explicit Placement Handle List|Correct handle association|
|FDP-NS-004|Create namespace without explicit list|Correct default behavior|
|FDP-NS-005|Minimum valid placement handles|Accepted|
|FDP-NS-006|Maximum supported placement handles|Accepted|
|FDP-NS-007|Invalid placement handle index|Rejected|
|FDP-NS-008|Duplicate placement handles|Follow exact list validity rules|
|FDP-NS-009|Attach namespace to controller|FDP association remains correct|
|FDP-NS-010|Detach and reattach namespace|No stale placement mapping|
|FDP-NS-011|Delete and recreate namespace|Old mapping not incorrectly reused|
|FDP-NS-012|Multiple namespaces sharing Endurance Group|Correct resource sharing|
|FDP-NS-013|Multiple Endurance Groups|Placement resources isolated|
|FDP-NS-014|Reset after namespace provisioning|Associations restored correctly|
|FDP-NS-015|Namespace format|Correct namespace and FDP behavior|
|FDP-NS-016|Namespace capacity boundary|Limits enforced|

### Critical multi-namespace test

Configure:
- Namespace 1: Placement Handles 0, 1
- Namespace 2: Placement Handles 1, 2
- Both namespaces associated with the same Endurance Group
    
Issue writes from both namespaces using the shared handle 1.

Verify that the correct shared RUH is selected, namespace logical addressing remains isolated, and RUH status reflects activity from both namespaces as applicable.

A Placement Handle is a namespace-level selection mechanism; it should not be confused with a unique physical Reclaim Unit.

## 6. Data Placement Directive validation

Objective: Verify the directive control mechanism and the correct interpretation of DTYPE and DSPEC in supported write commands.

### Important terminology

- DTYPE: Directive Type field, identifying the directive.
- DSPEC: Directive Specific field, whose interpretation depends on DTYPE.
- PID: Placement Identifier used by FDP to identify the placement destination.
- PHNDL: Placement Handle, which participates in identifying the selected RUH.
    
For the FDP Data Placement Directive, DSPEC carries the PID when the directive is applicable and enabled. DSPEC is not universally a PID for all directive types.

### Directive validation matrix

|Test ID|Scenario|Expected result|
|---|---|---|
|FDP-DIR-001|Identify directive support|Correctly advertised|
|FDP-DIR-002|Enable Data Placement Directive|Success|
|FDP-DIR-003|Read directive status|Enabled state reported|
|FDP-DIR-004|Disable Data Placement Directive|Success|
|FDP-DIR-005|Use valid DTYPE and PID|Correct placement|
|FDP-DIR-006|Use unsupported DTYPE|Specification-defined rejection|
|FDP-DIR-007|Use reserved DTYPE|Specification-defined rejection|
|FDP-DIR-008|Use valid PID with directive disabled|Defined disabled-directive behavior|
|FDP-DIR-009|Use DSPEC with no applicable directive|Defined command behavior|
|FDP-DIR-010|Disable directive during active workload|Safe state transition|
|FDP-DIR-011|Reset after directive enablement|Correct persistence behavior|
|FDP-DIR-012|Directive on unsupported command|Command-specific validation|
|FDP-DIR-013|Concurrent directive operations|No inconsistent state|
|FDP-DIR-014|Invalid directive command fields|Correct completion status|

### Example test

FDP-DIR-005 — Write with valid Data Placement Directive

1. Enable FDP.
2. Enable the Data Placement Directive.
3. Obtain a valid PID from the namespace's placement mapping.
4. Prepare a Write command.
5. Set the command's directive type to Data Placement.
6. Set DSPEC to the valid PID.
7. Submit the Write command.
8. Read back the data.
9. Query RUH status.
10. Compare firmware placement records, if available.
    
Expected result: Write succeeds, readback matches, and the selected placement is consistent with the PID and RUH mapping.

A successful Write completion alone does not prove correct physical placement.

## 7. Placement Identifier validation

Objective: Verify every legal PID encoding, mapping, and invalid-input boundary.

A PID identifies a placement selection involving the Reclaim Group Identifier (RGID) and Placement Handle (PHNDL). The RGIF field determines the partitioning of the PID.

Conceptually:

PID=(RGID << (16-RGIF)) | PHNDL

This expression assumes the standard 16-bit PID representation and a valid RGIF partition.

###### Example PID decoding

Suppose a test configuration advertises:
- NRG = 2
- NRUH = 8
- RGIF = 1
    
The resulting conceptual PID format is:

16-bit Placement Identifier

![[16 bit placement id.png]]

Illustrative encoding with RGIF = 1. Actual legal PIDs are restricted by the advertised configuration and namespace mapping.

|PID|RGID|PHNDL|Valid in example|
|---|---|---|---|
|0x0000|0|0|Yes|
|0x0007|0|7|Yes, if mapped|
|0x8000|1|0|Yes|
|0x8007|1|7|Yes, if mapped|
|0x0008|0|8|No|
|0x8008|1|8|No|

### PID test cases

|Test ID|Test condition|
|---|---|
|FDP-PID-001|Lowest valid PID|
|FDP-PID-002|Highest valid PID|
|FDP-PID-003|Every advertised valid PID|
|FDP-PID-004|First invalid PHNDL|
|FDP-PID-005|Invalid RGID|
|FDP-PID-006|All bits set: 0xFFFF|
|FDP-PID-007|PID not mapped to namespace|
|FDP-PID-008|PID mapped to another namespace|
|FDP-PID-009|PID after namespace recreation|
|FDP-PID-010|PID after FDP reconfiguration|
|FDP-PID-011|PID during concurrent RUH updates|
|FDP-PID-012|Same PID from multiple submission queues|
|FDP-PID-013|Different PIDs in one submission queue|
|FDP-PID-014|Randomized valid and invalid PIDs|
|FDP-PID-015|Every RGID/PHNDL boundary|

For invalid PIDs, verify the completion status and the FDP Invalid Placement Identifier event behavior specified for the applicable case.

Important: Do not assume every numerical PID below a maximum is valid. Validity also depends on the RGID, namespace Placement Handle List, and selected configuration.

## 8. Reclaim Unit Handle (RUH) validation

This is one of the most important areas of FDP validation because it tests the relationship between host placement decisions and the controller's internal allocation behavior.

### 8.1 RUH Status — I/O Management Receive

The RUH Status Management Operation returns status descriptors describing the RUHs associated with the namespace's placement identifiers.

Important fields include:

|Field|Meaning|Validation|
|---|---|---|
|NRUHSD|Number of RUH status descriptors|Matches returned descriptors|
|PID|Placement Identifier|Valid for namespace|
|RUH type|RUH allocation behavior|Matches configuration|
|EARUTR|Estimated Active Reclaim Unit Time Remaining|Valid according to type and state|
|RUAMW|Reclaim Unit Available Media Writes|Valid remaining-write capacity|

The RUH status operation and the RUH Update operation are demonstrated by publicly available FDP implementations.

### RUH status test matrix

|Test ID|Scenario|Expected result|
|---|---|---|
|FDP-RUHS-001|Retrieve status immediately after FDP activation|Valid initial descriptors|
|FDP-RUHS-002|Retrieve all descriptors|Correct count and format|
|FDP-RUHS-003|Request fewer descriptors|Correct truncated response|
|FDP-RUHS-004|Request more descriptors than available|Defined response|
|FDP-RUHS-005|Query invalid namespace|Command rejected|
|FDP-RUHS-006|Query with FDP disabled|FDP Disabled where specified|
|FDP-RUHS-007|Query after Write|RUAMW reflects media consumption|
|FDP-RUHS-008|Query after RU transition|Correct new RU status|
|FDP-RUHS-009|Query after reset|Valid reconstructed state|
|FDP-RUHS-010|Query during concurrent writes|Consistent, valid snapshot|
|FDP-RUHS-011|Query shared RUH from two namespaces|Shared state consistent|
|FDP-RUHS-012|Validate reserved bytes|Conform to specification|

### 8.2 RUAMW validation

RUAMW is particularly useful for observing FDP placement behavior.

### RUAMW-specific boundary tests

##### FDP-RUAMW-001 — Initial capacity

Query RUH status before any FDP write. Compare RUAMW against the expected initial media-write capacity.

##### FDP-RUAMW-002 — Single write

Write one logical block to the selected PID. Verify successful completion and the expected RUAMW behavior.

##### FDP-RUAMW-003 — Multiple writes

Issue 1, 10, 100, and 1,000 writes, using supported transfer sizes. Verify RUAMW progression.

##### FDP-RUAMW-004 — Exact Reclaim Unit boundary

Write enough data to reach the end of a Reclaim Unit. Verify the controller's transition behavior and status reporting.

##### FDP-RUAMW-005 — Write spanning RU boundary

Issue a Write that requires more media-write capacity than remains in the current RU.

Expected result: The controller handles the transition correctly, preserves the data, and reports valid RUH state.

##### FDP-RUAMW-006 — Overwrite

Write the same LBA repeatedly with the same PID. Verify data correctness and media accounting.

##### FDP-RUAMW-007 — Different PID

Write to PID A, then PID B. Confirm each maps to the intended RUH and that status changes are consistent with the allocation model.

##### FDP-RUAMW-008 — Controller garbage collection

Force internal GC and inspect RUAMW, internal mappings, and statistics. Verify that host-visible accounting follows the specification even if internal physical allocation changes.

## 9. RUH Update management validation

In pre-2.3 FDP terminology, the relevant operation is Reclaim Unit Handle Update, performed using I/O Management Send.

Its purpose is to modify the Reclaim Unit associated with a specified RUH, according to the operation's defined semantics.

This is not equivalent to changing the namespace's Placement Handle List.

### RUH Update test cases

|Test ID|Scenario|Validation|
|---|---|---|
|FDP-RUHU-001|Update valid RUH|Command succeeds|
|FDP-RUHU-002|Update unused RUH|Defined behavior|
|FDP-RUHU-003|Update partially written RU|Valid transition and event|
|FDP-RUHU-004|Update RU at capacity boundary|Correct handling|
|FDP-RUHU-005|Update invalid PID|Rejected|
|FDP-RUHU-006|Update PID outside namespace mapping|Rejected|
|FDP-RUHU-007|Update multiple valid PIDs|All selected handles processed|
|FDP-RUHU-008|Duplicate PID entries|Defined validation|
|FDP-RUHU-009|Invalid number of entries|Rejected|
|FDP-RUHU-010|Invalid buffer length|Rejected|
|FDP-RUHU-011|RUH Update with FDP disabled|FDP Disabled where required|
|FDP-RUHU-012|RUH Update during active writes|Data remains consistent|
|FDP-RUHU-013|Concurrent RUH Update commands|Serialized or otherwise safely handled|
|FDP-RUHU-014|Reset during RUH Update|Recoverable valid mapping|
|FDP-RUHU-015|Power loss during RUH Update|No acknowledged data loss|
|FDP-RUHU-016|RUH Update across shared namespaces|Correct shared-handle behavior|

### Example: Partially written Reclaim Unit

Initial state:
- PID = 0x0002
- RUH = 2
- Current Reclaim Unit = RU-A
- RU-A is 50% written

![[partially written RU.png]]

Expected validation:
1. The RUH Update operation completes with the correct status.
2. The RUH status reflects a valid post-update state.
3. Subsequent writes using the PID follow the new RU association.
4. Previously written data remains readable.
5. The Reclaim Unit Not Fully Written event is generated when its defined conditions are met.
6. No stale mapping or NAND resource leak occurs.
    
## 10. FDP log page validation

Validate each log both independently and in combination with I/O and management operations.

### 10.1 LID 20h — FDP Configurations
Covered in Section 3. Also test repeated reads during resets and configuration transitions.

### 10.2 LID 21h — Reclaim Unit Handle Usage
This log describes RUH usage attributes. It does not report per-RUH remaining capacity; RUAMW is obtained through RUH Status.

|Test ID|Scenario|
|---|---|
|FDP-LOG21-001|Retrieve usage log|
|FDP-LOG21-002|Verify NRUH|
|FDP-LOG21-003|Verify descriptor count|
|FDP-LOG21-004|Validate RUHA attributes|
|FDP-LOG21-005|Compare attributes with namespace provisioning|
|FDP-LOG21-006|Read after namespace creation|
|FDP-LOG21-007|Read after namespace deletion|
|FDP-LOG21-008|Read after controller reset|
|FDP-LOG21-009|Read with invalid Endurance Group|
|FDP-LOG21-010|Read with partial buffer|

### 10.3 LID 22h — FDP Statistics

Validate the three principal 128-bit counters:
- HBMW: Host Bytes with Metadata Written
- MBMW: Media Bytes with Metadata Written
- MBE: Media Bytes Erased

|Test ID|Scenario|Expected validation|
|---|---|---|
|FDP-STAT-001|Initial statistics|Correct baseline|
|FDP-STAT-002|Sequential writes|HBMW changes correctly|
|FDP-STAT-003|Random writes|Counters consistent with workload|
|FDP-STAT-004|Repeated overwrites|Media accounting correct|
|FDP-STAT-005|Write with metadata|Metadata included as defined|
|FDP-STAT-006|Write Zeroes|Correct command-specific accounting|
|FDP-STAT-007|Copy command|Correct accounting if supported|
|FDP-STAT-008|Deallocate|Correct accounting semantics|
|FDP-STAT-009|Forced garbage collection|MBMW/MBE reflect media operations|
|FDP-STAT-010|RUH Update|Correct counter behavior|
|FDP-STAT-011|Controller reset|Correct persistence/reset semantics|
|FDP-STAT-012|Power cycle|Correct persistence/reset semantics|
|FDP-STAT-013|Near-counter overflow|Defined saturation behavior|
|FDP-STAT-014|Concurrent log reads and writes|Valid counter snapshots|

### WAF validation

An approximate media-write amplification metric can be computed as:

WAF(media) = (ΔMBMW)/(ΔHBMW)

Use the specification's accounting definitions, include metadata correctly, and account for operations such as deallocation or Write Zeroes that can affect host accounting differently from physical programming.

Do not equate this ratio automatically with raw NAND program-page WAF under every workload.

### 10.4 LID 23h — FDP Events

Test both the host-event and controller-event reporting paths.

Validate event count, event type, event-specific data, PID/namespace association when applicable, ordering, retention, and log-read semantics.

## 11. FDP Events feature validation

Feature Identifier: 1Eh

The FDP Events feature controls event reporting associated with the applicable namespace Placement Handle and RUH. The exact Set Features/Get Features data-buffer and CDW12 semantics must be validated.

The pre-2.3 specification includes the following FDP event types.

|Event type|Event name|Trigger to validate|
|---|---|---|
|00h|Reclaim Unit Not Fully Written|RU transition with unwritten capacity under defined conditions|
|01h|Reclaim Unit Time Limit Exceeded|RU time limit exceeded|
|02h|Controller Level Reset Modified RUHs|Reset changes RU associations|
|03h|Invalid Placement Identifier|Write uses invalid PID|
|80h|Media Reallocated|Applicable media relocation|
|81h|Implicitly Modified RUH|Controller modifies RU association without direct host request|

### Event validation matrix

|Test ID|Scenario|
|---|---|
|FDP-EVT-001|Get supported event types|
|FDP-EVT-002|Enable one supported event|
|FDP-EVT-003|Enable all supported events|
|FDP-EVT-004|Disable one event|
|FDP-EVT-005|Disable all events|
|FDP-EVT-006|Configure events for multiple handles|
|FDP-EVT-007|Invalid namespace|
|FDP-EVT-008|Invalid placement handle|
|FDP-EVT-009|Unsupported event type|
|FDP-EVT-010|Reserved attribute bits|
|FDP-EVT-011|Feature access while FDP disabled|
|FDP-EVT-012|Event generated while enabled|
|FDP-EVT-013|Event suppressed while disabled|
|FDP-EVT-014|Read host-event log|
|FDP-EVT-015|Read controller-event log|
|FDP-EVT-016|Event log at capacity|
|FDP-EVT-017|Multiple events in rapid succession|
|FDP-EVT-018|Event behavior across reset|
|FDP-EVT-019|Event log partial read|
|FDP-EVT-020|Clear/retain behavior according to log rules|

### Event generation examples

Event 00h — Reclaim Unit Not Fully Written

1. Write a partial RU
2. Issue an applicable RUH Update
3. Retrieve the FDP Events log
4. Confirm that the expected event appears with valid fields
    

Event 03h — Invalid Placement Identifier

1. Enable the applicable event
2. Issue a Write with an invalid PID
3. Verify command completion behavior
4. Retrieve the FDP Events log
5. Verify event type 03h and its applicable context fields
    

Event 80h — Media Reallocated

1. Configure an applicable Initially Isolated RUH
2. Write known data
3. Trigger qualifying controller media relocation through a controlled firmware test mode or workload
4. Retrieve the controller-event log
5. Verify the event and data integrity

Event 81h — Implicitly Modified RUH

Trigger a controller-driven RUH reassignment that meets the event definition, without a host RUH Update. Confirm the event and the resulting RUH status.

Not every event is reliably inducible through standard host commands. Time-limit, media-relocation, and implicit-modification cases often need firmware hooks.

## 12. FDP I/O command validation

The I/O matrix should cover the FDP directive's applicability to each command, the command's normal NVMe semantics, and interactions with placement.

|Test ID|I/O operation|Validation|
|---|---|---|
|FDP-IO-001|Write, single block|Valid PID and data integrity|
|FDP-IO-002|Write, maximum transfer|Correct placement and completion|
|FDP-IO-003|Write, unaligned logical range|Normal LBA validation|
|FDP-IO-004|Sequential Write|Correct placement|
|FDP-IO-005|Random Write|Correct placement|
|FDP-IO-006|Overwrite same LBA|Latest data returned|
|FDP-IO-007|Write across RU boundary|Correct RU transition|
|FDP-IO-008|Write using alternating PIDs|Correct RUH selection|
|FDP-IO-009|Concurrent Writes using same PID|Correct serialization and accounting|
|FDP-IO-010|Concurrent Writes using different PIDs|Correct isolation|
|FDP-IO-011|Read after placed Write|Exact data match|
|FDP-IO-012|Flush after placed Write|Persistence|
|FDP-IO-013|Write Zeroes|Applicable directive and accounting rules|
|FDP-IO-014|Dataset Management/Deallocate|Correct invalidation behavior|
|FDP-IO-015|Copy|Applicable placement semantics|
|FDP-IO-016|Write with metadata|Metadata and protection integrity|
|FDP-IO-017|Fused operations|Applicable command semantics|
|FDP-IO-018|Write with invalid PID|Defined failure/event behavior|
|FDP-IO-019|Write during background GC|Data integrity and valid RU mapping|
|FDP-IO-020|Write after RUH Update|New association honored|

For Write Zeroes, Copy, fused operations, and other commands, use the applicable NVM Command Set revision to determine whether and how the Data Placement Directive applies. Do not assume that every write-like operation accepts a PID in the same way.

### I/O size and alignment sweep

Test at least:
- 1 LBA
- 2 LBAs
- 4 KiB
- 16 KiB
- 64 KiB
- 128 KiB
- 1 MiB where supported
- MDTS boundary
- Reclaim Unit remaining-capacity boundary
    
Repeat for supported LBA formats, metadata configurations, and protection-information modes.

## 13. Data integrity validation

Data integrity is a release-blocking requirement and should be checked throughout all FDP tests, not only in a standalone suite.

### Recommended verification mechanism

For every test write, generate deterministic data from:

Data=f(NSID,LBA,WriteSequence,Seed)
Data is a function of NSID, LBA, Write seq, seed etc.

Maintain a host-side reference map keyed by namespace and LBA. Record the latest successfully completed write, its payload hash, and any applicable metadata.

After reset, RUH Update, garbage collection, or power recovery, read the affected ranges and compare against the expected map.

### Integrity test matrix

|Test ID|Scenario|
|---|---|
|FDP-DATA-001|Write/read with every valid PID|
|FDP-DATA-002|Repeated overwrite with same PID|
|FDP-DATA-003|Overwrite using different PIDs|
|FDP-DATA-004|Read after RUH Update|
|FDP-DATA-005|Read after RU rollover|
|FDP-DATA-006|Read after garbage collection|
|FDP-DATA-007|Read after controller reset|
|FDP-DATA-008|Read after power cycle|
|FDP-DATA-009|Read after sudden power loss|
|FDP-DATA-010|Multi-namespace data isolation|
|FDP-DATA-011|Metadata integrity|
|FDP-DATA-012|Protection information validation|
|FDP-DATA-013|Large-scale random workload verification|
|FDP-DATA-014|Read after FDP disable/reconfiguration|
|FDP-DATA-015|Read after namespace detach/reattach|

Pass criteria: No silent corruption, no acknowledged data loss, no incorrect namespace-to-LBA mapping, and no unexpected protection-information failures.

For sudden power-loss testing, distinguish acknowledged writes with applicable durability guarantees from outstanding writes whose completion and persistence are not guaranteed.

## 14. Reset, shutdown, and power-loss validation

Objective: Validate FDP state persistence, recovery, and data integrity across all supported reset mechanisms.

### Reset coverage

|Test ID|Reset type|Test focus|
|---|---|---|
|FDP-RST-001|NVMe Controller Reset (CC.EN transition)|FDP state and RUH reconstruction|
|FDP-RST-002|PCIe Function Level Reset|Controller and FDP recovery|
|FDP-RST-003|PCIe hot reset|Recovery and state|
|FDP-RST-004|Subsystem Reset, if supported|All affected controllers|
|FDP-RST-005|Normal shutdown|Clean persistence|
|FDP-RST-006|Abrupt power removal|Power-loss recovery|
|FDP-RST-007|Power cycle after RUH Update|RUH metadata recovery|
|FDP-RST-008|Reset during active FDP Write|Data and command recovery|
|FDP-RST-009|Reset during garbage collection|Mapping consistency|
|FDP-RST-010|Reset during event generation|Event-state consistency|

### Power-loss injection points

Inject power loss during:
1. FDP feature enable/disable transition
2. Namespace provisioning
3. Placement Handle List initialization
4. A Write using a valid PID
5. A Write crossing an RU boundary
6. RUH Update
7. Garbage collection of a partially written RU
8. Firmware metadata persistence
9. FDP event-log update
10. Controller shutdown processing
    

After each recovery, verify:
- Controller becomes ready within its supported recovery requirements
- FDP feature state is valid
- Namespace associations remain correct
- RUH status can be retrieved
- Placement mappings do not reference invalid resources
- All guaranteed-persistent user data is intact
- Subsequent FDP writes succeed
- No permanent loss of reclaim capacity occurs
    
For power-loss testing, use both a controlled shutdown and a hardware power-cut mechanism capable of interrupting the DUT at randomized points.

## 15. Negative testing and error injection

This category validates controller robustness against malformed commands, invalid state transitions, and internal firmware failures.

### Protocol-level negative testing

|Category|Negative scenarios|
|---|---|
|Get Log Page|Invalid LID, invalid Endurance Group, invalid length, reserved fields|
|Set Features 1Dh|Invalid configuration, invalid Endurance Group, reserved bits|
|Set Features 1Eh|Invalid event, namespace, handle, attribute, buffer|
|Directives|Unsupported DTYPE, invalid DSPEC, disabled directive|
|I/O Management Receive|Invalid operation, invalid namespace, invalid length|
|I/O Management Send|Invalid operation, invalid PID list, invalid length|
|Write|Invalid RGID, invalid PHNDL, unsupported directive|
|Namespace Management|Invalid placement list, invalid Endurance Group, unsupported configuration|
|Concurrent commands|Conflicting operations and reset races|

### Firmware fault injection

For exhaustive SSD firmware qualification, inject internal faults at:
- RU allocation
- RUH-to-RU mapping update
- FTL mapping-table update
- NAND program
- NAND read
- NAND erase
- Garbage collection relocation
- Metadata journaling
- Metadata checkpointing
- Event-log persistence
- Recovery metadata reconstruction
    

Validate both recoverable and unrecoverable media errors, including correct propagation of NVMe error status and preservation of unaffected user data.

Do not accept a generic failure status as sufficient. Each negative test should assert the exact SCT/SC required by the target specification, including whether the command is aborted, whether data is transferred, and whether any state change is permitted.

## 16. Garbage collection and physical placement validation

This category is essential for verifying that FDP actually delivers its intended placement behavior.

Standard NVMe commands generally cannot reveal the physical NAND allocation used by an RUH. Firmware instrumentation is therefore needed for exhaustive internal validation.

### Recommended firmware diagnostic fields

|Diagnostic field|Purpose|
|---|---|
|Endurance Group ID|Identify resource domain|
|RGID|Identify Reclaim Group|
|PHNDL|Identify selected Placement Handle|
|PID|Correlate host placement request|
|RUH internal ID|Identify RUH object|
|RU internal ID|Identify active physical allocation|
|Physical block/superblock ID|Verify physical placement|
|RU remaining capacity|Cross-check RUAMW|
|RU state|Active, full, reclaiming, free|
|Valid-page count|Verify garbage collection|
|Invalid-page count|Verify invalidation|
|NAND bytes programmed|Measure physical write amplification|
|NAND bytes erased|Validate reclamation|
|GC relocation bytes|Quantify internal copying|

### Garbage collection test matrix

|Test ID|Scenario|Validation|
|---|---|---|
|FDP-GC-001|Single PID sequential workload|Correct RU allocation|
|FDP-GC-002|Multiple PID sequential workloads|Placement separation|
|FDP-GC-003|Hot/cold data separation|Lower relocation pressure|
|FDP-GC-004|Mixed lifetime data|Correct GC behavior|
|FDP-GC-005|RU rollover|No leaked capacity|
|FDP-GC-006|Partially written RU|Correct reclaim handling|
|FDP-GC-007|RUH Update with valid data|No data loss|
|FDP-GC-008|GC during RUH Update|Mapping consistency|
|FDP-GC-009|GC under low free-space conditions|Forward progress|
|FDP-GC-010|Multiple RGs|Correct resource accounting|
|FDP-GC-011|Media error during GC|Recovery|
|FDP-GC-012|Reset during GC|Metadata consistency|

### Workload comparison

Compare FDP disabled, FDP enabled with one PID, and FDP enabled with multiple PIDs.
Illustrative write amplification comparison

Hypothetical example only; not a specification requirement or measured DUT result.

![Chart](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAvAAAAFUCAYAAABCwwgfAAAQAElEQVR4Aez9faxtSXndC49xGl6Q+RIxNrd9kdx8SIBxEi42Br/8gRMFISKR8CIsfCOCooROGwORjUB5ZYK5QQ4hESJY2OI6EIgQYFuRor58KlH8B0hwMY1QgL6AQXyHj2BsY7obA93uc8d4atbca82zT/dZ52vPWXvMXU+NGk/NtfasX9VeZ67aq3efOZsjBEIgBEIgBEIgBEIgBEJgMwTOIEcIhEAIXBSBPCgEQiAEQiAEQuAkCOQG/iSo53uGQAiEQAiEwGkmkLGHQAhcEoHLdgP/J3/yJ/je9753SReTB4dACIRACIRACIRACIRACNw1gUu+gf/CF76A3/iN38C/+Bf/Ar/2a7+G3/7t38bZs2fv+rteRO83vvEN9Nh9eM9ZD8zvnj4/d54nnL0wvA562PfoOWvPWe172PfoOWvPWe172PfoOWvPWe172PfoOWvPWe172PfoOWvPWe172PfoOWvPWe172PfoOWvPWe172PfoOWvPWe172PfoOWvPWe172PfoOWvPWe172PfoOWvPWe172PfoOWvPWe172PfoOWvPWe172PfoOWvPWe172PfoOWvPWe172PfoOWvPWe172PfoOWvPWe172PfoOWvPWe172PfoOWvPWe172PfoOWvPWe172PfoOWvPWe172PfoOWvPWe172PfoOWvPWe172PfoOWvPWe172PfoOWvPWe172PfoOWvPWe172PfoOWvPWe172PfoOWvPWe172PfoOWvPWe172PfoOWvPWe172PfoOWvPWe172PfoOWvPWe172PfoOWvPWe172PfoOWvPWe172PfoOWvPWe172PfoOWvPWe172PfoOWvPWe172PfoOWvPWe172PfoOWvPWe172PfoOWvPWe172PfoOWvPWe172PfoOWvPWe172PfoOWvPWe172PfoOWvPWe172PfoOWvPWe172PfoOWvP7eol3cD/4Ac/wPve9z7843/8j+vG/W//7b+NT37yk/jEJz6x+z3SDoEQCIHBCGQ4IRACIRACIXByBC7pBv4v/uIv8A//4T/Eddddh3ve85548pOfXCO5z33uU3rnnXfipptuqvayct79y/z5/LXXXoseu+f0nDX5xigcwsE/Dz2yHrIe+lqwZj1kPXgd9Mh6OIH1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STXse8jOpeesc1IN+x6yc+k565xUw76H7Fx6zjon1bDvITuXnrPOSTXse8jOpeesc1IN+x6yc+k565xUw76H7Fx6zjon1bDvITuXnrPOSTXse8jOpeesc1IN+x6yc+k565xUw76H7Fx6zjon1bDvITuXnrPOSTXse8jOpeesc1IN+x6yc+k565xUw76H7Fx6zjon1bDvITuXnrPOSTXse8jOpeesc1IN+x6yc+k565xUw76H7Fx6zjon1bDvITuXnrPOyZ3GmZ32RTXvc5/71Ednrr/+ejzvec+r53jnO99Z6uopT3kKnva0p8E53+y/613vKv/Upz7V3ZsK73r13bDTpKdh3JtaiBd4sfe77/10j+4bcq9WPcgTKWnbucp1D7XnvBqVV65UPv2GAHQepWvgA9x6yy3IEQIhEAIhcPoIXPIN/C6yxz/+8Xj4wx8O/xWb3fwznvEMPPShD8XnPvc5POxhD4P9bv+FtK+99lr02D2/56xXMu/n9qalb4es7R/z9o+4ffLU7q3uFwXKPDbHR9e9W7yeemw17+smtUa9VWujiSEJqi3RZHnVyqif5JTnlFdfP18pUpW8z/ZE2yoD0rXOdYf6SYJqS6bnkcnzG0LxIHnZ+PhJr/2Jay1z9DVrnZNq2PeQnUvPWeekGvY9ZOfSc9Y5qYZ9D9m5XHvttSf6uu1rmi9GDfsesnPpOeucVMO+h+xces46J9Ww7yE7l56zzkk17HvIzqXnrHNSDfsesnPpOeucVMO+h+xces46J9Ww7yE7l56zzkk17HvIzqXnrHNSDfsesnPpOeucVMO+h+xces46J9Ww7yE7l56zzkk17HvIzqXnrHNSDfsesnPpOeucVMO+h+xG5msJAAAQAElEQVRces46J9Ww7yE7l56zzkk17HvIzqXnrHNSDfsesnPpOeucVMO+h+xces46J9Ww7yE7l56zzkk17HvIzqXnrHNSDfsesnPpOeucVMO+h+xces46J9Ww7yE7l56zzkk17HvIzqXnrHNSDfsesnPpOeucVMO+h+w55bLewPvZH/nIR+Le9763m3N89KMfxRe/+EXc6173whe+8AXYz50baujf3vkfX+q64zEMD4x8eKFqfKRXbTVcgdz1WHgufPpJoo5SnjyfuphUIRACITAqgYzrrghc9hv4z3zmM3jEIx4xf89PfepTeOMb34hHP/rReO1rXwt/Lt6+f05+PnELDW00qmgnrUpV8Zo4QVDZNA+NYtyinXEPzp/jtnoH3ZO175WZzku/V3P7/P8RrxXyqYtLFQIhEAIhcBoJnPcG/tZbb8UPf/jDc5h861vfwstf/vL62+8f//jH8da3vhXf/OY36zzrV7/6VTz72c8u/+Uvfxmvf/3rcd111+GFL3wh/B+9vuhFLyrv/Je+9KU6bzOVdt5UoK03l6hgqAzBAec5tp4m204xGSUHY7D1xZnrD4EQCIEQuGgC59zA+3/M9B//43+sPw15++234z/8h/+w95n2b3/72/BNvG++v//97+ODH/wgXvGKV+BVr3oV3vGOd+BlL3sZHvjAB9YFWR/3uMfhxS9+cd28O+mbeHvn3e/cZkI7lCreikNUsyYIKkPw0GiGLN5lT5zVz+uAMeSKHWpQGUwIhEAIXDEC59zA3//+98c/+kf/CL/7u79b8U/+yT/Z+586/dRP/RRe/epX4+lPfzqe8IQnwH9Z5pWvfCVe+tKX1o26P3Dfr9bP5b9O48++95zV/oYbbsADHvAA2+1E7eDpconFDny80BSEptWsaisegx4ktVYT5IAMBl2zGVYIhEAIhMDdEzjnBv7uH4LaYSdZp97vfvfDgx/8YNzznvcsP3Z1VrvN0wjVLFOqXKkqFbnqqiq+cKA4qFKpRKkqlVX4uojxKu++a/tZpYEur2HuqX6NsufTv89rxXw0VSkhEAIhEAKnkMBF3cCfQk41ZIJwqUpvYGhD6FBVparJU72EKsBVNasCylM1q1lVNasC1IM8P6gvl8bDNV0BTl5uPhjzIM2MIFkDJI9R5chz834AqbyClCpBHqPKkcfk+/nqI9MvHMfPg9iQF8fHz5kIgRAIgRA4fQRyA3/AnJ/1NnJtZKryrtye9367835Cq8+2dp9+EzEyk9AWp5or4uNpGjDazrrIa716ePFac155g/DwnJ6W+PDNX4Pjdb9/ExLrZvDhm792WpZlxhkCJ0YgN/AHoCeIVqLDccCYB6m1qskirQBp5aTxJAEQJOGDtHIz3tc8evzSy27EdX//d2B1vO73PoLEuhl4njxnfqM1+vrM+ELgpAiMewN/BYhqH7OeNepdTO+jj6U1ucNVbY48W21o3Td3bj79nUzTzmOprffk+fXrGE+92+6bQOt4ozsdI/IbLc9hbuRPx3xnlFeXQG7gD+CtfTnshXbr9vx+L8i77E3/ivgcsAw2diqn6yVYLU6KSYl2cPZsidk3y9mzJWbfLGfPlph9s5w9W2L2zXL2bInZN8vZsyVm3yxnz5aYfbOcPVti9s1y9gTgXNNWQ72tRbVQB3dqKNt9U2eg48i1FnWm0ipUYMd137T3HLnW4vwIDHz4hs+7uNQYSUJFIY1vHIimQNOVe9/I540YcoTAZSWQG/gDcHrnfS/O7jl/snY/0r/PY+lWxOeAZbCxU492jlvLa9ZDOFI77ySn3ySOuHQezq6RT7uuMWvf8HlkngP/dxt6qUCpkqXqKI1vXFbOQ9NUH4GynoLIEEPgqhDIDfwBmIl8jUrggGWwsVM9Y77kKI2hfobdoFrbVl/9iOHd93nHnQBJRZTcPgf/VgU5QiAELguB3MAfiLHtz7nWloce61bbnVuRn3a6dXnVyvV5lu56fsxq1Ggj9ypoI4wfg0MbxXh17b7XIlWlUis3qokWBJUt88jHaDSNKSFwmQjkBv4QkASoL3gnBLRRbeWkTlEV40WgFW6CBwY+tFw9FYgCWo0YhQOu0HGSTzvf4GmSCAJSVSpqQ7HjufA6SYXAlOek3S81/RQSxXk48Tx5PUjFvSxVpaL24nwuvE6Cj3mObRIhEAIXTSA38IegOzvt5M7qvRBticw+/SaAmcd2+ByyDLZ2rqZDv4nRXHip6uLjxWIADhrCcOXDN38ddY+uFdtfS6bZ0li1gJXvPv3i4R/mHSadjXWdfDSNKSMTyNiuIoHcwB8CW/+y9F0FkkAVV1ST8jzS9IuHC1XxiItaEBuSyrtYCX9BuVnVBq9ePwY+JpTYU7N1aNx7+e7d5+h+qe5zLPPdu8/R/VLd51jmu3efo/ulus+xzHfvPkf3S3WfY5nv3n2O7pfqPscy3737HN0v1X2OZb579zm6X6r6lBq0eHCEv6DFWQrX1HhZLVRNCaulxvGaxx/PpbIUNlZLjeP1MvMjCb9JQ44QCIFLJpAb+EMQasPDOxttw0NGZfZO7nrvjOz69BeqqszlSvHpnA9+/kMWwsbOFW+VGX0hckKhsp/30JQ0vjqv+6XqnPQXuqqMY4+HEwqVK9fvORk2ipxGV/vIx6qSKulvpM7lIDgq5+b7+epUufr9+qYpIRACl4FAbuAPgKjNA8AVAaubVsRvnwcGPRZrs6/ZqOZbEFQAVSrbU3hyMeRBamxViGqqUlH7yINQIZb57sHD+0lCRUE/GPQX1VRSBSQVzYNSVUqBpEJ+R0F5VUqBpEJ+R0F5VUqBpEJ+R0F5VUqBpEJ+R0F5VUqBpEJ+R0F5VUqBpEJ+R0F5VUqBpEJ+R0F5VUqBpEJ+R0F5VUqBpEJ+R0F5VUqBpAI6qEgJgRC4HARyA38AxbYrqf0Llfqc96TeeYvXTs6WeRywDjZ16jQnfe1GNXuCoKKd8aM1u0lfLzwaz4DFf+Pdw9MMaZ5UPEFay3O++3aSXn7decx56RcUlc5rqVeZj65kwNWaIYVAEbjqVW7gD0CuTQTUNoI3EWRU4g1hBB4Y9Ki5QS1bV56uczX9x3OhUQmOCgGb48+rrqquaj98URjyIDU2lyg8zaRhqLlx1QiQIwRC4PIQyA38ARy1x3O0gSCjIl+1tMpU6UmVVpGvWlplqtI/bfyIx0r4aEo2X44bwITXvAW7SlWVV6VSDytVpZJ+QVA54iBCe15GZRX9urQhS/HVyKKCoDIKBw0lJQRC4PIQyA38ARy1B4JpA2RWN47L9/PST1CMO4+lQok19OsSxyyCSxAozlY1YaVq2kzKSZ2iKsaLQCtcJQ9dFEY8RBtarqC+poJ9rx4C1NdUkP5dHiLDXa/2nueCV+tXVg0XtdgUpWqXVqUUsc/7gH7kCIEQuFwEcgN/AElvgvgzmE1r/01VfUoT00cLpfteidr8TL9QifVa+enSxixarLUitQCbeh6U1Krc8+kvIqpUtsHHMzniom30PTq1WtGcTF4D9rpVWjnVraidfnMRHrForapXyMfXmAiBELh0Aiu6gb/0wVzpZ6C+AUmwFJNy0ngCIAkCkKApJ1UOALlOj1EPAtQXijttQH25UcrWQvpBfbVCQA0qWqEs1aTURVpOWsUVK6NKhUDV0iquWBlVKgSqllZxxcqoUiFQtbSKK1ZGlQoB1S0w3HE0OkLLEqoVULi1UC48sH9e+vd5APv+BPggRwiEwGUhkBv4AzFqQ6MeES0M2u0ZQ9soxqxrJ04z1bTvVNq1VexWy27Yn8Lxec7GXLF9VG1lHo0zvv+ENkJb5dGuPnUIhMClEcgN/EH8qN2L9gBatD1UqnZTpl8sXFgV93nI0XlF6Yr46ZKGLDTz4qxW1DRAigUUG1fo+gddtKixaXwk57ZzZLw59CC3xgOrPHJRIbBFArmBP2jW+o7HtCdUnxtuT9B3RpqmvzhsiE+bxfFqr9j23x2oVfMRHYWH//ua8VasRuQXD63VGl9U/5gIyDAcNL+nqHz45q/B8brfvwmJdTPwPG1taeYG/sAZ035HPeLyKpDnQx0nxaG++YAVtbLm0G5dtZfaz1nmu09/I9B5LLX1giy65yrOk+/nX2Q/9PgBlyxAABobqYaCjI7DA6fi8M36dX//d/BLL7ux4nW/9xEk1s3Ac9XnbCs387mBPxUvJxnkaSWg/fYaeqk38lCV6h3VGenf4VHNqhqnFfPRpQEjVtpx1gzU7nNUEzwMD41l4OIbP98I+mZ94GEOPbR5DvVbk7UPNDfwh8yQd4J8ftQU4F0h+BiFh8cyWJDevYSmKgojGIwHBj1IerpARqk5JgkOoDUIjWPU4pv3P/p/vg5Nl4IKKNatZK5PCM6Zp9/SDbx/k4IVH7mBP2RytAtSp0cLQ31G1a1ReHgso4XnxluYUe3manJH46AhjVjqN0IaWNQ/vF66Y6imdNjSb/bqv7HxdOm1RkX/TGoVx2sRNw5b4uPfpHhHfq2LNjfwB8wMyXa2VU1SlTOTkjteTVJV+qG3tvBB7vBQk1QF9UxK7ng1SVXqvlqP97caLszQGKPAiBww5sGaLKgmfFCtKESB8EG1tqi+5hHDN+/esa2XWVUkAYWKhBWqXBTxJDfD53W/dxPWeuQG/oCZ8TvHOr3eVvsNpd9WK2NfsuPV3Ds//bUTIQwGp1CZuKljSguaW86rebX5+VsPF2bpQYmnpbNGfOHYPI82itNbZ+QhsAICH/7k13QVelFV0b9sre0X2fgjFhvl4R14hwayupIb+AOmhOzvGrl4Fx1PNgYCo9LaJCGjwjnUUFmfx6iH5oAemyoVCD6aqgkd6d8sD83ekGVen1qbnhxLU/W4KOIBUiBcFFCQrpwHVusx5uEbPDboqqlBsqnmxC2Uo4TVUqNp+huHqiksrJYaTa8wH1zI8+ucD9/8dazxyA38AbNSO8La0bT28A5eb1vj/Tm3o9gKjwOWwbZOrfWqS7ZKaj6W6l2i9Ht/SJtnWrsb4ePLHDG8HD0Z7fW0pkSVitZoy2vUOin9hmAuG+KhSx2t+ObdY2qvHK41L1qorjU7U2s37x77/qjul5r+NfDzNbTfsHg+1hW5gT9gPki9OyT0po1zgPtep4DkHGD6SYLssc8D3Pc6DSTnAK9OP07kuArftFjq+1glOE7F+Nh8Pz/9WCMfDHpMy03IqcBdRPr147wtPhjzoIZFEvU1Kxa+ekF2TX+RmHmUWyUfrPTIDfxBE6N3zXqTrFobmWrovXUrynh3SMYil36DEI9WRGTyFrnV8TloGWzsZK9UX3LUFLyfMoa2UYxZ+3XCM9VUrWnx7nu9khyb7+cP3q8X130efdxdVzj+MZerZkID01oUccGvUlV8Qdk8H83uKktu4A+aFr1D3Hm3CMSTYjBx2DIPDHpQ40pAK3S8wLAHQVKjo1QCK9UgKg0r44uDMJRuhIcvd8CodakpIF0BFrjphXCIigAAEABJREFURikQLwQb5YGVHpdyA7/SIV3Jy9L76dryiNbnT/W+ehS9kqvmJJ9b+x+apdoMqsuIbyzMwUCsPbbofc3jRZ8Rq0dn7RF/tILNZIs8fM2Dxe5UeGi73m3HMt+9+xzdL9V9jmW+e/c5ul+q+xzLfPfuc3S/VPc5lvnu3efofqnucyzz3bvP0f1S3edY5rt3n6P7pbrPscx37z5H90vtfc6vLHIDf9CEECT1CEYH46BJHbJQo+qhJnrberLe3x25HmBmAB2elx6yc59zx3nnxguPlho7NbSjoDJQLNU5xzLfvfsc3S/VfY5lvnv3Obpfqvscy3z37nN0v1T3OZb57t3n6H6p7nMs8927z9H9Ut3nWOa7d5+j+6W6z7HMd+8+x5HHeAehlQm4UnNfnVCo7OehQ8npn1I/FK6U2lcnFCr7eehQMo+fOEiuBD8/p596jZEb+INnpb8dizZ0o3Boo0kdAiFw0gT8mqLfctZltLZ3nd1q6lb6TcE8enTfyLS691nX0l/TeqnVCh/f+Jr0/sUt8933s7pfavobgSWX7lvvEe+e73q5+vvzrFFzA3/IrOjdLua3YzIq8QUBMJdqVgVszmPco7ZoNLyoIKiMwkFDGbJofkhCRWHtgXhBIbfLQxOIMY9pTjQ4Um0QKLUQ9bXwWHjSZx2dn/59HuS+v1p8NCNY65Eb+ENmpt7auTqrR0lV2vs/NxQq8YYwhWX+BLaMynr5aEpHLfXfbWhwUUFQGYWDhjJk0fzUf1uj1wtrD7+U9LY1Xq+mE6ut8ICud8g12xaja0+KtBavVKPVmOv3IaXxXgOb4tFmURO3vpIb+IPnhHpEDzXR29Z4bJYHxj20c1GDixYGnDiH6bXiUq+jjWa8WlxIMXKRktSUEXCZ2qSNUlKSICmjIiUZLwbg+njA14URj4IN1YDGSKqlUB1vDoriAh1qkyKjUI3Kq10KHWqT6lGoRuXVLoUOtUn1KFSj8mqXQofapHoUqlF5tUuhQ21SPQrVqLzapdChNqkehWonFOssuYE/eF70zroeEy0M87vTrfNooxmy1s5PjStaGLwDVI2t86hBDFrNLydTY5JzX26mjknSP62HmcfUmGQ1fKbLHEc6YI3IrysONau47Sijym2HmlXcdpRR5bZDzSpuO8oAcNsR3wiYhaO5+g3I/BrvnPscbjvcdrjtcNvhtsNth9sVO3Nbfj1VbuAPmAu9IdPZ9Bu1KEbjoCkdtJAEFOSRklSKyquoLaNiz1KyKZh+siCAPFKSkz9BPhjzoIelSgWCjKZqQoeMCpxsSjd3fDWrInR4niQ+qXm4WVXzXPjqqorQkcdfdj6iOmBhcSKlPTRKMp6cGGyVR82sLn6FJTfwB0xKe1N2dnpzd2p10PEfsBA2dmr7fKzXq3YStIjte3hHzm1P6q667Ui/N3SOuHVOZuM4UT4bW4cXermiXadGC4OXWDVG4VGDGa7S66vG5NcEz5NVVi8X5+bTP72mCpA5dR5LVfdq+Pla1hi5gT9gVvRGUmf73SRANlVLbdXxxUEkSslt8cGgBzUukpqTHthpOxdPmkOPDfHAFo+7v2bqlAQwKgMMebDNl19LND6SqgGSIACSIACSIADSdVO3SIIASIIASNdN3SIJAiAJAiBdN3WLJAiAJAiAdN3ULZIgAJIgANJ1U7dIggBIggBI103dIgkCIAkCIF03dYskCIAkCIB03dQtkiAAkiAA0nVTt0iCAEiCAEjXTd0iCQIgCQIgXTd1iyQIgCQIgHTd1C2SIACSIADSdVO3SIIASIIASNdNWwurPHIDf8C0aPOyzrY6+taI2z18wtz2W0olurfK6l2l34GqlX5BaCzMxuGE1XE1+fr7jhheYt4D8k6HSNfaazr1SNIvAtOCK9HC2+WlXnETqMp7lShTJ1qL5gn1+1rGDNP2yKKm4DU2lrbRpA6BAwnk9D0CuYHfw3HXxm/KdgN6a7brl+30A0smux4r4odBDyEWZoKkRshjVdlj84B69DjVIAmAIAmAIAmAIBVQSLtfqnpBEgBBEgBBEgBBKqCQdr9U9YIkAIIkAIIkAIJUQCHtfqnqBUkABEkABEkABKmAQtr9UtULkgAIkgAIkgAIUgGFtPulqhckARAkARAkARCkAgpp90tVL0gCIEgCIEhi5KOGp0oFGqxLVDBUts0B4x41NxpeqX48S+Onn100BZpujQ/WeeQG/pB50XaQN956aENOO2/aHZny8UcszGhLPC5wGWzuNC1NXbN3it2KHu2sj8BDUztqqenxetUA/WISXy+uRtH+0WlcNud12aOWmgsNrlTrtTS+lmu/FxCObXpf+AojN/CHTMr0rtHvrB2Ihzn0wIZ5HLIMtnSupkSXS5CMgiA5EAcNZdRS00TNlwboOWNTNyu5Sc+69Kp4SsejYY9aSLapJSblpPFCIxbb5YGVHrmBP3Bi+rtqv6P0Q+NNQaEdB9Xt3XU1XNWm0dSYZDpvdfza5Q1cL8D3CYhOc75VPtPljyY1HapU9CpSpap4zbQgqGySh65+3DL9xkgDbPcF8Y2DVuq0XrfqNaUnX465gtzAHwPl/Kn2DhJgFVd+Z2l1yhoPHRvko6seuxzNSRtnfP+Z3SyPduFD1iQ1PYroUByGXKzzoNp6hWdM67YpdByX55SX7J3f813TjxPng9UeuYE/aGr0NlJF7yerVBUvgoKgsmkeGsXYRROkUnNUA5VROcBPp9aD9AxSlSkZ799kXG0eoj5q0d6liKYejcCo67WNyy8Ay3DPMrfr09/+DdllstteAx9fwzojN/AHzYveFatA7whdjhTY9zpJpZKlQPqhQzBUYBh7CjhV1V5eRgU+SlWpwCfvKeBUVXt5GRX4KFWlAp+8pxj7mMZam0Ia+1KV0viJZb57UN2qul+qunQCscx3D6pbVfdLVZdOIJb57kF1q+p+qerSCcQy3z2oblXdL1VdOoFY5rsH1a2q+6WqSycQy3z3oLpVdb9UdekEYpnvHlS3qu67Ojtq0ONNDESgzeio69XjIikhSO4rFj79G+Ojy11pyQ38QRPjd4Z+wFKdcyzz3bvP0f1S3edY5rt3n6P7pbrPscx37z5H90t1n2OZ7959ju6X6j7HMt+9+xzdL9V9jmW+e/c5ul+q+xzLfPfuc3S/VPcNHNNw9z9/qH2PyqtS8ejTbwqdS1fBUXHP2vj4msaPCf480PgZRTW2xqMuesiqvT7odyY1JapU9Cqi3yR5uEf5fl7rUN7dMj3fVSn1pL8wCkbn0lWpq8hH32qF5dTcwF8O9qTfSUPvp60ESfggKCFISgHqq+p4+NgGD1/pgEHAy/D8wfTzrhhxtXww6OG1CgJN3QBgUUBJlT2PyitVSqQfAFUUMIzSI+8UlGvqBrDrW14pdcGm9Mg7BeWaugHs+pZXSl2wKe0eQx41RFUk25ClIEBScbyCx+f1EJDuTD9JsTieA3h8Xg8B6c7L14+VHrmBP2Bi2t+Q7u+p9c64vRXUG0G/R4zfNp8DFsKWTtXS9DL13ET1sysIKjiQxyrP39IyPORaPT96URVzPcpGa3jXO7Xr3Xa0vF+H/TjPtVUPVkn/EY/G6cibjaPlrzQ/zcmAxUusMVTLxTCj+hk2BK+1betal2xu4A+YGYJ1Nrv6XZ4ys+/5rukXHYgG4YNq7enK+PjahgsCxkwySrPgMBww6KGlClBFARkVGZX4grBtHhjy8NKEKtIVQO5qPLltHljpkRv4AyZGexN19qx+l63M7P0WfNenXzT07nvJpfuV8amLHa3SxodxT6i1I6IBKhcvDirFYcM8NIThiqbDLxptXDIq8QVBSKQqm+ahUaSEQAhcBgK5gT8AIknUl7WHM24v1TnHMt+9+xzdL9V9jmW+e/c5ul+q+xwgyLpqkDvqtgPKHafOOU5J/wHLYFunUper0FSqAZTOnguf/m3xwZCHVmVbl1CLcI2aFxBW1aXVw6rjxaFISAl9ETqINfLShQ1ZSPHWyNgjHjMLM9koD136aktu4A+YmvrcrLYzS711p/Due/me76o+b3emXwTEQrVwVN1UOTVMS7KTd0Z9ZqoOO8mV7z9gGWzr1Nqu04bdsSqux+b7+enXUqz5Pl5Pmk9d2pWuTuD5xbW+a7T9eI7GoSZ3uMr/Znm+5tCLxtzWaNPf1vHMZCN8NHWrLbmBP2BqSOodpcKqgEJOotptB47aWHhSfVBYFVDISVS77cBRGwtPqg8KqwIKOYlqtx04amPhSfVBYVVAISdR7bYDR20sPKk+KKwKKOQkqt124KiNhSfVB4VVAYWcRLXbDhy1sfCk+qCwKqCQk6h224GjNhaeVB8UVgUUchIesAo2duo0NA21LjxaGDTnY2gbxWg1QQ9Ji7VU7lhNv8gUqON1tXx8zeMFyZoHMkpNL0kQALlt1QCw1iM38IfMjN461ntIvXPUtrC2KV2c7BG/WT6HrIOcGwIhcAUJ6PXUz+7XWat+D1ey1PQ3LEsu3a+Wz3TZw8l51u1yHuKnmd8ar+myVyS5gT9kMgi9GavKDYWK3l0q6YZCJd4QprCcLC/yAr8/xj0I1uAYHYpDDWbISitVP7dklByLAabXIIx2EBqZK4KkDOEvqE0yXgwINg5qQ0FwOx7rPK7IDfxXvvIV3HHHHesc8aVcld45+3Nsbfdd7x7jhaJxUEO/ftD+u5j0tnUzvC5lXaz8sZqVusKo1qpIjMJBQxm0aIb0OuLXjhMOvYTlWi7nHOgfiTHXrF5atFI0PNc2URPQD5B+HzQAj5Wu2jNvfvObxViAj7nA2267DTfccMOx8fnPf35+xNve9ra9c97whjfgHve4x9w/TMPvGhXYCbK9i+w5Mr6zsJIb4THMIj13IAQryehQHGowQ1ZaqXrdIKPkWAwwvQZhtIPQyAhSAUXUFECKBRQbV1zUceUfdOZjH/sY/uAP/uDY7/SHf/iH+Bt/42/gec97Hn7lV36l4hd+4Rdw5swZPPShD63HfPe738UnPvEJPPe5z53jBS94QfUNWWlnqMYVLQx69zeGtlEMWddOiEYWbRsVo3DQlI5bptdX7/7WIOMnDG0N99fdrfHRFnWNY9RqlNeWjKP9nHUOa12vZ5785CfjQx/60LHXd91119VN++Mf/3j8zb/5NytuvfVWPPaxj62beD/ove99L+573/vimmuuqfyTnvQkPOQhD3EX7rzzTtx0003VXlbOu3+ZX7OnL67eSaoR1fvqgThoKKMVj4eaJbJqkNEiMAgHz++woTny2EhaoMULH2S8OWyVBzDNHwY8NDR6fITqqqIi0MqGeWC9xxnfbP/gBz/An/3Zn51zld59JwV+6vEN98c//nH83M/9XGX8Off3v//9+NrXvoa3vOUteMlLXoIPfOAD1efqs5/9LN70pjfhxhtvtJ3D3vlPf/rTc24LjXpPpp2gqGZrNA4a0ojFOwjepYuKQK3ZcXTE9eiFn/wAABAASURBVHo0pnqVlY0KgspV56Dv6XKlvq+fe7AoVKpU6jcNUU2wIKhsmodGsdZy5kd+5Efq2j73uc+V3lX1x3/8x7j99tvx0z/903WaP+f+mte8Bs9//vPxsz/7s7Xj/va3vx0333xz9T/qUY/Cs571LLzvfe/De97znsq9+93vLv/MZz4Tj3nMYyp3IdU3vvEN9Ng9v+esVzLv5663MnpDY62borN36jeZZ/0GE7iUvH5T4efz817S85zvevL8NU9n74IPFofXU4/drp6zrj1/y6231OsmSdSX9KxvYqcgKwuSE592c0smb07k+jnccsutu8twfo3cwvr0NTp2B3DLLVqz/s/ezkLrEjo8B1rG8qq1TgHCB6HpKe9KS7o8oDx0Zs7HGvloauA5Xs67fQ+f06PnrD1nte9h36PnrD1nte9h36PnrD1nte9h36PnrD13pHr9lDF3w7fWMtR6VtopuHIedeT8TfCpucL82jrZEq+DHpWYqp6zTqkS+x6VmKqes06pEvselVhUZ/zRF+e+853vWO4y/uiP/gg/9VM/hXve857zefe5z33qozPXX399fVbeHe985zstFU95ylPwtKc9Dc75Zv9d73pX+ac+9anVf95qhR1tsekfB11btVWpTD+el5Cfns/PpWY9X7VVqZS/pLwefFmeR0+icu71DPD8GsJw5X73vZ/mSjOmu5v6p0K6N0h36QzfAO3nfXZ17qV9nnusex163uPz/pnw8yh2HzCfv5tUe86f7/xl/nQ/P0ncWje8Yjdg0XKoUfVZjy8c/omtxlZ51MWPVs2L1K9JGpy8ShlrD0/e3NZp3S91PkcNFXf77Pn5nHC+q9uOXV8PUFLlnMc558j5haaqJY9dX5thBXRd1Rl/fMaXdO2111rOG96R8n/w2j8+c9yJ/qz8wx/+cHz961/f637GM55R/9Grd/kf9rCHwX7vhAswvr4eu6f3nPVK5v3c+vcSVMNxzZkzOHPNGVxz5sycc/81Z87gjOKaM8rrRJXqv+aM8tecyfnmoLjmzLr4aFr3itdTj92OnrOuPe/rOyPO0MIktRIV9j1QqapqzR7lCZKASs9ZoRyppMK+x/nzegqd6/5+rtWe9PPgPN+XF5iHnoquzjmfdB7nyfMC81j180PHtT+x/7rtddlD3XPpOeucVMO+h+xces46J9Ww7yE7l56zzkk17HvIzqXnrHNSjfvd737F/MwZltY0EpovKs4ouJN37ihI96H6/XivNavSyh2X7+ejnvfofC7OZ/WTy3y/HizO7/l+PurxeX6KE+E5Xs67fQ/sHD1n3UnDvsca8iBATvMOQksFquH1d4bKK2EvaXk16LzXOdRDPV6a8ycO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)

For a meaningful comparison, hold constant the total host-written bytes, data lifetime distribution, namespace capacity, over-provisioning, initial drive state, and steady-state criteria.

WAF reduction is an optimization objective, not a universal numerical conformance requirement. A DUT should not fail NVMe compliance solely because one workload shows no WAF improvement.

## 17. Performance and endurance validation

### Performance workloads

Use a factorial workload matrix:

|Dimension|Values|
|---|---|
|Transfer size|4 KiB, 16 KiB, 128 KiB, 1 MiB|
|Read/write mix|100% write, 70/30, 50/50|
|Access pattern|Sequential, random, mixed|
|Queue depth|1, 4, 16, 32, 64, supported maximum|
|PID count|1, 2, 4, 8, maximum supported|
|Namespace count|1, 2, 4, maximum practical|
|Data lifetime|Uniform, hot/cold, multi-temperature|
|Drive state|Fresh, preconditioned, steady state|
|GC state|Idle, background, foreground pressure|

Capture IOPS, throughput, average latency, p99/p99.9 latency, WAF, RU transitions, GC relocation bytes, and media errors.

### Long-duration stress

Recommended engineering campaign:
- 24-hour basic FDP I/O stress
- 72-hour mixed-PID stress
- 168-hour continuous workload
- Repeated RUH Update cycles
- Repeated controller reset cycles
- Repeated controlled power-loss cycles
- Capacity pressure and full-drive cycling
- Thermal corner testing
- Endurance aging at representative NAND wear levels
    
These durations are suggested qualification targets, not NVMe-mandated test durations.

## 18. Feature interoperability

Test FDP together with other supported NVMe features to expose shared state and firmware-resource conflicts.

|Feature|Validation focus|
|---|---|
|Multiple namespaces|Shared RUHs and namespace isolation|
|Endurance Groups|Correct group scoping|
|NVM Sets|Resource and placement interactions|
|Protection Information|Metadata and integrity|
|Volatile Write Cache|Durability and RU accounting|
|Flush|Correct persistence|
|Dataset Management|Invalidated data and GC|
|Write Zeroes|Accounting and placement semantics|
|Copy|Placement semantics where applicable|
|APST|FDP activity across power-state transitions|
|PCIe ASPM|No state corruption during link power management|
|Firmware activation|FDP configuration and state compatibility|
|Sanitize|Correct data removal and FDP state behavior|
|Format NVM|Correct namespace reset behavior|
|Reservations|Correct access control with FDP I/O|
|SR-IOV/multi-controller|Resource ownership and isolation|
|Telemetry|FDP faults correctly diagnosable|

Only require combinations that are supported and defined by the DUT's advertised capabilities and applicable specifications.

## 19. Automated test framework architecture

For scalable validation, separate the test framework into protocol, FDP model, workload, instrumentation, and assertion layers.

Test Orchestrator

pytest / test scheduler / parameter generator

NVMe command layer

Admin, I/O, Directives, IO Management

FDP reference model

Configurations, PIDs, RUHs, expected state

Workload generator

Data patterns, concurrency, resets

Diagnostic collector

Logs, NAND mapping, counters, traces

Assertion and result engine

Protocol compliance • State consistency • Data integrity • Physical placement

### Suggested Python test organization

```
fdp_validation/
├── core/
│   ├── nvme_transport.py
│   ├── nvme_admin.py
│   ├── nvme_io.py
│   ├── fdp_model.py
│   ├── fdp_parser.py
│   └── status_decoder.py
├── tests/
│   ├── test_fdp_discovery.py
│   ├── test_fdp_configurations.py
│   ├── test_fdp_enable_disable.py
│   ├── test_fdp_namespace.py
│   ├── test_fdp_directives.py
│   ├── test_fdp_pid.py
│   ├── test_fdp_ruh_status.py
│   ├── test_fdp_ruh_update.py
│   ├── test_fdp_logs.py
│   ├── test_fdp_events.py
│   ├── test_fdp_io.py
│   ├── test_fdp_reset.py
│   ├── test_fdp_power_loss.py
│   ├── test_fdp_gc.py
│   └── test_fdp_stress.py
├── workloads/
│   ├── sequential.py
│   ├── random.py
│   └── hot_cold.py
├── diagnostics/
│   ├── firmware_trace.py
│   └── nand_mapping.py
└── reports/
```

### Reference-model assertions

For each command, maintain the expected state before and after completion.

Examples:

- A rejected invalid-PID Write must not modify the target LBA.
- A successful RUH Update must not invalidate existing logical data.
- An RUH status response must contain valid descriptor counts and PIDs.
- An FDP event must match the configured reporting rules.
- A controller reset must not produce an impossible FDP state.
- Two namespaces must never return one another's data.
    
A model-based framework is more robust than asserting fixed RUAMW values or RU identities that may legitimately vary with firmware allocation decisions.

## 20. Test coverage and release exit criteria

An exhaustive plan should combine deterministic cases, boundary-value analysis, pairwise configuration coverage, randomized state-machine testing, and targeted fault injection.

Testing every mathematical combination is generally impractical. The goal is exhaustive coverage of defined requirements and known state transitions, supplemented by risk-based interaction testing.

### Suggested release gates

|Gate|Acceptance criterion|
|---|---|
|Specification compliance|100% applicable mandatory requirements pass|
|P0 tests|100% pass|
|Data integrity|Zero unexplained corruption or acknowledged data loss|
|Protocol negative tests|All required completion statuses match|
|RUH/PID model|No unexplained mapping inconsistencies|
|Reset recovery|All supported reset paths pass|
|Power-loss recovery|All required durability/recovery cases pass|
|Stress|No unresolved hangs, deadlocks, or leaks|
|Events and logs|Required fields and state transitions validated|
|Performance|Meets product-specific targets|
|WAF|Meets product-specific workload objectives|

### Recommended execution order

Phase 1

Discovery, configuration logs, FDP feature state

Phase 2

Namespace provisioning, directives, PID validation

Phase 3

RUH status, RUH Update, RU rollover

Phase 4

I/O, data integrity, FDP statistics and events

Phase 5

Reset, power loss, negative tests, fault injection

Phase 6

Garbage collection, performance, endurance and feature interactions

## 21. Most critical FDP validation scenarios

For an SSD firmware qualification program, I would give particular attention to these six scenarios:

1. PID-to-RUH mapping correctness: Verify that each valid PID selects the correct RUH across multiple namespaces and Reclaim Groups.
    
2. RU boundary transitions: Verify writes that exactly fill, nearly fill, and exceed remaining RU capacity.
    
3. RUH Update under active I/O: Verify mapping changes, outstanding commands, and preserved logical data.
    
4. Power loss during FDP metadata updates: Verify recovery of placement state without corruption.
    
5. GC with mixed data lifetimes: Verify valid-page relocation, correct event reporting, and physical placement accounting.
    
6. Multi-namespace shared RUHs: Verify resource sharing without cross-namespace data corruption or incorrect status reporting.
    

These are especially important because they exercise interactions among host-visible NVMe commands, firmware placement metadata, and the underlying NAND management layer.

The plan above is a comprehensive qualification framework, but it is not yet a clause-by-clause conformance suite. To make it fully traceable to NVMe 2.2, every case should be mapped to its normative specification figure and exact expected SCT/SC, with revision-specific field encodings and optionality verified.

Recommended next deliverable: A detailed, automation-ready FDP test specification containing individual test IDs, preconditions, exact command sequences, command DWORD values, expected CQE statuses, expected log changes, and pass/fail assertions. That would turn this validation framework into an executable SSD qualification plan.