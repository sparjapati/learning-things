# learning-things

This is a personal learning project for exploring programming concepts.

## Workflow

- When the user asks about a concept, explain it in the conversation with example with some real life analogies, then also write a summary of it to a markdown file.
- Concept files live inside a category folder (e.g. `system-design/`, `spring-boot/`), not at the root, once that category exists. A concept that doesn't fit any existing category stays at the root until enough related files accumulate to justify creating a new category folder for it — don't invent a new category folder for a single one-off file; ask the user before creating a new category if it's unclear which one (existing or new) a concept belongs to.
- Each category folder has its own `images/` subfolder alongside its concept files (e.g. `system-design/images/`), referenced from files in that folder with a relative path (e.g. `images/qr-code-anatomy.png`). Root-level (uncategorized) concept files keep using the root `images/` folder the same way.
- File naming convention: kebab-case, e.g. `event-loop.md`.
- Keep the root `README.md` continuously up to date as an index, grouped into a section per category (plus a "General" section for uncategorized root-level files): whenever a concept file is created, add a one-line entry for it under its category's section (linking the category-prefixed path, e.g. `system-design/event-loop.md`, with a short description of what it covers). Whenever an existing concept file gains a new section that changes what it covers, update that file's description line in `README.md` to match.
- Diagrams or example images that help explain a concept go in that concept file's category `images/` folder. This includes flow/sequence diagrams (e.g. showing steps between services, request/response flow, or topology comparisons like orchestration vs choreography) as well as structural/anatomy diagrams — generate one whenever a visual would make the steps or shape of a concept clearer than prose alone.
- Include short illustrative code or pseudocode snippets (in a fenced code block) when they'd make an abstract pattern concrete — e.g. showing what an orchestrator's step-by-step calls look like versus what a choreographed service's event handler looks like. Keep snippets minimal and language-agnostic/pseudocode unless a specific language is more illustrative; they're there to clarify the shape of the pattern, not to be a runnable implementation.

## Handling related questions across sessions

Before writing a new file, check if an existing `.md` file anywhere in this directory (its own category folder first, then other category folders and the root) already covers the same or a closely related concept (by filename and by skimming content). If one exists, decide as follows:

- **Small/tightly related addition** (a follow-up, clarification, or sub-topic that naturally belongs with the existing concept, e.g. an example, edge case, or "how do I decide" companion to an already-documented comparison) → add a new section to the existing file instead of creating a new one.
- **Big/standalone new concept** (substantial enough to be useful on its own, would make the existing file too long or unfocused, or someone might want to find it independently) → create a new file, and add a `See also: [existing-concept.md](existing-concept.md)` (or a short "Related" line linking the filename) pointing back to the existing one. Also add a matching backlink in the existing file if it doesn't already reference the new one. Place each "See also" line wherever it's contextually relevant in the file (e.g. right above the section it relates to) — not always at the top by default. Always use standard markdown link syntax (`[text](file.md)`), never `[[wikilink]]` syntax — GitHub's renderer doesn't convert double-bracket wikilinks into links, so they'd show up as literal brackets.

When unsure whether something counts as "small" or "big," lean toward: if the explanation is a few paragraphs or less and stays fully on the same topic, append; if it introduces a genuinely separate concept that stands on its own, create a new file with a cross-reference.
