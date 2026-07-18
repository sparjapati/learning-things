# learning-things

This is a personal learning project for exploring programming concepts.

## Workflow

- When the user asks about a concept, explain it in the conversation with example with some real life analogies, then also write a summary of it to a markdown file.
- Concept summary files live at the root of this directory (not in a subfolder).
- File naming convention: kebab-case, e.g. `event-loop.md`.
- Keep `README.md` continuously up to date as an index: whenever a concept file is created, add a one-line entry for it (linking the filename with a short description of what it covers). Whenever an existing concept file gains a new section that changes what it covers, update that file's description line in `README.md` to match.

## Handling related questions across sessions

Before writing a new file, check if an existing `.md` file in this directory already covers the same or a closely related concept (by filename and by skimming content). If one exists, decide as follows:

- **Small/tightly related addition** (a follow-up, clarification, or sub-topic that naturally belongs with the existing concept, e.g. an example, edge case, or "how do I decide" companion to an already-documented comparison) → add a new section to the existing file instead of creating a new one.
- **Big/standalone new concept** (substantial enough to be useful on its own, would make the existing file too long or unfocused, or someone might want to find it independently) → create a new file, and add a `See also: [[existing-concept]]` (or a short "Related" line linking the filename) at the top of the new file pointing back to the existing one. Also add a matching backlink in the existing file if it doesn't already reference the new one.

When unsure whether something counts as "small" or "big," lean toward: if the explanation is a few paragraphs or less and stays fully on the same topic, append; if it introduces a genuinely separate concept that stands on its own, create a new file with a cross-reference.
