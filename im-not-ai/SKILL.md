---
name: im-not-ai
description: Diagnose and revise Traditional-Chinese academic, thesis, technical, and research prose that sounds overly classical, conversational, formulaic, or AI-generated while preserving technical meaning. Use when users ask to reduce 文言、AI 套話、口語化、冗詞、名詞堆疊, improve academic Chinese flow, or audit phrases such as 「之」、「於」、「值得注意的是」、「進一步地」 in a draft.
---

# Im Not AI

Revise toward modern, formal, technically precise Chinese. Do not optimize for a lower count of any word or for a uniform style.

## Non-negotiable rules

- Treat trigger words as review candidates, never prohibited words.
- Read the target sentence and its preceding and following sentences before judging it. Read the adjacent paragraph's topic sentence when the target is at a paragraph boundary.
- Preserve technical terms, equations, variables, citations, labels, experimental conditions, evidence strength, and causal direction.
- Do not add claims, evaluations, citations, numerical implications, or causal links absent from the source.
- Do not replace every 「之」 with 「的」. Prefer clarifying the semantic relation through syntax or a precise verb.
- Do not edit the source file unless the user explicitly asks for edits. Otherwise return a diagnosis and proposed revisions.

## Writing target

Require all revisions to balance these properties:

| Property | Preserve | Avoid |
|---|---|---|
| Academic register | precise terms, explicit conditions, evidence-linked claims | archaic or bureaucratic wording |
| Natural Chinese | varied sentence rhythm and direct syntax | literal English syntax or rigid templates |
| Technical rigor | actors, actions, objects, conditions, evidence | vague evaluative language |

## Workflow

1. Protect non-prose spans: equations, LaTeX, variables, citations, labels, code, quoted material, table data, and established terminology.
2. Locate candidate phrases and inspect their local context.
3. Classify the actual issue, if any: classicalized syntax, nominalization, opaque relation, conversational wording, formulaic discourse marker, unsupported evaluation, redundancy, or no issue.
4. Decide whether to keep, minimally revise, or structurally rewrite. Keep a phrase when it carries a necessary logical, rhetorical, or technical function.
5. Give at most two revisions: a conservative version and, only if useful, a structural version.
6. Verify that the revision states the same technical proposition and fits the surrounding paragraph.

## Context-sensitive checks

### Classicalized syntax

Inspect 「之」、「於」、「其」、「所」、「藉由」、「基於」、「作為」 and patterns such as 「於……之下／中」、「A 與 B 之間」、「A 對 B 之影響」.

- For a simple modifier, use 「的」 only if it reads naturally.
- For a method, action, cause, or effect, expose the relation with a verb: 「A 對 B 之影響」 may become 「A 如何影響 B」.
- For time, place, or condition, prefer direct modern syntax: 「於非理想接觸條件下」 may become 「在非理想接觸條件下」.
- For headings and technical noun phrases, remove the particle only when the resulting term remains unambiguous.
- Preserve conventional terminology and occasional formal phrasing when it improves precision or rhythm.

### Formulaic or AI-like prose

Inspect phrases such as 「值得注意的是」、「需要說明的是」、「進一步地」、「此外」、「同時」、「綜上所述」、「不僅……更……」, and unsupported evaluations such as 「有效」、「顯著」、「明顯」、「良好」、「強健」.

- Identify the phrase's actual function: warning, contrast, causal link, scope limitation, transition, summary, or empty padding.
- Remove or replace it only when the surrounding syntax already conveys that function.
- Replace generic transitions with the specific relation when possible: cause, contrast, sequence, condition, comparison, or limitation.
- Require a figure, table, equation, statistical test, defined metric, or explicitly reported observation for evaluative claims. Otherwise soften, qualify, or flag for author confirmation.

### Conversational prose

Flag vague deictics and fillers such as 「這個」、「這些東西」、「很多」、「不太」、「其實」、「就是」 when they hide a technical referent or condition.

- Replace with the specific system, measurement, result, model, condition, or mechanism only when the antecedent is clear.
- Keep plain, direct sentences; do not compensate by adding archaic vocabulary.

## Decision output

For every flagged item, return:

```text
位置：章節／段落／句子
原文：……
脈絡功能：因果／限制／轉承／方法—產物／其他
判定：保留／建議修訂／人工確認
問題：……
保守改寫：……
結構改寫：（必要時提供）……
語義驗證：保留了……；未新增……
```

Return only high-value findings. State explicitly when no revision is warranted.

## Optional subagent workflow

Use this workflow when the user requests delegation, the draft is long, or an independent revision pass would materially improve reliability.

1. Send a diagnostic subagent the raw passage and ask it to identify candidates only. Require local-context reading and the decision output above. Do not ask it to rewrite.
2. Send approved candidates plus their context to a revision subagent. Require at most two revisions per item and prohibit new technical claims.
3. Have the main agent compare source, diagnosis, and revisions. Reject changes that alter terminology, technical meaning, evidence strength, or paragraph logic.

Do not use subagents for a short, unambiguous single-sentence edit. Do not let a revision subagent approve its own rewrite.

## Final checklist

- Can the reader identify the actor, action, object, condition, and evidence without inference?
- Does each transition name a real relation rather than fill space?
- Is a formal phrase retained because it serves meaning, not merely because it sounds academic?
- Does the paragraph retain natural variation rather than repeating a new template?
