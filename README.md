# IT Help Desk Lab: Jira Service Management Cloud (Free)

I built a working IT help desk in a real **Jira Service Management Cloud** site on the Free plan, for a simulated 50-person company with a 3-person IT team. All users, tickets, and data are simulated. I did the configuration in the Jira, Confluence, and Slack UIs, verified every feature with test tickets, and recorded the evidence in [`evidence/`](evidence/).

> **Status:** complete. Configuration, a 20-case UAT (20/20 passed), CSAT, and final reports are all verified.

## At a glance

| Area | What was built |
|---|---|
| Portal | 6 request types in 3 groups; access restricted to added customers; requests private by default |
| Workflow | One status set for all work types: Open → In progress ⇄ Waiting for customer → Resolved → Closed, plus Waiting for approval and Canceled |
| Approvals | Manager approval on hardware and software requests, with a hidden preset approver (separation of duties) |
| SLAs | Time to first response and Time to resolution, priority-based goals, Mon–Fri 9–5 America/New_York calendar |
| Automation | 5 active flows: category routing, an unavailable-agent sweep, customer-reply handling, a High-priority Slack alert, and SLA-breach Slack alerts (2 flows) |
| Integration | Slack incoming webhook to a dedicated test workspace |
| Knowledge base | Confluence space with 5 troubleshooting articles, linked to the portal |
| Reporting | Operations dashboard (filters + gadgets) and native JSM SLA/volume reports |
| Testing | Phase checkpoints plus a **20-case UAT** with negative tests, a defect log, and retests |
| Findings | **25 logged:** 21 fixed and verified, 1 Free-plan limitation, 2 design observations, 1 superseded |

