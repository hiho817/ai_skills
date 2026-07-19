---
name: fetch-thesis-experiment-data
description: Retrieve traceable thesis experiment metrics, reports, and figures from the remote Linux workspace at hiho817@100.102.127.73 for use in Thesis_value. Use when a thesis task needs physical-experiment results, RMSE or contact-state statistics, comparison tables, report evidence, or publication figures; when checking whether an experiment result is an official thesis result; or when copying approved analysis artifacts into Thesis_value/實驗 or Thesis_value/assets.
---

# Fetch Thesis Experiment Data

Use the remote analysis workspace as the data source while keeping thesis artifacts small, traceable, and reproducible.

## Fixed locations

- Connect to `hiho817@100.102.127.73` over SSH.
- Use `/home/hiho817/analysis_ws/thesis_exp/physical_exp/` as the only authoritative source for thesis physical-experiment results.
- Use `/home/hiho817/analysis_ws/experiments/` only to inspect all experiments, historical analysis methods, scripts, and report-generation patterns.
- Store analysis records and metric summaries under the active `Thesis_value/實驗/` directory.
- Store thesis figures under the active `Thesis_value/assets/` directory.
- Keep ROS 2 bags, VICON CSV files, and other large raw data on the remote host unless the user explicitly requests them.

Read [references/remote-layout.md](references/remote-layout.md) when locating experiments or interpreting `metrics.json`.

## Workflow

1. Confirm that the active workspace is `Thesis_value` and resolve its absolute root before writing locally.
2. Translate the thesis request into experiment type, experiment IDs or groups, required metrics, and required figures. Ask only when different choices would materially change the evidence.
3. Check SSH connectivity non-interactively with normal OpenSSH host-key verification. Inspect the remote Git SHA and `git status --short`; treat uncommitted changes as user work and never alter them implicitly.
4. Search the authoritative physical-experiment tree first. Prefer the aggregate report, then the experiment-specific `metrics.json`, report, and figures.
5. If the authoritative tree lacks the requested result, inspect `experiments/` only as a reference. State that the result is not yet an official thesis result. Obtain explicit approval before rerunning analysis, changing remote files, or promoting a result into `thesis_exp/physical_exp/`.
6. Read small Markdown and JSON files remotely before copying anything. Report exact values with their units, experiment ID, group, inclusion status, and metric definition.
7. Copy only the artifacts needed for the current thesis task. Use `scp` for files; do not recursively copy an experiment directory by default.
8. Verify every copied file using remote and local SHA-256 hashes. Do not overwrite an existing local file unless the user explicitly approves replacement.
9. Create or update the relevant record under `實驗/` with YAML frontmatter and a quantitative table. Include the remote source path, experiment ID or group, remote Git SHA, dirty-worktree status, retrieval date, metric definitions, inclusion/exclusion rule, and copied figure paths. For a derivative figure, also record the transformation tool, command or procedure, and parameters.
10. Check that any thesis-facing statement matches the authoritative report and metric files. Call out discrepancies rather than silently choosing one value.

## Source rules

- Treat `thesis_exp/physical_exp/results/analysis_report.md` as the first overview for official physical-experiment group statistics.
- Treat each official experiment's `results/<experiment-id>/metrics.json` as the structured source for its numeric results.
- Respect `exclude_stats`; do not include excluded trials in aggregate claims.
- Preserve the report's aggregation method. Distinguish trial-equal mean and sample standard deviation from pooled sample statistics.
- Do not infer that the newest dated experiment is official. Publication status is determined by presence in `thesis_exp/physical_exp/`, not by date.
- Do not mix values from `/experiments/` into official thesis tables without explicit promotion and provenance.
- Treat the authoritative path as the user's publication source, but disclose when its files come from a dirty worktree or cannot be tied to a clean commit. Do not imply commit-level reproducibility in that case.
- Require an explicit user request before promotion from `/experiments/`. Record the original source, analysis procedure, generated artifacts, approval context, and resulting authoritative paths.

## Local output rules

- Write analysis prose and tables in Traditional Chinese, using the terminology and formatting rules of the active thesis workspace.
- Put analysis records in `實驗/`. Include the workspace-required YAML frontmatter.
- Put copied PDF, PNG, or SVG figures in `assets/`. Prefer names containing the experiment ID and semantic figure purpose to avoid collisions.
- Keep original remote artifacts unchanged. If a figure needs editing, create a clearly named derivative locally and retain the source hash in the experiment record.
- Avoid copying `metrics.json` merely to quote a few numbers; record the needed values and provenance in Markdown. Copy raw JSON only when the user requests an archival snapshot.

## Remote safety

- Default to read-only SSH commands such as `find`, `sed`, `git status`, `git rev-parse`, and `sha256sum`.
- Keep OpenSSH host-key checking enabled. Never place passwords, private keys, access tokens, or other credentials in commands, logs, records, or skill files; use the user's existing SSH agent or key configuration.
- Never run analysis scripts merely to answer a retrieval request because they may overwrite generated results.
- Before any approved remote analysis run, inspect its entry point, required ROS 2 environment, output locations, and current working-tree changes.
- Never commit, reset, delete, clean, or rewrite the remote repository unless the user explicitly requests that separate action.

## Completion check

State which official experiments and metrics were used, which local files were created or updated, whether the remote tree was dirty, and whether any requested evidence remains unofficial or missing.
