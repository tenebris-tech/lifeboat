# Lifeboat — Requirements

A simple, self-hosted, single-tenant GRC (governance, risk & compliance)
platform.

**Purpose of this document:** state *what the application must do* —
capabilities and rules only. Architecture, technology, and implementation
choices are documented elsewhere.

**Priority key** (RFC 2119 sense):

| Level | Meaning |
|---|---|
| **MUST** | Required for v1. |
| **SHOULD** | Wanted for v1; may slip to a follow-on release if necessary. |
| **MAY** | Optional; a later release. |

**Requirement IDs.** Every requirement has a stable ID so it can be cited
elsewhere (design docs, issues, tests). The prefix tells you the section:

| Prefix | Section |
|---|---|
| `DM-n` | Domain Model (§4) |
| `FR-FW-n` | Functional — Frameworks & Criteria (§5.1) |
| `FR-CT-n` | Functional — Controls (§5.2) |
| `FR-TS-n` | Functional — Tests (§5.3) |
| `FR-EV-n` | Functional — Evidence Requirements & Evidence (§5.4) |
| `FR-ST-n` | Functional — Status, Health & Coverage (§5.5) |
| `FR-PL-n` | Functional — Policies (§5.6) |
| `FR-RK-n` | Functional — Risks (§5.7) |
| `FR-AU-n` | Functional — Users, Authentication & Roles (§5.8) |
| `FR-RP-n` | Functional — Reporting & Audit Trail (§5.9) |
| `FR-IF-n` | Functional — Interfaces (§5.10) |
| `NFR-n` | Non-Functional Requirements (§6) |

---

## 1. Purpose

Lifeboat lets a single organization manage its compliance posture across one
or more frameworks (SOC 2, ISO 27001, GDPR, …). It tracks the framework's
requirements, the organization's controls, the tests that verify those
controls, and the evidence those tests rely on. It also holds the
organization's security policies and a simple risk register, and it reports
readiness and gaps.

It is **single-tenant**: one organization per deployment. There is no tenant
isolation, and access control is correspondingly simple.

---

## 2. Scope

### 2.1 In scope for v1

- Multiple compliance frameworks, with SOC 2 and ISO 27001:2022 shipped out
  of the box.
- The requirement → control → test → evidence chain and its many-to-many
  mappings.
