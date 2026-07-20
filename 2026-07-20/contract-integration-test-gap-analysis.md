# Proposal: Contract & Integration Tests in CI

**Author:** Maria Rebelo (QAE, Core Engineering)
**Date:** 2026-07-20 (refresh of 2026-06-26 snapshot)
**Audience:** Engineering Manager, Core Engineering

---

## Executive Summary

In the last ~3 weeks (2026-06-27 → 2026-07-20), CORE filed **167 bugs**: **21 P0 blockers**, **32 P1 criticals**, **55 P2 majors**, **56 P3 minors**, and 3 P4s. The June proposal is now backed by a second month of data showing the same failure patterns — API contract violations, cross-service state drift, and iOS/Android parity gaps — this time including two new customer-facing sign-up outages and a MACSec toggle regression that reintroduces the exact class of bug the PoC branches were built to prevent.

This document refreshes the June evidence with July's bugs, and updates the proof-of-concept status: the Android PoC (`maria/QA-17148`) now has an opt-in `test:contract` CI job wired up and 3 real integration tests inline in `PortDetailViewModelTest`. The iOS PoC (`maria/QA-17147`) still holds 18 `@Test` scenarios (14 contract + 4 malformed-payload + 4 toggle/poll) — but the branch is 23K commits behind main and has never been PR'd. Concrete asks are at the end.

---

## The Problem: Bugs That Slip Through Unit Tests

Our current testing pyramid relies heavily on:
- Unit tests (logic correctness within a single module)
- Manual QA / E2E tests (full user journeys, slow and late in the cycle)

**The gap:** No automated validation that services agree on data formats, required fields, response codes, and state transitions at their boundaries.

---

## Evidence: Real Bugs from CORE (2026-06-27 → 2026-07-20)

### Category 1: API Contract Violations

These bugs occur when one service produces data that another service cannot consume correctly.

| Ticket | Priority | Summary | Preventable By |
|--------|----------|---------|----------------|
| CORE-31676 | P1 | Speedtest Edit override 409 mishandled — "Unknown error" toast, no refetch, every subsequent save also 409s | Contract test: client parses 409 conflict payload, shows conflict toast, refetches the version |
| CORE-32133 | P2 | GET /2.2/speedtest/overrides response `metadata.organizations` returns `{id}` only — `name` never populated | Contract test: overrides response schema declares `organizations[].name` as required |
| CORE-31936 | P3 | AmazonCloudLink `Left(e)` bypasses status-code match — 400 body still logs ERROR + throws (~56k events) | Contract test: every declared 4xx status maps to WARN-level regardless of body-decode branch (Left/Right) |
| CORE-31728 | P3 | FutureRetry re-throws Akka timer `IllegalStateException` at shutdown instead of the ORIGINAL error (60,911 events since Oct 2024) | Contract test: FutureRetry surfaces the original exception when scheduling fails, not the akka timer error |
| CORE-31862 | P3 | NodeSwitchConfigController logs ERROR on empty `configId` — 23,968 events since 2026-04-14 | Contract test: `ConfigStatusReport` proto requires non-empty configId; empty payload is WARN-level, not ERROR |
| CORE-31800 | P3 | `network_admins.role` stores 19 distinct non-enum values (owner, admin, viewer…) → all degrade to InvalidNetworkRole | Contract test: writers to `network_admins.role` must supply one of {network-owner, temporary-admin, standard-admin} |
| CORE-32137 | P3 | hardwaredata MAC-collision returns opaque 500 instead of 409 on `idx_eeros_mac_address` unique-violation | Contract test: factory-registration MAC conflict returns 409 with a typed body, not 500 |
| CORE-31867 | P3 | `EventStreamProcessor.createView` unsafe `Option.get` on BaseEvent field — one bad event crashes the batch | Contract test: BaseEvent renderer tolerates any optional field being empty (getOrElse, not .get) |

### Category 2: Data Sync / State Mismatch Between Systems

These bugs occur when two systems hold contradictory state about the same entity.

