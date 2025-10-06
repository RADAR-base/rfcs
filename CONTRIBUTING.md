Contributing to RADAR-base RFCs
===============================

Scope
-----
Use RFCs for substantial, cross-cutting, or risky changes that benefit from design discussion: new services, breaking API changes, architectural refactors, data model changes, or security/privacy-affecting work. Smaller, incremental changes can go straight to code repositories.

Roles
-----
- Author: Writes the RFC, owns iteration.
- Reviewers: Subject-matter experts providing feedback.
- Maintainers: Facilitate process, ensure quality, decide acceptance.

Process
-------
1. Open a "Pre-RFC discussion" issue to validate scope and gather early feedback.
2. Fork and branch. Copy `rfcs/0000-template.md` to `rfcs/NNNN-title.md`. If the RFC clearly belongs to an area, place it under `rfcs/platform/`, `rfcs/backend/`, or `rfcs/mobile/`.
3. Fill all sections. Keep the tone clear and concise; prefer diagrams where helpful.
4. Open a Pull Request using the RFC PR template. Link the pre-RFC issue.
5. Incorporate feedback. Major unresolved concerns should be documented under "Open questions".
6. Maintainers assign status: Accepted/Rejected/Superseded when ready.
7. After implementation, send a follow-up PR to mark status as Implemented and link code PRs/releases.

Timelines
---------
Typical review cycles run 1–2 weeks for medium changes, longer for large/complex proposals. Urgent items can be expedited by maintainers.

Decision making
---------------
Consensus seeking. Maintainers hold final call; dissenting opinions are recorded with rationale.

Style
-----
- Keep RFCs self-contained. Use headings from the template.
- Use clear language; prefer examples.
- Include diagrams in `rfcs/media/` and link them.

Labels and reviewers
--------------------
- Required labels: one `status:` label and at least one `area:` label. PR checks will fail without them.
- Auto-labeling: PRs are labeled based on file paths under area folders.
- Reviewers: Domain reviewers are auto-requested via CODEOWNERS and workflow automation.

License
-------
By contributing, you agree that your contributions are licensed under the repository's `LICENSE`.


