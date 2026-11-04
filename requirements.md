# Lifeboat — Requirements

Lifeboat is a simple, self-hosted, single-tenant GRC (governance, risk and
compliance) application.

This document states what the application must do: its capabilities and
rules. Architecture, technology and implementation choices are covered in a
separate design document.

**Priority**

| Level | Meaning |
|---|---|
| **MUST** | Required for version 1. |
| **SHOULD** | Wanted for version 1; may move to a later release if necessary. |
| **MAY** | Optional; a later release. |

**Requirement IDs.** Every requirement has an ID so it can be cited in
design documents, issues and tests. The prefix gives the section:

| Prefix | Section |
|---|---|
| `DM` | 4. Domain model |
| `FW` | 5.1 Frameworks and Criteria |
| `PD` | 5.2 Observation periods and collection periods |
| `CT` | 5.3 Controls |
| `TS` | 5.4 Tests |
| `EV` | 5.5 Evidence and assessments |
| `ST` | 5.6 Status, Health and Coverage |
| `PL` | 5.7 Policies |
| `RK` | 5.8 Risks |
| `FN` | 5.9 Findings |
| `RP` | 5.10 Reports and exports |
| `LOG` | 5.11 Audit trail |
| `AU` | 5.12 Users, sign-in and roles |
| `IF` | 5.13 Interfaces |
| `SEC` | 6. Security |
| `NFR` | 7. Non-functional requirements |

---

## 1. Purpose

Lifeboat lets one organization manage its compliance across one or more
frameworks, such as SOC 2 and ISO 27001. It records each framework's
requirements, the organization's controls, the tests that check those
controls, and the evidence the tests rely on. It also holds the
organization's policies and a simple risk register, and it reports
readiness and gaps.

Lifeboat is **single-tenant**: one organization per installation.

---

## 2. Scope

### 2.1 In scope for version 1

- Several compliance frameworks at once, each loaded from a definition file.
  SOC 2 and ISO 27001:2022 ship as definition files. The application knows
  nothing about any framework beyond what its file contains (FW-3).
- The chain from framework requirement to control, test and evidence, with
  many-to-many links at each step.
- Evidence freshness, based on each test's collection cadence and each
  framework's observation period (§5.2); recorded pass/fail assessments; and
  the resulting readiness and gap reporting.
- A policy repository: governance documents linked to controls.
- A risk register with inherent, residual and effective risk on the NIST SP
  800-30 scales, reduced by linked controls.
- Username and password sign-in, with three roles.
- An HTTP API and an MCP server that carry every capability. The web UI and
  the CLI work through the HTTP API (IF-1).
- Evidence and assessments added by other applications, including AI tools,
  through the API or MCP server (EV-16), and evidence requests sent to
  another application's web address (TS-8).
- CSV exports for auditors: current coverage (RP-2); evidence by collection
  period (RP-3, RP-4); Statement of Applicability (RP-7); risk register and
  findings log (RP-8); document register (RP-9); audit log (LOG-1); and users
  and bearer tokens (AU-11). Also, as single downloads, the evidence files
  behind an RP-3 export (RP-6) and the approved policy files behind the
  document register (RP-10).

### 2.2 Out of scope for version 1

Any of these could come in a later release.

- Multi-tenancy.
- SSO, SCIM, an OAuth2 authorization server, invitations, fine-grained
  permissions, and second-factor sign-in.
- Automated or continuous testing, built-in connectors, and built-in AI.
  Other applications can do this from outside through the API (EV-16, TS-8);
  Lifeboat itself does not collect evidence or assess controls.
- Formal auditor workpapers: sample selection, population definition and
  auditor exception logs. The organization's own assessments, exception
  notes and findings log (FN-1) are in scope.
- Vendor management, trust center, privacy tooling (DPIA, data mapping) and
  asset inventory.
- Quantitative or automated risk scoring, threat modeling, and risk
  treatment workflow (task tracking, approval routing) beyond the records in
  RK-7.
- Formatted (PDF) reports.
- Notifications and integrations (email, Slack, ticketing), apart from RP-12
  and evidence requests (TS-8).

---

## 3. Terminology

| Term | Meaning |
|---|---|
| **Framework** | A compliance standard, such as SOC 2 or ISO 27001. Has an observation period. |
| **Criterion** | One requirement of a Framework, such as SOC 2 `CC6.1` or ISO 27001 `A.5.1`. Belongs to exactly one Framework. Loaded from the definition file and editable by the organization. Its display label comes from the definition (FW-9). |
| **Control** | A safeguard the organization operates. Not tied to one Framework; it may meet Criteria in several. |
| **Test** | A reusable procedure that checks that one or more Controls work. Has a collection cadence. |
| **Evidence Requirement** | A named slot on a Test that must be filled with evidence, such as "AWS" or "Okta" in a per-system access review. |
| **Evidence** | A file, link or written statement. Each version has a collection date (EV-5) and a date added (EV-6). Evidence lives in a shared repository and can be attached to several Evidence Requirements. |
| **Observation period** | The window under review for a Framework, with a start and end date set by an admin, such as a SOC 2 audit period or an ISO 27001 audit cycle. Its name comes from the Framework definition (PD-1). |
| **Cadence** | How often a Test's evidence must be collected: every 1, 3, 6 or 12 months. |
| **Collection period** | One cadence-length window, counted from the start of the Framework's current observation period (PD-3). |
| **Collection state** | The state of one collection period of an Evidence Requirement: not yet due, due, done or overdue (EV-11). |
| **Assessment** | A judgment recorded on an Evidence Requirement: pass, fail or exception. A period with no current assessment shows as not assessed (EV-14). |
| **Health** | Green, Yellow or Red for a Control in one Framework, based only on whether evidence is current (ST-4). |
| **Coverage** | Covered or not covered for a Criterion in one Framework, based on whether its Controls pass (ST-3). |
| **Applicability** | Whether a Criterion applies to the organization: Applicable or Not applicable, with a justification (FW-7). |
| **Active range** | Optional dates between which a Control, Test or Evidence Requirement is in operation (TS-7). |
| **Policy** | A controlled document the organization maintains, such as a policy or procedure, linked to the Controls it directs. |
| **Risk** | A security risk with inherent, residual and effective levels, reduced by linked Controls. |
| **Finding** | A nonconformity, exception, observation or improvement opportunity, with its corrective action (FN-1). |

---

## 4. Domain model

- **DM-1 (MUST)** Frameworks, Criteria, Controls, Tests, Evidence
  Requirements, Evidence, Policies, Risks and Findings are distinct records.
- **DM-2 (MUST)** A Framework contains many Criteria. Each Criterion belongs
  to exactly one Framework.
- **DM-3 (MUST)** Controls and Criteria are linked many-to-many, across
  Frameworks. One Control may meet Criteria in several Frameworks, and one
  Criterion may be met by several Controls.
- **DM-4 (MUST)** Tests and Controls are linked many-to-many. A Test is
  defined once and reused across Controls.
- **DM-5 (MUST)** A Test has one or more Evidence Requirements, and each
  Evidence Requirement belongs to exactly one Test. A simple Test has one; a
  per-system Test, such as a quarterly access review of AWS, Okta and the
  production database, has one per system.
- **DM-6 (MUST)** Evidence and Evidence Requirements are linked
  many-to-many. One Evidence item may be attached to requirements on
  different Tests, and one requirement may hold many items. Freshness is
  judged per attachment, against that requirement's Test cadence and the
  Framework's collection periods, so the same item can be done for one Test
  and overdue for another.
- **DM-7 (MUST)** Each Evidence Requirement has two independent states: a
  collection state, which the system works out from the evidence, and an
  assessment, which a user records.
- **DM-8 (MUST)** Policies and Controls are linked many-to-many. Policies do
  not affect Coverage or Health.
- **DM-9 (MUST)** Risks and Controls are linked many-to-many. Risks have no
  Tests or Evidence of their own; their effective level comes from their
  linked Controls.
- **DM-10 (MUST)** Frameworks, Controls, Tests, Evidence Requirements,
  Evidence, Policies, Risks and Findings each have a human-readable
  reference, such as `CTL-014`.
  - It is unique within its record type.
  - It never changes: not on edit, rename, retirement, upgrade or restore.
  - It is never reused, even after deletion.
  - It appears in every export.
  A Criterion's reference comes from its definition and is unique within its
  Framework. Every import, export and cross-Framework reference to a
  Criterion also names its Framework.