- Evidence freshness (cadence-based collection periods anchored to each
  framework's audit date), assessment (pass/fail), and the resulting
  coverage/readiness and gap reporting.
- A policy repository (governance documents linked to controls).
- A risk register (inherent/residual/effective risk on NIST SP 800-30
  scales, mitigated by mapped controls).
- Authentication with three flat roles.
- Three interfaces to the same capabilities: web UI, CLI, and MCP server.
- CSV export of coverage/status for auditors.

### 2.2 Out of scope for v1

Any of these could be a later release.

- Multi-tenancy and organization isolation.
- SSO, SCIM, OAuth2 authorization server, invitations, complex RBAC.
- Automated or continuous testing; connector-based or AI-driven evidence
  collection.
- AI or natural-language assessment of controls.
- Formal auditor workpapers: sampling, population definition, exception logs.
- Vendor/third-party management, trust center, privacy tooling (DPIA, data
  mapping), asset inventory.
- Quantitative or automated risk scoring; threat modeling; risk treatment
  plans beyond the accepted flag (FR-RK-8).
- Formatted (PDF) reports.
- Notifications and integrations (email, Slack, ticketing) beyond FR-RP-3.

---

## 3. Terminology

| Term | Meaning in Lifeboat |
|---|---|
| **Framework** | A compliance standard (SOC 2, ISO 27001, …). Carries an **anchor date** that phases all collection periods for that framework. |
| **Criterion** | A single requirement of a Framework (e.g. SOC 2 `CC6.1`, ISO 27001 `A.5.1`). Owned by exactly one Framework; seeded from the definition file and editable by the organization. Its display label is set per Framework (FR-FW-7). |
| **Control** | A safeguard the organization implements. Framework-agnostic; may satisfy Criteria in several Frameworks. |
| **Test** | A reusable procedure that verifies one or more Controls are operating. Carries a collection **cadence**. |
| **Evidence Requirement** | A named slot on a Test (e.g. "AWS", "Okta") that must be filled with evidence. Tracks two independent axes: **Collection** and **Assessment**. |
| **Evidence** | An artifact — uploaded file or external link — with a collection date. Lives in a shared repository and may be attached to many Evidence Requirements. |
| **Collection state** | Derived per Evidence Requirement: **current / due / overdue**, based on the newest attached evidence versus the current collection period. |
| **Assessment** | A judgment recorded on an Evidence Requirement: **not assessed / pass / fail / exception**, with who and when. |
| **Collection period** | A window of the Test's cadence length (1, 3, 6, or 12 months), phased from the Framework's anchor date. |
| **Health** | Per Control, per Framework: **Green / Yellow / Red**, derived from evidence *freshness* only. |
| **Coverage** | Per Criterion, per Framework: **covered / not covered**, derived strictly from Control satisfaction. |
| **Policy** | A governance document (file or link) the organization maintains, linked to the Controls it directs. |
| **Risk** | A security risk with inherent, residual, and derived effective levels, mitigated by mapped Controls. |

---

## 4. Domain Model

- **DM-1 (MUST)** Frameworks, Criteria, Controls, Tests, Evidence
  Requirements, Evidence, Policies, and Risks are distinct entities.
- **DM-2 (MUST)** A Framework contains many Criteria; each Criterion belongs
  to exactly one Framework.
- **DM-3 (MUST)** Control ↔ Criterion is many-to-many, across Frameworks: one
  Control may satisfy Criteria in several Frameworks; one Criterion may be
  satisfied by several Controls.
- **DM-4 (MUST)** Test ↔ Control is many-to-many. A Test is defined once and
  reused across Controls without duplication.
- **DM-5 (MUST)** A Test has one or more Evidence Requirements; each Evidence
  Requirement belongs to exactly one Test. A simple test has one requirement;
  a per-system test (e.g. quarterly access review of AWS, Okta, and the
  production database) has one per system.
- **DM-6 (MUST)** Evidence ↔ Evidence Requirement is many-to-many. Evidence
  items live in a shared repository; one item may be attached to requirements
  on different Tests, and one requirement may hold many items. Freshness is
  judged **per attachment** (against that requirement's Test cadence and
  Framework grid), so the same item can be current for one Test and overdue
  for another.
- **DM-7 (MUST)** Each Evidence Requirement carries two independent states:
  a derived **collection state** and a recorded **assessment**.
- **DM-8 (MUST)** Policy ↔ Control is many-to-many. Policies do not
  participate in coverage or Health.
- **DM-9 (MUST)** Risk ↔ Control is many-to-many. Risks have no Tests or
  Evidence of their own; their effective level derives from their mapped
  Controls.
- **DM-10 (SHOULD)** Controls, Tests, Evidence Requirements, Evidence,
  Policies, and Risks each carry a human-readable reference (e.g. `CTL-014`)
  in addition to the system identifier.

---

## 5. Functional Requirements

### 5.1 Frameworks & Criteria

- **FR-FW-1 (MUST)** The system supports more than one Framework at once.
- **FR-FW-2 (MUST)** Frameworks and their Criteria are loaded from
  **JSON framework definition files**. Adding a Framework means adding a
  definition file, not writing code.
- **FR-FW-3 (MUST)** A SOC 2 definition ships out of the box with the full
  Trust Services Criteria.
- **FR-FW-4 (SHOULD)** An ISO 27001:2022 definition ships out of the box
  with the 93 Annex A controls as its Criteria. Clause 4–10 management-system
  requirements MAY be added later as a separate definition or section.
- **FR-FW-5 (MUST)** Users can view Frameworks and browse their Criteria.
- **FR-FW-6 (MUST)** Once loaded, a Framework is the organization's own
  copy: users can edit it — add, edit, and remove Criteria, and annotate them
  — without changing the definition file. Reloading a definition file does
  not overwrite the organization's edits.
- **FR-FW-7 (MUST — requirement label)** Each Framework definition carries
  its own singular/plural label for the requirement level (SOC 2 →
  "Criterion/Criteria", ISO 27001 → "Annex A Control(s)", GDPR →
  "Article(s)"). The UI resolves the label from the Framework in context.
  The label must be distinct from the organization's own "Controls" (hence
  "Annex A Controls", not "Controls", for ISO 27001).
