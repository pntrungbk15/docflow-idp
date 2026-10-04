# Technical overview

[← Back to README](../README.md)

## Problem shape

Logistics packets are hard for page-by-page extraction:

- **Mixed documents in one file** — the page sequence has to be segmented before any schema applies.
- **Continuations** — a container list or invoice line items run over several pages; later pages repeat headers, carry subtotals forward and lack most header fields.
- **Non-content pages** — terms and conditions, blank backs.
- **Scans** — some pages are images with an OCR layer that confuses `0/O`, `1/I`, `5/S`.
- **Numbers that must agree** — totals vs. rows, quantity × price, check digits, references between documents.

## Classification and segmentation

Each page is classified on its own into a type plus two flags — *continuation* (it continues the previous page) and *self-contained* (a complete one-page document) — with a confidence. Segmentation is then a simple, predictable pass over the sequence:

- a continuation joins the open document of the same type;
- anything else starts a new document, so two bills of lading back to back stay separate;
- `IGNORE` pages are folded into the preceding document, so the terms page stays attached to its bill of lading;
- an orphan continuation starts a new document with lowered confidence.

Separating *classify* (a model decision per page) from *segment* (a deterministic rule over decisions) keeps errors local and explainable.

## Grouped, continuation-aware extraction

Schemas are split into small groups by section, with each table as its own group. Smaller groups mean shorter prompts and fewer missed fields when a VLM backend is used. Page 1 of a document is asked for the whole schema; continuation pages are asked only for fields that are still empty, plus table rows. This prevents a continuation page — which often repeats a partial header — from overwriting good page-1 values.

## Localisation

Finding *where* a value is printed is a separate pass. For each extracted value the localiser returns candidate boxes in 0–1000 page coordinates; non-maximum suppression keeps the best non-overlapping candidate, preferring candidates next to the field's printed label and whole phrases. Table cells are searched only inside their row's band. A value that cannot be placed is flagged rather than given a guessed box.

## Multi-page table aggregation

Rows collected page by page are aligned into one table: wrapped description lines are merged into their row, "carried forward" / "brought forward" lines are removed, repeated rows (headers repeated on continuation pages) are kept once, and every row keeps the page it came from — so the UI can highlight a row's cells on its source page.

## Derived fields and validation

| Rule family | Examples |
|---|---|
| Arithmetic | total = subtotal + freight + insurance; table totals = row sums (with tolerances); quantity × unit price |
| Identifiers | ISO 6346 container check digit; value formats per field kind |
| Dates | issue / shipped-on-board / arrival order |
| Completeness | required fields per document type |
| Cross-document | an arrival notice must refer to a bill of lading in the packet |

Missing totals are derived from rows when possible. A failing rule names its fields and its numbers, and all of its fields are flagged together.

## Confidence

Confidence is per value and is lowered for concrete, displayable reasons: an OCR digit/letter repair, an approximate label match ("Invoice Dated" vs. "Invoice Date"), a weak document title, or a value that could not be located. Anything below 0.80 goes to review.

## Text-layer baseline backend

The default backend answers the same contract without a model: pages are classified from large-type titles, keywords, "page n of m" and continuation cues (softmax over scores as confidence); values are read next to or below their labels and checked against the field kind; tables are read by locating the header line and assigning phrases to columns. It is a baseline and a deterministic test harness, not a general document reader — image-only pages need a VLM backend.

## Testing approach

A seeded packet generator renders packets with bills of lading in two layouts, invoices that may run over two pages, packing lists that may be "scanned", arrival notices and terms pages, and records the ground truth of every value and box. On the 4 demo packets and 24 random seeds (136 pages, 96 documents, 5,251 values) the pipeline reproduced every page type, document split and value; localised boxes overlap the generator's boxes with a mean IoU of 0.79 (minimum 0.74), the gap coming from font-metric boxes vs. per-glyph boxes. These are **synthetic consistency figures**, not accuracy on real documents.

## Technology

FastAPI · SQLite · PDF rendering and text layer with per-character boxes · vanilla JavaScript and CSS with inline SVG overlays (no framework, no build step) · Playwright for screenshot capture · optional OpenAI-compatible and Gemini VLM backends.
