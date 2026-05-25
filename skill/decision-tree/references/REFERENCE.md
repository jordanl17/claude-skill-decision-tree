# Decision tree navigator reference

## Payload schema

The runtime payload is a JSON object with three top-level keys: `topic`, `submit_instruction`, and `tree`. Claude `JSON.stringify`-s this object and substitutes the result into the single `{{NAVIGATOR_DATA}}` placeholder in the bundled widget. The `tree` is a recursive structure. Each level has a question, a basis line, and branches. Each branch has metadata and an optional `sub` pointing to the next level. Leaves have `next_hint: null` and `sub: null`.

```json
{
  "topic": "short string label for the decision",
  "submit_instruction": "the prompt that should run when the user commits a path",
  "tree": {
    "question": "the question at this level, short and conversational",
    "basis": "one sentence on why this question matters given the path here",
    "branches": [
      {
        "id": "kebab-case-id",
        "title": "3 to 5 words",
        "summary": "one to two sentences",
        "next_hint": "short phrase about what the next decision explores (null for leaves)",
        "sub": {
          "question": "...",
          "basis": "...",
          "branches": [ ... ]
        }
      }
    ]
  }
}
```

Size targets:

- 2 to 4 branches at each level. Pick the count that fits the decision.
- Tree depth: exactly 3 levels (L0 -> L1 -> L2). L2 nodes are terminal leaves.
- A clean binary at L0 (2 branches) is fine if each branch leads to a genuinely different L1 question; 4 is the ceiling.

## Generation prompt template

Adapt this to the user's topic. The filter should mirror constraints the user has stated or that follow naturally from context. Run this as an internal generation call before the user-facing tree appears.

```
Generate a complete decision tree for [TOPIC].

Filter applied to every leaf:
- [CONSTRAINT 1]
- [CONSTRAINT 2]
- [CONSTRAINT 3]

Structure:
- 2 to 4 branches at each level, exactly 3 levels deep.
- L0: top-level directions, must be meaningfully different.
- L1: narrowing within the L0 commitment.
- L2: concrete, committable specifics (terminal leaves).

L2 leaves must be concrete and committable. Not "a tool that helps with X" but specific enough to start acting on. For ideation tasks, name the tool and sketch the MVP. For architecture tasks, name the pattern and sketch its trade.

Critically: the L1 question under each L0 branch must be a DIFFERENT question, not the same question with different filters. If you find yourself writing the same L1 question for multiple L0 branches, the topic is wrong for a tree.

Respond ONLY with JSON matching this shape, no preamble, no markdown fences. The top-level object must include `topic`, `submit_instruction`, and `tree`:

{
  "topic": "short label for the decision (e.g. 'micro-SaaS ideation')",
  "submit_instruction": "the prompt that should run when the user commits a path",
  "tree": {
    "question": "L0 question (conversational, short)",
    "basis": "one sentence on why we start here",
    "branches": [
      {
        "id": "kebab-case",
        "title": "3 to 5 words",
        "summary": "one sentence",
        "next_hint": "what L1 explores under this branch",
        "sub": {
          "question": "L1 question for this branch",
          "basis": "one sentence on why this matters",
          "branches": [
            {
              "id": "kebab-case",
              "title": "3 to 5 words",
              "summary": "one sentence",
              "next_hint": "what L2 explores",
              "sub": {
                "question": "L2 question",
                "basis": "one sentence",
                "branches": [
                  {
                    "id": "kebab-case",
                    "title": "concrete and committable",
                    "summary": "one to two sentences with specifics",
                    "next_hint": null,
                    "sub": null
                  }
                ]
              }
            }
          ]
        }
      }
    ]
  }
}
```

## Token budget

The full tree at 3 L0 / 3 L1 / 2 L2 sizing is roughly 1500 to 2500 tokens of JSON. Use `max_tokens: 4096` or higher for the generation call. If the tree truncates, drop branch counts (e.g. 2-2-2).

## Widget substitution

The bundled widget HTML at `assets/widget-bundled.html` contains a single placeholder, `{{NAVIGATOR_DATA}}`, inside an inline `<script id="navigator-data" type="application/json">` tag. Claude reads the bundled file, replaces the placeholder with the JSON-stringified payload `{ topic, submit_instruction, tree }`, and passes the result to `visualize:show_widget`.

`widget.ts` reads `document.getElementById('navigator-data').textContent` and `JSON.parse`s it. The header line in the rendered widget is set from the `topic` field at runtime - no separate HTML interpolation needed.

## Edge cases

**User has pre-committed to one L0 branch.** Drop L0 entirely. Start at the user's stated L0 and ask only L1 plus L2. Adjust the workflow accordingly.

**User mid-walk wants to reframe.** Suggest restarting with a fresh framing-confirmation conversation. The widget is path-independent once rendered, and mid-walk reshaping is the failure mode this skill explicitly avoids.

**User adds annotations during the walk.** Annotations get included in the final submission payload. They do NOT cause regeneration of branches mid-walk. They influence the final artifact Claude produces from the committed path.

**Tree quality is bad after rendering.** User asks for a different tree via chat. Generate fresh and render a new widget.

**User insists on rendering after a variance check fails.** Offer the list-style alternative once more, briefly. If the user insists again, do not render against the gate. The skill exists to protect the user from a widget that looks authoritative but has no real variance.

## Anti-patterns to avoid

- Rendering without doing the variance check internally first (the check must still happen; it just no longer appears in chat)
- Trees deeper than 3 levels
- Promising per-tap dynamic generation (the tree is static once rendered)
- Trees with low-variance L1 questions
- Rendering cold without context
- Treating annotations as branch-reshaping signals
