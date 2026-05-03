# Workbench Knowledge Base

The canonical knowledge base for Workbench installations. Contains roles, guides, process documents, runbooks, requirements, and standards.

## Structure

| Directory | Contents |
|-----------|----------|
| `roles/` | Agent role definitions — seed Claude session plan files at launch |
| `guides/` | How-to guides for using the Workbench |
| `process/` | Team process documents (debugging, feature development, runbook execution) |
| `runbooks/` | Step-by-step operational procedures |
| `requirements/` | System requirements documents |
| `standards/` | Work product standards (requirements, HLD, test plans, code, test code) |

## Customising for Your Organisation

Fork this repo into your organisation's account. Point your Workbench install at your fork via Admin Settings → Knowledge Base. Your customisations stay on your fork; pull upstream changes with:

```bash
git fetch upstream
git rebase upstream/main
```
