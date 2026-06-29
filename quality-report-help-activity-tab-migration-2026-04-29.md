# Quality Report: Help and Activity Tab Migration (Milestone 3)

**Date**: 2026-04-29 | **Release**: 26.6 | **Epics**: [CORE-27097](https://eeroinc.atlassian.net/browse/CORE-27097) (Android), [CORE-27096](https://eeroinc.atlassian.net/browse/CORE-27096) (iOS)

---

## Feature Overview

Migration of the Activity tab to a Help tab with troubleshooting and contact support sections, plus UI fixes including Security & Privacy relocation, data usage improvements, and icon/color theming updates across Android and iOS.

---

## QA Sign-off

| Scope | Verdict | Justification |
|-------|---------|---------------|
| Android | 🔴 NO-GO | 8 P0/P1 bugs remain open; 5 P0 stories are still in PR; 2 Defects are currently in QA validation |
| iOS | 🔴 NO-GO | 2 P0 bugs are in QA validation; 1 P0 and 1 P1 are in Dev Completed awaiting QA; P0 stories remain in PR |

### Blocking Issues

| Key | Summary | Severity | Scope | Status |
|-----|---------|----------|-------|--------|
| CORE-29213 | Scans metrics aggregation diverging between platforms | P1 | Android | Investigating |
| CORE-28964 | Dark mode theming of Troubleshooting icons mismatched | P0 | Android | Dev Completed |
| CORE-29060 | Help Tab Icons color update needed | P0 | Android | In Pull Request |
| CORE-29210 | UI issues on Colors screen | P1 | Android | QA In Progress |
| CORE-29064 | Info alert banner incorrect color scheme (T-mobile) | P1 | Android | QA In Progress |
| CORE-29125 | Software update alert banner misplaced | P0 | iOS | QA In Progress |
| CORE-29123 | Data usage in Unassigned profile inaccurate | P0 | iOS | QA In Progress |
| CORE-29215 | eero Plus upsell prompted incorrectly (T-mobile) | P0 | iOS | Dev Completed |
| CORE-29033 | Help tab options menu wrong labels for ISP networks | P1 | iOS | Dev Completed |

---

## Development Status

| Metric | Android (46 issues) | iOS (38 issues) |
|--------|--------------------:|----------------:|
| Done | 24 (52%) | 22 (58%) |
| In Progress | 13 (28%) | 6 (16%) |
| To Do | 4 (9%) | 7 (18%) |
| Won't Do | 5 (11%) | 3 (8%) |
| Open P0/P1 Bugs | 8 | 4 |

---

## Risks

| Risk | Severity | Impact |
|------|----------|--------|
| Scans metrics diverge between Android and iOS | High | Users see different numbers per platform |
| Data usage inaccurate for Unassigned profile (iOS) | High | Wrong data shown to users |
| No TestRail suite — coverage unknown | High | Cannot verify regression coverage |
| Test automation 40% complete (2/5 screens covered) | Medium | Limited automated regression for release validation |
| Help tab options menu broken for ISP networks | Medium | ISP users get wrong labels + broken nav |
| eero Plus upsell shown to T-mobile users incorrectly | Medium | Unnecessary purchase prompts |
| T-mobile/ISP flows under-tested (3 bugs found) | Medium | Potential escapes post-launch |

---

## QA Resources

| Resource | Link |
|----------|------|
| Test Plan | [Google Doc](https://docs.google.com/document/d/1dlnoFn0OQVItR0ZH8_Xohl-BinojCJVXaKTslKEmplQ/edit?tab=t.0) |
| Test Automation | [CORE-28779](https://eeroinc.atlassian.net/browse/CORE-28779) (40% complete — 2/5 screens) |
| TestRail Suite | Pending |
| Jira (Android) | [CORE-27097](https://eeroinc.atlassian.net/browse/CORE-27097) |
| Jira (iOS) | [CORE-27096](https://eeroinc.atlassian.net/browse/CORE-27096) |

---

## Conditions for Sign-off

**Before sign-off can proceed:**
- 4 P0 bugs are currently in QA validation (CORE-29125, CORE-29123, CORE-29210, CORE-29064) — these need verified fixes to clear the NO-GO.
- 2 P0 bugs are in Dev Completed (CORE-28964, CORE-29215) and have not yet entered QA.
- 5 P0 stories remain in Pull Request and are not yet available for testing.
- The platform divergence in scans metrics (CORE-29213) is still under investigation.

**Outstanding items not blocking, but carrying risk into release:**
- 2 P1 bugs (CORE-29641, CORE-29097) are in Backlog with no scheduled fix.
- A TestRail suite does not yet exist for this feature — regression coverage cannot be confirmed.
- T-mobile and ISP-specific flows have surfaced 3 bugs, suggesting limited test coverage in those areas.

**Observations for future milestones:**
- Android has approximately 2x the bug density of iOS, indicating the Android migration may need additional stabilization.
- Test automation covers 2 of 5 screens (Internet, Troubleshooting). Profile screen is in PR, Home screen is in progress, and Device details has not started.
- Legacy screens and feature flags from the migration are still present in the codebase.
