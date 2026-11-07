---
inclusion: always
---

# Change Management & Version History

## 1. Core Principle

Every generated feature specification and its implementation must maintain a version history.

The initial generated specification is Version 1 (V1).

Any subsequent changes to the requirements, design, scope, or implementation requested after the initial specification has been generated must be recorded as a new version.

## 2. Versioning Rules

- Initial specification: V1
- First requested modification: V2
- Second requested modification: V3
- Continue incrementing sequentially.
- Never overwrite or silently discard previous decisions.
- Do not increment the version for typo fixes or formatting corrections.
- Group related changes from a single revision request into one version.

## 3. Changelog

Every specification must have a CHANGELOG.md file in its spec directory.

Example:

specs/
  feature-name/
    requirements.md
    design.md
    tasks.md
    CHANGELOG.md

The changelog must record:

- Version number
- Date of change
- Summary of changes
- Reason for change
- Affected requirements
- Affected design components
- Affected implementation tasks

## 4. Revision Workflow

Whenever a user requests changes to an existing specification:

1. Read the current requirements, design, tasks, and changelog.
2. Identify the current version.
3. Determine which requirements, designs, and tasks are affected.
4. Increment the version number.
5. Update the relevant specification documents.
6. Update affected implementation tasks.
7. Append a changelog entry.
8. Preserve previous version information.

Do not generate a completely new specification when the request is a revision of an existing feature.

## 5. Implementation Changes

Changes requested after implementation must also be recorded.

The changelog must distinguish between:

- Specification changes
- Design changes
- Implementation changes
- Bug fixes
- Breaking changes

A code change does not automatically require a specification version increment unless it changes the agreed behaviour, scope, or design.

## 6. Changelog Format

Use this format:

# Changelog

## V1 - Initial Release
- Initial requirements and design generated.
- Initial implementation scope established.

## V2 - YYYY-MM-DD
### Changes
- Description of changes.

### Reason
- Why the changes were requested.

### Impact
- Affected requirements, design components, and tasks.

## 7. Important Rules

- Never silently modify an approved specification.
- Never remove historical changelog entries.
- Never claim a previous version was implemented unless verified.
- Keep version numbers consistent across related specification documents.
- Ask for clarification when a requested revision conflicts with an existing requirement.
- Treat approved changes as the new source of truth while retaining historical records.