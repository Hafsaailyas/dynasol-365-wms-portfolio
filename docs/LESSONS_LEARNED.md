[← Back to README](../README.md)

# Lessons Learned

> 📌 **Portfolio Document** — Technical challenges and solutions from building the Warehouse Management System.

---

## Overview

Building a production-grade mobile ERP integration surfaces problems that don't appear in tutorials or toy projects. This document captures the five most significant technical challenges encountered during development, the root causes behind each, and the solutions that made it to production. It also includes broader process lessons and what I would do differently on a second implementation.

---

## Challenge 1: Barcode Scanner Flicker and False Triggers

### Problem
The initial scanner implementation produced a poor user experience: the scanner would flicker between "scanning" and "idle" states multiple times per second, and the same barcode would be processed two or three times in rapid succession. In a warehouse environment where a single scan should produce exactly one result, duplicate processing caused quantities to be added multiple times.

### Root Cause
CameraX delivers frames to the ML Kit analyser on every camera frame — typically 30 frames per second. The first implementation processed every positive detection immediately. A barcode held in frame for even half a second would trigger 10–15 consecutive detections, each dispatching an event to the ViewModel.

### Solution
Introduced a debounce mechanism in the scanner layer combined with a "cooldown" flag that is set on the first successful detection and cleared only after the user performs a deliberate action (tapping "Done" or navigating away). The analyser discards subsequent detections of the same barcode value within the cooldown window. A separate `lastScannedValue` tracker prevents the same barcode from re-triggering even after the cooldown clears, unless the value changes.

### Result
Zero duplicate scans in production. Scan responsiveness remained under 300ms for first detection — fast enough to maintain warehouse workflow pace.

---

## Challenge 2: Offline Data Consistency Across App Restarts

### Problem
Early in development, data would disappear unpredictably when the Android OS killed the app process under memory pressure. A warehouse operator could spend 20 minutes scanning items into a cart, walk away from their phone, return to find the app cold-started, and lose all pending adjustments. This was a critical reliability failure.

### Root Cause
The initial architecture held all state in ViewModel instances, which are destroyed when the process is killed. SharedPreferences was considered for lightweight state but is unsuitable for structured, relational data. The app had no durable persistence layer at the time.

### Solution
Replaced all in-memory state in the manager classes with Room Database-backed persistence. Each manager (`CartManager`, `GlobalScanManager`, `ScanSessionManager`) initialises by reading from Room on first access and writes every state change back to Room immediately and synchronously within the manager's internal coroutine scope. The app's first action after login is to restore all per-user state from the database. Because Room uses SQLite under the hood, state survives process death, device reboots, and even APK updates.

Additional design detail: all Room entities include a `userId` field so that state is isolated per user on shared devices. A user logging out does not clear another user's pending adjustments.

### Result
Complete elimination of data loss due to process kills. Operators can put their phone down mid-session, receive a phone call, restart the device, and resume exactly where they left off.

---

## Challenge 3: Permission Granularity Without Complexity Explosion

### Problem
The client's warehouse has distinct roles — some staff should only be able to scan and update quantities, others should only be able to trigger posting, and supervisors need both. The initial design used a simple `canPost` boolean flag, which quickly proved insufficient when the client requested separate controls for shipment vs receipt operations.

### Root Cause
Boolean flags do not compose well. Two document types × two operation types = four flags, and each combination needed to be validated both on the client (to show/hide UI) and on the Business Central server (to authorise the operation). With flags, the validation logic becomes a series of `if/else` branches that must be kept in sync between two codebases.

### Solution
Replaced boolean flags with a single `AccessType` enum with 9 discrete values covering every meaningful combination of permissions. Each access type maps to a precise set of allowed operations. Validation in Business Central is a `case` statement on the enum value — one code path per access type, no branching combinations. The Android app reads the access type from the certificate and shows or hides UI controls accordingly. Adding a new permission set requires adding one enum value and one `case` branch, rather than coordinating multiple flags across two systems.

### Result
Clean, auditable permission model. Every user's capabilities are expressed in a single field. The access type table (see Architecture doc) is readable by non-technical stakeholders. No permission-related bugs in production.

---

## Challenge 4: Batch API Performance Under High Volume

