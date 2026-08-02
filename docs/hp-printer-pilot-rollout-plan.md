# HP Printer Rollout — End-to-End Pilot Plan (5 Locations)

## Purpose

This plan defines the end-to-end approach for piloting an HP printer rollout across 5 locations before a company-wide (big-bang) deployment. It covers direct/IPP printing under Windows Protected Print Mode (WPP), badge-based secure/pull print, colour vs. black-and-white (B/W) governance, and the fleet management architecture (HP Web Jetadmin on Azure, firewall connectivity, and the HP Insights portal).

A companion architecture diagram input file is provided at `docs/hp-printer-pilot-architecture-input.txt` — paste it into `diagram-generator.html` (in this repo) to generate a `.drawio` diagram of the pilot topology.

## 1. Environment Summary

| Component | Detail |
|---|---|
| Print delivery | Direct print via IPP; Windows 11 clients using **Windows Protected Print Mode (WPP)** — not a traditional driver/print-server queue model |
| Secure/pull print | Badge/card release at the device (HP Access Control-style solution) |
| Fleet management | HP Web Jetadmin (WJA), self-hosted on **Azure**, reaching on-prem printers across a **firewall** |
| Cloud management | **HP Insights portal** (fleet monitoring, analytics, firmware) |
| Print policy | Colour vs. B/W control, tracking, and chargeback |

## 2. Objectives & Scope

- Validate direct IPP printing under WPP on Windows 11.
- Validate badge-release secure print end-to-end (badge → identity lookup → job release → colour/BW policy applied).
- Validate WJA (Azure-hosted) ↔ on-prem printer connectivity through the firewall, and HP Insights cloud manageability.
- Prove colour/BW governance (permissions, cost tracking, default-to-BW) before scaling.
- Produce a go/no-go decision framework for the big-bang rollout.

## 3. Pilot Site Selection Criteria

Select 5 sites that together cover:

1. A site on ExpressRoute/site-to-site VPN to Azure.
2. A site on internet breakout only (no private circuit).
3. A site with higher latency / lower bandwidth to Azure.
4. A high-volume, colour-heavy user population (e.g., marketing/design) alongside a low-volume B/W site.
5. A site with an existing legacy print server (to test coexistence/migration) and one greenfield site.
6. A site with guest/BYOD or hot-desk users (to test badge enrollment for transient users).

## 4. Target Architecture

- **WJA server**: Azure VM, with NSG + Azure Firewall rules scoped to only what's required: SNMP (161/162) for discovery/traps, HTTPS (443) for management UI and HP Insights connector traffic, IPP (631) if WJA proxies any print traffic, plus HP Access Control / badge-auth-server ports.
- **Site connectivity**: printers reach Azure over VPN/ExpressRoute or internet with firewall allow-listing by printer subnet. Document every required inbound/outbound rule explicitly — undocumented firewall gaps are the most common pilot blocker.
- **Client → printer path**: Windows 11 clients print **directly via IPP** to the device (WPP-compliant) — not through a print-server queue. WJA's role is fleet management/monitoring, not the print path itself.
- **HP Insights**: cloud fleet visibility, firmware, and supplies telemetry (device → cloud). Confirm whether it needs firewall allowances separate from WJA's.
- **Badge release**: card reader/accessory at each MFP, integrated with the org's identity system. Confirm where the "hold queue" actually lives (on-device, a release server, or WJA/HP Insights-integrated secure print) — WPP removes the traditional print-server hold-queue model, so this must be validated explicitly, not assumed.

## 5. Scenario Catalogue

### A. Network & Firewall
- WJA → printer reachability per site (discovery, firmware push, config push) across the firewall.
- Firewall rule drift/misconfiguration detection and alerting.
- WAN/Azure path failure: confirm direct IPP printing keeps working locally even if WJA/Azure is unreachable.
- Split-tunnel/VPN-off laptops printing directly to on-prem IPP printers.
- Separate rule sets for WJA management traffic vs. HP Insights cloud telemetry.