- **DM-11 (MUST)** Deleting a record removes it from active use. It never
  deletes shared records or history.
  - Before any deletion, the system lists the records affected.
  - Deleting a Framework is admin-only and retires its Criteria and links.
    Their references and assessments stay readable in history, and other
    Frameworks that share its Controls and Tests are unchanged.
  - A Control, Test or Evidence Requirement that has attached Evidence or a
    recorded assessment is, on deletion, ended on the deletion date (TS-7),
    with its links as they stood. It stays in history, in the period view
    (EV-14), in gaps (ST-7) and in RP-3, marked "deleted", for every period
    that started before the deletion. A record with neither is removed.
  - Deleting Evidence and Policy documents follows EV-9.

---

## 5. Functional requirements

### 5.1 Frameworks and Criteria

- **FW-1 (MUST)** The system supports several Frameworks at once.
- **FW-2 (MUST)** Frameworks are loaded from JSON definition files. Adding a
  Framework means adding a file, not writing code.
  - A definition file may contain only Criteria, or Criteria plus Controls,
    Evidence Requirements and Tests, with their links to Criteria and to each
    other. Both kinds load the same way.
  - Loading a file never creates Evidence or assessments.
- **FW-3 (MUST)** The application contains no behaviour specific to any
  Framework. Everything that distinguishes one Framework from another comes
  from its definition file: its Criteria, their types and sections, the
  requirement labels (FW-9), the observation-period name (PD-1), which
  Criteria cannot be excluded (FW-7), and which Criterion type a Statement
  of Applicability covers (RP-7). The system behaves the same for any
  Framework loaded from a valid definition file. The shipped SOC 2 and ISO
  27001 files are data, not features.
- **FW-4 (MUST)** Loading a definition file:
  - is admin-only;
  - states the definition's source and version, and the stable reference of
    each Criterion;
  - checks the whole file before changing anything.
  Loading the same definition again creates no duplicate Criteria,
  Controls, Evidence Requirements or Tests, and does not reset links,
  applicability or local edits.
  Loading an updated definition:
  - lists its additions, removals and changed text before it is applied;
  - refuses a conflict it cannot resolve, with no change;
  - retires, rather than deletes, a Criterion the new version removes. The
    retired Criterion keeps its reference, links and history, and no longer
    counts toward current status;
  - replaces a locally edited value only when the admin chooses that for the
    value;
  - treats a Criterion whose reference has changed as a removal plus an
    addition, without carrying its links over.
- **FW-5 (MUST)** Users can view Frameworks and browse their Criteria.
- **FW-6 (MUST)**
  - Admins can add, edit and annotate Criteria and set their applicability
    (FW-7).
  - A Criterion added by the organization takes one of its definition's
    Criterion types, chosen by the admin, or no type if the definition has
    none. It can be deleted.
  - Criteria from the definition file cannot be deleted, only marked Not
    applicable.
  - A Framework loaded by mistake is removed by deleting the Framework
    (DM-11).
- **FW-7 (MUST)** Each Criterion is Applicable (the default) or Not
  applicable.
  - Marking a Criterion Not applicable requires an exclusion justification,
    entered or explicitly confirmed in the same change. An existing inclusion
    justification is never reused as the exclusion justification without
    that confirmation.
  - An Applicable Criterion may have an inclusion justification. The
    Statement of Applicability (RP-7) shows whether each justification is for
    inclusion or exclusion.
  - An admin may set applicability for one Criterion, or for every Criterion
    in a section (FW-8) at once. A justification is still required.
  - A definition can mark Criteria as not excludable; the system refuses to
    mark them Not applicable. The shipped ISO 27001 file marks clauses 4–10;
    the SOC 2 file marks CC1–CC9.
  - Not applicable Criteria are left out of Coverage, readiness and gaps.
    They stay visible and exportable with their justification.
  - Changes are admin-only and logged (LOG-1).
- **FW-8 (MUST)** A Criterion may belong to a section given by its
  definition file. Sections are used for section-wide applicability (FW-7)
  and per-section readiness (ST-6).
  - In a definition that has sections, Criteria without one form a group
    called "no section". A definition with no sections loads normally, and
    its readiness shows the total only.
  - Section identifiers are unique within a definition; a file with
    duplicates is refused with no change. Display labels may repeat.
  - The shipped ISO 27001 file uses sections 'Clause 4' to 'Clause 10' and
    'A.5' to 'A.8'. The shipped SOC 2 file uses CC1–CC9, A1, C1, PI1 and
    P1–P8.
- **FW-9 (MUST)** Each definition gives a singular and plural label for its
  Criteria, optionally one per Criterion type. The interface uses the label
  of the Framework being viewed, and the same label heads the Criterion
  columns of every per-Framework export (RP-2, RP-3, RP-7). Labels must differ
  from "Control" and "Evidence Requirement". The shipped labels are:
  - SOC 2: "Trust Services Criterion / Criteria";
  - ISO 27001: "Clause(s)" for clauses, "Annex A Control(s)" for Annex A, and
    "ISO 27001 Requirement(s)" where both are listed together.
- **FW-10 (MUST)** A SOC 2 definition file ships, labelled "2017 Trust
  Services Criteria (revised points of focus, 2022)".
  - It has 61 Criteria (CC1.1–CC9.2, A1.1–A1.3, C1.1–C1.2, PI1.1–PI1.5,
    P1.1–P8.1), each with its section.
  - CC1–CC9 are marked not excludable.
  - Points of focus are guidance text on their Criterion, not separate
    Criteria.
  - The shipped text is limited to IDs and short titles written for
    Lifeboat. The organization can add the full Criterion text and points of
    focus (FW-6).
  - It names no Statement of Applicability type, so RP-7 and the Statement
    of Applicability gap in ST-7 do not apply to it.
- **FW-11 (MUST)** An ISO/IEC 27001:2022 definition file, including
  Amendment 1:2024, ships with 123 Criteria:
  - 30 of type "Clause": 4.1–4.4, 5.1–5.3, 6.1.1–6.1.3, 6.2, 6.3, 7.1–7.4,
    7.5.1–7.5.3, 8.1–8.3, 9.1, 9.2.1, 9.2.2, 9.3.1–9.3.3, 10.1 and 10.2;
  - 93 of type "Annex A": A.5.1–A.5.37, A.6.1–A.6.8, A.7.1–A.7.14 and
    A.8.1–A.8.34.
  Its Statement of Applicability type is Annex A. Criterion type is a
  general, optional part of the definition format (FW-3). The shipped text
  is limited to IDs and short titles written for Lifeboat; the organization
  can add text (FW-6).
- **FW-12 (MUST)** The shipped SOC 2 and ISO 27001 files also include a
  starter set of Controls, Evidence Requirements and Tests, linked to their
  Criteria. Once loaded, these are ordinary records that users can edit or
  delete. For version 1, about 10 Controls in each file is enough. The set
  can grow later by editing the files, without code changes (NFR-5).

### 5.2 Observation periods and collection periods

- **PD-1 (MUST)** Each Framework has a current **observation period**, with
  a start date and an end date set by an admin.
  - The admin enters it when loading the definition; there is no default.
  - Its name comes from the definition file: "Audit period" in the shipped
    SOC 2 file and "Audit cycle" in the shipped ISO 27001 file. A definition
    that gives no name uses "Observation period".
  - Both dates are inclusive. An end date before the start date is refused.
    Otherwise the admin may set any dates, and observation periods may
    overlap.
  - Starting a new observation period closes the current one. A closed
    observation period never changes: its collection periods, the placement
    of its evidence and its assessments stay as they were. Evidence and
    assessments recorded after it closed are never placed in it, whatever
    their dates.
  - Setting or changing an observation period, or starting a new one, is
    admin-only and logged with old and new values (LOG-1). It never changes
    recorded collection dates or assessments; its effect on collection
    periods follows PD-5.
  - The readiness view and every export state the observation period and
    time zone used.