| Ticket | Priority | Summary | Preventable By |
|--------|----------|---------|----------------|
| CORE-31743 | P0 | MACSec Port Security toggle first render shows disabled while shield icon shows enabled — flips after back navigation | Integration test: port-security toggle reflects device's true port state on first render, not after re-entry |
| CORE-32132 | P0 | Node ACKs `throttle_backup_bandwidth` (speed=0, wmob0 disconnected) but LTE data keeps flowing past 110GB cap (>200GB observed) | Integration test: after throttle action + node ACK, wmob0/ECM stays disconnected until the action is cleared |
| CORE-31999 | P0 | 800+ Retrograde 5G networks stuck unregistered → LTE backup never provisioned, "No Retrograde" outcome for profile-switching | Integration test: RG-eligible networks reach registered=true before the backup-availability check reads their state |
| CORE-32155 | P0 | Thread toggle hidden in iOS/Android/Insight for Firefly ALC even though `otbr-agent` is active on the device | Integration test: device-reported otbr capability surfaces the Thread toggle across all clients |
| CORE-32150 | P0 | `node_update_outcomes` missing every `.wrx` firmware OTA record — Looker OTA history blank for wormhole/xenia/wormholepoe/xeniapoe | Integration test: firmware records with `.wrx` suffix flow end-to-end from update event → outcomes table → Looker |
| CORE-32049 | P1 | App/Cloud reports leaf Kilimanjaro on 24/7 LTE backup — `ring_settings.lte_state='on'` stale, node isn't on LTE | Integration test: `ring_settings.lte_state` stays synchronized with device-reported LTE mode within N seconds |
| CORE-32221 | P1 | Non-SamKnows speedtests (Eero_UDP, MLab, provider-less) shipped to SamKnows-owned S3 bucket after retry-path removal | Integration test: only speedtests where provider ∈ {SamKnows, SamKnows_UDP} are written to the SamKnows S3 logger |
| CORE-31871 | P3 | Backup AP reorder API returns 200 but `backup_ap` book order unchanged in DB 2s later | Integration test: after reorder POST, subsequent GET returns the requested order within N seconds |

### Category 3: Interface Parity / Schema Drift Between Platforms

These bugs occur when iOS/Android/Web disagree on how to interpret shared API data.

| Ticket | Priority | Summary | Preventable By |
|--------|----------|---------|----------------|
| CORE-32109 | P0 | Static-IP gateway replacement fails on Android (node never gets WAN IP, drops to setup) — iOS works on same eeroOS | Integration test: static-IP gateway replacement completes end-to-end on both iOS and Android against the same eeroOS build |
| CORE-31876 | P1 | Android runs placement test for an ethernet-connected leaf; iOS correctly skips it | Contract test: placement-test skip predicate derived from a shared API field and honored on iOS and Android |
| CORE-31750 | P1 | iOS "Connected to" shows the leaf's own port; Android shows the gateway port the leaf is wired to | Contract test: `connected_to.port` semantic is fixed in the API spec and asserted identically by iOS and Android |
| CORE-32024 | P1 | Notification set for the same event stream disagrees between iOS and Android | Contract test: notification-category → alert-string map is a single shared source read by both platforms |
| CORE-32156 | P1 | iOS "Scans/Ads" detail screen missing Profiles tab + percentage that AOS renders from same data | Contract test: Scans/Ads detail view renders same required fields (profiles, percentage) on iOS and AOS |
| CORE-31458 | P3 | Android renders idle devices as "Idle"; iOS renders "↓0 kbps ↑0 kbps" — same API values | Contract test: idle-device throughput rendering is defined in the shared schema (zero-values format identically) |
| CORE-32164 | P3 | PoE row label reads "PoE allocation" on one platform, "PoE usage" on the other — same underlying value | Contract test: PoE row label maps to a single shared localization key across iOS and Android |
| CORE-32048 | P3 | Crane Port details after disabling PoE — iOS shows "PoE disabled", Android shows "Power disabled" | Contract test: post-toggle PoE label uses a shared localization key across iOS and Android |

---

## Impact Assessment

### Scale (2026-06-27 → 2026-07-20, CORE only)

