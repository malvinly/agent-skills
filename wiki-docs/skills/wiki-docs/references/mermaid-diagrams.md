# Mermaid diagrams in wiki-docs pages

Diagrams are part of high-quality documentation, not decoration. Read this file before writing or fixing any ```mermaid fence.

## Choosing a diagram type

- `sequenceDiagram` — runtime and request flows across components (auth flows, request lifecycles, agent tool loops).
- `stateDiagram-v2` — lifecycles and state machines (job states, connection states, run phases).
- `erDiagram` — the data model: entities and their relationships.
- `flowchart TD` — branching control flow and decision logic.

## Discipline

- Ground every diagram in inspected source. Do not invent participants, states, entities, or relationships the code does not support.
- Cover the high-value cases: a page documenting a request/runtime flow, a call sequence, a lifecycle or state machine, or a data model deserves a diagram. A typical wiki has several such diagrams, not one overall. Skip pages that are navigation, reference tables, or pure configuration.
- Prefer a few strong diagrams over decorating every page. Give each diagram a one-line caption directly below it stating what it shows.
- Keep labels short; move explanation into the surrounding prose or the caption.

## Syntax safety

Most renderers (GitHub included) fail the whole fence on one bad label. When in doubt, rephrase the label.

- Never place semicolons or pipes inside node, message, or edge labels.
- Never place unescaped angle brackets in labels; write "returns Promise of User" instead of "returns Promise<User>".
- In `flowchart`, wrap any label containing parentheses, brackets, or other punctuation in double quotes: `A["calls foo(bar)"]`.
- In `flowchart`, never use the bare word `end` as a node id, and never start a node id with `o` or `x` followed by a dash (both are edge-marker syntax); rename the node.
- In `sequenceDiagram`, participant names with spaces or punctuation need an alias: `participant AS as Auth Service`.
- Never use a Mermaid reserved word as a participant name, alias, or node id: `note`, `end`, `loop`, `alt`, `opt`, `par`, `and`, `else`, `activate`, `deactivate`, `class`, `state`, `click`, `link`. A notification participant must be `Notifier`, not `Note`.
- In `erDiagram`, entity and attribute names must be single identifier-like tokens; put human phrasing in the relationship label.

## Update runs

- A wrong diagram is a stale claim, not existing structure to preserve. If a source change makes a diagram inaccurate, update the diagram in the same edit as the surrounding prose.
- Do not rewrite a diagram that is still accurate; regenerating unchanged diagrams creates diff noise.