- **PD-2 (MUST)** Dates and time zone:
  - The organization has one time zone. Changing it is admin-only and logged
    with old and new values (LOG-1). It never changes stored dates or
    recorded times.
  - Observation-period dates and evidence collection dates are dates without
    a time.
- **PD-3 (MUST)** A Framework's **collection periods** for a Test are
  windows of the Test's cadence length (TS-3), counted from the start of the
  Framework's current observation period.
  - Each period includes its start date and ends the day before the next
    period starts.
  - Month-based boundaries are counted from the observation period's start
    date, using the last day of the month when a month is shorter. Periods
    are never realigned to calendar boundaries; an observation period that
    starts on 1 January gives calendar-aligned periods.
  - Periods continue with the same timing before the start and after the end
    of the observation period, until a new observation period is started. A
    period that runs past the end date is left as it is. The end date has no
    effect on collection state.
  - "Current period" and "preceding period" always refer to this sequence.
- **PD-4 (MUST)** One Test may serve Controls in several Frameworks with
  different observation periods. Freshness is therefore judged separately in
  each Framework: the same Test and Evidence are measured against each
  linked Framework's collection periods. A Control's Health and a
  Criterion's Coverage can differ between Frameworks, and each Framework's
  dashboard uses its own collection periods. A Test has one cadence; if two
  Frameworks need the same check at different frequencies, they use two
  Tests.
- **PD-5 (MUST)** A Test's cadence and a Framework's current observation
  period can be changed at any time, including after evidence exists. When
  either changes, the system recalculates the collection periods from the
  start of the current observation period, in each Framework affected.
  - Evidence and assessments in the current observation period are placed in
    the new periods by their collection dates, and every derived state is
    updated by the next read (ST-11).
  - Closed observation periods do not change (PD-1).
  - The change takes effect when it is made. There is no separate effective
    date and no transition period.
  - The change is logged with old and new values (LOG-1).
- **PD-6 (MUST)** Recalculating periods (PD-5) keeps every reset caused by a
  withdrawal, deletion or other change under EV-14. Moving or merging
  periods never makes a reset assessment count again. A later valid
  assessment may count under the normal rules.

### 5.3 Controls

- **CT-1 (MUST)** Users can create, view, edit and delete Controls.
- **CT-2 (MUST)** A Control has a name, a description and an owner. The owner
  is a user, or a role or person named as text. A Control that is not N/A
  and has no owner, or whose owner is a disabled user (AU-5), can be saved
  and is listed as a gap (ST-7).
- **CT-3 (MUST)** Each Control has an **implementation status**, set by a
  user: Not Implemented, Implemented or Not Applicable (N/A). An N/A Control
  can have a justification.
- **CT-4 (MUST)** A global setting, when on, changes a Control from Not
  Implemented to Implemented the first time it is satisfied (ST-2) in every
  active Framework it serves. A Control that serves no Framework is never
  changed. The change is logged as made by the system (LOG-1). It is one-way:
  the system never changes a Control back to Not Implemented; lapsed
  evidence shows in Health instead. When the setting is off, implementation
  status is set by users only.
- **CT-5 (MUST)** An N/A Control is left out of Coverage and Health: it
  neither meets nor blocks a Criterion.
- **CT-6 (MUST)** Users can link a Control to Criteria in one or more
  Frameworks, and unlink them, from either the Control or the Criterion.
- **CT-7 (SHOULD)** For any Control, users can see every Criterion, Test,
  Policy and Risk linked to it.

### 5.4 Tests

- **TS-1 (MUST)** Users can create, view, edit and delete Tests.
- **TS-2 (MUST)** A Test has a name, a description and an optional
  procedure.
- **TS-3 (MUST)** Each Test has a **collection cadence** of 1, 3, 6 or 12
  months. The cadence sets the length of its collection periods; their
  timing comes from each Framework's observation period (PD-3). Every Test
  has a cadence, so evidence never stays current forever.
- **TS-4 (MUST)** Users can link a Test to one or more Controls and unlink
  them.
- **TS-5 (SHOULD)** For any Test, users can see every Control it checks and
  all Evidence attached under it.
- **TS-6 (MUST)** A user can create a Test with one Evidence Requirement,
  and attach Evidence to it, without creating that Evidence Requirement
  separately. More Evidence Requirements can be added later without
  recreating the Test.
- **TS-7 (MUST)** A Control, a Test and an Evidence Requirement can each
  have an optional **active range**: an active-from date and an active-to
  date, with a note.
  - Both dates are inclusive. A missing date means no limit. An end before a
    start is refused.
  - Along one chain of Control, Test and Evidence Requirement, the chain is
    in operation in a period when the period overlaps the dates common to
    all three active ranges.
    - A chain with no common part counts for nothing toward Test passing,
      satisfaction or Health for that Control.
    - A Test shared by several Controls is judged separately along each
      chain.
    - Only Evidence dated inside the chain's common range counts for that
      chain.
  - A period wholly outside the active range is left out of collection
    state, Health, Coverage, Test passing, gaps and the work list (EV-17). It
    appears in RP-3 as "not in operation". A period partly inside the range
    counts as in operation.
  - Evidence and assessments from inside the range are kept. Period history
    names the active range that excludes a chain.
  - The dates can be changed or cleared, and each change is logged (LOG-1).
    If a change places a period that holds evidence or an assessment wholly
    outside the range, that period keeps its rows in RP-3 and ST-7, marked
    "not in operation — range changed on <date> by <user>", with the
    period's evidence and current assessment.
- **TS-8 (MUST)** A Test can have an **evidence-request address**: a web
  address, set only by an admin (SEC-9). When one is set, the system sends a
  request for evidence to it:
  - starting a set number of days before the last day of the current
    collection period, and repeating at a set interval until the request is
    satisfied;
  - both numbers are set per Test. The days before must be zero or more and
    the interval at least one day;
  - each request names the Test and each of its Evidence Requirements that
    has no evidence for the current period, with that period's last day. A
    requirement drops out once it has such evidence, and requests stop when
    none is left;
  - requests continue at the same interval after the last day has passed;
  - a Test that serves several Frameworks uses the earliest last day among
    them;
  - the receiving application responds by adding evidence through the API
    (EV-16). The request itself never changes anything in Lifeboat;
  - a request contains no Evidence content, no collection instructions and
    no secret;
  - every request, and whether it was delivered, is recorded and shown on the
    Test.
- **TS-9 (SHOULD)** The receiver of an evidence request can verify that it
  came from this installation and was not altered.

### 5.5 Evidence and assessments

Each Evidence Requirement tracks two independent things:

- **Collection**: is there evidence for the period? The system works this
  out.
- **Assessment**: has that evidence been judged pass or fail? A user records
  this.

Evidence can be collected but not yet assessed, or collected and assessed
as a fail, meaning it shows the Control is not working. The one link between
the two is EV-13: a pass counts only for a period that contains the
collection date of an item it judged.

- **EV-1 (MUST)** Users can define one or more named Evidence Requirements on
  a Test, such as "AWS", "Okta" and "Prod DB".
- **EV-2 (MUST)** An Evidence Requirement can have an owner, who must be a
  user. Ownership assigns work; it grants no permission (AU-2).
  - A requirement with no owner, or whose owner is disabled (AU-5), appears
    in the work list as unassigned. The disabled owner's name stays on past
    records.
  - A requirement whose owner has the viewer role shows "owner cannot submit
    evidence" in the work list and wherever its owner is shown or assigned.
    This is not a gap, and nothing is refused.
- **EV-3 (MUST)** An Evidence Requirement can have collection instructions
  (free text), shown wherever the requirement is presented for collection:
  the work list, the attach action, and CLI and MCP output. An Evidence item
  has an editable title, which defaults to the file name, the link address
  or the first line of a statement.
- **EV-4 (MUST)** Users add evidence by hand; Lifeboat does not collect it.
  In one action, a user attaches evidence to one or more Evidence
  Requirements by:
  - uploading one or more files. Each file becomes an Evidence item with the
    same collection date and attachments. Each file is checked against the
    size limit (SEC-5), and the whole action succeeds or fails as one
    (NFR-8);
  - entering a link;
  - writing a statement. A statement records its author and when it was
    written, cannot be edited afterwards, and is marked as a statement
    wherever items are marked by kind (SEC-6);
  - choosing an existing item from the shared evidence repository.
  Other applications may add evidence on a user's behalf (EV-16).
