RADAR-base RFCs
================

Purpose
-------
This repository hosts the Request for Comments (RFC) process for proposing and collaborating on substantial changes to RADAR-base. RFCs are lightweight design documents that capture motivation, design, and impact of changes across components.

Quick start
-----------
1. Fork this repository.
2. Create a new branch for your RFC.
3. Copy `rfcs/0000-template.md` to a new file `rfcs/NNNN-your-title.md` where `NNNN` is the next available RFC number. Place it under an area folder if applicable (e.g., `rfcs/platform/NNNN-your-title.md`, `rfcs/backend/NNNN-your-title.md`, `rfcs/mobile/NNNN-your-title.md`, `rfcs/data-analytics/NNNN-your-title.md`, or `rfcs/general/NNNN-your-title.md`).
4. Fill in all sections.
5. Open a Pull Request using the provided template.
6. Apply labels: one `status:` label and at least one `area:` label (see below). Labels are auto-suggested based on file paths.
7. Link any related issues or discussions directly in the RFC body, such as Pre-RFC discussions (e.g., `RADAR-base/<repo>#<123>`) using the `Discussion` field in the RFC template.

Lifecycle
---------
- Draft: PR opened, under active review.
- Accepted: Consensus reached and approved by maintainers.
- Rejected: Not accepted (with rationale).
- Superseded: Replaced by a later RFC.
- Implemented: Shipped in codebases (link references to PRs/releases).

Index
-----
All merged RFCs live in `rfcs/`. Add an entry to the table below in your PR.

| Number | Title | Status | Author | Link |
|--------|-------|--------|--------|------|
| 0001 | Sample RFC – Example Structure and Content | Draft | RADAR-base Team | rfcs/0001-sample-rfc.md |
| 0002 | Just In Time Interventions Platform | Draft | RADAR-base Team | rfcs/platform.0002-just-in-time-intervention-platform.md |

Governance
----------
The RFC process is facilitated by the RADAR-base maintainers. See `CONTRIBUTING.md` for details on roles, timelines, and decision procedures.

Areas and ownership
-------------------
- Area folders: `rfcs/platform/`, `rfcs/backend/`, `rfcs/mobile/`, `rfcs/data-analytics/`, `rfcs/general/`
- CODEOWNERS: PRs touching these folders auto-request domain reviewers
- Labels: use one of `area: platform`, `area: backend`, `area: mobile`, `area: data-analytics`, `area: general`

Status and label requirements
-----------------------------
- One `status:` label is required: `status: draft | accepted | rejected | superseded | implemented`
- At least one `area:` label is required
- PRs without required labels will be flagged by automation


