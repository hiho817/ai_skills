# Thesis Notation Rules

## Canonical typography

| Type | Rule | LaTeX example |
|---|---|---|
| Scalar | Lowercase italic | `m`, `t`, `r`, `\theta` |
| Vector | Lowercase bold | `\mathbf{p}`, `\mathbf{v}`, `\boldsymbol{\omega}` |
| Matrix | Uppercase bold | `\mathbf{R}`, `\mathbf{M}`, `\mathbf{J}` |
| Coordinate frame | Place the frame identifier at the upper right | `\mathbf{v}^{B}`, `\mathbf{p}_{i}^{W}` |

Use `\mathbf{}` for Latin vectors and matrices. Use `\boldsymbol{}` for Greek vectors because `\mathbf{\omega}` does not reliably bold Greek glyphs. Preserve italic lowercase for Greek scalars.

Interpret uppercase bold objects by semantics: rotation, inertia, covariance, Jacobian, transition, observation, gain, selection, and identity objects are matrices. Do not convert a vector to uppercase merely because it is bold.

## Script ordering

- Put the coordinate-frame identifier in the right superscript: `\mathbf{r}_{C,i}^{B}`.
- Put point, leg, component, and time indices in the right subscript: `C`, `i`, `x`, `k`.
- Keep estimator-status modifiers such as prior/posterior, transpose, correction, or inverse distinct from coordinate frames. Examples include `\hat{\mathbf{x}}_{k}^{-}`, `\mathbf{R}^{\mathsf{T}}`, and `\mathbf{P}^{-1}`.
- When a symbol needs both a frame and another superscript, use an unambiguous documented order or a named operator. Do not silently interpret every superscript as a frame.
- Do not use a left superscript for a coordinate frame under this thesis convention.

## Consistency and uniqueness

Create a thesis-wide registry for semantic variables. Apply both directions:

1. One meaning, one symbol: flag the same physical quantity written with different base identifiers or typography.
2. One symbol, one meaning: flag a base identifier assigned to different physical quantities, even across chapters.

Compare the complete notation before declaring a collision. `\mathbf{p}^{W}` and `\mathbf{p}^{B}` may be the same quantity expressed in different frames; `\mathbf{p}_{C}^{B}` and `\mathbf{p}_{G}^{B}` may denote different points and are not duplicates when the subscripts are defined. Conversely, changing only boldness does not make a reused identifier safe.

Allow conventional local notation when its role is stable and locally clear, including indices `i,j,k`, dimensions `n,m`, differential elements, summation indices, dummy integration variables, and standard operators. Flag local reuse only when it creates ambiguity in the same derivation or conflicts with a declared thesis-wide semantic variable.

Treat accents and modifiers as related families, not automatically identical meanings:

- `\mathbf{x}`, `\hat{\mathbf{x}}`, and `\delta\mathbf{x}` are true state, estimate, and error state.
- `\mathbf{P}`, `\mathbf{P}^{-}`, and `\mathbf{P}^{+}` are one covariance family with different update status.
- A dot or double dot denotes time derivatives only when stated: `\dot{\mathbf{q}}`, `\ddot{\mathbf{q}}`.

## Definition checks

Require each nonstandard semantic symbol to be defined at first formal use. The definition should identify:

- physical or mathematical meaning;
- scalar, vector, or matrix type;
- dimension when useful;
- coordinate frame for spatial vectors and frame-dependent matrices;
- indices and estimator modifiers.

Do not treat acronym terminology tracking as a replacement for a mathematical symbol registry. Consult both when the workspace maintains both.

## Severity

- **Critical:** wrong or ambiguous coordinate frame, or a symbol collision that can change equation meaning.
- **Major:** wrong scalar/vector/matrix type, inconsistent symbol for the same quantity, or undefined symbol essential to a derivation.
- **Minor:** typography drift, inconsistent transpose style, or a definition that is understandable but incomplete.

## Safe revision rules

- Revise all occurrences belonging to the same semantic variable within scope, including prose definitions and downstream equations.
- Preserve `\label{}` and `\ref{}` relationships and the workspace's Markdown equation-label convention.
- Avoid global substitutions for single letters. Search by LaTeX structure and inspect every replacement in context.
- Do not alter quoted source notation, bibliography titles, code identifiers, filenames, or external API names unless explicitly requested.