- **167 total bugs**: 21 P0 · 32 P1 · 55 P2 · 56 P3 · 3 P4
- **~25% preventable overall** by contract or integration tests
- **~24% of the P0+P1 slice** (13 of 53) map cleanly to contract or integration failures — and every P0 in this refresh with a cross-service cause is in that group

**Mix shift vs the June snapshot:** June's overall preventable share was ~40%. The July batch is heavier on pure UI polish (missing icons, misaligned dividers, single-platform copy issues), agentic-code-review / Reviewbot / Drones tooling regressions, and CI/K8s infra ops — none of which contract or integration tests can meaningfully cover. **The P0/P1 slice, however, is still dominated by cross-service failures**: state sync (throttle, RG registration, Thread capability, LTE state), platform parity (static-IP replacement, placement test, alert-set), and consumer-side contract mishandling (409 conflict, provider routing). Nothing in the underlying failure mix has improved — it's the volume of low-severity polish work that grew.

### Chronic Offenders (still firing)

| Ticket | Events | Active Since | Note |
|--------|--------|--------------|------|
| CORE-31728 | 60,911 | Oct 2024 | Akka scheduler shutdown race in FutureRetry — masks the real error that triggered the retry |
| CORE-31936 | ~56,000 | Jul 2026 | Regression from the CORE-30606 fix — ~28k events on each of two Sentry issues within 8 days of release |
| CORE-31862 | 23,968 | Apr 2026 | Proto change on 2026-04-13 causes nodes to post empty `configId`; every such payload is a Sentry ERROR |
| CORE-31800 | ongoing | Jul 2026 | 19 non-enum role names stored at rest; each GET /users read silently degrades to InvalidNetworkRole |
| CORE-31697 | recurring | Jun 2026 | DynamoDB throughput/hot-key exceptions on captiveportal access-control table since eeroOS 6.24 |

Two of these (CORE-31728, CORE-31862) alone represent >84,000 Sentry events over their lifetimes. Same story as June: **systemic interface mismatches that persist because nothing validates the contract between producer and consumer.**

### Customer Impact (P0/P1 highlights)

- **CORE-32013 (P0):** New Icelandic users (+3546 MSISDN) can't sign up — Twilio SMS verification never delivered; ~20 blocked cases per day, escalating
- **CORE-32020 (P1):** Xtrim Ecuador (Claro) subscribers can't verify their phone — no OTP SMS, accounts stuck unverified
- **CORE-31999 (P0):** 800+ Retrograde 5G networks never provide LTE backup — cloud registration never completed; dogfood hitting "No Retrograde"
- **CORE-32132 (P0):** Ripple Fiber customers exceed the 110 GB LTE cap (top offenders >150 GB) — throttle ACKed by the node but not enforced on wmob0
- **CORE-32253 (P1):** Customers can't disable eero Hotspot / cellular backup — toggle reverts to ON immediately after switching off
- **CORE-32109 (P0):** Android customers can't complete static-IP gateway replacement — new Wormhole never obtains WAN IP; iOS works fine on the same eeroOS

---

## Proposed Solution

### Phase 1: Merge and expand the existing PoC (immediate)

The Android PoC branch already has:
- 12 `CT-*` contract tests in `app/src/test/kotlin/com/eero/android/contract/portsecurity/PortSecurityContractTest.kt` (CT-001..003, CT-006..014 — CT-004/005 intentionally omitted, see case study)
- 3 real INT-001..003 integration tests inline in `PortDetailViewModelTest.kt`
- An opt-in `test:contract` CI job with diff-scoped selection (commit `1a5d323`)
- Draft PR #13181 open on eero-inc/android (last commit 2026-07-10)

The iOS PoC branch has 18 `@Test` scenarios (14 CT + 4 malformed) covering the same tech-spec matrix, but is 23K commits behind main and has never been PR'd. Rebase + open PR is the first ask.

### Phase 2: Contract Tests (2-4 weeks)

**What:** Schema-level tests that validate:
- Request/response field types, required fields, allowed values
- Error code mappings (4xx vs 5xx for client errors)
- Enum completeness (no `IllegalArgumentException` on valid values)
- Key format consistency (read matches write)