- **FR-FW-8 (MUST — anchor date)** Each Framework has a settable **anchor
  date** (e.g. the SOC 2 audit period start). All collection periods for that
  Framework are phased from this date: a 12-month cadence's period runs
  anchor-to-anchor, a monthly cadence's periods start on the anchor's
  day-of-month, and so on. Periods are never aligned to arbitrary calendar
  boundaries.
- **FR-FW-9 (MUST — per-framework evaluation)** Because one Test may serve
  Controls in several Frameworks with different anchors, freshness is
  evaluated **per Framework context**: the same Test/Evidence is judged
  against each linked Framework's period grid. Consequently a Control's
  Health and a Criterion's Coverage are Framework-scoped and may differ
  between Frameworks. Each Framework's dashboard uses that Framework's grid.
  A Test's cadence is single-valued; if two Frameworks require different
  collection frequencies for the same verification, model them as two Tests.

### 5.2 Controls

- **FR-CT-1 (MUST)** Users can create, view, edit, and delete Controls.
- **FR-CT-2 (MUST)** A Control has a name and description, and SHOULD have
  an owner.
- **FR-CT-3 (MUST — implementation status)** A Control carries a manually
  set **implementation status** with exactly these values:
  **Not Implemented / Implemented / Not Applicable (N/A)**. N/A SHOULD
  support an optional justification.
- **FR-CT-4 (MUST — auto-implement)** A **global** setting, when enabled,
  automatically transitions a Control from Not Implemented → Implemented the
  first time it becomes *satisfied* (FR-ST-2). The transition is one-way: the
  system never auto-reverts to Not Implemented (evidence lapse is shown by
  Health, not by changing status). When the setting is disabled, status is
  purely manual.
- **FR-CT-5 (MUST — N/A exclusion)** A Control marked N/A is excluded from
  Coverage and Health: it neither satisfies nor blocks a Criterion.
- **FR-CT-6 (MUST)** Users can map a Control to one or more Criteria (across
  several Frameworks at once) and unmap them.
- **FR-CT-7 (SHOULD)** For any Control, users can see all Criteria it maps
  to, all Tests attached to it, and all Policies and Risks linked to it.

### 5.3 Tests

- **FR-TS-1 (MUST)** Users can create, view, edit, and delete Tests.
- **FR-TS-2 (MUST)** A Test has a name and description, and SHOULD define a
  procedure.
- **FR-TS-3 (MUST — cadence)** Each Test has a **collection cadence** of
  **1, 3, 6, or 12 months**. The cadence sets the period *length*; the period
  *phase* comes from each Framework's anchor date (FR-FW-8). Every Test
  has a cadence; evidence never stays current indefinitely.
- **FR-TS-4 (MUST)** Users can map a Test to one or more Controls and unmap
  them.
- **FR-TS-5 (SHOULD)** For any Test, users can see all Controls it verifies
  and all Evidence attached under it.
- **FR-TS-6 (SHOULD)** Creating a Test with a single Evidence Requirement is
  effortless (e.g. a default requirement is created), so simple tests do not
  feel ceremonious.

### 5.4 Evidence Requirements & Evidence

Each Evidence Requirement tracks two **independent** axes:

- **Collection** — is current evidence present? (derived)
- **Assessment** — has that evidence been judged pass/fail? (recorded)

Evidence can be collected but not yet assessed, or collected and assessed
*fail* (present, but showing the control is not working). The one hard link
between the axes is FR-EV-8: a *pass* requires current evidence.

- **FR-EV-1 (MUST)** Users can define one or more named Evidence Requirements
  on a Test (e.g. "AWS", "Okta", "Prod DB").
- **FR-EV-2 (MUST)** Evidence is collected **manually by users**: a user
  attaches Evidence to an Evidence Requirement by uploading a file, entering
  an external link/URL, or selecting an existing item from the shared
  evidence repository. The system does not collect evidence itself.
- **FR-EV-3 (MUST)** Each Evidence item records a **collection date** and MAY
  carry notes. The collection date may be set to a past date (back-dating);
  all freshness calculations use the recorded collection date.
- **FR-EV-4 (MUST — collection state)** Each Evidence Requirement derives a
  **collection state** from its newest attached Evidence, evaluated against
  the current collection period (FR-TS-3, FR-FW-8) in the Framework context
  being viewed:
  - **current** — newest evidence was collected within the current period.
  - **due** — newest evidence was collected within the immediately preceding
    period; collection is due this period but not yet done.
  - **overdue** — no evidence, or newest evidence is older than the preceding
    period.