### Problem
The initial implementation sent one API request per scanned item. In a busy shipment with 40–60 line items, the save operation was taking 45–90 seconds and occasionally timing out under poor warehouse WiFi conditions. Users were abandoning saves mid-operation, leaving the system in inconsistent states.

### Root Cause
HTTP request overhead dominates when payloads are small. Each request included OAuth token validation, OData routing, and a BC session acquisition — fixed costs that dwarfed the actual data transfer. Forty requests meant forty full round-trips at ~1–2 seconds each.

### Solution
Redesigned the payload format to support batching at the application layer: a single POST to the `/apiBuffers` endpoint now carries an array of line adjustments for an entire document. The Business Central buffer processor iterates over the array and applies each line update within a single transaction. On the Android side, the cart drawer's "Save All" operation serialises the entire pending queue into one JSON payload.

Additionally, the buffer table's `OnAfterInsertEvent` was already providing async processing — this meant the mobile client only waits for the buffer insert to confirm (fast), not for BC to finish processing all line updates (potentially slow).

### Result
Save time for a 50-line shipment dropped from ~60 seconds to under 3 seconds. Zero timeout-related failures in production since the change.

---

## Challenge 5: Multi-Match Barcode Resolution

### Problem
The barcode reference data in Business Central is not guaranteed to be unique across documents. An item reference number can appear on multiple open shipments simultaneously — for example, if the same SKU is included in two separate transfer orders. The initial implementation arbitrarily picked the first matching document, causing operators to unknowingly update the wrong shipment.

### Root Cause
The `GlobalBarcodeRepository` was built with the assumption that each barcode maps to exactly one document. In practice, the same item reference can appear on multiple open warehouse documents, particularly in multi-warehouse or multi-order environments.

### Solution
Changed the lookup return type from a single document to a list of documents. When the result contains exactly one item, the existing auto-resolve path is used (no user interaction required). When the result contains two or more items, a `DocumentSelectorDialog` is presented to the user with the document number, document type, and item description for each match, allowing them to select the correct target. The selection is remembered within the current scan session so that subsequent scans of the same barcode within the same session auto-resolve to the previously selected document.

### Result
Elimination of wrong-document scan errors. Operators have explicit confirmation of which document they are updating, and the session memory prevents the dialog from becoming an annoyance for high-frequency scans of the same item.

---

## Process Lessons

**Write the permission model first.** Permission requirements expand in ways that are expensive to retrofit. The shift from boolean flags to an enum added a week of rework that could have been avoided by asking more detailed questions about role requirements during the discovery phase. On future projects, I will spec the full permission matrix before writing a line of code.

**Offline-first cannot be bolted on.** The decision to retrofit Room persistence mid-project was painful. The data model, the manager classes, and the sync logic all needed to be redesigned around persistence. Starting with "every state change writes to Room" from day one would have saved two weeks of rework and avoided the data-loss bugs that reached early testers.

**ERP integration requires pessimistic assumptions.** Business Central's API behaviour differs between cloud and on-premises, between versions, and between configurations. Never assume that a pattern that works in a sandbox environment will behave identically in a production tenant. Build validation and detailed error logging into every API interaction from the start.

**Design for the network you have, not the network you want.** Warehouse WiFi is patchy, intermittent, and occasionally non-existent in cold-storage areas. Every feature should be designed to function fully offline with a clearly understood sync point. Features that require constant connectivity are a support burden in warehouse environments.

---

## What I'd Do Differently

**Separate the scanner state machine into its own class from the beginning.** The scanner logic grew organically inside the ViewModel before being extracted. A dedicated scanner state machine with explicit states (`Idle`, `Scanning`, `Cooldown`, `Processing`) would have made the debounce and duplicate-prevention logic cleaner and easier to test.

**Use a single serialisable state object per manager.** The current Room schema has seven tables spread across three managers. With hindsight, a more compact representation — a serialised state blob per user per session — would be easier to migrate and back up, at the cost of some query flexibility.

**Build a mock BC API server for development.** API development was blocked whenever the Business Central sandbox was unavailable for maintenance. A local mock server that replicated the OData response structure would have removed this dependency and allowed faster iteration on the network layer.

**Instrument the app with analytics from day one.** Post-launch, understanding which features are used most, which errors appear most frequently, and where operators abandon workflows required digging through BC audit logs. Lightweight telemetry in the Android app would have surfaced these insights much faster.

---

[← Back to README](../README.md)