- **EV-5 (MUST)** Each Evidence version has a **collection date**: the date
  the activity it shows was performed, or the date the condition it records
  was true. Examples: an access review performed on 3 July and uploaded on
  5 July has 3 July; a configuration export taken on 31 March has 31 March;
  minutes of a meeting held on 29 September that were available only on
  5 October have 29 September.
  - The user can give the date when adding evidence, on every interface. If
    none is given, a new item uses the date it was added, and a correction
    (EV-7) uses the previous version's collection date.
  - The date cannot be later than the date the version was added (EV-6), in
    the organization's time zone (PD-2).
  - Freshness and period placement use the collection date of the item's
    current version.
  - Changing only the date creates a new version with the same content
    (EV-6), logged with old and new values (LOG-1). Earlier versions keep
    their dates and the assessments that judged them. The effect on
    assessments follows EV-14.
  - Wherever a version's collection date differs from the previous
    version's, the item and RP-3 also show the previous date.
  - An item can have notes.
- **EV-6 (MUST)** An Evidence item or Policy document is one item with one
  or more versions.
  - The content of a stored version (a file, a statement or a link address)
    never changes. A correction creates a new version, and earlier versions
    stay retrievable with their collection date and contributor.
  - Each version records when it was added and by whom. The system sets
    these values, separately from the collection date, and no role can edit
    them on any interface. They are shown wherever the item appears,
    including RP-3.
- **EV-7 (MUST)** Users can download or open Evidence items. Correcting an
  item (wrong file or wrong link) creates a new version under the same
  reference: the earlier content is kept, the change is logged (LOG-1), and
  any assessment that relied on it is reset in the current observation
  period (EV-14). The user sees one item with its versions. Deleting an
  item takes it out of use (EV-9). Deleting or detaching an item that is
  attached to other requirements warns the user and names them.
- **EV-8 (MUST)** Users can browse and search the shared evidence repository
  by title, reference, kind and collection date, and see every requirement
  an item is attached to. The same search, without deleted items, is used
  when choosing an existing item (EV-4).
- **EV-9 (MUST)** Deleting an Evidence item, an Evidence version or a Policy
  document version takes it out of use: it is detached everywhere, no
  longer counts for collection state, and leaves current views and exports.
  - Its content stays readable, marked deleted. Assessments that judged it
    still show it (EV-14), RP-3 marks it deleted, and backups include it.
  - Only an admin can purge it, giving a reason. A purge removes the content
    only. The name, dates, contributor, purge reason and audit entries stay,
    and history shows "purged".
  - Closed observation periods are not affected (EV-14).
  - Deleting a version follows EV-10.
- **EV-10 (MUST)** Deleting an Evidence item is different from deleting one
  of its versions.
  - Deleting an older version leaves the item's current version and
    attachments unchanged, and resets the assessments that judged it
    (EV-14).
  - Deleting the current version takes the item out of use. It does not make
    an earlier version current.
  - No deletion brings back an earlier assessment or document approval.
    Closed observation periods do not change, and deletion markers are kept.
- **EV-11 (MUST)** Each collection period of an Evidence Requirement has one
  of four **collection states** in the Framework being viewed:
  - **not yet due**: a future period. Nothing is required for it yet;
  - **due**: the current period, with no evidence dated in it yet;
  - **done**: the period has evidence dated in it;
  - **overdue**: the period has ended with no evidence dated in it. It
    becomes done if evidence dated in it is added later.
  The requirement's own state is the state of its current period, except
  that it is overdue while the preceding period is overdue. A period that
  started before the requirement became active is never overdue, so a new
  requirement stays due until the end of its first period.
  - For each requirement, the system also shows, per Framework, the last day
    of the current collection period.
  - Only attached, non-deleted Evidence counts.
  - A requirement whose Test serves no Framework shows "no Framework", with
    no collection state and no deadline.
  - Attaching evidence never requires choosing a Framework; Frameworks matter
    only when viewing states.
- **EV-12 (MUST)** Users record **assessments** on an Evidence Requirement,
  with a result of **pass**, **fail** or **exception**.
  - Each assessment records who made it, through which interface (SEC-8),
    when, and a note. A fail or exception requires a note describing the
    deviation and, where known, the fix.
  - An exception means the evidence shows a documented deviation. It does not
    count as a pass.
  - "Not assessed" is not a result users record. It is shown for any
    requirement or period with no current assessment (EV-14).
  - A user may withdraw a mistaken assessment, with no note required. It
    stays in history, and the period no longer passes.
  - A user may record a fail with no evidence, for example when a review was
    not done. It stays in the requirement's history but belongs to no
    collection period, so it changes no period's assessment, Test passing,
    Coverage or effective risk; the missed period still shows as overdue. It
    can be withdrawn later, or followed by a new assessment once evidence is
    attached.
  - A pass always needs evidence (EV-13).
- **EV-13 (MUST)** An Evidence Requirement can be assessed as a pass only on
  at least one attached Evidence item. In each Framework, a pass counts only
  for a collection period that contains the collection date of an item it
  judged. No assessment makes a period without such an item pass. This rule
  applies on every interface and cannot be overridden.
- **EV-14 (MUST)** Every assessment is kept as a separate record, holding:
  - the result: pass, fail or exception;
  - the assessor, the interface and the bearer token used, if any (SEC-8);
  - the time it was recorded, and the note;
  - the exact Evidence items and versions it judged, with each version's
    collection date.
  Assessments belong to periods as follows:
  - An assessment names no Framework and no period. It may judge evidence for
    a period that has already ended.
  - In each Framework, an assessment belongs to every collection period that
    contains the collection date of a version it judged, except periods of
    an observation period that was closed before it was recorded. A fail
    with no evidence belongs to no period (EV-12).
  - A period's **current assessment** is the latest assessment that belongs
    to it and was recorded after the period was last reset. If there is
    none, the period is not assessed.
  These changes reset a period, so its earlier assessments no longer count:
  - a new version of an item that was judged. If the collection date
    changed, the periods of both the old and the new date are reset;
  - detaching or deleting an item that was judged;
  - attaching another item dated in the period;
  - withdrawing an assessment, which resets each period in which it was the
    current assessment.
  Items dated in other periods do not reset a period. A reset never makes an
  earlier assessment count again. Resets apply only to periods of the
  current observation period; nothing changes the current assessment of a
  period in a closed observation period.
  History and reporting:
  - Each reset is shown in the period view and in RP-3, with its time, the
    change that caused it and the user who made that change.
  - No later assessment, reset, new period or evidence change alters or
    removes an earlier record.
  - For a chosen Framework, users can view and export each requirement period
    by period: the items dated in the period, every assessment of them,
    every reset, and the current assessment.
- **EV-15 (SHOULD)** A user can record the same result and note on several
  Evidence Requirements in one action: every requirement of one Test, or
  every requirement a chosen Evidence item is attached to. One assessment
  record is written per requirement, and EV-13 and SEC-7 apply to each.
- **EV-16 (MUST)** Other applications, including AI tools, can add Evidence
  and record assessments through the API (IF-3) or the MCP server.
  - Each one acts as a user, through that user's bearer token (AU-9), with
    the same role, checks, limits and EV-13 rule as a person.
  - The Evidence and assessments it adds record the user, interface and
    bearer token (EV-6, SEC-8). An application with its own user account is
    therefore always distinguishable from a person.
- **EV-17 (MUST)** The system shows a **work list** of every Evidence
  Requirement that is in operation (TS-7) and that, in any Framework:
  - is due or overdue (action: collect);
  - has evidence dated in the current period, and a current-period
    assessment that is not assessed (action: assess) or is a fail or
    exception (action: resolve);
  - has an ended collection period in the Framework's current observation
    period, inside its active range, with evidence dated in it and no
    current assessment (action: assess).
  Each entry shows:
  - the Test name, the requirement name, the collection instructions (EV-3),
    the owner or "unassigned", and the action;
  - each Framework and period concerned, with the reason;
  - for a collect entry, a deadline: the earliest last day of the current
    period among the Frameworks in which the requirement is not done. If the
    active-to date is earlier, that date is the deadline and is marked as
    such. Assess and resolve entries have no deadline;
  - for a requirement whose Test serves no Framework, "no Framework" and no
    deadline.
  From an entry, a user can attach or assess directly, without opening the
  Test or choosing a Framework.
  The list can be filtered by action and by owner: the signed-in user, any
  named user, or "unassigned". Entries are sorted by deadline, earliest
  first, followed by entries without a deadline; ties are sorted by Test
  reference, then requirement name. The list, filters and order are the same
  on every interface.
