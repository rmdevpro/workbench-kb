# Roles

Role files define the purpose, domain, and working principles for a specific type of session. They are the required prefix for all plan files.

## How Role Files Are Used

Every plan file begins with a role description copied from here. The role section describes what the session is, what it is responsible for, and how it works. It does not change as work progresses.

After the role section, a plan file may include dynamic content — active work items, findings, decisions, and progress notes. That section evolves as the session works. The role section does not.

**Plan file structure:**
```
# [Role Title] — [Project or Task Name]

[Role description — copied from the relevant role file, with project-specific
placeholders filled in]

---

[Optional: dynamic plan content below the line — active tasks, findings,
decisions. This section changes. The role section above does not.]
```

## Available Roles

| File | Role |
|---|---|
| `software-developer.md` | Implements features and fixes bugs |
| `software-tester.md` | Executes test plans and reports findings |
| `devops-engineer.md` | Manages deployment, infrastructure, and operations |
| `project-manager.md` | Coordinates work, tracks issues, and manages delivery |
| `document-manager.md` | Creates, reviews, and edits project documentation |

## Project-Specific Placeholders

Some role files contain placeholders (e.g. `[PROJECT_REPO]`) that must be filled in when the role is used in a plan file. Replace all placeholders with the actual values for the project before committing the plan file.