### B. Direct/IPP Print & Windows Protected Print Mode
- Confirm every pilot printer model/firmware supports IPP Everywhere/Mopria — WPP requires it; legacy PCL/PS-only drivers will not work once WPP is enabled.
- Test WPP enablement across representative Windows 11 builds.
- **Critical open risk**: WPP restricts third-party print drivers, port monitors, and print processors — many secure-print/badge-release agents historically hook into the print pipeline this way. Explicitly test whether the chosen badge-release solution is WPP-compatible, or whether it needs an IPP/HP Insights-native release mechanism instead.
- Mixed-mode coexistence: some devices on WPP, some not, with a defined rollback path.
- Printer discovery/add-printer self-service experience for end users.
- Mobile/AirPrint/guest device printing to the same fleet.

### C. Secure (Badge/Pull) Print
- Badge release at the "home" printer and at other pilot sites (roaming release).
- Unreleased job expiry and secure deletion.
- New employee/contractor badge enrollment end-to-end.
- Lost/forgotten badge fallback (PIN, mobile release, helpdesk override) with audit logging.
- Badge release when identity/AD lookup is delayed or unavailable.
- Job hold-queue location and latency under WPP's direct-IPP model.

### D. Colour / B/W Governance
- Default-to-B/W enforcement per user/group/site, with an exception process.
- Colour usage tracking/chargeback reconciled across device counters, WJA, and HP Insights.
- Confirm the enforcement point (device-level, IPP attribute-level) still works without legacy drivers under WPP.
- Reporting cadence for colour cost control during the pilot.

### E. Fleet Management (WJA) & HP Insights
- Firmware rollout/rollback via WJA without disrupting production printing.
- Consistent alerting (consumables, faults, patch status) across WJA and HP Insights — avoid two conflicting sources of truth.
- Role-based access for helpdesk/IT in both portals.
- HP Insights licensing/subscription scope validated for pilot device count.

### F. Security & Compliance
- TLS enforced for WJA↔printer and printer↔HP Insights traffic.
- Encryption at rest for held/secure-print jobs; defined retention/purge policy.
- Badge/identity data storage location and privacy compliance.
- Firewall/NSG least-privilege review before scale (close any "make it work" rules opened during build).
- Print audit trail (who printed what, when, released where) meets compliance requirements.

### G. Resilience & Failure Modes
- WJA server outage/restart: confirm printing and badge release are unaffected (direct-IPP path is independent of WJA).
- HP Insights portal unreachable: confirm local management fallback via WJA/device web UI.
- WAN link failure at a site: local printing continues; cloud/WJA visibility degrades gracefully with alerting.
- Printer local queue behavior when Azure connectivity is degraded mid-print.

### H. User Experience & Change Management
- Comms/training for the new badge-release flow and default-B/W behavior.
- Helpdesk runbook for common issues (badge not recognized, job stuck in hold, WPP install prompts).

### I. Migration/Cutover
- Parallel run alongside legacy print servers where applicable.
- Pending-job handling at cutover.
- Per-site rollback plan with a defined trigger/threshold.

## 6. Detailed Test Scenarios

Each test case below should be tracked with: **ID | Scenario | Preconditions/Steps | Expected Result | Priority (P1/P2/P3)**.

### T1 — Network & Firewall (WJA ↔ Azure ↔ Printer)
| ID | Scenario | Priority |
|---|---|---|
| T1.1 | Discover a printer via WJA across the firewall from a cold state (new device, no prior rule) | P1 |
| T1.2 | Push a firmware update from WJA to a printer at each of the 5 sites; confirm success and rollback | P1 |
| T1.3 | Remove a firewall rule for one site; confirm WJA loses visibility but client printing is unaffected | P1 |
| T1.4 | Validate SNMP trap/alert delivery from printer → WJA across the firewall | P2 |
| T1.5 | Validate HP Insights telemetry path independently of the WJA management path | P2 |
| T1.6 | Print from a client on VPN vs. off-VPN/internet-breakout at each site profile | P1 |
| T1.7 | Audit deployed firewall rules against the documented required rule set | P2 |