- **EV-18 (SHOULD)** From a collect entry in the work list, a user can open
  the evidence the requirement held in its most recent earlier period that
  had any, with the item's title, collection date and notes.
- **EV-19 (MUST)** After each action, the response states what it did to
  collection periods, in each Framework of the requirements affected:
  - after an attach: the period each item counts for, whether that is the
    current period, the requirement's new collection state, and every period
    whose assessment became not assessed;
  - after a correction, detach, deletion, collection-date change or
    withdrawal: every period whose assessment became not assessed (EV-14);
  - after an assessment: every period it counts for, and whether that
    includes the current period.
  The information is the same on every interface; the wording may differ.

### 5.6 Status, Health and Coverage

The system tracks four kinds of status:

| Status | Applies to | Values | Set by |
|---|---|---|---|
| **Assessment** | Evidence Requirement | pass, fail, exception; "not assessed" when none applies | a user (EV-12); reset by the system (EV-14) |
| **Implementation status** | Control | Not Implemented, Implemented, N/A | a user, or automatically (CT-3, CT-4) |
| **Health** | Control, per Framework | Green, Yellow, Red | worked out from evidence freshness only (ST-4) |
| **Coverage** | Criterion, per Framework | covered, not covered | worked out from Control results (ST-1 to ST-3) |

Coverage answers "is it passing?"; Health answers "is the evidence
current?". A Control can be Green but not satisfied, when current evidence
was assessed as a fail. It can never be satisfied without current evidence.

- **ST-1 (MUST)** A Test is **passing** only when every active Evidence
  Requirement is done (EV-11) and has a pass as its current assessment for
  the current period (EV-14). A pass from an earlier period never carries
  forward. If any requirement is due, overdue, not assessed, failed or an
  exception, the Test is not passing, and the reason is shown.
- **ST-2 (MUST)** A Control is **satisfied** only when it has at least one
  Test and every one of its Tests is passing. A Control with no Tests is not
  satisfied.
- **ST-3 (MUST)** A Criterion is **covered** only when it has at least one
  linked Control that is not N/A, and every such Control is satisfied. A
  Criterion with none is not covered.
- **ST-4 (MUST)** Each Control that is not N/A has a **Health**, taking the
  worst state across every Evidence Requirement of every linked Test, in the
  Framework being viewed:
  - **Green**: every requirement is done;
  - **Yellow**: none is overdue, but at least one is due;
  - **Red**: at least one is overdue.
- **ST-5 (MUST)** Empty or excluded sets never produce a pass or 100 %
  readiness.
  - A Framework or section with no applicable Criteria shows "no applicable
    requirements", with no percentage.
  - An Applicable Criterion with no active, non-N/A linked Control is not
    covered.
  - An active, non-N/A Control with no active Tests is Red and not satisfied.
  - A Test with no active Evidence Requirements is not passing, does not
    count as an active Test for its Controls, and is listed as a gap.
  - A requirement with no evidence is not assessed. Current evidence that
    has not been assessed is listed as a gap.
  - N/A Controls and inactive records (TS-7) count for nothing toward
    Coverage or risk reduction.
- **ST-6 (MUST)** For each Framework, a **readiness view** shows the share of
  Applicable Criteria (FW-7) that are covered, in total and per section
  (FW-8).
- **ST-7 (MUST)** The system lists **gaps**:
  - Criteria with no linked Controls;
  - Controls with no Tests;
  - Controls that are not N/A and have no owner, or a disabled owner (CT-2);
  - Applicable Criteria of the definition's Statement of Applicability type
    (RP-7) that have neither an inclusion justification nor a Risk linked to
    one of their Controls;
  - Evidence Requirements that are due or overdue, done but not assessed, or
    assessed as a fail or exception.
  For a date range the user chooses, it also lists every ended collection
  period that overlaps the range, is not wholly outside the requirement's
  active range (TS-7), and either has no evidence dated in it or has a
  current assessment of not assessed, fail or exception, even if later
  periods have evidence. This includes requirements ended by deletion
  (DM-11), for periods that started before the deletion.
- **ST-8 (SHOULD)** Every current assessment of fail or exception with no
  linked Finding (FN-1) is listed as a gap. The gap clears when a Finding of
  any status, raised on or after the date the assessment was recorded, is
  linked to that Evidence Requirement.
- **ST-9 (MUST)** The status view shows, for each Criterion, Control, Test
  and Evidence Requirement, both its freshness (Health; done, due or overdue)
  and its assessment (pass, fail, exception or not assessed, rolling up to
  Coverage), wherever each applies (see the table above).
- **ST-10 (SHOULD)** The Control list and the status view can be filtered by
  Control owner, Health and implementation status, in the Framework being
  viewed.
- **ST-11 (MUST)** Every derived state (collection state, Health, Coverage,
  readiness, effective risk) reflects every relevant change by the next read
  on any interface. Relevant changes are to evidence, assessments,
  applicability, links, active ranges, cadences, observation periods, and
  the passing of each day in the organization's time zone. Recalculation
  never changes recorded assessments.

### 5.7 Policies

- **PL-1 (MUST)** Users can create, view, edit and delete Policies. A Policy
  is any controlled document: a policy, a procedure, the ISMS scope
  statement, a method, or an approved Statement of Applicability.
  - A Policy has a name, an owner and one or more versions. Each version has
    a document (file or link), a version identifier, and a status of draft
    or approved.
  - A version is approved only when an approver name and date are recorded.
    The approval applies to that version only (PL-7).
  - The current version is the latest approved one.
  - The system records which user entered each approval; no role can edit
    that value.
- **PL-2 (MUST)** Users can link a Policy to one or more Controls and unlink
  them.
- **PL-3 (MUST)** Policies are for information only: an out-of-date Policy
  does not by itself change Coverage, Health or any other derived state.
  Periodic policy review is checked with an ordinary Test, such as "Annual
  policy review" with one Evidence Requirement per policy.
- **PL-4 (MUST)** Replacing a Policy document keeps earlier versions, with
  their document, date and author.
- **PL-5 (SHOULD)** For any Policy, users can see the Controls it directs.
- **PL-6 (SHOULD)** A Policy can be linked to one Evidence Requirement that
  checks its periodic review (PL-3). A second link is refused. Wherever the
  Policy is listed, and in the document register (RP-9), it shows:
  - the last review date (the newest collection date of evidence on that
    requirement) and the user who added that evidence;
  - the current assessment of that period;
  - the current version's approval date, separately.
  A Policy with no linked requirement shows "no review requirement".
- **PL-7 (MUST)** Each document approval and each risk acceptance (RK-7),
  including its treatment plan, identifies the exact version or values
  approved. Changing those contents or values keeps the earlier approval in
  history and marks the record as awaiting renewed approval. Unrelated edits
  do not affect the approval.

### 5.8 Risks

Risks use the qualitative scales of **NIST SP 800-30 Rev. 1**: likelihood
and impact are each rated Very Low, Low, Moderate, High or Very High, and
the risk level comes from the 800-30 likelihood × impact matrix (Table I-2).

- **RK-1 (MUST)** Users can create, view, edit and delete Risks. A Risk has a
  name, a description and an owner. A Risk without an owner can be saved and
  is flagged in the register.
- **RK-2 (MUST)** Each Risk has an inherent likelihood and impact, before any
  Controls, set by a user. The inherent risk level comes from the matrix.
- **RK-3 (MUST)** Each Risk has a residual likelihood and impact, assuming
  every linked Control is working, set by a user. The residual risk level
  comes from the matrix.
- **RK-4 (MUST)** Users can link a Risk to the Controls that reduce it. Every
  linked Control is required: the residual rating assumes all of them are
  working.