- **FR-EV-5 (MUST — assessment)** Each Evidence Requirement carries an
  **assessment** with a result of **not assessed / pass / fail / exception**,
  recording who assessed it, when, and an optional note. *Exception* means
  the evidence shows a documented deviation; it does not count as passing.
- **FR-EV-6 (SHOULD)** Users can download/open, replace, and delete Evidence
  items. Deleting or detaching a shared item warns when it is attached to
  other requirements.
- **FR-EV-7 (SHOULD)** Users can browse and search the shared evidence
  repository and see, for any item, every requirement it is attached to.
- **FR-EV-8 (MUST — no pass without evidence)** An Evidence Requirement
  cannot be assessed **pass** unless its collection state is **current** at
  the time of assessment. This rule cannot be overridden.

### 5.5 Status, Health & Coverage

The system tracks four distinct status axes:

| Axis | On | Values | How set |
|---|---|---|---|
| **Assessment** | Evidence Requirement | not assessed / pass / fail / exception | recorded by a user (FR-EV-5) |
| **Implementation status** | Control | Not Implemented / Implemented / N/A | manual, or auto-implement (FR-CT-3/4) |
| **Health** | Control, per Framework | Green / Yellow / Red | derived from freshness only (FR-ST-4) |
| **Coverage** | Criterion, per Framework | covered / not covered | derived, strict (FR-ST-1..3) |

Coverage answers *"is it passing?"*; Health answers *"is the evidence
current?"*. A Control can be Health-Green yet not satisfied (fresh evidence
assessed *fail*), but never satisfied without current evidence (FR-EV-8).

- **FR-ST-1 (MUST — Test status)** A Test is **passing** only when **every**
  Evidence Requirement is both **current** (FR-EV-4) **and** assessed
  **pass** (FR-EV-5). If any requirement is due, overdue, not assessed,
  fail, or exception, the Test is not passing, and the reason is
  distinguishable.
- **FR-ST-2 (MUST — Control satisfaction)** A Control is **satisfied** only
  when it has at least one Test **and every** attached Test is passing. A
  Control with no Tests is not satisfied.
- **FR-ST-3 (MUST — Criterion coverage)** A Criterion is **covered** only
  when it has at least one mapped non-N/A Control **and every** mapped
  non-N/A Control is satisfied. A Criterion with no mapped non-N/A Controls
  is not covered.
- **FR-ST-4 (MUST — Health)** Each non-N/A Control derives a **Health**
  indicator, worst-wins across every Evidence Requirement of every attached
  Test, in the Framework context being viewed:
  - **Green** — every requirement is *current*.
  - **Yellow** — none is *overdue*, but at least one is *due*.
  - **Red** — at least one is *overdue*.
- **FR-ST-5 (MUST — readiness)** For each Framework, the system presents a
  **readiness view** showing the proportion of Criteria covered.
- **FR-ST-6 (MUST — gaps)** The system identifies gaps: Criteria with no
  mapped Controls, Controls with no Tests, Evidence Requirements that are due
  or overdue, and Evidence Requirements assessed fail or exception.
- **FR-ST-7 (MUST — two lenses)** The status dashboard lets users toggle
  between two lenses over the same Criteria/Controls/Tests:
  - **Collection view** — freshness: Health (Green/Yellow/Red) and per
    requirement current/due/overdue.
  - **Assessment view** — pass/fail: per requirement pass/fail/exception/not
    assessed, rolling up to Coverage.
- **FR-ST-8 (SHOULD)** All derived states update automatically as Controls,
  Tests, Evidence, mappings, and dates change.

### 5.6 Policies

- **FR-PL-1 (MUST)** Users can create, view, edit, and delete Policies. A
  Policy has a name and a current document (uploaded file or link), and
  SHOULD have an owner, a version identifier, and a last-reviewed date.
- **FR-PL-2 (MUST)** Users can link a Policy to one or more Controls and
  unlink them.
