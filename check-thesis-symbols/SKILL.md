---
name: check-thesis-symbols
description: Audit mathematical notation in thesis and research Markdown or LaTeX drafts. Use when checking or revising coordinate-frame superscripts, scalar/vector/matrix typography, symbol definitions, reused identifiers, cross-chapter notation consistency, or a thesis symbol glossary. Trigger on requests such as 論文符號檢查、符號統一、座標系代號、向量矩陣粗體、代號重複、notation audit, or symbol consistency.
---

# Check Thesis Symbols

Audit mathematical symbols by meaning and scope, not by blind text replacement. Read [references/notation-rules.md](references/notation-rules.md) before performing an audit or revision.

## Workflow

1. Read the workspace instructions and any symbol list, terminology memo, outline, or earlier chapter that governs the target draft. Treat explicit user and project conventions as authoritative.
2. Establish scope. Default to the requested files; when asked for thesis-wide consistency, scan all formal draft chapters and exclude notes, quotations, code, bibliography entries, and generated files unless requested.
3. Build a symbol inventory with the rendered symbol, normalized base identifier, mathematical type, physical meaning, coordinate frame, indices, first definition, file, and line. Keep accents such as `\hat{}` and `\delta` in the rendered form but compare their base variables deliberately.
4. Classify each occurrence from its definition, dimensions, and equation role. Do not infer type from letter case alone.
5. Check frame placement, typography, consistency, collisions, missing definitions, and ambiguous reuse using the reference rules.
6. Report findings before editing unless the user explicitly requested revision. Cite exact file and line, show current and proposed notation, and separate definite violations from items requiring author judgment.
7. If revision is authorized, make the smallest consistent edits across definitions, equations, prose, and later references. Never change equation labels or physical meaning merely to satisfy typography.
8. Read modified files back and verify LaTeX backslashes, `\times`, subscripts, superscripts, matrices, equation labels, and references. Check for tabs or control-character damage.

## Audit Output

Provide:

- A brief scope and governing convention statement.
- A symbol registry or delta from the existing registry.
- A findings table with severity, location, current form, proposed form, and reason.
- Collision groups: one symbol with multiple meanings, and one meaning with multiple symbols.
- Unresolved questions only where context cannot determine the intended semantics.
- A verification summary after edits.

Prioritize semantic collisions and incorrect coordinate frames over typography-only findings. Do not flag standard local indices or operators as thesis-wide collisions when their conventional role is unchanged.
