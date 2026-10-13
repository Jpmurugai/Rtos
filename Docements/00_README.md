# LIA Project – Documentation Pack (before Vibe Coding)

## What is this?
Before coding with AI ("vibe coding"), we write 6 documents so the AI (and the team) knows
exactly WHAT to build, HOW, and in WHICH order. Each file here is an **empty template**.

| # | Document | Owner | File | Answers the question |
|---|----------|-------|------|----------------------|
| 1 | Product Requirements (PRD) | Ossian | 01_Product_Requirements.md | What & why? For whom? |
| 2 | Technical Requirements (TRD) | Ossian | 02_Technical_Requirements.md | With what tech? |
| 3 | App Flow Document | Ossian | 03_App_Flow.md | How does the user move through the app? |
| 4 | Design Brief | Gayatri | 04_Design_Brief.md | How does it look & feel? |
| 5 | Backend Schema | Gayatri | 05_Backend_Schema.md | How is data stored / which APIs? |
| 6 | Implementation Plan | Gayatri | 06_Implementation_Plan.md | In what order do we build it? |

> Note: "Background Schema" in the task = **Backend Schema** (database + API structure).

## Shared format rules (so all forms look the same)
1. Every document starts with the same **Document Info** table.
2. Headings: `#` title, `##` numbered sections, `###` sub-sections.
3. Empty fields are marked `_____` or `[ ]` – fill them in, do not delete sections (write "N/A").
4. IDs: Requirements `FR-01`, `NFR-01`, Screens `SCR-01`, Tables `TBL-01`, Tasks `T-01`.
   Use the same IDs in Jira so tickets link to documents.
5. Update the **Change Log** at the bottom on every change.

## What YOU (Gayatri) need to do
1. Wait for/review Ossian's docs 1–3 (your docs depend on them).
2. Fill **04 Design Brief** – colors, fonts, screens from App Flow (SCR-xx).
3. Fill **05 Backend Schema** – tables from the PRD features + TRD database choice.
4. Fill **06 Implementation Plan** – phases/tasks; each task becomes a Jira ticket.
5. Agree with Ossian on the format (this README) and review each other's documents.

## Jira mapping
- Epic = Phase in Implementation Plan
- Story = Feature (FR-xx) / Screen (SCR-xx)
- Task/Sub-task = T-xx