### T2 — Direct/IPP Print & Windows Protected Print Mode
| ID | Scenario | Priority |
|---|---|---|
| T2.1 | Add printer on a WPP-enabled Windows 11 device via IPP discovery/URL; confirm no legacy driver installed | P1 |
| T2.2 | Print from a non-WPP legacy Windows 11 device to the same printer (coexistence) | P2 |
| T2.3 | Attempt printing with a legacy PCL driver on a WPP-enabled device; confirm it is blocked with a clear error | P2 |
| T2.4 | Print immediately after a printer firmware update | P2 |
| T2.5 | Print from a mobile device (AirPrint/Mopria) | P2 |
| T2.6 | Print with VPN disconnected, printer reachable only on local LAN | P1 |
| T2.7 | Toggle WPP off and confirm rollback to standard driver-based printing without a device rebuild | P1 |
| T2.8 | First-time printer add/discovery with no IT assistance (self-service UX) | P3 |

### T3 — Secure/Badge (Pull) Print
| ID | Scenario | Priority |
|---|---|---|
| T3.1 | Badge release at the "home" printer immediately after submission | P1 |
| T3.2 | Badge release at a different pilot site than where the job was submitted (roaming) | P1 |
| T3.3 | Unreleased job auto-purge/expiry and secure deletion at policy-defined time | P2 |
| T3.4 | New employee/contractor badge enrollment end-to-end, including first release | P1 |
| T3.5 | Lost/forgotten badge fallback (PIN/helpdesk override) with audit log confirmation | P2 |
| T3.6 | Badge release with identity/AD lookup artificially delayed or failing | P1 |
| T3.7 | **Go/no-go gate**: confirm the badge-release agent functions correctly with WPP enabled | P1 |
| T3.8 | Concurrent release stress test — multiple users badging at the same device in quick succession | P2 |
| T3.9 | Colour-restricted user attempts to release a colour job — confirm policy enforcement | P1 |

### T4 — Colour / B/W Governance
| ID | Scenario | Priority |
|---|---|---|
| T4.1 | Standard user submits a colour job; confirm default policy (convert/block/approve) triggers | P1 |
| T4.2 | Exception-list user (e.g., design team) submits a colour job; confirm it prints in colour | P1 |
| T4.3 | Colour usage reconciles across device counters, WJA, and HP Insights (no drift) | P2 |
| T4.4 | Colour/BW enforcement works via IPP attributes/device policy under WPP (no legacy driver) | P1 |
| T4.5 | Chargeback/cost report spot-check against actual device counters for one full pilot week | P2 |

### T5 — WJA / HP Insights Fleet Management
| ID | Scenario | Priority |
|---|---|---|
| T5.1 | Helpdesk role-based access verified in both WJA and HP Insights (least privilege) | P2 |
| T5.2 | WJA server restart/patching — confirm printing and badge release continue uninterrupted | P1 |
| T5.3 | Consumables/fault alerts appear consistently in both WJA and HP Insights | P2 |
| T5.4 | HP Insights licensing/device-count scope matches the actual pilot fleet | P2 |
| T5.5 | Patch/firmware compliance dashboard accuracy after a manual firmware change | P3 |

### T6 — Security & Compliance
| ID | Scenario | Priority |
|---|---|---|
| T6.1 | TLS enforced (no plaintext fallback) for WJA↔printer and printer↔HP Insights | P1 |
| T6.2 | Held/secure print job data encrypted at rest | P1 |
| T6.3 | Badge/identity data storage location and retention meet privacy requirements | P1 |
| T6.4 | End-of-pilot firewall/NSG rule review to catch temporary "make it work" rules | P2 |
| T6.5 | Print audit trail (who/what/when/where) meets compliance/retention requirements | P2 |