**Where to start (top 5 target interfaces, prioritized by P0/P1 volume in this refresh):**
1. **Speedtest overrides API** — schema + 409 handling + provider routing (CORE-32133, CORE-31676, CORE-32221)
2. **MACSec / Port Security state sync** — toggle mismatch, cross-network banner leakage, phantom blocked (CORE-31743 and the June-baseline CORE-31519 / CORE-31321)
3. **Retrograde 5G / LTE-backup pipeline** — RG registration, throttle enforcement on wmob0, `ring_settings.lte_state` freshness (CORE-31999, CORE-32132, CORE-32049)
4. **iOS ↔ Android setup + detail parity** — static-IP replacement, placement-test skip, "Connected to" semantics, live-data idle rendering, alert-set (CORE-32109, CORE-31876, CORE-31750, CORE-32024, CORE-31458)
5. **Scala Cloud error-code / enum contracts** — AmazonCloudLink Left(e), FutureRetry shutdown, NodeSwitchConfig empty configId, network_admins.role, hardwaredata 500→409 (CORE-31936, CORE-31728, CORE-31862, CORE-31800, CORE-32137)

**How:** Same golden-fixture pattern as the PoC — client-side contract tests using the production decoder. No Pact, no new infrastructure. See the [MACsec case study](./macsec-contract-test-case-study.html) for the working template.

### Phase 3: Integration Tests (4-8 weeks)

**What:** Cross-service data flow tests validating state transitions complete end-to-end, and UI state reflects backend state after mutations.

**Where to start:** The three P0 state-sync flows above (throttle, RG registration, Thread capability) plus the DARS / SSID / port-toggle flows from the June snapshot. The Android `PortDetailViewModel` INT-001..003 pattern (fake API client + emitted state assertion) is directly reusable.

**How:** Run against staging or against faked API clients, in existing test suites, in CI on every PR.

---

## ROI Estimate

| Metric | Current State | With Contract/Integration Tests |
|--------|---------------|-------------------------------|
| Chronic Sentry bugs (>1 month old) | 5+ bugs with >1,000 events each — two >20K events, one >60K events | Caught at PR time |
| P0/P1 bugs from cross-service failures | 13 of 53 in the last 3 weeks (~24%) | Reduced to near-zero for covered interfaces |
| Time-to-detect | Days to months (Sentry alerts, customer complaints, dogfood escalation) | Minutes (CI failure) |
| Dev cost of late-detected bugs | High (debugging prod, hotfixes, dogfood rollbacks) | Low (failing test shows exact contract) |
| Existing PoC | 30 contract tests + 3 real integration tests, 0 CI dependencies, ready to merge | — |

---

## Next Steps

1. **Rebase and open PR for iOS PoC** (`maria/QA-17147-port-security-tests`) — 23K commits behind, needs a merge before it's reviewable
2. **Land Android PR #13181** — `test:contract` job is opt-in and diff-scoped; zero blast radius on the default pipeline
3. **1 QAE sprint** — replicate the pattern for the top-5 interfaces above (fixtures + assertions), same golden-fixture approach as the PoC
4. **Weekly fixture refresh** — small CI job that re-records fixtures from staging, alerts on schema drift
5. **Review checkpoint** — at end of sprint, compare new PR-time contract-test failures against Sentry events on the same interfaces

---

## Appendix: Bug Source Data

All bugs sourced from Jira project **CORE** (Core Engineering), 2026-06-27 → 2026-07-20. Full ticket details at `https://eeroinc.atlassian.net/browse/CORE-XXXXX`. Sentry event counts referenced from linked Sentry issues in ticket descriptions.

**Not counted as preventable (~127 of 167):** UI polish (missing icons, alignment, blinking toggles, single-platform copy), agentic-code-review / Reviewbot / Drones tooling regressions, CI/K8s infra ops (stage-eerosign HPA, gitlab-runner unready, K8s panic mode, EBSByteBalance), and single-service runtime crashes with no cross-service linkage (Compose NoSuchMethodException, PopupLayout window-manager, structured-logging interpolation, captiveportal DynamoDB throughput).