- **RK-5 (MUST)** The system works out each Risk's **effective risk level**:
  - if every linked Control is working, the effective level is the residual
    level;
  - if any linked Control is not working, or the Risk has no linked
    Controls, the effective level is the inherent level, the Risk is flagged
    **elevated**, and the Controls that are not working are named.
  A Control is **working** when it has at least one active Test and, in
  every Framework it serves, each Evidence Requirement of its Tests:
  - is not overdue; and
  - has a pass as the current assessment of the period used. The period used
    is the current collection period if it has a current assessment, and
    otherwise the preceding one.
  Not assessed, fail or exception in the period used means the Control is
  not working. Only evidence within the active range counts (TS-7). A
  Control that serves no Applicable Criterion of a loaded Framework is not
  working.
- **RK-6 (MUST)** An N/A Control does not reduce risk. Linking an N/A Control
  to a Risk is flagged as an inconsistency, and the Control counts as not
  working for RK-5.
- **RK-7 (MUST)** Each Risk records:
  - an owner;
  - a treatment option: Mitigate, Accept, Avoid or Transfer;
  - for any option other than Accept, a treatment plan: planned actions, a
    responsible person and a target date. Linked Controls that are Not
    Implemented are listed as planned actions. A plan may have a completion
    date;
  - the risk owner's approval of the treatment plan and acceptance of the
    residual risk: the owner's name, a date and an optional note, plus the
    user who entered it, which the system records and no one can edit;
  - the date the risk was last assessed, entered by a user.
  The approval applies to the residual rating, option and plan in force when
  it was recorded (PL-7). Changing any of them marks the approval "renewal
  needed" and keeps the earlier approval in history.
  The register and its export (RP-8) show the last-assessed date, and flag
  Risks with no owner, an incomplete plan, a plan past its target date with
  no completion date, or an approval that is missing or needs renewal.
  Nothing is refused.
- **RK-8 (SHOULD)** An organization setting gives the highest residual risk
  level that may be accepted without a treatment plan. The register and its
  export (RP-8) flag every Risk whose option is Accept and whose residual
  level is above that setting. Nothing is refused.
- **RK-9 (MUST)** The system shows a **risk register** listing each Risk with
  its inherent, residual and effective levels and its linked Controls and
  their state, with elevated Risks highlighted.

### 5.9 Findings

