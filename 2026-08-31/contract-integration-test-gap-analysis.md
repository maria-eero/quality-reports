# Proposal: Contract & Integration Tests in CI

**Author:** Maria Rebelo (QAE, Core Engineering)
**Date:** 2026-08-31 (second refresh — extends the 2026-06-26 baseline and the 2026-07-20 refresh)
**Audience:** Engineering Manager, Core Engineering · Cloud Engineering
**Companion:** [Contract & Integration Testing — Implementation Guide](./contract-integration-testing-implementation-guide.html) — the how-to handoff for the cloud side

---

## Executive Summary

Between **2026-07-21 and 2026-08-31** (6 weeks), CORE filed **325 bugs**: **57 P0 blockers**, **69 P1 criticals**, **79 P2 majors**, **115 P3 minors**, and 5 P4s. Of those, **48 are contract, state-sync, or platform-parity failures — 24 of them P0 or P1.**

The headline finding is not the share. It is that **the rate is flat and the interfaces repeat.**

- Categorized bugs held at **8.0/week** (August) versus **7.0/week** (July) — the pattern is structural, not a spike.
- The interface this proposal named as **target #1 in July — Speedtest provider overrides — produced 7 more bugs in this window**, including a P1 Insight crash and two API-validation defects that accept and persist bad data.
- **Multi-WAN / WAN links produced 19 bugs, 9 of them P0.** It has displaced Speedtest overrides as the highest-volume boundary in CORE and is the single strongest candidate for the next reference implementation.
- **CORE-33235 is a MACsec Port Security recurrence** — on the exact interface the shipped contract tests cover. It escaped them, and *why* is the most useful thing in this document: cloud serves `port_security` in two representations that disagree, and no test anywhere asserted they should agree. See [Where the shipped tests hit their ceiling](#where-the-shipped-tests-hit-their-ceiling).

The reference implementation shipped: Port Security contract + integration tests are **merged on both platforms** — Android (PR #13181, 2026-07-30, commit `dd9f38f4`) and iOS (PR #14285, 2026-08-03, commit `74221e2c`). They run in the standard unit suite on every PR, with zero new dependencies.

What is new in this refresh: the evidence now points at **the cloud side of the boundary**, not the client side. 15 of the 48 are producer-side defects a client-only test can never catch. That is what the companion implementation guide is for.

---

## The Problem: Bugs That Slip Through Unit Tests

Our current testing pyramid relies heavily on:
- Unit tests (logic correctness within a single module)
- Manual QA / E2E tests (full user journeys, slow and late in the cycle)

**The gap:** No automated validation that services agree on data formats, required fields, response codes, and state transitions at their boundaries.

Each side's unit tests mock the other side. The mock is whatever the developer imagined. When the two imaginations differ, nothing fails until a build is running end to end — usually after the bug has shipped.

---

## Evidence: Real Bugs from CORE (2026-07-21 → 2026-08-31)

### Category 1: API Contract Violations

One service produces data another cannot consume: schema, nullability, error codes, enums, key formats, required fields.

| Ticket | Priority | Summary | Preventable By |
|--------|----------|---------|----------------|
| [CORE-33144](https://eeroinc.atlassian.net/browse/CORE-33144) | P0 | DynamicKV/throttle reads during service startup silently return defaults — AppConfig warming is async and `get()` is non-blocking, so any read before the first poll completes falls back to the default. Regression from CORE-31102 (2026-07-22) removing `RoutingDynamicKVService`; Consul's `get()` used to block until warm | Contract test: a DKV read before warm-up either blocks or signals unavailability — it must never be indistinguishable from a real "default" value |
| [CORE-32415](https://eeroinc.atlassian.net/browse/CORE-32415) | P1 | Insight Speedtest overrides page crashes on an offline network — `TypeError: Cannot read properties of null (reading 'network_group')`. The API returns **200 OK with valid data**; the frontend assumes `gateway` is always non-null | Contract test: overrides response schema declares `gateway` nullable, and the consumer renders a network with no gateway |
| [CORE-33202](https://eeroinc.atlassian.net/browse/CORE-33202) | P1 | `POST /organizations/{id}/users` returns 400 `error.organization.user.not.created` for **every** email, including guaranteed-unique fresh addresses, on org 60034. Blocks node-automation's `isp_tech_login` suite | Contract test: a valid create request with a unique email returns 201; the failure code is typed and distinguishable from a uniqueness conflict |
| [CORE-32551](https://eeroinc.atlassian.net/browse/CORE-32551) | P1 | All `/cellular_profile/:id/verizon/delete` calls time out and fail with HTTP 500 (with retries), blocking Verizon prod profile-switching suites | Contract test: the delete endpoint returns a typed 2xx/4xx for known inputs — a 500 is never the contracted outcome |
| [CORE-32502](https://eeroinc.atlassian.net/browse/CORE-32502) | P2 | WAN links created without a name break Insight. **Cloud marks `name` optional; the Apollo GraphQL contract declares it required** — the query fails whenever it is absent | Contract test: producer's optionality and consumer's schema are asserted against one shared source; `name`, `port`, `eero_id`, `priority` agree on both sides |
| [CORE-33044](https://eeroinc.atlassian.net/browse/CORE-33044) | P2 | On boot, a Multi-WAN node fetches `/config` and the response **omits the `wan` field**, silently disabling Multi-WAN. A subsequent `config` action returns it correctly | Contract test: `/config` always includes `wan` for a Multi-WAN-enabled node, including the boot-path response |
| [CORE-33380](https://eeroinc.atlassian.net/browse/CORE-33380) | P2 | `MonolithDataService/GetNetworkNodes` returns gRPC `INTERNAL` (13) when `node-session-stage` DynamoDB reads are throttled — 1,744 of ~171k calls in the 19:00 UTC hour on 2026-08-28. Pages core on-call; alert has flapped 212 times | Contract test: a throttled downstream maps to a retryable status, never `INTERNAL` |
| [CORE-32316](https://eeroinc.atlassian.net/browse/CORE-32316) | P2 | Per-port "Test Ethernet cable" runs the check on **ALL** ports — the port parameter is ignored, value-independently | Contract test: the per-port cable-check request scopes to exactly the port in the payload |
| [CORE-32295](https://eeroinc.atlassian.net/browse/CORE-32295) | P2 | Speedtest override resolution: when the highest-priority layer holds a runtime-ineligible provider, resolution falls all the way to Default instead of ceding to the next-priority valid layer — silently discarding a working override | Contract test: resolution over the 6 targeting layers cedes to the next eligible layer, asserted layer by layer |
| [CORE-32964](https://eeroinc.atlassian.net/browse/CORE-32964) | P2 | `/nodes/:id` redirect only special-cases 403/404. Unauthenticated users are not redirected to login, and an out-of-range node id shows a bare "Error loading data" instead of not-found | Contract test: every declared response status maps to a defined client outcome — no fall-through |
| [CORE-32695](https://eeroinc.atlassian.net/browse/CORE-32695) | P3 | `PUT /2.2/networks/:id/settings` does not strictly validate `nat_port_randomization`: string `"true"` is accepted (200), coerced, stored, **and reboots the network**; `null` returns 200; only a number returns 400 | Contract test: non-boolean values for a boolean field are rejected 400 — no coercion, no side effects |
| [CORE-32631](https://eeroinc.atlassian.net/browse/CORE-32631) | P3 | Speedtest override endpoints never validate that the network exists. `GET` on a phantom id returns 200 with `metadata:null` instead of 404; `PUT` returns 200 and **persists a real orphan record** | Contract test: writes to a non-existent resource return 404 and persist nothing |
| [CORE-33268](https://eeroinc.atlassian.net/browse/CORE-33268) | P3 | `remote-nodes.phi-accrual-value:∞\|g` — the phi-accrual detector emits `Double.PositiveInfinity`, `DecimalFormat` renders it as the Unicode infinity glyph, and Telegraf's statsd parser cannot read it as float64. Paged PROD | Contract test: every emitted gauge is a finite float64 on the wire; non-finite values are clamped or dropped at the emit boundary |
| [CORE-33094](https://eeroinc.atlassian.net/browse/CORE-33094) | P3 | Speedtest config-override duplicate guard is bypassable via firmware-version normalization: the field accepts `7.13.0` and `v7.13.0`, the backend normalizes to `v7.13.0`, but the client conflict check compares the **raw** string — so the PUT is sent and rejected 400 instead of blocked inline | Contract test: firmware-version keys are normalized identically on both sides of the boundary before comparison |
| [CORE-32903](https://eeroinc.atlassian.net/browse/CORE-32903) | P3 | Dejavu is absent from `HwModelCapabilityService.wlanRateLimitPolicyCapable` (which allows Crane, Novo, Snowbird) with no firmware gate — a PUT subnet carrying `ratelimitConfig` is rejected `InvalidRateLimitParameters` on any Dejavu node | Contract test: capability tables are exhaustive over the hw-model enum — adding a model fails the test until every table is reviewed |

### Category 2: Data Sync / State Mismatch Between Systems

Two systems hold contradictory state about the same entity.

| Ticket | Priority | Summary | Preventable By |
|--------|----------|---------|----------------|
| [CORE-33057](https://eeroinc.atlassian.net/browse/CORE-33057) | P0 | A non-blocking antenna warning during Connect to Leo lets setup report **success** while the network is created in the backend only — offline in admin, Ladro attached to nothing, network never appears in the app | Integration test: setup reports success only after the network is reachable and visible through the read path the app uses |
| [CORE-33304](https://eeroinc.atlassian.net/browse/CORE-33304) | P0 | A failed WAN setup leaves an **orphaned WAN connection** in the backend. Android does not render an inactive entry; iOS does — so the same orphan is invisible on one platform and visible on the other | Integration test: a failed WAN setup leaves no persisted connection, and both clients render the post-failure state identically |
| [CORE-33341](https://eeroinc.atlassian.net/browse/CORE-33341) | P0 | With primary WAN disconnected, a secondary **VLAN** WAN carries traffic and the network is online, but reports `active: false` — so the app shows the connection as "Ready". Reproduces on DHCP+VLAN, Static+VLAN, PPPoE+VLAN; not without VLAN | Integration test: `active` on a WAN link tracks the actual default route, across every connection type including VLAN variants |
| [CORE-33320](https://eeroinc.atlassian.net/browse/CORE-33320) | P0 | Retail Residential customers cannot change SSID or password from App or Insight. **The change is written to the audit log but never applied**; support could not do it either and had to fall back to Admin | Integration test: after an SSID/password write returns success, the subsequent read returns the new value — audit-log success is not acceptance |
| [CORE-33188](https://eeroinc.atlassian.net/browse/CORE-33188) | P0 | On a business network, the guest bandwidth limit cannot be disabled — toggling off and saving silently retains the previous value | Integration test: a save that clears a limit is reflected on the next read; disable is not a no-op |
| [CORE-32917](https://eeroinc.atlassian.net/browse/CORE-32917) | P1 | Backup WAN reports `active: true` while the default route is still on the primary (eth1) | Integration test: exactly one WAN link reports `active: true`, and it is the one holding the default route |
| [CORE-33148](https://eeroinc.atlassian.net/browse/CORE-33148) | P1 | Mirror image: after the backup link reaches `CONFIGURED`, the primary stays `active: false` despite state `ONLINE` and carrying the traffic | Integration test: same invariant, asserted from both directions after a link-state transition |
| [CORE-32327](https://eeroinc.atlassian.net/browse/CORE-32327) | P1 | Foghorn and Retrograde accessories are **still returned by `GetEerosDetail` after removal** from the network — but only while they remain plugged in | Integration test: after an accessory removal returns success, it is absent from the read path regardless of power state |
| [CORE-32515](https://eeroinc.atlassian.net/browse/CORE-32515) | P1 | "Resume Profile" does not unpause devices when a scheduled pause is active and a device is also individually paused. **Ongoing ~2 years**, confirmed by T2 on 2026-07-03 | Integration test: manual resume clears the effective block regardless of which pause sources are stacked |
| [CORE-32463](https://eeroinc.atlassian.net/browse/CORE-32463) | P1 | Saving a new network name in bridge mode does not reflect the change in the app or Insight | Integration test: SSID write → read round-trip asserted in bridge mode, not just routed mode |
| [CORE-32859](https://eeroinc.atlassian.net/browse/CORE-32859) | P1 | The physical port LED is off while the app reports PoE enabled — the user cannot visually confirm which ports deliver power | Integration test: rendered PoE state derives from the device-reported port state, not from the requested configuration |
| [CORE-33055](https://eeroinc.atlassian.net/browse/CORE-33055) | P1 | Verizon activation sometimes fails and **issues multiple ICCIDs** for one device | Integration test: a retried activation is idempotent — one device converges on one ICCID |
| [CORE-32766](https://eeroinc.atlassian.net/browse/CORE-32766) | P2 | When the Consul KV read for traffic policies fails, xds swallows the error, substitutes an **empty** policy map, and pushes a snapshot reverting **the entire fleet** to `DefaultTrafficPolicy`. Introduced in envoy-control-server #37 on **2020-06-11** | Integration test: a KV read failure aborts the snapshot push and retains the last good policy map — empty is never a valid substitute |
| [CORE-33364](https://eeroinc.atlassian.net/browse/CORE-33364) | P2 | Manual→Automatic IP switch is not applied uniformly: Guest and both Business subnets revert to the default range while the main network (`br-lan`) keeps the old manual IP, leaving the gateway inconsistent | Integration test: an addressing-mode change applies to every subnet or to none |
| [CORE-33309](https://eeroinc.atlassian.net/browse/CORE-33309) | P3 | 23 Trieste serials — **already sold to customers** — have no node properties in `hardwaredata` at all. The known-eero-sync forward-only cursor advances past rows not yet present on a lagging replica, leaving them permanently below the watermark. Explicit recurrence of CORE-32469 in the same window; backfilled via cloud PR #29287 | Integration test: every node written on the factory line is readable from cloud within N minutes; the sync cursor cannot advance past unread rows |
| [CORE-32694](https://eeroinc.atlassian.net/browse/CORE-32694) | P3 | NAT Port Mode is applied as a separate non-atomic mutation. When DHCP succeeds and NAT fails, the form closes with a generic toast and **no indication that the DHCP change was applied** — split state is hidden | Integration test: a partial-failure save reports which half applied, or rolls back |

### Category 3: Interface Parity / Schema Drift Between Platforms

iOS, Android, Insight, and Web disagree on how to interpret the same data.

| Ticket | Priority | Summary | Preventable By |
|--------|----------|---------|----------------|
| [CORE-32690](https://eeroinc.atlassian.net/browse/CORE-32690) | P0 | For an active eero Plus subscriber, "Additional wifi networks" appears in Settings on AOS and is **absent entirely on iOS** — same account, same network | Contract test: the Settings row set is derived from a shared entitlement predicate asserted identically on both platforms |
| [CORE-32310](https://eeroinc.atlassian.net/browse/CORE-32310) | P0 | The "BroadbandNow" link inside the WAN IP support article returns **403 from CloudFront on Android**; the same flow works on iOS | Contract test: outbound support-article link targets resolve on both platforms from the same article source |
| [CORE-33269](https://eeroinc.atlassian.net/browse/CORE-33269) | P0 | Adaptive Ethernet (WAN Power Optimization) is missing on **both Mobile and Insight** for the Dejavu model only, blocking the whole WAN Power Optimization run (TestRail 170838) | Contract test: capability→UI surfacing is exhaustive over the hw-model enum on every client (see CORE-32903, the cloud-side twin) |
| [CORE-33185](https://eeroinc.atlassian.net/browse/CORE-33185) | P1 | Activate Subscription license-key field: iOS accepts space characters while typing and only errors on Next; Android rejects the space as text outright | Contract test: field-level input validation for the license key is one shared rule, asserted on both platforms |
| [CORE-32748](https://eeroinc.atlassian.net/browse/CORE-32748) | P1 | On a B2B network, the eco-efficiency page is **entirely different** on AOS — adaptive ethernet and power-saving-schedule rows missing. 5/5 reproducibility | Contract test: eco-efficiency row set for a B2B network is one shared list read by both platforms |
| [CORE-32666](https://eeroinc.atlassian.net/browse/CORE-32666) | P1 | A wired leaf on physical Port 1 is reported as "Port 2" by Android; iOS correctly shows Port 1. Same eeroOS 7.17.0-12695. **Recurrence of CORE-31750** from the July snapshot | Contract test: `connected_to.port` semantics are fixed in the API spec and asserted identically by iOS and Android |
| [CORE-32595](https://eeroinc.atlassian.net/browse/CORE-32595) | P1 | The Data Pack "running low" alert fires at 80% on both iOS and Android where the spec is ≥90% — both clients hardcode the same wrong threshold | Contract test: the alert threshold is a single shared constant sourced from the API, not a per-client literal |
| [CORE-33165](https://eeroinc.atlassian.net/browse/CORE-33165) | P1 | Restarting one leaf makes **all** eeros show "Restarting" on the Android home screen; iOS correctly scopes the status to the restarted node | Contract test: per-node status is keyed by node id on both platforms — a status event never fans out to siblings |
| [CORE-33235](https://eeroinc.atlassian.net/browse/CORE-33235) | P2 | **MACsec recurrence — and a two-representation contract split.** The app shows Port Security enabled on the Novo WAN port (`eth1`). Cloud serves port security in **two independent representations**: `state_data`/`/eeros` is faithful (`macSecStatus=None`, `isWanPort=True`), while `/connections` sets `enabled` straight from the persisted `NodeEthernetPortSetting.portSecurityOn` DB flag with **no WAN guard, no `capable` guard and no live-status gating** — and never clears it. The port rows read the second one. Reproduces on Novo-Crane and Novo-Snowbird, not Hornbill-Crane, because Novo's `eth1` is both the WAN port and PORT_SECURITY-capable. **Primary fix is cloud-side** | Contract test: the two representations of `port_security` agree for the same port, and `enabled` in the connections view is gated on `capable`, live status, and not-WAN. See the ceiling section below |
| [CORE-32446](https://eeroinc.atlassian.net/browse/CORE-32446) | P2 | The iOS IP Addresses info popup omits the IPv6 summary section that Android renders inline | Contract test: info-popup section set comes from one shared content map |
| [CORE-32517](https://eeroinc.atlassian.net/browse/CORE-32517) | P3 | On first network creation Android presets the timezone to Pacific Standard Time while iOS shows "No timezone set" — divergent defaults for a value time-dependent features read | Contract test: the new-network default timezone is one shared value; "unset" is either valid on both platforms or neither |
| [CORE-32470](https://eeroinc.atlassian.net/browse/CORE-32470) | P3 | With eero Plus enabled, the Internet page shows a "Mobile backup" row on AOS and omits it on iOS | Contract test: the Connections row set is one shared list keyed by entitlement |
| [CORE-32473](https://eeroinc.atlassian.net/browse/CORE-32473) | P3 | Android renders an "Idle" subtext where the value belongs on the live-data-usage screen. **Recurrence of CORE-31458** from the July snapshot | Contract test: zero-value throughput formatting is defined in the shared schema and rendered identically |
| [CORE-32320](https://eeroinc.atlassian.net/browse/CORE-32320) | P3 | Internet backup dismissal control and the added-backup-network heading differ between Android and iOS | Contract test: heading and control affordance map to shared localization keys |
| [CORE-32472](https://eeroinc.atlassian.net/browse/CORE-32472) | P3 | Sort-by option labels differ between AOS and iOS | Contract test: sort-option labels map to shared localization keys |
| [CORE-32664](https://eeroinc.atlassian.net/browse/CORE-32664) | P3 | The Android port-forwarding "already in use" error message is inconsistent with iOS for the same server-side conflict | Contract test: a given conflict response maps to one shared user-facing string on both platforms |
| [CORE-32747](https://eeroinc.atlassian.net/browse/CORE-32747) | P3 | Top Devices and Profile headers missing on the Scans/Threats/Filters sessions across AOS and iOS. **Recurrence of CORE-32156** from the July snapshot | Contract test: the detail-view required-field set is asserted from the shared schema on both platforms |

---

## Impact Assessment

### Scale (2026-07-21 → 2026-08-31, CORE only)

- **325 total bugs**: 57 P0 · 69 P1 · 79 P2 · 115 P3 · 5 P4
- **48 contract / state-sync / parity examples** across the three categories — **24 of them P0/P1**
- **14.8%** of all bugs, **19.0%** of the P0/P1 bar

### The rate is flat — this is structural, not a spike

The August window is 42 days against July's 24, so raw counts are not comparable. Normalized:

| | June 26 baseline | July 20 refresh | Aug 31 refresh |
|---|---|---|---|
| Window | 60 days | 24 days | 42 days |
| Total CORE bugs | — | 167 (48.7/wk) | 325 (54.2/wk) |
| Categorized | 12 | 24 (7.0/wk) | 48 (8.0/wk) |
| Categorized share of all | ~40%¹ | 14.4% | **14.8%** |
| Categorized share of P0/P1 | — | 24.5% (13/53) | **19.0% (24/126)** |

¹ The June figure used a narrower, hand-picked "defensibly preventable" set over a 60-day window and is not rate-comparable; it is shown for continuity only.

Three windows, three months, and the categorized rate lands between 7 and 8 per week every time. The P0/P1 *share* softened from 24.5% to 19.0% because overall P0/P1 volume grew faster (53 → 126) than the categorized slice did — but the absolute count of preventable P0/P1s nearly doubled, 13 → 24. Nothing here is improving on its own.

### The interfaces repeat — and they are the ones we named

**Speedtest provider overrides** was target interface **#1** in the July proposal. In this window it produced **7 more bugs**:

| Ticket | Priority | Defect |
|--------|----------|--------|
| CORE-32415 | P1 | Page crash on null `gateway` — API returns 200 |
| CORE-32295 | P2 | Resolution falls to Default instead of ceding to next eligible layer |
| CORE-32296 | P3 | No-op Save fires a persisting PUT and bumps the version — churns shared-scope records |
| CORE-32631 | P3 | PUT to a non-existent network returns 200 and persists an orphan |
| CORE-33092 | P3 | Bundle offers `EeroUdpAutomatic` as a target provider (should be excluded) |
| CORE-33094 | P3 | Duplicate guard bypassable via firmware `v`-prefix normalization |
| CORE-32628 | P3 | Override drawers do not manage focus (WCAG 2.4.3 / 2.1.2) — not contract-related, listed for completeness |

Six of those seven are contract defects. Zero contract tests were added to that interface between the July proposal and today.

**Multi-WAN / WAN links** is the new highest-volume boundary: **19 bugs, 9 of them P0** (CORE-32502, 32613, 32614, 32660, 32870, 32917, 33032, 33033, 33044, 33081, 33148, 33186, 33248, 33269, 33304, 33305, 33341, 33355, 32283). It is under active development, it spans node → cloud → iOS → Android → Insight, and it is producing exactly the three failure shapes this proposal is about. **It should be the next reference implementation.**

### Chronic offenders (still firing)

| Ticket | Age / scale | Note |
|--------|-------------|------|
| CORE-32766 | Since **2020-06-11** | A single transient Consul KV read error reverts the entire fleet's traffic policies to defaults. Six years old, fleet-wide, still open |
| CORE-32515 | ~**2 years** | Scheduled-pause + manual-resume interaction leaves devices paused. Confirmed by T2 on 2026-07-03 |
| CORE-33309 | Recurrence of CORE-32469 **in the same window** | Same forward-only cursor watermark bug, now losing all node properties instead of just the MAC — for 23 nodes already sold to customers |
| CORE-33380 | 212 alert state changes | Throttling surfaced as gRPC `INTERNAL`; pages core on-call and flaps at its threshold |
| CORE-33144 | Introduced **2026-07-22** | Removing `RoutingDynamicKVService` turned a blocking read into a silent default. Six weeks from merge to P0 |
| CORE-32666 / CORE-32473 / CORE-32747 | Recurrences of CORE-31750 / CORE-31458 / CORE-32156 | Three July parity bugs reopened as new tickets on the same interfaces |

Same story as June and July: **systemic interface mismatches persist because nothing validates the contract between producer and consumer.**

### Customer impact (P0/P1 highlights)

- **CORE-33320 (P0):** Retail Residential customers cannot change their SSID or password from App or Insight — the change is audit-logged and never applied; support agents could not do it either
- **CORE-33057 (P0):** Users complete Leo setup, are told it succeeded, and have no working network and no error — the network exists in the backend only, offline and invisible to the app
- **CORE-33341 (P0):** A VLAN WAN carries live traffic while reporting `active: false`, so the app tells the customer the connection is merely "Ready"
- **CORE-33309 (P3 by priority, high by consequence):** 23 nodes shipped to customers with no properties in cloud — they cannot be set up correctly
- **CORE-32515 (P1):** Two years of customers tapping "Resume Profile" and watching their kids' devices stay blocked
- **CORE-32766 (P2 by priority, fleet-wide by blast radius):** One transient KV read failure silently reverts every service's traffic policy

---

## Where the shipped tests hit their ceiling

This is the most important section for the cloud handoff, and it is an argument against over-claiming for the approach that shipped.

**CORE-33235** landed against MACsec Port Security — the interface with 12 contract tests and 3 integration tests merged on each platform. The tests passed. The bug shipped.

The mechanism matters, and it is not what the ticket first looked like. The reporter's three-source snapshot showed node and cloud `state_data` both correct and concluded the defect was in the app. A code-level root-cause investigation (writeup available from Maria) found something more interesting:

**Cloud serves port security in two independent representations, and the port UI reads the wrong one.**

| | `state_data` / `/eeros` — "status view" | `/connections`, `/ethernet_ports` — "connections view" |
|---|---|---|
| Fields | `status` only | `capable`, **`enabled`**, `status` |
| `enabled` source | — emitted only when the node reports a live `macSecStatus` book | the persisted **`NodeEthernetPortSetting.portSecurityOn` DB flag** |
| Guards on `enabled` | n/a — faithful | **none**: no WAN guard, no `capable` guard, no live-status gating |
| Who reads it | Home MACsec banner only | **the port rows / port detail** — the buggy screen |
| Result on the WAN port | correct: `None` | **stale `enabled=true` on `eth1`** |

The flag is set by `enablePortSecurity` and `propagateToPeerOnEnable`, both of which gate only on `isMACSecCapable`, neither of which excludes a WAN port, and neither of which clears the flag when a port later becomes WAN. Novo reproduces and Hornbill does not because **both of Novo's ports advertise PORT_SECURITY capability**, so its WAN port can carry a stale `enabled`. **The primary fix is cloud-side.**

Why the shipped tests missed it:

| What the shipped tests assert | Why that was not enough |
|---|---|
| The client **decodes** a `port_security` payload correctly — field types, malformed-input tolerance, enum handling | The client decoded faithfully. The value it decoded was already wrong upstream |
| Golden fixtures represent payload shapes captured from the API | No fixture paired `enabled=true` with `status=DISABLED` on a WAN port, because the fixtures treated `enabled` as ground truth |
| Integration tests assert the ViewModel emits the state the API reported | It did exactly that. Faithfully rendering a bad flag is indistinguishable from correct behaviour at this boundary |

Three honest conclusions:

1. **A fixture test cannot catch a producer-side staleness bug, by construction.** The fixture *is* the assertion. If the recorded payload carries `enabled=true`, every test built on it agrees. There is no assertion a client-side test could have made here without already knowing the answer.
2. **The one client-side test that would have helped is a cross-field invariant, not a decode test.** Assert that `enabled` is never true where `capable` is false, where live `status` is `DISABLED`, or on a WAN port — a contradiction check inside a single payload, which does not depend on knowing which value is right. That is cheap and should be added to both platforms.
3. **This is exactly where provider verification earns its cost.** Two cloud representations of the same fact disagreed, and nothing anywhere asserted they should agree. A provider-verified contract stating "`/connections.port_security.enabled` is true only for a port that is `capable`, not WAN, and reporting live MACsec status" would have failed in **cloud** CI — which is also where the fix has to land.

That third point is the bridge to the companion guide, and CORE-33235 is the strongest argument in this document for it: the bug presented as a client rendering defect, was reported as a client defect, and is actually a producer emitting two contradictory answers to the same question.

---

## Proposed Solution

### Phase 1: Reference implementation — DONE

Merged on both platforms, running in the standard unit suite on every PR, zero new dependencies:

- **Android** — PR #13181, merged 2026-07-30, commit `dd9f38f4`. 12 `CT-*` tests in `app/src/test/kotlin/com/eero/android/contract/portsecurity/PortSecurityContractTest.kt` plus fixtures in `PortSecurityFixtures.kt`; 3 `INT-001..003` tests in `PortDetailViewModelTest.kt`
- **iOS** — PR #14285, merged 2026-08-03, commit `74221e2c`. 12 `@Test` scenarios in `Eero/EeroNetworking/Tests/Port Security Tests/PortSecurityContractTests.swift`; 3 `INT-001..003` tests in `PortDetailsTests.swift`

**Follow-up from CORE-33235:** add the cross-field contradiction test described above — `enabled` is never true where `capable` is false, where live `status` is `DISABLED`, or on a WAN port. The reference implementation is a good template and an incomplete one. Note that this only makes the client refuse to render a bad flag; **the flag itself is a cloud-side fix** and is tracked separately.

### Phase 2: Extend to the top interfaces (2–4 weeks)

Reprioritized by this window's P0/P1 volume:

1. **Multi-WAN / WAN links** — 19 bugs, 9 P0. `active` flag versus default route (CORE-32917, CORE-33148, CORE-33341), `/config` omitting `wan` (CORE-33044), orphaned connections on failure (CORE-33304, CORE-33305), `name` optionality across cloud and Apollo (CORE-32502)
2. **Speedtest provider overrides** — 6 contract defects this window alone. Nullability (CORE-32415), existence validation (CORE-32631), layer resolution (CORE-32295), dirty-check (CORE-32296), provider enum (CORE-33092), firmware-key normalization (CORE-33094)
3. **Capability tables over the hw-model enum** — the Dejavu pair (CORE-32903 cloud-side, CORE-33269 client-side) is one root cause surfacing twice. Exhaustiveness tests over the model enum catch the whole class
4. **Write→read round-trips on settings** — SSID/password (CORE-33320, CORE-32463), bandwidth limit (CORE-33188), accessory removal (CORE-32327), addressing mode (CORE-33364), profile resume (CORE-32515)
5. **Cloud error-code and emit-boundary contracts** — gRPC status mapping (CORE-33380), boolean field validation (CORE-32695), non-finite gauges (CORE-33268), DKV warm-up semantics (CORE-33144), redirect status handling (CORE-32964)

**How:** the golden-fixture pattern from the reference implementation for client-side work; provider-side verification for items 1 and 5, which no client-only test can reach. The [implementation guide](./contract-integration-testing-implementation-guide.html) covers both.

### Phase 3: Integration tests (4–8 weeks)

**What:** cross-service state-transition tests — the invariant that a write is visible on the read path, and that a reported status matches physical reality.

**Where to start:** the WAN `active` invariant (CORE-32917, CORE-33148, CORE-33341), setup-success honesty (CORE-33057, CORE-33304), factory→cloud node sync (CORE-33309, CORE-32469), and activation idempotency (CORE-33055).

**How:** against staging or faked API clients, in existing suites, on every PR. The Android `PortDetailViewModel` INT pattern is directly reusable for the client half; the cloud half needs the provider-side work in the guide.

---

## ROI Estimate

| Metric | Current state | With contract/integration tests |
|--------|---------------|--------------------------------|
| Categorized bugs per week | 8.0, flat across three snapshots | Reduced to near-zero for covered interfaces |
| P0/P1 from cross-service failures | 24 of 126 in 6 weeks (19.0%) | Reduced to near-zero for covered interfaces |
| Repeat bugs on already-flagged interfaces | 7 on Speedtest overrides after it was named target #1 | Caught at PR time on the first regression |
| Chronic defects | One 6-year-old fleet-wide policy revert; one 2-year-old pause bug; a same-window sync recurrence | Caught at PR time |
| Time-to-detect | Days to years — Sentry, customer escalation, support cases, factory ASN checks | Minutes (CI failure) |
| Cost of the reference implementation | 12 CT + 3 INT per platform, 0 new dependencies, runs in the existing unit job | — |
| Known limit | Fixture tests cannot catch producer-side staleness by construction — CORE-33235 | Provider verification closes this specific gap; a cross-field contradiction check narrows it on the client |

---

## Next Steps

1. ✅ **Reference implementation merged** — Android #13181 and iOS #14285, gating every PR
2. **Add the CORE-33235 cross-field contradiction test** to both platforms, and **route the cloud-side fix** for the ungated `portSecurityOn` flag — the client test stops the symptom, the cloud fix removes the cause
3. **Pick Multi-WAN as the next reference interface** — 9 P0s and active development make it the highest-value target
4. **Hand the cloud side to Cloud Engineering** — the [implementation guide](./contract-integration-testing-implementation-guide.html) documents both approaches, verified against `eero-inc/cloud` at Scala 2.13.18 / sbt 1.12.1 / Play 2.9.10
5. **Review checkpoint at end of sprint** — compare new PR-time contract-test failures against Sentry events and new CORE tickets on the same interfaces

---

## Appendix: Bug Source Data

All bugs sourced from Jira project **CORE** (Core Engineering), created 2026-07-21 → 2026-08-31, `issuetype = Bug`, 325 issues retrieved via the Jira Cloud JQL search API. Full ticket details at `https://eeroinc.atlassian.net/browse/CORE-XXXXX`.

Each of the 48 categorized bugs was read at description level before inclusion; the "Preventable By" column states a specific assertion, not a general aspiration. Sentry event counts, alert-flap counts, and defect ages are quoted from the linked evidence in each ticket. CORE-32595 has an empty description field and is categorized from its summary and title metadata only.

Priority mix for the window: P0 57 (17.5%) · P1 69 (21.2%) · P2 79 (24.3%) · P3 115 (35.4%) · P4 5 (1.5%).
