# Sprint Readiness Audit — CNM - Sprint 26.7

**Project**: CORE | **Sprint**: CNM - Sprint 26.7 | **Total Tickets**: 90 | **Audit Date**: 2026-06-10

---

## 🏥 Sprint Health: 🔴 RED

51% of tickets lack QA assignment — this single metric exceeds the >25% critical gap threshold. Combined with 18% missing story points and 2 unassigned P1s, the sprint is not trackable, not fully testable, and has unowned critical work 18 hours before close.

| Critical Gap | % | Threshold Breach |
|---|---|---|
| Missing QA assignee | **51%** | >25% ✗ |
| Missing components | **70%** | >25% ✗ |
| Missing story points | **18%** | 10–25% (Yellow alone) |
| Unassigned P1s | 2 tickets | Binary risk — unacceptable |

---

## Gap Summary

| Gap Type | Count | % of Sprint | Severity |
|----------|------:|------------:|----------|
| Missing Story Points | 16 | 18% | 🔴 Critical |
| Missing Description | 4 | 4% | 🔴 Critical |
| Empty QA Assignment | 46 | 51% | 🟡 High |
| Empty Bug Reason | 26 | 29% (of bugs) | 🟡 High |
| Empty Bug Source | 19 | 21% (of bugs) | 🟡 High |
| No Components | 63 | 70% | 🟠 Medium |
| Unassigned | 8 | 9% | 🟠 Medium |

---

## 📋 Program Manager Assessment

### Scope Clarity Score: 73%

66 of 90 tickets have both a description and story points. The remaining 24 (27%) are either blind spots or container-level items. However, the 4 tickets with no description AND no points represent true zero-visibility work that will stall or surprise the team.

### Delivery Risk

**Blind spots** (no owner + no points + no description):
- CORE-30832 (P1, In PR, no assignee, no description) — this is actively in development with zero traceability
- CORE-30079 and CORE-30075 (Epics, no description) — container items that can't be validated for completion criteria

These are work that **will surprise the team** because nobody can confirm what "done" means.

### Process Gap Pattern

The dominant pattern is **QA assignment neglect at scale** — 51% is not a one-off; this is a filing discipline issue. The team does not enforce QA assignment as part of sprint commitment. Secondary pattern: Bug Reason is consistently empty on L10n and customer-reported bugs (26 of 40 bugs), indicating reporters don't have a clear Bug Reason taxonomy or it's not required in the workflow.

**Root cause:** No enforced Definition of Ready. Tickets enter the sprint without meeting minimum field requirements.

---

## 🧪 QA Readiness Assessment

### QA Coverage Risk: 51% Unassigned — CRITICAL

46 tickets have no QA assignee. Features most at risk:
- **iOS MACsec Port Security** (7 stories, 0 QA) — the sprint's largest feature initiative ships untested unless QA is assigned today
- **All L10n/i18n bugs** (12+ bugs, 0 QA) — localization bugs that went through Dev Completed with no QA verification planned
- **Cloud/Backend subtasks** (8 tickets, 0 QA) — event stream refactoring with no test coverage
- **Password Strength Enforcement** (initiative + epic, 0 QA) — regulatory feature with no QA

**QA load imbalance**: Maria Rebelo carries ~69% of all QA-assigned tickets (20+ of 29 assigned). This is a single-point-of-failure risk.

### Bug Metadata Health: 65% of bugs missing Bug Reason

- 26 of 40 bugs (65%) lack Bug Reason — escape analysis for this sprint is **impossible**
- 19 of 40 bugs (48%) lack Bug Source — cannot determine if bugs escaped from specific test phases
- **Impact**: Sprint defect dashboards will show incomplete data. Any "defects by root cause" report presented to leadership will be inaccurate. Escape analysis for release 26.7 is blocked.

### Testability Flags — 4 Untestable Tickets

