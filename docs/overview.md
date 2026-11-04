# Lifeboat: Overview

Lifeboat is a self-hosted, open-source GRC (governance, risk and compliance)
application. One installation serves one organization. It tracks what each
compliance framework requires, which of the organization's controls meet
those requirements, how each control is tested, and the evidence behind each
test. From that it shows, at any moment, what is passing, what is missing and
what is about to fall due.

This page is a short introduction. The full requirements are in
[requirements.md](../requirements.md).

## Principles

- **Frameworks are data.** SOC 2 and ISO 27001 ship as JSON definition files.
  Adding a framework means adding a file, not writing code. The application
  knows nothing about any framework beyond what its file contains.
- **One control, many frameworks.** A control is written once and can meet
  requirements in several frameworks at the same time.
- **API and MCP first.** The HTTP API and the MCP server carry every
  capability. The web UI and the CLI use the API, so a person, a script and an
  AI tool can all do the same things with the same permissions.
- **Evidence collection stays outside.** Lifeboat has no built-in connectors.
  Other tools, including AI tools, push evidence and assessments in through
  the API or MCP using a user's bearer token. Lifeboat can send an evidence
  request to another application's web address when evidence falls due.
- **Strict passing.** Nothing passes on good intentions. A test passes only on
  current evidence that someone has assessed as a pass, and a pass never
  carries over from an earlier period.
- **Simple to run.** Single VM, no external SaaS or AI service required, no
  outbound connections except evidence requests.

## Data model

```
Framework ──1:n── Criterion ──n:m── Control ──n:m── Test ──1:n── Evidence Requirement ──n:m── Evidence
                                      │  │
                         Policy ──n:m─┘  └─n:m── Risk
```

| Record | What it is |
|---|---|
| **Framework** | A compliance standard, loaded from a definition file. Has an observation period (for example a SOC 2 audit period) with start and end dates set by an admin. |
| **Criterion** | One requirement of a framework, such as SOC 2 `CC6.1` or ISO 27001 `A.5.1`. Belongs to one framework. Can be marked not applicable, with a justification. |
| **Control** | A safeguard the organization operates, with an owner. Linked to criteria in any number of frameworks. |
| **Test** | A reusable procedure that checks one or more controls. Has a cadence: evidence every 1, 3, 6 or 12 months. |
| **Evidence Requirement** | A named slot on a test that must be filled, such as "AWS" or "Okta" in a quarterly access review. A simple test has one; a per-system test has one per system. |
| **Evidence** | A file, link or written statement, with the date it was collected. Kept in a shared repository; one item can fill slots on several tests. |
| **Assessment** | A judgment on an evidence requirement: pass, fail or exception, with who recorded it and which evidence it judged. |
| **Policy** | A controlled document (policy, procedure, scope statement), versioned, linked to the controls it directs. |
| **Risk** | An entry in the risk register, rated for likelihood and impact, linked to the controls that reduce it. |
| **Finding** | A nonconformity, exception, observation or improvement opportunity, with its corrective action. |

Every record has a permanent reference (such as `CTL-014`) that never changes
and is never reused. Deleting a record takes it out of use but keeps its
history.

## How status is worked out

Time is divided into **collection periods**: windows of the test's cadence,
counted from the start of the framework's observation period. A quarterly
test in a calendar-year audit period has four.

Each period of each evidence requirement is in one of four **collection
states**:

| State | Meaning |
|---|---|
| not yet due | a future period |
| due | the current period, no evidence dated in it yet |
| done | evidence dated in the period has been added |
| overdue | the period ended with no evidence dated in it |

Separately, a user records an **assessment** of the evidence. The two answer
different questions:

- **Health** (Green, Yellow, Red) asks "is the evidence current?" A control is
  Green when all its evidence is done, Yellow when something is due, Red when
  something is overdue.
- **Coverage** asks "is it passing?" It rolls up strictly:
  - a **test passes** when every evidence requirement is done and assessed as
    a pass for the current period;
  - a **control is satisfied** when it has at least one test and all of them
    pass;
  - a **criterion is covered** when it has at least one control and all of
    them are satisfied.

Empty sets never pass: a control with no tests, or a criterion with no
controls, is not satisfied or covered.

**Risks** follow the controls. Each risk has an inherent level (no controls)
and a residual level (all linked controls working), on the NIST SP 800-30
scales. While every linked control is satisfied the risk sits at its residual
level; if any is not, it reverts to its inherent level and is flagged
elevated, naming the controls at fault.

## Example

The organization runs a quarterly access review covering AWS and Okta.

1. The SOC 2 audit period is set to 1 January to 31 December, so the review
   has four collection periods.
2. Control `CTL-014` "User access is reviewed quarterly" meets SOC 2 `CC6.2`
   and ISO 27001 `A.5.18`. Test `TST-009` "Quarterly access review" is linked
   to it, with two evidence requirements: AWS and Okta.
3. Fifteen days before the end of Q1, Lifeboat sends an evidence request to
   the team's automation. It uploads both exports through the API.
4. Both requirements are now done for Q1. A reviewer (or an AI tool acting
   under its own user account) assesses each as a pass.
5. The test passes, the control is satisfied and Green, and `CC6.2` and
   `A.5.18` are covered. If the Okta export had been missing on 31 March, the
   control would turn Red, the criteria would show as not covered, and any
   risk relying on that control would be flagged elevated.

## Users and access

Username and password sign-in, with three roles: **viewer** (read and
export), **contributor** (create and edit records, evidence and assessments)
and **admin** (users, frameworks and settings). Other applications use a
bearer token tied to a user and get exactly that user's role. Every change is
recorded in an audit log.

## Reports

CSV exports for auditors: coverage, evidence by collection period, Statement
of Applicability, risk register and findings log, document register, audit
log, and users and tokens. Evidence and approved policy files can be
downloaded as a bundle.

## Not in version 1

Multi-tenancy, SSO and second-factor sign-in, built-in connectors or AI,
auditor workpapers, vendor management, quantitative risk scoring, PDF reports,
and notifications beyond evidence requests.