- **FN-1 (MUST)** The system keeps a findings log. Each Finding has:
  - a reference and a description;
  - a type: nonconformity, exception, observation or opportunity for
    improvement;
  - for a nonconformity, an optional grade: major or minor;
  - a source: internal audit, external audit, assessment, incident,
    management review or other;
  - the date raised and an owner;
  - optional links to Criteria, Controls and Evidence Requirements;
  - free-text fields for correction, cause, corrective action (including
    management's response) and effectiveness;
  - an optional target date;
  - a status of open or closed, and a closure date.
  The log flags open Findings past their target date, closed nonconformities
  with no cause, and closed Findings with no effectiveness note. Nothing is
  refused. Changes are logged (LOG-1).

### 5.10 Reports and exports

- **RP-1 (MUST)** These rules apply to every export (RP-2 to RP-10, LOG-1,
  AU-11):
  - Any role may produce them. Each export is logged with who, what and
    when.
  - Every export reflects one consistent state, even if an edit happens
    while it runs, or it fails with the reason stated. Separate exports need
    not reflect the same state.
  - Every CSV starts with a header giving the export date and time, the time
    zone and the user who produced it, as well as that export's own header
    values.
  - In every CSV, no cell is treated as a formula when opened in a
    spreadsheet, values stay readable, and delimiters, quotes and line
    breaks inside values do not shift columns.
- **RP-2 (MUST)** A **coverage report** exports current state as CSV, per
  Framework.
  - One row per Criterion × Control × Test × Evidence Requirement. Criteria
    with no Controls, and Controls with no Tests, appear with the missing
    columns empty.
  - Columns:
    - Criterion: reference, text, applicability and justification;
    - Control: reference, name, description, owner and implementation
      status;
    - Test: reference, name, cadence and procedure;
    - Evidence Requirement: name, collection state and current-period
      assessment.
  - Header: Framework, definition version, observation period, time zone,
    and evaluation date and time.
- **RP-3 (MUST)** An **evidence-by-period report** exports, for a chosen
  Framework and date range, a CSV covering every collection period that
  overlaps the range.
  - One row per Criterion × Control × Test × Evidence Requirement × period ×
    Evidence item.
    - A period with no item has one row marked "no evidence".
    - A period wholly outside the active range has one row marked "not in
      operation". If such a period holds evidence or an assessment, those
      rows stay, marked as TS-7 describes.
    - Records ended by deletion (DM-11) appear, marked deleted, for periods
      that started before the deletion.
  - Columns:
    - Criterion (labelled as in FW-9): reference, applicability and
      justification;
    - Control: reference and name;
    - Test: reference, name, procedure, and the cadence in force for the
      period;
    - Evidence Requirement: name and collection instructions;
    - period: start and end;
    - Evidence item: reference, current version, title or address, kind,
      notes, and the original file name. For a statement, the full text of
      the current version and of each version the current assessment judged;
    - the collection date, the earlier collection date where it changed
      (EV-5), the date added and who added it (EV-6);
    - the period's current assessment: result, assessor, interface and
      bearer token (SEC-8), recording time, note, and the versions it judged.
      Where the period is not assessed because of a reset, the reset's
      reason, time and user (EV-14);
    - the number of earlier assessments in the period.
  - Header: Framework, definition version, each observation period that
    overlaps the range, every cadence or observation-period change (PD-5)
    made in the range with its date, the time zone, and a statement that
    links are shown as of the export.
  - A range of one date covers the period that contains it.
- **RP-4 (SHOULD)** The RP-3 report can also be produced grouped by Control:
  one row per Control × Test × Evidence Requirement × period × Evidence
  item, with the linked Criterion references of the chosen Framework in one
  cell.
- **RP-5 (SHOULD)** RP-3, RP-4 and RP-6 can be limited to chosen Criteria or
  Controls of the chosen Framework, as well as to a date range. An RP-6
  download uses the same selection as its RP-3 report. The header states the
  selection.
- **RP-6 (MUST)** The RP-3 report can be downloaded together with the stored
  files behind it, all from one consistent state.
  - The download holds one file per distinct stored version the report
    shows: each item's current version, and every version judged by an
    assessment the report includes. Each file appears once, even if it
    serves several rows.
  - Each file is named by Evidence reference and version, followed by the
    original file name reduced to letters, digits, ".", "-" and "_", with no
    path.
  - Links and statements appear in the CSV only.
  - A purged or unavailable version is left out, and its row says so.
  - Where SEC-11 is delivered, a fingerprint index covers every file in the
    download.
- **RP-7 (MUST)** A **Statement of Applicability** exports as CSV for any
  Framework whose definition names a Statement of Applicability type (FW-3).
  The shipped ISO 27001 definition names Annex A.
  - One row per Criterion of that type, whether from the definition or added
    by the organization.
  - Columns: ID, title, applicability, justification, implementation status
    (worked out as below), linked Control references and names, and the
    references of Risks linked to those Controls. Column labels follow FW-9.
  - Header: Framework, definition version, export date and time, and time
    zone.
  - Implementation status:
    - Not applicable: the Criterion is Not applicable;
    - Implemented: at least one non-N/A Control is linked and all of them
      are Implemented;
    - Partially implemented: some, but not all, are Implemented;
    - Not implemented: otherwise.
- **RP-8 (MUST)** The following export as CSV:
  - the **risk register**: reference, name, owner, inherent, residual and
    effective levels, the elevated flag, linked Control references and their
    state, and the RK-7 fields;
  - the **findings log** (FN-1).
- **RP-9 (MUST)** A **document register** exports as CSV, with one row per
  non-deleted Policy:
  - reference, name and owner;
  - the current approved version identifier, its approver, its approval date
    and the user who entered the approval (PL-1);
  - whether a newer draft exists;
  - linked Control references;
  - a state: "no approved version", "awaiting renewed approval" (PL-7) or
    "approved";
  - where PL-6 is delivered, the last review date and its current
    assessment.
- **RP-10 (SHOULD)** The document register (RP-9) can be downloaded together
  with the file of each listed Policy's current approved version. Files are
  named and indexed as in RP-6. A Policy whose current approved version is a
  link appears in the CSV only.
- **RP-11 (MAY)** The readiness view and the coverage report (RP-2) can be
  produced as of a chosen past date, using the assessments, evidence dates
  and applicability in force on that date. The output states that date and
  that current links were used.
- **RP-12 (MAY)** The system notifies owners of Evidence Requirements that
  are due or overdue.

### 5.11 Audit trail

- **LOG-1 (MUST)** The system records every creation, change and deletion
  of:
  - Frameworks, Criteria, applicability and observation periods;
  - Controls and implementation status, including automatic changes (CT-4),
    which are attributed to the system;
  - Tests and cadences;
  - Evidence Requirements, owners and active ranges;
  - Evidence, including versions, collection dates, link addresses,
    attaching, detaching, deletion and purge;
  - assessments, approvals and acceptances;
  - Policies, Risks, Findings and links;
  - global settings.
  Each entry records the actor, the bearer token used if any, the time, the
  interface, the record, and the values before and after.
  - Entries can only be added. They survive deletion and purge of the
    record, never contain a secret, and are kept indefinitely.
  - On every interface, a user can list these entries together with the
    security events (LOG-2), filtered by date range, record type, record
    reference, actor and interface. A user can also open any record's
    history from the record.
  - The same filtered list exports as CSV (RP-1), one row per entry: entry
    number, time, actor, interface, source address, bearer token, record
    type, record reference, action, and values before and after. Entry
    numbers follow the recording order and are never reused or changed. The
    list and the export show where an entry is missing from the sequence.
- **LOG-2 (MUST)** The system records these security events, each with the
  actor, time, interface and source address:
  - sign-in success and failure (a failure for an unknown account is
    recorded as "unknown account", without the name tried), and sign-out;
  - password changes and resets;
  - bearer-token creation and revocation;
  - requests refused for an invalid, expired or revoked bearer token, naming
    the token where it can be identified, never the secret presented;
  - user creation, role change, disabling and re-enabling;
  - changes to security and global settings;
  - exports, backups, restores, upgrades, integrity-check failures and
    purges;
  - actions refused for lack of permission.
  No role can edit or delete an entry on any interface. Entries are never
  removed automatically and are kept indefinitely. No entry contains a
  secret.
- **LOG-3 (SHOULD)** The system records each download or opening of an
  Evidence or Policy file version, with the user, bearer token, time,
  interface, item reference and version. These records can be viewed and
  exported with the audit log (LOG-1).

### 5.12 Users, sign-in and roles

- **AU-1 (MUST)** Every interface requires sign-in, including CLI and MCP
  sessions started on the server itself. The only exceptions are the
  operator paths for the first admin and recovery (AU-4).
- **AU-2 (MUST)** There are three roles: **admin**, **contributor** and
  **viewer**. There are no per-record permissions.
- **AU-3 (MUST)**
  - A **viewer** can read everything, including deleted items not yet purged
    and the audit log; produce exports (RP-1); run integrity checks (SEC-11,
    where delivered); and manage their own password and bearer tokens
    (AU-9).
  - A **contributor** can do everything a viewer can, and also create, edit
    and delete (EV-9) Controls, Tests, Evidence Requirements, Evidence,
    Policies, Risks, Findings, links, assessments and implementation status.
    A contributor can record a risk acceptance on behalf of the named owner;
    the record keeps both names (RK-7).
  - An **admin** can do everything a contributor can, and also manage users
    and roles, revoke any bearer token, manage Frameworks (load definitions,
    edit Criteria, set applicability and observation periods), change
    global and security settings, purge (EV-9), and back up and restore
    (NFR-6).
  - No role can edit or delete audit entries or times recorded by the
    system. Ownership assigns work; it grants no permission.
- **AU-4 (MUST)**
  - The first admin can be created only by someone with access to the server
    itself, never over the network, and with no credential shared between
    installations. This path closes once an admin exists.
  - A documented recovery lets someone with administrator access to the
    server's operating system restore admin access without losing records.
    The recovery is logged (LOG-2).
- **AU-5 (MUST)** A user who has recorded any action is disabled, never
  deleted. Usernames cannot be changed or reused, and past actions show the
  username. A disabled user stays the owner of what they owned, and those
  items appear as gaps marked "owner disabled". The system refuses to
  disable or demote the last active admin.
- **AU-6 (MUST)** Users sign in with a username and password.
  - The API has a sign-in endpoint that checks the username and password and
    returns a signed token (JWT) recording when the user signed in. The web
    UI and the CLI sign in through it and send the token with each request.
  - A sign-in token ends after an idle time (default 30 minutes) and after a
    maximum lifetime (default 12 hours). An admin can change both.
  - Signing out ends the token in the system, so it is refused from then on,
    not only once it expires.
  - When users change their own password, their other sign-in tokens end.
    Their bearer tokens stay valid, and the response lists them with their
    creation and last-use dates.
  - An admin password reset, or disabling the user, ends all of that user's
    sign-in tokens and revokes their bearer tokens.
  - A role change applies from the next request, on every interface.
  - Nothing a client held before sign-in grants access after sign-in.
  - After an admin resets a user's password, the user must set a new one
    before their next sign-in completes.
- **AU-7 (MUST)** Passwords are stored one-way: two users with the same
  password have different stored values, and recovering a password from a
  copy of the storage requires guessing it. Passwords:
  - must meet an admin-set minimum length (default 12 characters);
  - may be at least 64 characters long;
  - are refused if they appear on a common-password list shipped with
    Lifeboat.
  No password, bearer token or sign-in token appears in any log, audit
  entry, export, error message or URL. Backups hold credentials only in
  stored form.
- **AU-8 (MUST)** Repeated failed sign-ins for an account are slowed or
  temporarily refused. Each refusal lasts a limited time, and failures from
  one source do not stop the owner signing in from another source once that
  time has passed. The sign-in response does not reveal whether the account
  exists or is disabled.
- **AU-9 (MUST)** The MCP server, and other applications that use the API,
  sign in with **bearer tokens**, separate from passwords. Each bearer token
  belongs to one user, and a user may have several. Each token:
  - has a name given by its user, shown in every audit entry made with it;
  - can be created only as AU-10 allows;
  - acts with its user's current role;
  - expires within an admin-set maximum (default 90 days). Lowering the
    maximum makes each existing token expire no later than its creation date
    plus the new maximum, or at once if that date has passed;
  - has a secret shown once, at creation;
  - shows when it was created, when it expires and when it was last used;
  - can be revoked by its user or by any admin, from the next request on.
  The CLI never requires a password or secret on the command line. Any token
  or secret the CLI stores can be read only by the operating-system user who
  stored it.
- **AU-10 (MUST)** These actions require a recent sign-in:
  - creating a bearer token;
  - changing one's own password;
  - creating a user, changing any user's role, re-enabling a user, or
    resetting another user's password;
  - purging (EV-9);
  - making or restoring a backup through the application (NFR-6);
  - setting a security setting weaker than its default.
  A sign-in token older than an admin-set age (default 15 minutes) cannot
  perform them: the API refuses and says a new sign-in is needed, and the
  web UI and CLI then ask for the password, sign in again and retry. A bearer
  token can never perform these actions, and the MCP server refuses them.
  Backups and recovery done by the operator on the server (AU-4) are not
  affected.
- **AU-11 (MUST)** Any role can view, and export as CSV (RP-1):
  - every user, with role, active or disabled status, creation date and last
    sign-in;
  - every bearer token, including expired and revoked ones, with its user,
    name, creation date, expiry, revocation date and last use.
  No secret is ever shown.

### 5.13 Interfaces

- **IF-1 (MUST)** The HTTP API (IF-3) and the MCP server are the primary
  interfaces. The web UI and the CLI work through the HTTP API, with no other
  path to the data.
- **IF-2 (MUST)** All four interfaces (web UI, CLI, MCP server and HTTP API)
  offer the same capabilities and enforce the same sign-in and roles
  (§5.12). Anything a user can do in the web UI, they can do from the CLI,
  the MCP server or the API with the same role, except that the MCP server
  refuses the actions listed in AU-10.
- **IF-3 (MUST)** The HTTP API serves the web UI, the CLI and other
  applications alike, and is documented for other applications to use.
  - It accepts sign-in tokens (AU-6) from the web UI and the CLI, and bearer
    tokens (AU-9) from other applications. SEC-3 applies to every request
    from the web UI.
  - Audit entries and assessments record which interface was used: web UI,
    CLI, MCP or API (SEC-8). For a sign-in token, whether it is the web UI
    or the CLI is as stated by the client at sign-in.
  - Every rule this document sets for the CLI and MCP server also applies to
    the API.
- **IF-4 (SHOULD)** In its description of each capability, the MCP server
  marks every capability that changes or deletes data. Its read
  capabilities never change data.

---

## 6. Security

- **SEC-1 (MUST)** Every network connection to the web UI, CLI, MCP server
  and API is encrypted.
  - An unencrypted connection is accepted only from the server itself, or
    after an admin explicitly allows it. While it is allowed, the status page
    shows "network (unencrypted)".
  - Encryption may be provided by Lifeboat or by a front end the operator
    runs.
  - The CLI refuses a connection if it cannot verify the server's identity,
    either through a trust source the operator configures or through an
    identity the user has explicitly trusted. It never silently accepts a
    failed check.
- **SEC-2 (MUST)** A new installation:
  - has no default or shared credentials;
  - cannot be reached from another machine until an admin allows network
    access;
  - has every security setting at its documented default;
  - creates its data, stored files and backups so that other
    operating-system users cannot read them, except accounts the operator
    explicitly grants access to.
  The status page lists every security setting weaker than its default.
- **SEC-3 (MUST)** The web UI refuses any state-changing request that another
  site causes the user's browser to send, and its pages cannot be shown
  inside another site's frame.
- **SEC-4 (MUST)** All text entered by users or imported is shown as text on
  every interface and never runs as active content. This covers names,
  descriptions, notes, Criterion text from definition files, Findings and
  statements.
- **SEC-5 (MUST)** Uploads:
  - An uploaded file never runs as active content in the web UI. Opening it
    either downloads it or displays it so that no script in it can act with
    the user's session.
  - The maximum sizes for an Evidence upload, a Policy upload, a statement, a
    definition file and an import file, and the maximum number and total
    size of files in one submission, are stated, have defaults, can be
    changed by an admin, and are enforced on every interface.
  - A submission that is too large, invalid or interrupted fails, names the
    limit or reason, and leaves no partial item, attachment or definition.
  - A file's name, type or content never makes it replace or change another
    stored item, or any file outside the store. The original file name is
    kept as data only.
- **SEC-6 (MUST)** Links:
  - Link Evidence and Policy links accept only http and https addresses.
  - The system never fetches a link's content.
  - Every item is marked as a file, link or statement wherever it appears,
    including exports.
  - Changing a link address creates a new version (EV-6).
  - Wherever a link is offered for opening, its full address is shown, not
    only its title.
- **SEC-7 (MUST)** Bulk and batch operations (loading definition files,
  attaching to several requirements at once, and scripted CLI, MCP and API
  calls) enforce the same role checks, validation, limits and EV-13 rule as
  single operations, and write one audit entry per record changed. Loading
  a file never records assessments.
- **SEC-8 (MUST)** Every assessment records three values that the system
  sets and no role can edit: the signed-in user as assessor, the interface
  used, and the bearer token used, if any. The period view (EV-14) and RP-3
  show all three.
- **SEC-9 (MUST)** Evidence requests (TS-8) go only to https addresses, with
  two exceptions: an http address on the server itself, and any http
  address after an admin has explicitly allowed unencrypted requests. While
  an http address is in use, the status page shows "evidence requests
  (unencrypted)". For every request, the system:
  - verifies the receiver's identity in the same way as the CLI (SEC-1);
  - never follows a redirect;
  - ends each delivery attempt within a stated time limit;
  - records only "delivered" or "not delivered" with the status returned,
    never the content of the response.
  Setting or changing an address is admin-only on every interface, including
  in definition files (FW-2), and is logged with old and new values (LOG-1).
- **SEC-10 (MUST)** Lifeboat makes no outbound network connections other
  than evidence requests (TS-8). It sends no usage data and does not check
  for updates on its own.
- **SEC-11 (SHOULD)** The system records a fingerprint of each stored file
  version when it is added, and shows it on the item, in the RP-3 report and
  in the RP-6 index.
  - A user can check whether stored files still match their fingerprints. A
    mismatch names the item and version and is logged (LOG-2).
  - The recipient of an export can check each file against the index
    without access to Lifeboat. The RP-6 index covers every version in the
    download.
  - The audit entry for each added file version (LOG-1) includes its
    fingerprint.
- **SEC-12 (SHOULD)** The status page and the CLI show the running version.
  Release notes mark the changes that fix security defects.
- **SEC-13 (MUST)** Lifeboat ships with a security description that states:
  - how sign-in and sign-in tokens work;
  - what each role can do, and that every role can read all evidence,
    policies and the audit log;
  - what the audit log records, and that it is protected against changes
    through the application only: someone who controls the server can
    rewrite it, which can be detected only by comparing with an earlier
    export;
  - how integrity is checked, if SEC-11 is delivered;
  - every security default;
  - that Lifeboat does not encrypt backups;
  - that a purge does not remove content from backups made before it;
  - that uploaded files are not scanned for malicious content;
  - that times recorded by the system come from the server clock;
  - that, for sign-in tokens, the interface recorded (web UI or CLI) is as
    stated by the client;
  - that text returned through the API and MCP server was entered by users
    and may contain instructions aimed at AI tools, so each application
    should have its own user with the lowest role it needs;
  - how security fixes are announced and which versions receive them
    (SEC-12);
  - what the operator must provide: a front end for encryption if one is
    used, protection of stored data, protected backup storage, correct time,
    and protection of the server.

---

## 7. Non-functional requirements

- **NFR-1 (MUST)** One organization per installation, with no tenant
  separation.
- **NFR-2 (MUST)** Lifeboat is self-hosted and self-contained: it needs no
  external SaaS, AI or LLM service to run.
- **NFR-3 (MUST)** Data is stored durably, and uploaded Evidence and Policy
  files can always be retrieved.
- **NFR-4 (SHOULD)** Lifeboat handles at least 300 Controls, 300 Tests, 5,000
  Evidence items and 10 named users, with every list view returning in under
  two seconds on the server resources stated in the documentation.
- **NFR-5 (MUST)** Framework definitions and starter data are JSON data
  files, not code.
- **NFR-6 (MUST)** A documented, tested backup and restore procedure
  recovers a consistent copy of everything into a clean, compatible
  installation: records and references, files and earlier versions, deleted
  items not yet purged, assessment and change history, the audit log,
  fingerprints (SEC-11), Framework customizations, users and settings.
  - Backup and restore require admin or operator access, report success or
    failure, and never report an incomplete restore as successful.
  - The procedure states how recent the recovered data is guaranteed to be.
  - Link items keep their address; the linked content is not kept.
- **NFR-7 (MUST)** A restore:
  - records in the restored audit log who restored, when, and from which
    backup;
  - ends every sign-in token and revokes every bearer token, so users sign in
    again and each user and connected application needs a new bearer token;
  - where the installation being replaced is still readable, keeps the audit
    entries made after that backup retrievable by an admin, and lists every
    purge recorded after the backup, with its item and reason, so the admin
    can repeat them.
- **NFR-8 (MUST)** Each operation either completes all its changes or makes
  none. Permissions and assessment rules are checked against the state
  actually changed, on all four interfaces. If evidence changes while an
  assessment is being submitted, the assessment is either refused or
  recorded only as a judgment of the earlier evidence. When two users edit
  the same descriptive field at once, the later edit wins and both appear in
  history.
- **NFR-9 (MUST)** A supported upgrade keeps all records, content,
  references, links, assessments with the evidence they judged, approvals,
  settings, history and the audit log, and is itself logged. An unsupported
  upgrade is refused before any data changes. Any change in how a derived
  state is calculated is stated in the release notes and never rewrites
  recorded assessments.