| Key | Summary | Status | Why Untestable |
|-----|---------|--------|----------------|
| [CORE-30754](https://eeroinc.atlassian.net/browse/CORE-30754) | Wrong Icon color for T-mobile | In Pull Request | No description — cannot verify fix |
| [CORE-30832](https://eeroinc.atlassian.net/browse/CORE-30832) | Adding request on screen opening | In Pull Request | No description, no assignee — zero traceability |
| [CORE-30079](https://eeroinc.atlassian.net/browse/CORE-30079) | Android UI Gate for Unsupported Versions | In Execution | Epic with no description — cannot validate scope |
| [CORE-30075](https://eeroinc.atlassian.net/browse/CORE-30075) | iOS UI Gate for Unsupported Versions | In Execution | Epic with no description — cannot validate scope |

---

## 🔴 Critical Gaps — Immediate Action Required

### Missing Story Points (16 tickets)

| Key | Summary | Type | Assignee |
|-----|---------|------|----------|
| [CORE-29679](https://eeroinc.atlassian.net/browse/CORE-29679) | Home screen loading twice | Bug | Devika Shendkar |
| [CORE-28262](https://eeroinc.atlassian.net/browse/CORE-28262) | Non-descriptive placeholder name | Bug | Jon Criswell |
| [CORE-27217](https://eeroinc.atlassian.net/browse/CORE-27217) | "Last restart date" showing inaccurate timestamp | Bug | Jean Sandrin |
| [CORE-30281](https://eeroinc.atlassian.net/browse/CORE-30281) | Echo built-in bouncing between profiles | Bug | Conor Giambona |
| [CORE-29246](https://eeroinc.atlassian.net/browse/CORE-29246) | Color of status light partially localized | Bug | Unassigned |
| [CORE-30212](https://eeroinc.atlassian.net/browse/CORE-30212) | "Connection" tab missing in gateway node | Bug | Saumya Sinha |
| [CORE-30405](https://eeroinc.atlassian.net/browse/CORE-30405) | Port information missing in device info tab | Bug | Jean Sandrin |
| [CORE-29874](https://eeroinc.atlassian.net/browse/CORE-29874) | Crash when accessing scans screen | Bug | Osvaldo Jr |
| [CORE-29911](https://eeroinc.atlassian.net/browse/CORE-29911) | Security & Privacy section shifts | Bug | Devika Shendkar |
| [CORE-29457](https://eeroinc.atlassian.net/browse/CORE-29457) | App crashing on eero Device Detail page | Bug | Devika Shendkar |
| [CORE-27762](https://eeroinc.atlassian.net/browse/CORE-27762) | Cannot enable Die Hard via Local Mode | Bug | Luke McClure |
| [CORE-30506](https://eeroinc.atlassian.net/browse/CORE-30506) | Password Strength Enforcement (BR, SG) | Epic | Nathan Wang |
| [CORE-30508](https://eeroinc.atlassian.net/browse/CORE-30508) | Password Strength Enforcement (Initiative) | Initiative | Nathan Wang |
| [CORE-30078](https://eeroinc.atlassian.net/browse/CORE-30078) | UI-Screen Blocker for unsupported versions | Initiative | Rexell Kurniawan |
| [CORE-29760](https://eeroinc.atlassian.net/browse/CORE-29760) | Update toast design | Story | Luke McClure |
| [CORE-28914](https://eeroinc.atlassian.net/browse/CORE-28914) | iOS Align Test Tags with QA Spec | Story | Devika Shendkar |

### Missing Description (4 tickets)

| Key | Summary | Type | Assignee |
|-----|---------|------|----------|
| [CORE-30754](https://eeroinc.atlassian.net/browse/CORE-30754) | Wrong Icon color for T-mobile | Bug | Osvaldo Jr |
| [CORE-30832](https://eeroinc.atlassian.net/browse/CORE-30832) | Adding request on screen opening | Bug | Unassigned |
| [CORE-30079](https://eeroinc.atlassian.net/browse/CORE-30079) | Android Add UI Gate for Unsupported Versions | Epic | Rexell Kurniawan |
| [CORE-30075](https://eeroinc.atlassian.net/browse/CORE-30075) | iOS Add UI Gate for Unsupported Versions | Epic | Rexell Kurniawan |

---

## 🟡 High Gaps — Address Within 24h

### Empty QA Assignment (46 tickets)

The majority of tickets (51%) have no QA assignee. Notable patterns:
- **All iOS MACsec stories** (CORE-29534 through CORE-29539, CORE-30440) — no QA assigned
- **All L10n bugs** — no QA assigned
- **Cloud/Backend subtasks** (CORE-30067 through CORE-30679) — all unassigned
- **New feature epics** (CORE-30506, CORE-30079, CORE-30075) — no QA coverage planned

### Empty Bug Metadata (26 tickets missing Bug Reason)

| Pattern | Count | Examples |
|---------|-------|----------|
| Bugs with neither Bug Reason nor Bug Source | 19 | CORE-30754, CORE-30832, CORE-30281, CORE-30202 |
| Bugs with Bug Source but no Bug Reason | 7 | CORE-29679, CORE-29286, CORE-29277 |

---

## 🟠 Medium Gaps

- **No Components**: 63 tickets (70%) — affects routing and reporting
- **Unassigned**: 8 tickets including CORE-30832 (P1), CORE-29246 (P1), and 6 sub-tasks (CORE-30679, CORE-30678, CORE-30671, CORE-30070, CORE-30069, CORE-30068)

---

## ✅ Decisions & Actions — SDM Action Plan

> Do today, in this order. Sprint closes tomorrow.

| # | Action | Owner | Deadline | What It Unblocks |
|---|--------|-------|----------|-----------------|
| **1** | **Assign or defer the 2 unassigned P1s** (CORE-30832, CORE-29246). Confirm scope, assign owner, or pull from sprint. | SDM | Next 2 hours | Prevents fire drill at sprint close; removes highest-severity delivery risk |
| **2** | **QA triage of 46 unassigned tickets.** Bucket into: (a) assign QA now, (b) accept risk with Tech Lead sign-off, (c) tag for post-release smoke. | QA Lead + SDM | End of day | Unblocks test execution; creates explicit risk register for anything shipping untested |
| **3** | **Bulk-update bug metadata (26 Bug Reason, 19 Bug Source).** Run a 30-min session with dev leads to populate fields on all 40 bugs. | Dev Leads | End of day | Unlocks sprint defect reports, escape analysis, and retro data |
| **4** | **Remove or block the 4 description-less tickets.** Mark untestable, return to PO. Do not count as Done. | Product Owner | Next 3 hours | Stops false "Done" inflation; prevents untested scope from shipping |
| **5** | **Point the 16 unpointed tickets or flag as untrackable.** Quick-size with dev leads (T-shirt → points) or annotate as excluded from velocity. | Scrum Master + Dev Leads | End of day | Restores sprint commitment accuracy for closeout reporting |

### Capacity Impact

16 unpointed tickets (18%) corrupt commitment accuracy. If average ticket = 3 pts, ~48 story points are invisible to the burndown — likely 15–20% of total committed capacity untracked. **Velocity this sprint is unreportable without footnoting the unpointed tickets.** Quick-size them today — imperfect data beats no data for planning Sprint 26.8.

---

## Process Fix (Post-Sprint — Flag for Retro)

- **No enforced Definition of Ready** — tickets enter sprint without owner, points, description, QA assignee, component
- **Bug transition gate needed** — Bug Reason + Bug Source required before moving to Done
- **QA load balancing** — 69% concentration on one engineer (Maria Rebelo) is a single-point-of-failure risk

---

*Generated by Sprint Readiness Audit (3-Agent Pipeline: PM III + QA Lead + SDM III) | 2026-06-10*
