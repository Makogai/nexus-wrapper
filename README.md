# Nexus-wrapper

This folder is a NEXUS workspace.

* `nexus-workspace.json` — portable workspace manifest (identity, schema, template, projects).
* `constitutions/`, `skills/`, `workflows/`, `integrations/` — how Claude works here.
* `knowledge/`, `project-context/` — decisions, notes, specs and per-project knowledge.
* `projects/` — default location for cloned project repositories (ignored by Git).

The workspace can be a Git repository so it follows you to other machines. It never contains
credentials, session transcripts or the NEXUS database.
