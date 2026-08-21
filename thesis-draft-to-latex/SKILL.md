---
name: thesis-draft-to-latex
description: Convert thesis Markdown drafts into LaTeX while preserving source meaning, wording, equations, citations, and structure. Use when working from `Thesis/論文初稿`, `Thesis_value/論文初稿`, thesis outline Markdown, or other thesis draft Markdown files into `Thesis_overleaf` LaTeX chapters; especially when the user requires strict consistency with the Markdown source, no edits to the Markdown draft, chapter-by-chapter conversion, source commit SHA comments, and PDF review after conversion.
---

# Thesis Draft to LaTeX

## Core Rule

Treat the Markdown thesis draft as the source of truth. Do not edit, normalize, rewrite, or "improve" the Markdown draft unless the user explicitly asks for Markdown edits. In the LaTeX repository, only make formatting, escaping, labeling, and environment changes required to represent the Markdown content correctly.

## Content Availability Gate

Before adding LaTeX prose or lower-level headings, classify each source as one of:

- Formal chapter draft: a Markdown file or set of files clearly intended to be converted into the target chapter.
- Outline only: `論文大綱.md` entries, planning bullets, or heading lists.
- Local note or partial scratch draft: isolated subfolder notes such as `草稿.md`, implementation notes, TODO text, or section-specific fragments that are not explicitly requested by the user.

Only formal chapter drafts may produce body text. Outline-only sources may provide at most `\chapter` and `\section` skeletons when the user asks to prepare empty chapters. Do not create `\subsection` entries from outline-only sources. Do not convert local notes or partial scratch drafts into thesis prose unless the user explicitly identifies that file or section as content to convert.

Common failure to avoid: treating "available Markdown somewhere under the chapter folder" as chapter content. A chapter with no formal draft must remain a sparse skeleton; it must not be expanded with prose, subsection headings, TODO sentences, or inferred explanations from outlines and notes.

## Figure and Placeholder Gate

Before finishing a conversion, scan the Markdown source for every image reference, figure caption, and placeholder marker:

- Markdown image links such as `![](path/to/file.pdf)` are real source assets. Copy the referenced asset into the LaTeX repository's figure directory, convert the Markdown image and adjacent caption into a `figure` environment, and use `\includegraphics` with a stable label.
- Do not leave a real image as a LaTeX comment or text-only caption. If the source references a PDF/PNG/JPG/SVG asset, the converted LaTeX must include that asset unless the user explicitly asks to omit it.
- Placeholder comments such as `<!-- 圖 4.x 預留位置：... -->` or source text that clearly says a figure is reserved but provides no asset must become a visible placeholder in the PDF using the repository's existing `\placeholderfigure{...}` command when available.
- Placeholder figures should still have a `figure` environment, caption, and label so the figure number and list of figures remain stable.
- Do not substitute unrelated or older asset variants for a placeholder-only figure. If multiple unused assets exist but the Markdown does not identify which one belongs to the placeholder, keep a visible placeholder and report the ambiguity.

Common failures to avoid: converting figure captions while dropping the actual PDF/image, leaving placeholder comments invisible in the PDF, or silently replacing a missing figure with a guessed asset.

## Required Sources

Before converting, read the relevant project instructions if present:

- `Thesis_value/AGENTS.md` or `Thesis_value/agent.md`
- `Thesis_overleaf/agent.md` or `Thesis_overleaf/AGENTS.md`
- `Thesis_value/論文初稿/論文大綱.md`
- The Markdown chapter or section being converted
- The target LaTeX chapter file and nearby style examples

Also check whether `Thesis/論文初稿` or `Thesis_value/論文初稿` has changed since the last conversion. Prefer `git status`, `git log -1`, and `git rev-parse HEAD` in the draft repository when available.

## Conversion Workflow

1. Identify exactly one chapter to convert unless the user explicitly asks for a smaller unit. Avoid multi-chapter conversion in one pass.
2. Record the draft repository commit SHA before editing LaTeX. If the source is not a git repository, record the source path and timestamp instead.
3. Compare the thesis outline with the Markdown draft content.
4. Edit only the LaTeX target files needed for the selected chapter.
5. Add a LaTeX comment near the converted chapter or section recording the source draft path and commit SHA, for example:

```tex
% Source draft: /home/.../Thesis_value/論文初稿/chapter03.md @ abc1234
```

6. Build the PDF using the repository's existing build command.
7. Review the generated PDF after each conversion. Check at minimum: chapter/section order, missing text, broken formulas, figure placement, references, and obvious layout overflow.
8. Report the changes grouped by chapter and section.

## Fidelity Requirements

- Preserve the Markdown draft's meaning, order, terminology, and technical claims.
- Do not add new arguments, new citations, new results, or explanatory prose not present in the Markdown source unless the user explicitly asks.
- Do not delete source content because it is awkward or redundant. If content appears problematic, flag it in the final summary instead of silently changing it.
- Keep Traditional Chinese wording aligned with the source. Preserve English technical terms such as Proprioceptive, Leg-Wheel, Transformable, EKF, Jacobian, IMU, GMO, and ROS 2 when the source uses them.
- Convert Markdown structure to LaTeX structure without inventing empty lower-level headings.
- If the outline contains a subsection but the formal Markdown chapter draft has no content for it, create only the corresponding `\chapter` and `\section`; do not create empty `\subsection` entries.

## LaTeX Formatting Rules

- Convert display math from Markdown `$$ ... $$` into the target repository's preferred LaTeX environments, usually `equation`, `align`, or related environments.
- If the Markdown has a label comment immediately before display math, move the label into the LaTeX math environment.
- Preserve semantic labels such as `eq:lowercase-kebab-case`; do not rename labels unless needed to avoid a concrete collision.
- Check that every `\ref{eq:...}` has a matching `\label{eq:...}` and that labels are not duplicated.
- Protect backslashes and high-risk symbols such as `\theta`, `\tau`, `\text`, and `\times`.
- Do not leave blank lines immediately before or after display math environments in LaTeX.
- Keep figures readable. Do not force side-by-side subfigures if they become too small.
- Avoid unnecessary `\FloatBarrier` after every figure.
- Convert figure placeholders with the local `\placeholderfigure{...}` command when it exists, matching the repository's established placeholder style.

## Consistency Review

After editing LaTeX, perform a source-to-target review:

- Re-read the edited LaTeX and compare against the Markdown source for missing paragraphs, reordered content, changed equations, and altered technical claims.
- Verify that each Markdown image reference and each placeholder marker has a visible LaTeX counterpart.
- Search for damaged LaTeX control sequences, accidental tabs in formulas, unmatched braces, duplicate labels, and unresolved references.
- Build and review the PDF. If the PDF cannot be built or inspected, state exactly what was attempted and what blocked validation.

## Final Response

Summarize by chapter and section. Include:

- Source Markdown path and commit SHA used
- LaTeX files changed
- Chapter/section content converted
- PDF build and review result
- Any source/LaTeX consistency concerns that remain
