---
name: read-doc-pdf
description: Use when turning a user-provided technical PDF in a project folder into source-grounded Vietnamese study materials using that project's README.md and init/DETAIL.md.
---

# Read technical PDFs for deep understanding

Create source-grounded Vietnamese study material from a technical PDF. The skill supplies a reusable reading method; each project's `README.md` and `init/DETAIL.md` specify the actual source, deliverables, depth, and scope.

When setting up a new project, create its `README.md` and `init/` first. Keep the original user-provided source files in `init/` without altering them, and write the project's concrete generation request in `init/DETAIL.md`. If a source has not been supplied, record that gap rather than inventing or replacing it.

## Project context comes first

1. Identify the project folder named in the request. **Before reading the PDF or generating notes, read that folder's `README.md`.** For DDIA, this is `DDIA/README.md`.
2. Follow the README to `init/DETAIL.md` (or the project's explicitly named requirements file) and read it in full. Inspect the original source files in `init/` identified there. The project detail is required even when this skill already describes the topic.
3. If the README, detail, or source is missing or ambiguous, report the exact gap and complete only work supported by the available material. Do not substitute generic skill instructions for missing project requirements.

## Verify and explain the source

Inspect the PDF's title/copyright pages and table of contents to establish its edition, chapter structure, and page numbering. Distinguish printed page numbers from PDF page indices. For scanned pages or unreliable extraction, inspect page images, figures, and tables. Never infer an edition, heading, argument, or quotation from memory.

Use the supplied PDF as the primary source and cite pages or sections for substantive claims where possible. Paraphrase in natural Vietnamese rather than copying long passages. Label outside explanations **Bổ sung ngoài sách** and invented examples **Ví dụ minh họa do AI xây dựng**.

Follow the project's requested output structure and the source's order of reasoning. For DDIA, use `DDIA/init/DETAIL.md`; for HighPerformanceSQL, use `HighPerformanceSQL/init/DETAIL.md`. Explain What, Why, How, examples, trade-offs, and design or operational decisions where they help the reader understand the concept; use Mermaid and comparison tables when useful.

Before finishing, compare the requested unit against its source outline and `init/DETAIL.md`. Check technical accuracy, omitted arguments, terminology, links, and labels. State any unverified details or unreadable pages.
