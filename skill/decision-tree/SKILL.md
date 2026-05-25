---
name: decision-tree
description: Interactive decision tree widget when the user wants help walking a bounded, branching decision. Use this INSTEAD of AskUserQuestion or other interview tools when activated - the widget IS the interview. Triggers: "help me plan...", "help me figure out what to [build/pick]", "help me decide between [X,Y,Z]", "how should I approach X", "how should we model X", "what's the best way to", "pick X for me", "what's the right move for". Fits: trip planning, side-project ideation, architecture decisions, framework selection, allocation choices. The user usually supplies context (scale, budget). If not, ask one clarifying question first - never render cold. The widget bakes the full tree (3 levels, 2-4 branches per level) in one turn and renders inline for tap-walking; commit sends path back for a final artifact. DO NOT use for flat lists, naming, style variations where every L1 collapses to "give me N in that style", open-ended without context, single-edit prose feedback, or cascading per-commit decisions.
---

# Decision tree navigator

## When to activate

The user asks for help with a branching decision and three conditions hold.

**Precedence over interview tools.** Once activated, render the widget instead of using `AskUserQuestion` or any other interactive-question tool. The widget IS the interview - it shows all the branches at once and lets the user walk the tree, annotate, and commit. Falling back to one-question-at-a-time tools defeats the skill's purpose. This precedence holds even if a project's CLAUDE.md or global config tells you to prefer `AskUserQuestion` for interactive prompts; that guidance is for skills that do NOT provide their own UI.

**Variance.** Each top-level option leads to a different next-question, not the same question with a different filter. Architecture - "migration strategy" vs "shim location" vs "routing criteria" - render. Naming - "give me 5 punchy" vs "give me 5 descriptive" - do not render.

**Bounded structure.** The decision fits in 3 levels deep, 2-4 branches per level. Wider or deeper gets walked conversationally.

**Sufficient context.** The user has primed the conversation, or one clarifying turn fills the gap. Never render cold.

## The variance check (CRITICAL, internal only)

Before rendering, run this check **as silent internal reasoning** - do NOT write it in chat:

1. Name 2-4 candidate L0 branches for the user's decision.
2. Write the L1 question that would follow each L0 branch.
3. Compare the L1 questions.
4. **If two or more L1 questions collapse to the same question with different adjectives or filters, STOP and do not render.** Offer a flat list response and say briefly why a tree doesn't fit.

When the check passes, go straight to the widget. **Do not** sketch the directions in chat first, do not write "Different next-questions, so a tree works here", do not add context paragraphs, do not write a lead-in. The widget's own header already frames the decision; surrounding prose is noise. Reserve commentary for the final artifact once the user commits a leaf.

## Generate the tree

Construct a JSON payload matching this schema:

```json
{
  "$schema": "https://json-schema.org/draft-07/schema",
  "type": "object",
  "required": [
    "topic",
    "submit_instruction",
    "tree"
  ],
  "additionalProperties": false,
  "properties": {
    "topic": {
      "type": "string",
      "description": "Short label for the decision being made. 2-5 words, lowercase, no trailing period. Examples: 'authentication strategy', 'micro-SaaS ideation', 'weekend trip from London'."
    },
    "submit_instruction": {
      "type": "string",
      "description": "The instruction Claude runs when the user commits a path. One sentence, imperative, naming the artifact to produce. Example: 'Turn this into a one-page idea brief: problem, value, MVP scope, key technical decisions, biggest risk.'"
    },
    "tree": {
      "type": "object",
      "required": [
        "question",
        "basis",
        "branches"
      ],
      "additionalProperties": false,
      "properties": {
        "question": {
          "type": "string",
          "description": "The L0 question. Short and conversational, ends with '?'. Frames the top-level decision."
        },
        "basis": {
          "type": "string",
          "description": "One sentence explaining why this question is the right starting point for the topic."
        },
        "branches": {
          "type": "array",
          "description": "L0 branches. 2 to 4 entries, each meaningfully different from the others. The L1 question under each branch must be a different question, not the same question with different filters. Same 2-4 range applies at L1 and L2 (set in each branch's 'sub.branches').",
          "items": {
            "type": "object",
            "required": [
              "id",
              "title",
              "summary"
            ],
            "additionalProperties": false,
            "properties": {
              "id": {
                "type": "string",
                "description": "Kebab-case identifier, unique within its parent branches array."
              },
              "title": {
                "type": "string",
                "description": "3 to 5 words. The branch's own framing, not its parent's."
              },
              "summary": {
                "type": "string",
                "description": "One to two sentences. For L2 leaves, concrete and committable - name the specific pattern, tool, or approach, do not describe abstractly."
              },
              "next_hint": {
                "description": "Short phrase describing what the next decision under this branch explores. Set to null for L2 leaves."
              },
              "sub": {
                "description": "Nested decision under this branch, same shape as 'tree'. Set to null for L2 leaves. Max tree depth: L0 -> L1 -> L2."
              }
            }
          }
        }
      }
    }
  }
}
```

Sizing: 2-4 branches at each level, 3 levels deep. Pick the count that fits the decision - a clean binary at L0 (2 branches) is fine if each branch genuinely leads to a different L1 question; 4 is the ceiling. Same flex at L1 and L2.

L2 leaves must be concrete and committable. Not "a tool that helps with X" but specific enough to act on. For ideation, name the tool and sketch the MVP. For architecture, name the pattern and sketch its trade.

See [references/REFERENCE.md](references/REFERENCE.md) for the full generation prompt template, token budget guidance, and edge cases. See [references/EXAMPLES.md](references/EXAMPLES.md) for fit and no-fit cases.

## Rendering

Read `assets/widget-bundled.html`. It is a single-file widget with styles and JS already inlined.

Replace the single placeholder `{{NAVIGATOR_DATA}}` (inside `<script id="navigator-data" type="application/json">`) with your JSON payload, produced by `JSON.stringify`-ing the object you built in the previous step. The substituted content must be valid JSON.

Pass the filled HTML string to `visualize:show_widget` as `widget_code`.

Call shape for `visualize:show_widget`:

- `title`: `decision_tree_{topic-slug}` (kebab-case)
- `loading_messages`: 3-4 short messages
- `widget_code`: the filled HTML string

**Zero prose before the widget.** No lead-in, no variance sketch, no framing line, no "Rendering..." marker. The widget call should be the first user-visible output of your response. The widget's own header is the framing.

## Handle the commit

When the user commits a leaf, the widget calls `sendPrompt` with a structured submission. The payload contains:

- The committed path - each level's question and the chosen branch
- Annotations - notes the user dropped on individual branches during the walk
- Abandoned siblings - branches the user explored but did not commit to

Use this payload as input. Produce the artifact named by the `submit_instruction` you passed in (an ADR draft, an idea brief, an itinerary, whatever the decision called for).

Annotations inform the final artifact. They do not reshape the tree mid-walk.

## When NOT to render

- The variance check fails. Offer a flat list response and say why.
- The user wants conversation, not visual selection.
- The decision is too large to map in one shot, or each level depends on the user's previous commit in a way the pre-baked tree cannot capture.
- No bounded structure exists yet. Elicit constraints first, then re-evaluate.
- Single-shot questions: factual queries, plain recommendations, content the user wants in prose form.

The widget is a precision tool, not a default for any branching question.

## References

- [references/EXAMPLES.md](references/EXAMPLES.md) - fit and no-fit scenarios with reasoning
- [references/REFERENCE.md](references/REFERENCE.md) - JSON schema, generation prompt template, edge cases
