---
inclusion: always
---

# Development Workflow

Apply this order to every change request: **Design and Task Updates → Requirement Consistency → Specification Isolation**.

Spec locations:
- Working specs: `.kiro/specs/<feature>/` (`requirements.md`, `design.md`, `tasks.md`, `CHANGELOG.md`).
- Isolated, versioned specs: root `specs/` folder.

## 1. Design and Task Updates
- Reflect every requested change in the `design.md` and `tasks.md` of the affected spec.
- Update `requirements.md` too when behavior or scope changes.
- Do this automatically; do not wait for the user to ask.
- Follow the `change-management` steering for version increments and CHANGELOG entries.

## 2. Requirement Consistency
- After the change, check requirements, design, and tasks for contradictions or gaps: conflicting rules, orphaned tasks, requirements without design coverage.
- If any are found, stop and notify the user before continuing. Present options, a recommendation, and the reasoning behind it.
- Proceed to step 3 only when requirements are consistent.

## 3. Specification Isolation
- Translate the consistent requirements from `.kiro/specs/` into the root `specs/` folder, separated by domain and by version.
- Never delete or overwrite earlier version folders. Put changes under the new version folder.
- Keep the existing domain file naming `NN-DOMAIN.md` (e.g. `00-OVERVIEW`, `01-INVENTORY`).
- A version folder contains only the domains changed in that version, except `v1`, which holds the full baseline.
- Keep `specs/CHANGELOG.md` consistent with the version numbers in the spec-level changelogs.

Expected output:

```text
specs/
├── v1/
│   ├── 00-OVERVIEW.md
│   └── ...            # one file per domain
├── v2/
│   └── [domain files that changed in v2]
└── CHANGELOG.md
```

## 4. Requirement Creation (New Specs)
1. Read the requirements of all existing specs first.
2. Flag any inconsistencies or overlaps with the new requirement.
3. Ask the user, with a recommendation on how to refine, before creating the requirement.