## Contents
1. [Business scenario](#business-scenario)
2. [Environment and people](#environment-and-people)
3. [Portal and request design](#portal-and-request-design)
4. [Workflow](#workflow)
5. [SLAs](#slas)
6. [Automation and routing](#automation-and-routing)
7. [Slack alerts](#slack-alerts)
8. [Knowledge base](#knowledge-base)
9. [Reporting](#reporting)
10. [Testing and UAT](#testing-and-uat)
11. [Findings log](#findings-log)
12. [Subscription limitations](#subscription-limitations)
13. [Stretch goals: CSAT and Assets](#stretch-goals-csat-and-assets)
14. [What I configured and learned](#what-i-configured-and-learned)
15. [Repository layout](#repository-layout)

## Business scenario
- A fictional company of about 50 employees in five departments (Sales, Finance, Operations, HR, Marketing).
- The IT team is 3 agents, matching the Free plan's 3-agent limit; the lab admin is one of them.
- Employees raise requests through a self-service portal. Simulated employees use Gmail plus-addressing (`name+empNN@…`), so one inbox can stand in for many people.
- **Scope:** incidents, service requests, approvals, SLAs, routing, notifications, a knowledge base, and reporting. Asset tracking is evaluated only (it needs a paid plan).

## Environment and people

| Item | Value |
|---|---|
| Product / plan | Jira Service Management Cloud (Service Collection), **Free** |
| Space | IT Help Desk (`ITHD`), company-managed, Basic IT service management template (the full ITSM template requires Premium) |
| Customer access | Restricted: only customers added by IT can raise requests |
| Default sharing | New requests are private to the requester (customers can share them manually) |

| Persona | Type | Role |
|---|---|---|
| Lab admin (Lead) | Agent | Space admin; owns Accounts & access; fallback owner |
| Alex Chen | Agent | Hardware |
| Sam Patel | Agent | Software |
| Maria Lopez (Sales), Kevin Brooks (Operations), Priya Shah (HR) | Customers | Requesters |
| Dana Kim (Finance Manager) | Customer | Approver |

- **License vs. role:** an agent **license** (product access in Atlassian Administration) is separate from **space membership** (the Service Desk Team role). Licensed agents couldn't see the space until they were added to the role.
- Agents have no administrator access (least privilege).
- Departments are modeled as JSM **organizations**.

## Portal and request design
Request types are what employees see; work types are how IT processes them.

| Portal group | Request type | Work type | Form notes |
|---|---|---|---|
| Fix a problem | Report a hardware problem | Incident | Priority question with defined levels |
| Fix a problem | Report a software problem | Incident | Priority question with defined levels |
| Fix a problem | Something else / not sure (listed last) | Service request | Catch-all that lands in triage |
| Request something | Request new hardware | Service request with approvals | Required business justification; approver preset and hidden |
| Request something | Request software installation | Service request with approvals | Required business justification; approver preset and hidden |
| Accounts & access | Account access or password help | Service request | "Which system?" + "What's happening?" |

- **Priority design:** ITIL derives priority from impact × urgency. On Free, the incident forms ask one plain-language question ("How badly is this affecting your work?") with defined levels, and agents validate priority during triage. Other request types default to Medium. A matrix automation was left out to save the automation allowance.
- Template request types the lab doesn't need were **hidden, not deleted**, so the change can be rolled back.
- The default comment type is *internal note*, so agents must deliberately choose *Reply to customer*.

## Workflow
All work types use one set of statuses (standardized through the workflow scheme):

```
[approval types only] Waiting for approval --Approved--> Open
                                           --Declined--> Resolved (resolution: Declined)
Open -> In progress -> Resolved -> Closed      (Closed is final: no transitions out)
Open / In progress -> Waiting for customer --customer reply (automation)--> Open
Resolved --customer reply (automation) or agent--> In progress   (reopen; resolution cleared)
Any active status -> Canceled (resolution: Canceled) -> Closed
```

- **Resolutions:** Done, Won't Do, Duplicate, Declined, Canceled. No default is set, so a resolution is never applied silently.
- A ticket counts as "done" because the **Resolution field** is set, not because of the status name. Queues and reports filter on `resolution`, not on status lists.
- *Canceled* is kept as an extra status so that withdrawn requests stay separate from fixed ones in reports.

**Queues**

| Queue | JQL |
|---|---|
| Unassigned – triage | `resolution = Unresolved AND assignee IS EMPTY AND status != "Waiting for approval"` |
| My open tickets | `assignee = currentUser() AND resolution = Unresolved` |
| High priority | `priority in (Highest, High) AND resolution = Unresolved` |
| Waiting for customer | `status = "Waiting for customer"` |
| Waiting for approval | `status = "Waiting for approval"` |
| Recently resolved (7 days) | `resolution != Unresolved AND resolved >= -7d` |

## SLAs
**Calendar:** *NY Business Hours*, Mon–Fri 09:00–17:00 America/New_York (1 business day = 8 business hours). SLA math uses the calendar's time zone.

| Priority | Time to first response | Time to resolution |
|---|---|---|
| Highest / High | 1 business hour | 8 business hours |
| Medium (default) | 4 business hours | 16 business hours (2 days) |
| Low / Lowest | 8 business hours | 24 business hours (3 days) |
| Anything else (catch-all) | 4 business hours | 16 business hours |

| | Time to first response | Time to resolution |
|---|---|---|
| **Start** | Work item created | Work item created, or *Resolution: Cleared* (a reopen starts a new cycle) |
| **Pause** | Waiting for customer, Waiting for approval | Waiting for customer, Waiting for approval |
| **Stop** | *Comment: For Customers* (an agent's **public** reply; internal notes don't count), or *Resolution: Set* | *Resolution: Set* |
| **Cycles** | Once per ticket | A new cycle per reopen, each with the full goal |

**How the timers behave**
- A public agent reply stops first response; an internal note doesn't (verified).
- *Waiting for customer* pauses resolution. The customer's reply moves the ticket back to Open (automation), and the due time moves later by the paused time (verified: 13:28 → 13:47 after a 19-minute wait).
- *Waiting for approval* pauses both timers, so the manager's decision time isn't charged to IT.
- Escalating priority applies the tighter goal **measured from creation time**.
- A decline or cancel sets a resolution, so those tickets count as SLA *met*.
- Outside business hours, the timers show as paused.

**Testing method:** the timer logic (S1–S9) was tested on temporary short goals on a 24/7 calendar, and then the real goals were restored and verified. The calendar itself was verified separately: a ticket created on Saturday started its clock Monday 09:00. Live breaches on real goals were also tested in UAT (T09: first response breached exactly 1 business hour after creation, with one Slack alert).

## Automation and routing

| Flow | Trigger | What it does | Loop / duplicate protection |
|---|---|---|---|
| Route new requests by category | Work item created | Hardware → Alex; Software → Sam; Access → Lead; anything else stays unassigned (triage) | Only runs on create |
| Routing: unavailable-agent sweep (hourly) | Scheduled, JQL `project = ITHD AND resolution = Unresolved AND assignee is not EMPTY AND assignee not in membersOf("it-agents-available")` | Clears the assignee and adds an internal note, so the ticket goes to triage | Changes only tickets that match |
| Customer reply: resume waiting request or reopen resolved request | Comment added | Waiting for customer → Open; Resolved (and not Declined) → In progress | Ignores comments from the Service Desk Team and Administrators roles; a declined request can't be reopened by commenting |
| Slack: alert on High-priority tickets | Priority changed (create or edit) | Posts to Slack and adds the label `slack-high-notified` | JQL `priority in (Highest, High) AND (labels IS EMPTY OR labels != slack-high-notified)` |
| Slack: SLA breach – first response / resolution | SLA threshold breached | Posts to Slack | The trigger fires once per breach |

- **Unavailable agents:** marking an agent out of office means removing them from the group `it-agents-available`; no rule edit is needed. Their open tickets move to triage within an hour.
- **Why a scheduled sweep:** an event-time check proved unreliable. JQL conditions read Jira's search index, which lags a brand-new assignment by a few seconds (F-25). I diagnosed this with audit logs, a split-rule experiment, and a manual JQL search that proved the logic correct.
- Every automation fix was verified in the **automation audit log**.

## Slack alerts
Alerts go to `#it-helpdesk-alerts` in a dedicated, free Slack test workspace, through an incoming webhook (Slack app *JSM Alerts*). **The webhook URL is treated as a secret**: it isn't in this repo or in any screenshot.

Verified behavior:
- **A new High ticket** sends exactly one 🚨 alert. Editing the ticket, adding notes, or flipping priority sends no duplicate.
- **A Low ticket escalated to High** sends one 🚨 alert, at the moment of escalation.
- **A live SLA breach** sends one ⚠️ alert per breach, for first response and resolution separately.
- A breach created retroactively by changing goals doesn't fire the trigger. Breaches crossed in real time do. This only affects testing.

## Knowledge base
- **Space:** *IT Help Desk Knowledge Base* (Confluence Free), linked to the portal as **External** knowledge. It's a dedicated space containing only customer-facing articles, because an External link exposes every page in the space (template sample pages were removed).
- **Articles** (sources in [`kb/`](kb/)), each with Symptoms → Steps → Verify it worked → When to contact IT, and each pointing to the right portal request type:
  1. How to reset your password or unlock your account
  2. Printer not working: quick fixes
  3. VPN won't connect: troubleshooting
  4. How to request software installation
  5. Computer running slow: what to try first
- **Agents:** *Related articles* on tickets suggests the right article; a VPN ticket ranked the VPN article first. Agents share articles in replies.
- **Customers (Free-plan limitation, F-20):** for customers to read articles, the knowledge base's *Who can view* must be *All logged-in users*. That needs the Confluence global permission *JSM access → Use Confluence*, which is **locked on Confluence Free**. The portal search as a customer returned no articles. Customer self-service needs **Confluence Standard or above**; it wasn't trialed, to keep the lab at $0.

## Reporting
**Dashboard *IT Help Desk – Operations*.** Its gadgets read from saved filters: *ITHD – Open requests* (`project = ITHD AND resolution = Unresolved`) and *ITHD – All requests* (`project = ITHD`).

| Metric | Tool | Why this tool |
|---|---|---|
| Ticket volume | Created vs. Resolved gadget, plus the native report | Both work; the gadget sits on the dashboard |
| Open by priority | Pie chart gadget (Priority) | A live snapshot; the native *Incidents by priority* report covers incidents only |
| Open by category | Pie chart gadget (Work type) | The gadget didn't offer *Request Type*; per-request-type trends need a native custom report with JQL series |
| Workload by agent | Pie chart gadget (Assignee), plus the native *Workload* report | Live open work per agent |
| SLA compliance | Native **SLA met vs breached** and **SLA success rate** reports | Only native JSM reports understand SLA goals, pauses, and cycles |
| Resolution time | Native **Time to resolution** report | The *Resolution Time* gadget measures whole days (same-day tickets show 0) |

Reports available on Free: Workload, Satisfaction, Requests deflected, Requests resolved, Created vs Resolved, Time to resolution, SLA met vs breached, Incidents by priority, SLA success rate, and Service requests.

Known reporting caveats:
- Declined and canceled requests count as SLA *met*.
- Private-by-default sharing leaves the Organization field empty, so department breakdowns don't work for new tickets (F-24).

**Final results after UAT** (past 7 days, all 39 tickets; this includes the configuration test tickets ITHD-1 to ITHD-15, so it's lab data rather than a performance claim). Screenshots: `evidence/final/`.

| Report | Result |
|---|---|
| Created vs Resolved | 39 created / 39 resolved; the backlog was cleared at the end of UAT |
| SLA met vs breached | Resolution: 37 met / 1 breached (the deliberate S6 breach). First response: 35 met / 2 breached (S6 and UAT T09) |
| SLA success rate | First response 100.0% / resolution 97.4% |
| Workload | 0 items in progress per agent |

Reporting observations:
- **The two SLA reports disagree on first response:** 2 breaches vs 100%. They count differently (by completion date and cycle), so the numbers have to be reconciled before anyone quotes them.
- **The Workload report lists integration accounts** (Slack, Automation for Jira, and others) as "agents". Filter to the 3 real agents when reading it.
- With the backlog cleared, the dashboard's open-ticket gadgets show *No Data Available*. Their populated state is in `evidence/phase6/`.

## Testing and UAT
Testing happened in two layers:
1. **Phase checkpoints** during the build:
   - workflow (8 tests)
   - SLA timers (S1–S9)
   - routing (R1–R5)
   - Slack (N1–N3, S6)
   - knowledge base
2. **UAT:** the full plan and results are in [`uat/UAT-matrix.md`](uat/UAT-matrix.md). It ran on two business days with the real SLA goals, 20 test cases, and 4 customer personas. **Result: 20/20 passed**, 16 on the first run and 4 after a fix or retest.

| UAT result | Tests |
|---|---|
| Passed on first run | T01, T04, T05, T06, T07, T09, T10, T11, T12, T13, T15, T16*, T17, T18, T19, T20 |
| Failed, fixed, retested, passed | T02 (F-21), T08 (F-23), T14 (F-25) |
| Step skipped on first run, retested, passed | T03 (with design observation F-22) |

\*T16 passed for agents; customer article access is blocked by the documented Free-plan limitation (F-20).

**Negative tests included:**
- unauthorized access (a customer opening an agent URL is redirected to the portal)
- self-approval (the requester has no approve control)
- Closed is final
- declined requests can't be reopened by commenting
- duplicate Slack notifications
- cross-department privacy (customers can't search other departments)

## Findings log
Every finding from the build and from UAT, in ID order.

| ID | Found in | Finding | Outcome |
|---|---|---|---|
| F-01 | Phase 1 | No SLA timers: the template created SLA fields but no goals | Fixed: two SLAs on the business-hours calendar |
| F-02 | Phase 1 | Tickets default to Medium, but the SLA spec covered only High and Low | Fixed: Medium goals plus a catch-all row |
| F-03 | Phase 1 | Customer organization named after a person, not a department | Fixed: five department organizations |
| F-04 | Phase 1 | Portal intro text showed the old space name | Fixed |
| F-05 | Phase 1 | Template statuses didn't match the target workflow | Superseded by F-08 to F-10 |
| F-06 | Phase 2 | Hardware and software requests skipped approval (wrong work type) | Fixed: new request types on the approval work type, approver preset and hidden |
| F-07 | Phase 2 | New requests were shared with the requester's whole department by default | Fixed: private by default |
| F-08 | Phase 2 | Inconsistent status names across workflows | Fixed: one status set through the workflow scheme; *Waiting for support* replaced by *Open* |
| F-09 | Phase 2 | Incidents had no *Waiting for customer*, so incident SLAs could never pause | Fixed: incidents moved to the standard workflow |
| F-10 | Phase 2 | Regression: *In progress* had no transition to *Waiting for customer* | Fixed: added a *Respond to customer* transition |
| F-11 | Phase 2 | A customer reply didn't move the ticket out of *Waiting for customer* (nothing triggered the transition) | Fixed: customer-reply automation |
| F-12 | Phase 2 | The site had **no resolution values**, so tickets couldn't be properly resolved | Fixed: five resolutions, no default |
| F-13 | Phase 2 | A customer could reopen a manager's **declined** request just by commenting | Fixed: the reopen branch excludes Declined |
| F-14 | Phase 3 | The first-response SLA stopped on *customer* comments instead of agent public replies | Fixed in configuration review |
| F-15 | Phase 3 | The Low resolution goal was on the 24/7 calendar | Fixed in configuration review |
| F-16 | Phase 4 | The account-access form still used "request a new account" wording | Fixed: relabeled form fields |
| F-17 | Phase 4 | Routing: the availability check read stale data | Fixed: re-fetch (later superseded by F-25) |
| F-18 | Phase 4 | The automation *User condition* didn't detect group membership | Fixed: JQL condition (later superseded by F-25) |
| F-19 | Phase 4 | Duplicate Slack alert: the dedupe label value was invalid (`ADD: slack-high-notified`) | Fixed and retested |
| F-20 | Phase 5 | Customers can't read KB articles: Confluence Free locks *JSM access* | **Plan limitation**, documented |
| F-21 | UAT T02 | The software form asked *Urgency* (template field), not *Priority*, so High requests weren't prioritized or alerted | Fixed and retested |
| F-22 | UAT T03 | Any customer comment on a Resolved ticket (even "Thanks!") reopens it and starts a new SLA cycle | **Design observation**; possible improvement: auto-close Resolved tickets after N days |
| F-23 | UAT T08 | *Cancel request* set the resolution to Won't Do instead of Canceled | Fixed and retested |
| F-24 | UAT T13 | Private-by-default sharing leaves Organization empty, so department reporting doesn't work | **Design observation**; possible fix: an automation that copies the reporter's organization |
| F-25 | UAT T14 | The unavailable-agent fallback failed intermittently: JQL conditions read a search index that lags new assignments | Fixed: hourly scheduled sweep; retested |

## Subscription limitations
Verified against Atlassian documentation and in the site itself:
- **3 agents maximum** on JSM Free.
- People can be added to the built-in space roles, but **role permissions, permission schemes, and issue security can't be customized**. Agents also see the option to create spaces.
- **Automation** runs on a monthly step allowance, so test runs and scheduled flows have to be budgeted.
- **100 outbound emails per day**, which is why UAT was split over two days.
- **Assets** isn't included on Free.
- **Confluence Free** locks global permissions, so portal customers can't read KB articles (F-20).
- Paid features seen in the UI but not used: Journeys, Playbooks, AI agents, the full ITSM template.

## Stretch goals: CSAT and Assets
**Customer satisfaction (CSAT): supported on Free and verified.**
- **Setting:** *Space settings → Request management → Customer satisfaction → Collect satisfaction (CSAT) feedback* is **on** (the default for new service spaces). The survey asks "How was our service for this request?" on a 5-point scale from Very poor to Very good.
- **Test:** resolving ITHD-39 (UAT T19) sent the requester the *resolved* email at 16:33 with the embedded 5-star survey. Results appear in the native **Satisfaction** report.
- **Notes:**
  - The classic survey fires when a ticket reaches a *Done*-category status, so declined and canceled requests also get a survey.
  - Atlassian's newer automation-driven **Surveys** feature is paid-only and wasn't used.

Evidence: `evidence/csat/`

**Hardware asset tracking (evaluation only):**
- **What it would add:** an asset record (for example `LAP-0042`, a laptop assigned to Maria) linked to the *Affected hardware* field on hardware tickets, so agents can see an asset's ticket history and warranty.
- **Plan requirement:** native **Assets** isn't available on JSM Free. Atlassian's pricing lists it on paid plans (sources conflict on whether Standard or Premium is the minimum, so confirm before buying). Third-party Marketplace asset apps are also paid.
- **Free-plan alternative used in this lab:** the free-text *Affected hardware* field on the hardware form (computer name or asset tag). This is searchable with JQL but has no asset records or relationships.
- **Recommendation:** for a real 50-person company, use Assets on a paid tier, or a lightweight asset register (for example a spreadsheet keyed by asset tag) until the help desk outgrows it.

## What I configured and learned
**Configured, all by hand in the UI:**
- the space, portal, request types, and forms
- organizations and roles
- the workflow scheme, transitions, and resolutions
- queues
- 2 SLAs and a business-hours calendar
- 5 automation flows
- a Slack webhook integration
- a Confluence knowledge base with 5 articles
- a dashboard and reports
- a 20-case UAT

**Lessons I'd bring to a help desk role:**
- **The resolution field defines "done"**, not the status name. A missing resolution list (F-12) silently breaks reports and SLAs.
- **A public reply and a status change are separate actions.** Only public replies count as a first response, and only status changes pause timers.
- **Automation has to be verified, not assumed.** Audit logs exposed stale data (F-17), a bad label value (F-19), and index timing (F-25). Testing the same JQL by hand in search isolated the real cause.
- **Templates hide defects.** Leftover fields, labels, and transitions from templates caused F-16, F-21, and F-23.
- **Loop and duplicate protection is part of the design:** role conditions, label stamps, and exclusions for declined requests.
- **Privacy settings have reporting costs**, and those trade-offs should be documented (F-24).
- **Know the plan's limits** before promising a feature (F-20, Assets).

**Portfolio wording** (no unmeasured outcome claims):
- Built a Jira Service Management help desk lab for a simulated 50-person company: 6 request types, manager approvals, department organizations, and a restricted self-service portal.
- Configured business-hours SLAs (first response and resolution, priority-based) with pause, stop, and reopen rules, and validated them with live breach tests and Slack breach alerts.
- Built Jira automation for category routing, an unavailable-agent fallback, customer-reply handling, and de-duplicated Slack alerts, troubleshooting failures through audit logs.
- Planned and executed a 20-case UAT including negative tests. Logged 25 findings; fixed and retested every confirmed defect, and documented the plan limitations.

## Repository layout
```
README.md            this document
uat/UAT-matrix.md    20-case UAT plan, results, defect log
kb/                  source of the 5 knowledge base articles
evidence/phase1-6/   build-phase screenshots
evidence/uat/        UAT screenshots
evidence/final/      post-UAT dashboard and reports
evidence/csat/       customer satisfaction survey setting and email
```