- **FR-PL-3 (MUST)** Policies are informational for Coverage and Health: an
  out-of-date Policy does not by itself change any derived state. Periodic
  policy review is verified with an ordinary Test (e.g. "Annual policy
  review" with one Evidence Requirement per policy).
- **FR-PL-4 (SHOULD)** Replacing a Policy document retains prior versions
  (document, date, who).
- **FR-PL-5 (SHOULD)** For any Policy, users can see the Controls it directs.

### 5.7 Risks

The risk model uses the **NIST SP 800-30 Rev. 1** qualitative scales:
likelihood and impact are each rated **Very Low / Low / Moderate / High /
Very High**, and a risk level is derived from the 800-30 likelihood × impact
matrix (Table I-2).

- **FR-RK-1 (MUST)** Users can create, view, edit, and delete Risks. A Risk
  has a name and description, and SHOULD have an owner.
- **FR-RK-2 (MUST — inherent risk)** Each Risk carries a manually set
  inherent likelihood and impact (before any controls); the inherent risk
  level is derived via the matrix.
- **FR-RK-3 (MUST — residual risk)** Each Risk carries a manually set
  residual likelihood and impact (assuming all mapped mitigating Controls are
  functioning); the residual risk level is derived via the matrix.
- **FR-RK-4 (MUST — mitigating controls)** Users can map a Risk to the
  Controls that mitigate it. Every mapped Control is required: the
  residual rating presumes all of them are functioning.
- **FR-RK-5 (MUST — effective risk)** The system derives each Risk's
  **effective risk level**:
  - If every mapped Control is functioning → effective = **residual**.
  - If any mapped Control is not functioning, or the Risk has no mapped
    Controls → effective = **inherent**, the Risk is flagged **elevated**,
    and the non-functioning Controls are identified.
  A Control is **functioning** when it is *satisfied* (FR-ST-2) in **every**
  Framework context it serves (worst case across Frameworks).
- **FR-RK-6 (MUST)** A Control marked N/A does not mitigate. Mapping an N/A
  Control to a Risk is flagged as an inconsistency and treated as not
  functioning for FR-RK-5.
- **FR-RK-7 (MUST — risk register)** The system presents a risk register
  listing each Risk with its inherent, residual, and effective levels, its
  mitigating Controls and their state, and highlighting elevated Risks.
- **FR-RK-8 (SHOULD)** A Risk supports an **accepted** flag with a
  justification note, shown in the register.

### 5.8 Users, Authentication & Roles

- **FR-AU-1 (MUST)** All access requires authentication, on every interface
  (§5.10).
- **FR-AU-2 (MUST)** Three flat roles: **admin**, **contributor**, **viewer**.
  There is no policy engine or per-object permission.
- **FR-AU-3 (MUST)** Role capabilities:
  - **viewer** — read-only access to everything.
  - **contributor** — viewer, plus create/edit/delete Controls, Tests,
    Evidence Requirements, Evidence, Policies, Risks, mappings, assessments,
    and Control implementation status.
  - **admin** — contributor, plus manage users, Frameworks (load
    definitions, edit Criteria, set anchor dates), and global settings.
- **FR-AU-4 (MAY)** First run bootstraps an initial admin account.

### 5.9 Reporting & Audit Trail

- **FR-RP-1 (MUST)** The system exports a coverage/status report as **CSV**
  per Framework — Criteria, mapped Controls, Tests, and each Evidence
  Requirement's collection state and assessment — suitable for sharing with
  an auditor.
- **FR-RP-2 (SHOULD)** The system records who changed what and when for
  Controls, Tests, Evidence, Policies, Risks, mappings, and assessments.
- **FR-RP-3 (MAY)** The system notifies owners of due and overdue Evidence
  Requirements.

### 5.10 Interfaces

- **FR-IF-1 (MUST)** The system is accessible through three interfaces: a
  **web UI**, a **CLI**, and an **MCP server**.
- **FR-IF-2 (MUST)** All three interfaces expose the same capabilities and
  enforce the same authentication and roles (§5.8). Anything a user can do in
  the web UI can be done from the CLI or via MCP with the same role.

---

## 6. Non-Functional Requirements

- **NFR-1 (MUST)** Single-tenant: one organization per deployment; no tenant
  isolation logic.
- **NFR-2 (MUST)** Self-hosted and self-contained: runs with no mandatory
  external SaaS, AI, or LLM dependency.
- **NFR-3 (MUST)** Data persists durably; uploaded Evidence and Policy files
  are stored and retrievable.
- **NFR-4 (SHOULD)** Handles realistic single-organization scale — hundreds
  of Controls and Tests, thousands of Evidence items — with a responsive UI.
- **NFR-5 (MUST)** Framework definitions and seed data are maintained as
  JSON data files, not code.

