# Proposal: Contract & Integration Tests in CI

**Author:** Maria Rebelo (QAE, Core Engineering)
**Date:** 2026-06-26
**Audience:** Engineering Manager, Core Engineering

---

## Executive Summary

In the last 30 days alone, CORE has filed **30 P0 blockers** and **23 P1 critical** bugs. A significant portion of these stem from interface contract violations and integration gaps between services — failure modes that unit tests cannot catch and that manual QA only discovers after deployment.

This proposal presents data from real CORE Jira bugs demonstrating that **contract tests** (API schema/response validation) and **integration tests** (cross-service data flow verification) would have caught these issues before they reached staging or production.

---

## The Problem: Bugs That Slip Through Unit Tests

Our current testing pyramid relies heavily on:
- Unit tests (logic correctness within a single module)
- Manual QA / E2E tests (full user journeys, slow and late in the cycle)

**The gap:** No automated validation that services agree on data formats, required fields, response codes, and state transitions at their boundaries.

---

## Evidence: Real Bugs from CORE (June 2026)

### Category 1: API Contract Violations

These bugs occur when one service produces data that another service cannot consume correctly.

| Ticket | Priority | Summary | Preventable By |
|--------|----------|---------|----------------|
| CORE-31617 | P2 | Rate-Limiting Configuration Machine ID Read/Write Mismatch — IDs are inserted with one format but queried with another prefix convention | Contract test: assert read/write key formats match |
| CORE-31612 | P3 | Empty `eimTranId` written to DynamoDB GSI key → DynamoDbException on writes | Contract test: validate non-empty GSI keys before write |
| CORE-31564 | P3 | Null `baseMac` fails ValidationException in RRM data ingest — 2,004 Sentry events | Contract test: assert upstream always provides non-null baseMac |
| CORE-31449 | P3 | MissingRequiredAcGroup client error returns 500 instead of 400 — 80 Sentry events since April | Contract test: verify error responses map to correct HTTP status codes |
| CORE-31436 | P3 | `SubnetTypeId.fromSubnetType` throws on GENERIC — 266 events since July 2025 | Contract test: enum mapping covers all valid subnet types |
| CORE-31269 | P3 | `IllegalStateException: Unexpected hw model None` — 247,773 events since Oct 2024 | Contract test: device provisioning API guarantees non-None hw model |
| CORE-31568 | P3 | `None.get` in UserEeroActivationController when `organizationId` is missing | Contract test: activation endpoint requires organization context |
| CORE-31527 | P3 | Invalid client MAC crashes Kafka consumer — malformed MAC already on topic | Contract test: validate MAC format at producer before publishing |

### Category 2: Data Sync / State Mismatch Between Systems

These bugs occur when two systems hold contradictory state about the same entity.

| Ticket | Priority | Summary | Preventable By |
|--------|----------|---------|----------------|
| CORE-31596 | P1 | DARS Lambda never writes Status attribute — requests stuck at PENDING forever, dedup logic resurfaces dead requests | Integration test: verify status transitions from PENDING → COMPLETED/FAILED |
| CORE-31519 | P0 | Wormhole: UI shows port enabled, switch has it disabled (state out of sync) | Integration test: assert UI state matches device state after toggle |
| CORE-31270 | P3 | Subnet SSID config mismatch — password in network_subnets doesn't match SSID Config table — 121,355 events | Integration test: verify SSID config consistency across tables |
| CORE-31608 | P2 | Thread credential sync not always working across multiple networks | Integration test: verify credential propagation after permission grant |
| CORE-31321 | P0 | MACSec blocked state not reflected in app — banner shows stale data | Integration test: assert blocked-state change propagates to client model |

### Category 3: Interface Parity / Schema Drift Between Platforms

These bugs occur when platforms (iOS/Android/Web) disagree on how to interpret shared API data.

