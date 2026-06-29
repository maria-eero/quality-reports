# Quality Report: Help Tab and UI Fixes (Milestone 3)

**Date**: 2026-04-29
**Author**: Kiro (Lead QA Engineer agent)
**Scope**: CORE-27097 (Android), CORE-27096 (iOS)

## Feature Overview
**Feature**: Help Tab and UI Fixes — Milestone 3
**Description**: Migration of the Activity tab to a Help tab with troubleshooting and contact support sections, plus various UI fixes across the app including Security & Privacy section relocation and data usage improvements.
**Scope**: Android and iOS mobile platforms
**Epics**: [CORE-27097](https://eeroinc.atlassian.net/browse/CORE-27097) (Android), [CORE-27096](https://eeroinc.atlassian.net/browse/CORE-27096) (iOS)

## Timeline
| Item | Detail |
|------|--------|
| Target Release | TBD (not specified) |
| Target Date | TBD |
| Project Tracking | [Tech Spec](https://docs.google.com/document/d/1*3Gvi2Q4CL5ligY6g7Tklfbj3jaNCtDYn084kmRNJOQ/edit?usp=sharing) |

## Executive Summary

Both epics are currently "In Execution" with significant progress — Android is 64% complete and iOS is 68% complete (excluding Won't Do). However, multiple P0/P1 bugs remain open on both platforms, including critical issues in QA validation. The feature is **not ready for release** on either platform due to active P0 blockers still in development or QA.

## QA Sign-off

| Scope | Verdict | Justification |
|-------|---------|---------------|
| Android | 🔴 NO-GO | 4 open P0/P1 bugs not yet resolved; 5 P0 stories still in PR; 2 P1 Defects in QA validation |
| iOS | 🔴 NO-GO | 2 P0 bugs in QA validation; 1 P0 bug in Dev Completed awaiting QA; P0 stories still in PR |

### Blocking Issues (NO-GO areas)

| Key | Summary | Severity | Scope | Status |
|-----|---------|----------|-------|--------|
| CORE-29641 | Tapping info icon on data usage redirects back to client details | P1 | Android | Backlog |
| CORE-29097 | Missing eero plus section cards from Unassigned profile screen | P1 | Android (Cloud) | Backlog |
| CORE-29213 | Scans metrics aggregation for network diverging from iOS | P1 | Android | Investigating |
| CORE-29210 | UI issues on Colors screen | P1 | Android | QA In Progress |
| CORE-29064 | Info alert banner incorrect color scheme | P1 | Android (T-mobile) | QA In Progress |
| CORE-28964 | Dark mode theming of Troubleshooting icons mismatched | P0 | Android | Dev Completed |
| CORE-29060 | Update Color For Help Tab Icons | P0 | Android | In Pull Request |
| CORE-29125 | Software update alert banner misplaced on Internet page | P0 | iOS | QA In Progress |
| CORE-29123 | Data usage in Unassigned profile not showing accurate data | P0 | iOS | QA In Progress |
| CORE-29215 | eero Plus upsell prompted on Usage screen via Profile entry point | P0 | iOS (T-mobile) | Dev Completed |
| CORE-29033 | Help tab more options menu shows wrong labels for ISP networks | P1 | iOS | Dev Completed |

## QA Resources
| Resource | Link |
|----------|------|
| Tech Spec | [Google Doc](https://docs.google.com/document/d/1*3Gvi2Q4CL5ligY6g7Tklfbj3jaNCtDYn084kmRNJOQ/edit?usp=sharing) |
| Jira Epic (Android) | [CORE-27097](https://eeroinc.atlassian.net/browse/CORE-27097) |
| Jira Epic (iOS) | [CORE-27096](https://eeroinc.atlassian.net/browse/CORE-27096) |
| TestRail Suite | Not provided |
| Test Plan | Not provided |

## Development Status

### Android (CORE-27097)
| Metric | Value |
|--------|-------|
| Total issues | 46 |
| Done | 24 (52%) |
| Won't Do | 5 (11%) |
| In Progress (Dev Completed / In PR / QA) | 13 (28%) |
| To Do (Backlog / Investigating) | 4 (9%) |
| Bugs/Defects (total) | 28 |
| Bugs/Defects (open, excl. Won't Do) | 14 |
| Open P0/P1 bugs | 8 |

### iOS (CORE-27096)
| Metric | Value |
|--------|-------|
| Total issues | 38 |
| Done | 22 (58%) |
| Won't Do | 3 (8%) |
| In Progress (Dev Completed / In PR / QA / Dev In Progress) | 10 (26%) |
| To Do (Backlog) | 7 (18%) |
| Bugs/Defects (total) | 19 |
| Bugs/Defects (open, excl. Won't Do) | 5 |
| Open P0/P1 bugs | 4 |

## Test Coverage Summary
| Metric | Value |
|--------|-------|
| Total test cases | Not available (TestRail suite not provided) |
| Automated | N/A |
| Manual | N/A |

*Note: No TestRail data was provided for this trial run. In a real execution, test coverage metrics would be populated from the TestRail suite.*

## Risk Analysis
| Dimension | Risk | Details |
|-----------|------|---------|
| Coverage gaps | High | No TestRail data available; unable to verify test coverage for migrated screens |
| Code churn | Medium | Multiple UI fixes and migrations happening simultaneously across both platforms |
| Bug density | High | 28 bugs on Android, 19 on iOS — high density for a UI migration feature |
| Dependency risk | Medium | T-mobile/ISP-specific behaviors require separate validation paths |
| Platform risk | High | Android and iOS have different progress levels and different blocking issues; parity not achieved |
| Automation debt | Unknown | No TestRail data to assess automation status |

## Post-Launch Risks
| Risk | Severity | User Impact | Workaround | Tracking |
|------|----------|-------------|------------|----------|
| Scans metrics aggregation diverging between platforms | High | Users see different security scan numbers on Android vs iOS | None | CORE-29213 |
| Data usage in Unassigned profile inaccurate (iOS) | High | Users see wrong data usage for unassigned devices | None | CORE-29123 |
| Help tab options menu wrong labels for ISP networks | Medium | ISP users see incorrect menu options, broken navigation | None | CORE-29033 |
| eero Plus upsell incorrectly shown to T-mobile users | Medium | T-mobile users prompted to buy eero Plus they don't need | None | CORE-29215 |
| Info icon navigation broken on data usage screen (Android) | Medium | Users tapping info get redirected to wrong screen | None | CORE-29641 |

## Known Issues

| Key | Summary | Severity | Status | Platform |
|-----|---------|----------|--------|----------|
| CORE-29641 | Tapping info icon redirects to client details | P1 | Backlog | Android |
| CORE-29097 | Missing eero plus section cards from Unassigned profile | P1 | Backlog | Android |
| CORE-29213 | Scans metrics aggregation diverging from iOS | P1 | Investigating | Android |
| CORE-29210 | UI issues on Colors screen | P1 | QA In Progress | Android |
| CORE-29064 | Info alert banner incorrect color scheme (T-mobile) | P1 | QA In Progress | Android |
| CORE-29593 | Help "?" icon too thick | P1 | Dev Completed | Android |
| CORE-28805 | Speed test history screen title UI issues | P1 | Dev Completed | Android |
| CORE-28964 | Dark mode theming of Troubleshooting icons mismatched | P0 | Dev Completed | Android |
| CORE-29060 | Help Tab Icons color update needed | P0 | In Pull Request | Android |
| CORE-29540 | 3 dots support icon size mismatch | P3 | In Pull Request | Android |
| CORE-29281 | Adding test tags for the whole app | P3 | Dev Completed | Android |
| CORE-29068 | Fix icons on snapshot tests | P3 | Dev Completed | Android |
| CORE-29125 | Software update alert banner misplaced | P0 | QA In Progress | iOS |
| CORE-29123 | Data usage in Unassigned profile inaccurate | P0 | QA In Progress | iOS |
| CORE-29215 | eero Plus upsell prompted incorrectly (T-mobile) | P0 | Dev Completed | iOS |
| CORE-29033 | Help tab options menu wrong labels for ISP networks | P1 | Dev Completed | iOS |

## Recommendations

### Immediate (before release)
- Resolve all P0 blockers currently in QA validation (CORE-29125, CORE-29123, CORE-29210, CORE-29064)
- Complete QA validation on Dev Completed P0 bugs (CORE-28964, CORE-29215)
- Investigate and resolve the scans metrics divergence between platforms (CORE-29213)
- Merge pending P0 stories still in PR (caching, security & privacy section, device screen updates)

### Short-term (next sprint)
- Address P1 bugs in Backlog (CORE-29641, CORE-29097) — these are currently unscheduled
- Fix Help tab options menu for ISP networks (CORE-29033)
- Validate T-mobile-specific flows end-to-end on both platforms
- Provide TestRail suite link for proper coverage tracking

### Medium-term (next quarter)
- Achieve platform parity — Android has significantly more bugs than iOS, suggesting the Android migration needs more stabilization
- Complete backlog cleanup tasks (legacy screen removal, feature flag deprecation)
- Establish automated regression suite for the migrated Help tab flows

## Appendix

### Uncovered Areas
- TestRail coverage data not available for this report
- No GitHub repo provided — code churn analysis not performed
- T-mobile/ISP-specific flows appear under-tested based on bug patterns

### Won't Do Issues (excluded from metrics)
- **Android**: CORE-27108, CORE-27331, CORE-27330, CORE-28968, CORE-28952, CORE-28930
- **iOS**: CORE-28951, CORE-28959, CORE-28967
