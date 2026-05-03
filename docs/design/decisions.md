# Design Decisions

Prior decisions with rationale. Check here before questioning why something works the way it does.

## XML tags in the default template

The default template uses XML tags (`<defs for="name">`, `<memories for="name">`, `<embed>`, `<component>`) as structural delimiters rather than markdown headers.

**Why:** Entity facts and Discord messages both contain markdown. Markdown headers used as structural framing would visually blend with the content they're framing. XML tags are unambiguous as structure and don't conflict with LLM tool-call syntax in practice.

**Corollary:** `embed.toJSON()` dumps the full embed JSON inside `<embed>` tags. This is correct — do not replace it with a prose summary. The JSON is what the LLM needs, and "verbosity" complaints are not a valid reason to downgrade fidelity.

## Two-layer system prompt

The LLM receives two distinct system instruction channels:
1. **`system` parameter** — rendered from per-entity system template (or empty by default).
2. **System-role messages** — entries in the messages array emitted by the main template via `send_as('system')`. Carry entity definitions, memories, and response instructions.

**Why:** Separating the meta-instructions (system param) from the entity context (system-role messages) gives template authors control over both layers independently. It also maps cleanly to how most LLM providers treat the `system` field vs system-role history entries.

## Variable unification (one-directional)

`$if` expressions and Nunjucks templates share variables, but unification is one-directional: templates receive everything from `ExprContext`, not the reverse.

- `createBaseContext()` in `src/logic/expr.ts` — adds a variable to both `$if` and templates.
- `src/ai/template.ts` rendering — adds template-only variables (entities, others, memories, history, char, user). Not available in `$if`.
- Fact macros (`{{entity:ID}}`, `{{char}}`, etc.) — expanded by string replacement in `src/ai/prompt.ts` before evaluation. Separate mechanism from expression variables.

**Why:** `$if` conditions are evaluated per-fact, before the full template context is assembled. Giving `$if` access to template-only variables would require assembling the full context first, creating a circular dependency.

## Entity-everything

No distinct character/location/item types. Everything is an entity with facts.

**Why:** A rigid type hierarchy requires predicting what distinctions will matter. Freeform facts with emergent conventions scale better for collaborative worldbuilding where users invent their own schemas.

## Template grouping (shared LLM calls)

Entities sharing the same template (including the default `null`) share a single LLM call. Entities with different templates each get their own call.

**Why:** Grouping minimizes API calls in the common case (most entities use the default template). Per-template calls are isolated so a custom template can't interfere with entities using a different one.