| Ticket | Priority | Summary | Preventable By |
|--------|----------|---------|----------------|
| CORE-31458 | P3 | Android shows "null" for idle devices, iOS shows "0 kbps" — same API, different interpretation | Contract test: shared response schema with explicit null handling |
| CORE-31457 | P2 | Content Filter naming: iOS uses "Pre-K/Teen", Android uses "Most Restrictive" — T-Mobile app is consistent | Contract test: localization keys match across platforms |
| CORE-31592 | P2 | JSON Dumps page fires API without required query params → 400 for every user | Contract test: page knows required params from API spec |
| CORE-31560 | P1 | Speedtest overrides GET returns 500 for repurposed gateways — frontend renders empty | Integration test: verify API response for all gateway provenance types |
| CORE-31306 | P2 | Provider override dropped in gRPC path — comparison tests run wrong provider | Integration test: verify override propagation through gRPC |

---

## Impact Assessment

### Scale (last 30 days, CORE only)
- **30 P0 blockers**, **23 P1 criticals**, **77 P2 majors** filed
- Of the bugs I analyzed in detail: **~40% are preventable** by contract or integration tests

### Chronic Offenders (still firing)
| Ticket | Events | Active Since |
|--------|--------|--------------|
| CORE-31269 | 247,773 | Oct 2024 (20 months!) |
| CORE-31270 | 121,355 | Jan 2026 |
| CORE-31565 | 4,673 | Active |
| CORE-31564 | 2,004 | Active |
| CORE-31436 | 266 | Jul 2025 (12 months) |

These are not transient bugs — they represent **systemic interface mismatches** that persist because nothing validates the contract between producer and consumer.

### Customer Impact
- **CORE-31596 (P1):** Self-service DSAR requests stuck at PENDING — user privacy requests never complete
- **CORE-31519 (P0):** Port physically disabled but UI says enabled — network admin makes decisions on wrong info
- **CORE-31321 (P0):** Security feature (MACSec) state not accurately reflected — security posture mismatch
- **CORE-31617 (P2):** Rate-limiting configurations silently ineffective — service lacks intended protection

---

## Proposed Solution

### Phase 1: Contract Tests (Quick Win, 2-4 weeks)

**What:** Schema-level tests that validate:
- Request/response field types, required fields, and allowed values
- Error code mappings (4xx vs 5xx for client errors)
- Enum completeness (no `IllegalArgumentException` on valid values)
- Key format consistency (read matches write)

**Where to start:** The Sentry-sourced bugs above directly point to the exact code locations and contracts that need tests.

**How:** Consumer-driven contract tests (e.g., Pact, or lightweight JSON schema validators) run in CI on every PR.

### Phase 2: Integration Tests (4-8 weeks)

**What:** Cross-service data flow tests that validate:
- State transitions complete end-to-end (PENDING → COMPLETED)
- Data written by service A is readable by service B in expected format
- UI state reflects backend state after mutations

**Where to start:** The DARS status flow (CORE-31596), the port enable/disable sync (CORE-31519), and SSID config consistency (CORE-31270).

**How:** Run against staging environment in CI, using lightweight fixtures.

---

## ROI Estimate

| Metric | Current State | With Contract/Integration Tests |
|--------|---------------|-------------------------------|
| Chronic Sentry bugs (>1 month old) | 5+ bugs with >1,000 events each | Caught at PR time |
| P0/P1 bugs from data mismatch | ~5-8 per month | Reduced to near-zero for covered interfaces |
| Time-to-detect | Days to months (Sentry alerts or manual QA) | Minutes (CI failure) |
| Dev cost of late-detected bugs | High (debugging prod, hotfixes, rollbacks) | Low (failing test shows exact contract) |

---

## Next Steps

1. **Prioritize top 5 interfaces** from the bugs above (DARS status flow, subnet config sync, MAC validation, rate-limit key format, enum mappings)
2. **Spike: 1 week** — write contract tests for CORE-31617 (read/write mismatch) and CORE-31436 (enum coverage) as proof of concept
3. **Present results** — demonstrate how the existing bugs would have been caught
4. **Expand** — add contract tests to CI template for all new service interfaces

---

## Appendix: Bug Source Data

All bugs sourced from Jira project **CORE** (Core Engineering), June 2026. Full ticket details available at `https://eeroinc.atlassian.net/browse/CORE-XXXXX`.

Sentry event counts referenced from linked Sentry issues in ticket descriptions.
