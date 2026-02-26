# Requirements Review: PII Anonymizer Sidecar
## Schema & Automated Validation
The validation script passes 23/23 — no schema errors, no flag violations, all IDs unique, all parent/child links intact. The traceability graph shows 23 trace links with no orphan L2s and no childless L1s.

##Interface Coverage (Context Diagram → Requirements)
All 5 interfaces from the context diagram are fully mapped with no gaps and no extras:
Interface	L1 Requirements
IF-HTTPS-API	SYS-001, SYS-002, SYS-003
IF-TLS	SYS-004
IF-VAULT	SYS-005, SYS-006
IF-CLI	SYS-007, SYS-008
IF-ADMIN	SYS-009, SYS-010
Clean 1:1 coverage. No interface is over-specified or missing.

## Issues Found
1. IF-VAULT is architecturally internal — not a system boundary interface
This is the most significant structural issue. Looking at the context diagram, SQLite is inside the Docker container boundary, not an external system. IF-VAULT is a subsystem-to-subsystem interface, not a system boundary crossing. SYS-005 and SYS-006 are therefore technically L2-level concerns dressed as L1 requirements.
The practical consequence: the L1 test cases for SYS-005 and SYS-006 require stopping/restarting the Docker container (acceptable at L1) but SYS-006's test procedure references direct DB manipulation — a BLACK_BOX_VIOLATION.
Options: Either promote the SQLite vault to an explicit external system on the context diagram (as a persistent volume mounted from the VPS host, which is genuinely outside the container boundary), or demote SYS-005/SYS-006 to L2 and add a single L1 requirement like "The system shall survive restart without session data loss" at IF-ADMIN or IF-HTTPS-API.

2. Black-box violations in two L1 test cases
TC-SYS-003 (step 3): verifies no session was created by calling GET /v1/sessions/{sessionKey} from loopback. Since IF-ADMIN is a defined system interface, this is defensible — but only because admin is explicitly modeled. This is borderline; document the justification.
TC-SYS-006 (step 2 and pass criterion): the test procedure says "trigger the purge job (either wait for interval or invoke directly in test)" and the pass criterion calls GET /v1/sessions/{sessionKey}/mappings. "Invoke directly" implies calling an internal function — a clear BLACK_BOX_VIOLATION. Fix: restrict the test to either waiting for the TTL to expire and observing via the admin interface, or explicitly adding a POST /v1/admin/purge trigger endpoint so the invocation is at a defined boundary.

3. SYS-001 cold-start latency exceeds spec
The verification result notes cold-start at 5.56s vs the 5s requirement. This is currently marked PASS, which is incorrect — it should be PARTIAL or FAIL with a waiver. The requirement as written has no cold-start exception. Either:
* Add a cold-start carve-out to the SYS-001 description ("after initial model load"), or
* Create a separate SYS-001a for first-request latency with a 15s allowance, or
* Accept the waiver and change verification status to FAIL_WAIVED with documented rationale

4. SYS-004 is BLOCKED with no mitigation path
No domain, no TLS. This is fine for the current dev phase but SYS-004 and SUB-SEC-003 need a clear acceptance condition for when they become unblocked. Suggest adding a precondition note in the verification result: "Blocked pending domain acquisition. Target: Phase 6 packaging milestone."

5. Missing L1 requirement: request size limit
SUB-SEC-002 (payload size limit) exists as an L2 but there is no L1 parent that covers it. Its parent_ids points to SYS-003 (reject unauthenticated requests), which is wrong — payload size rejection is not an authentication concern. There should be an L1 like: "The system shall reject any /v1/ request whose body exceeds 1MB with HTTP 413."* This is observable at IF-HTTPS-API. SUB-SEC-002 should derive from it.

6. SUB-INT-002 is orphaned from SYS-007 child_ids
SUB-INT-002 (CLI --json flag) lists SYS-007 as its parent, but SYS-007.child_ids only lists ['SUB-CTRL-001', 'SUB-CTRL-003', 'SUB-INT-001'] — SUB-INT-002 is missing. Broken bidirectional link.

## Verification Status Summary
ID	Status	Action
SYS-001	Marked PASS but 5.56s > 5s	Fix requirement wording or status
SYS-004	BLOCKED	Add unblock condition to verification notes
All others	PASS	✓

## What's Missing
Comparing the SPEC to the requirements, two SPEC features have no requirements coverage yet:
* Streaming restore guidance (rolling-buffer approach, SPEC §Streaming) — P1 in the SPEC, no requirement exists
* Allowlist/denylist options (options.allowlist, options.denylist in POST /v1/anonymize) — no L1 or L2 covering it
* mappingVersion in response — SYS-001 describes the endpoint but no requirement verifies this field is present and incrementing
These are likely intentionally deferred, but worth flagging so they land on the backlog.
---6 issues found, ranked by severity:
1. IF-VAULT boundary classification — SQLite is inside the Docker container; IF-VAULT is not a system boundary interface. Decide: promote vault to external entity on context diagram, or demote SYS-005/006 to L2.
2. TC-SYS-006 black-box violation — "invoke purge directly" is internal access. Fix by exposing an admin trigger endpoint or relying purely on TTL wait.
3. Missing L1 for payload size limit — SUB-SEC-002 has no valid L1 parent. SYS-003 is the wrong parent. Add a new L1 at IF-HTTPS-API.
4. SYS-007 missing SUB-INT-002 in child_ids — broken bidirectional link.
5. SYS-001 verification marked PASS at 5.56s > 5s — needs a waiver or requirement fix.
6. SPEC features without requirements — streaming restore, allowlist/denylist, and mappingVersion incrementing have no coverage.
The validation report and traceability graph are attached above.