### T7 — Resilience & Failure Modes
| ID | Scenario | Priority |
|---|---|---|
| T7.1 | Planned WJA VM outage — confirm direct IPP printing at all 5 sites is unaffected | P1 |
| T7.2 | HP Insights portal connectivity blocked — confirm local device web UI/WJA still manageable | P2 |
| T7.3 | Simulated WAN link failure at one site — local printing continues, cloud visibility degrades gracefully | P1 |
| T7.4 | Printer local queue/storage behavior when Azure connectivity is degraded mid-print | P2 |

### T8 — User Experience & Change Management
| ID | Scenario | Priority |
|---|---|---|
| T8.1 | New user onboarding end-to-end (discovery, enrollment, first secure print) — measure time-to-first-print | P2 |
| T8.2 | Helpdesk dry-run against seeded issues (stuck job, badge not recognized, WPP prompt) — measure resolution time | P2 |
| T8.3 | Structured user feedback survey at each site after 1 week of live use | P3 |

### T9 — Migration/Cutover (where legacy print servers coexist)
| ID | Scenario | Priority |
|---|---|---|
| T9.1 | Parallel run: legacy queue and new direct-IPP path both available, no user confusion/duplication | P2 |
| T9.2 | Cutover rollback drill: revert one site to legacy queue within a defined window | P1 |
| T9.3 | Pending-job handling at the moment of cutover | P2 |

## 7. Pilot Phases & Timeline

1. **Discovery & Design (2–3 wks)** — finalize architecture, firewall rule matrix, badge-release/WPP compatibility confirmation, site sign-off.
2. **Build (2 wks)** — provision WJA rules, configure HP Insights, install/test badge readers, stage WPP policy via Intune/GPO.
3. **Site Readiness (1–2 wks per wave)** — network validation, printer install/firmware baseline, badge enrollment per site.
4. **Pilot Execution (4 wks)** — run the full scenario/test catalogue at each site; daily/weekly issue triage.
5. **Hypercare (2 wks)** — heightened support, KPI monitoring, residual fixes.
6. **Go/No-Go Review** — score against success criteria; produce big-bang readiness report.

## 8. Success Criteria / KPIs

- ≥99% job success rate for direct IPP print across pilot sites.
- Badge release success rate and average release latency within target (e.g., <5s).
- Zero unresolved P1 security findings.
- Colour/BW policy compliance rate and accurate cost reporting reconciliation.
- Declining helpdesk ticket volume by end of hypercare.
- Confirmed WPP compatibility outcome for the badge-release solution — the single biggest technical risk and a hard go/no-go input.

## 9. Risk Register

| Risk | Severity |
|---|---|
| Badge-release/secure-print agent incompatible with Windows Protected Print Mode | High |
| Firewall rule gaps between WJA/Azure and site networks causing inconsistent management visibility | Medium |
| Colour/BW enforcement gap once legacy driver-based restriction is unavailable under WPP | Medium |
| HP Insights and WJA reporting divergence causing confusion in fleet status/chargeback data | Low-Medium |

## 10. RACI

Sponsor / Project Lead / Network & Security team / Identity team / Helpdesk / Site champions / HP TAM or partner (for WPP + badge-release compatibility guidance).

## 11. Communication Plan

Kickoff → pre-cutover notice per site → badge-enrollment scheduling → go-live notice → hypercare feedback channel → go/no-go readout to stakeholders.

## 12. Open Questions to Confirm With Vendor/Stakeholders

- Does the chosen badge-release (HP Access Control or equivalent) solution officially support Windows Protected Print Mode? This should be confirmed directly with HP/the vendor before pilot build, not assumed.
- Where does the secure-print hold queue live once WPP removes the traditional print-server model — on-device, a dedicated release server, or HP Insights-managed?
- Are HP Insights and WJA firewall/port requirements documented separately, and do they differ per printer model/firmware?
