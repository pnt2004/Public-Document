---
name: read-doc-pdf
description: Use when studying a user-provided technical PDF in this project, especially Designing Data-Intensive Applications (DDIA), and creating source-grounded Vietnamese chapter notes about concepts and system design.
---

# Read technical PDFs for deep understanding

Turn a supplied technical book or document into Vietnamese Markdown study notes. For DDIA, prioritize distributed systems, database internals, and system design reasoning. Explain concepts and decisions, not language-specific implementation or framework usage.

## Establish the source

1. Locate the PDF the user supplied in the relevant project folder. Inspect its title/copyright pages and table of contents to identify the edition, chapter structure, and PDF page numbering. Distinguish printed page numbers from PDF page indices when they differ.
2. If the source or a required page is unavailable or unreadable, state exactly what is missing and continue only with work supported by the available pages. Never infer an edition, heading, argument, or quotation from memory.
3. Use the supplied book as the primary source. When outside knowledge helps explain a point, label it **Bổ sung ngoài sách**. Label invented teaching examples **Ví dụ minh họa do AI xây dựng**. Paraphrase in natural Vietnamese; do not reproduce long passages.
4. Record page or section references for substantive claims where the source permits. For a scanned PDF, inspect the actual page image when text extraction is unreliable, including figures and tables.

## Write in source order

For DDIA, store notes under `DDIA/ddia-notes/`: `README.md`, `glossary.md`, `chapter-XX.md`, and `system-design-concepts.md`. Split an unusually long chapter by its actual sections when useful, and link the parts from the notes README. Keep the book PDF outside the generated notes.

Start by recording the verified edition and table of contents in the notes README. Then write the first chapter, or the specific chapter or section the user requested. Finish and check that unit before moving to another; do not generate the entire book in one pass without a new request.

For each chapter, provide:

- **Tổng quan:** the chapter's question, purpose, and connection to earlier chapters.
- **Giải thích theo mục sách:** follow the author's headings and reasoning, including difficult steps. For each important concept, explain **What** (definition and English terminology), **Why** (origin of the problem and limits of alternatives), **How** (mechanism and data flow), **Example** (simple then realistic), **Trade-offs**, and **System Design Perspective**. Adjust depth to the concept instead of filling a rigid form mechanically.
- **Sơ đồ và so sánh:** use Mermaid only when it clarifies architecture, data flow, process, or relationships. When the book compares approaches, include a table of mechanism, advantages, limitations, trade-offs, and suitable conditions.
- **Tư duy thiết kế:** show how requirements and constraints change a decision through concrete scenarios.
- **Kiến thức cốt lõi** and **5–10 câu hỏi tự kiểm tra**, favoring why, what-if, and comparison questions. Put explained answers in a separate section or linked file so readers can attempt the questions first.

Update the glossary and cross-chapter system-design index as relevant concepts appear. Define unfamiliar terms in Vietnamese while retaining their English names; connect new ideas to concepts already covered. Write enough for understanding without a fixed length target or code examples.

## Check before finishing

Compare the notes against the verified source outline and pages for the requested unit. Check the author's argument, technical accuracy, terminology, internal links, diagrams, and the labels for outside knowledge and invented examples. Report any source gaps or unverified details explicitly.
