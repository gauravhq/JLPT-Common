# QA Sweep Checklist (cross-level workbook builds)

Purpose: open every QA round with ONE full-category sweep instead of chasing the
first-spotted defect and discovering the rest next round. Derived from the actual
defect history of the N4 Kanji "Adventure" build (reviewer rounds 1-6, VQA batches,
design passes). Reusable for N5/N3/N2 editions.

**How to use:** run Part A (it is mechanical, no judgment). Then sweep Part B
template-wide, not page-wide. Then confirm Part C before sending anything out.

---

## Part A - already automated: DO NOT hand-check, just run the gate

These fire on every build regardless of whether anyone mentions them. Briefing adds
nothing here. A sweep of Part A means "run the pipeline and read the counts."

| Category | Owning check |
|---|---|
| Answer leakage into the question region | `html_qa` H03 |
| Answer keys appearing before the divider | `html_qa` H08 |
| Answer number != keyed correct option | `html_qa` EH03 |
| Explanation text drift from the bank | `html_qa` EH09 |
| Item filed under the wrong 問題 heading | `html_qa` H14 |
| Copy-paste residue in explanations | `html_qa` EXPL-TGT |
| Explanation template uniformity | `html_qa` EXPL-FMT |
| Distractor validity (4 opts, one CORRECT at the keyed position) | `html_qa` AK04/AK05 |
| Item counts + type distribution drift | `qa_selfaudit` A11a/A11b, A13b/A13c |
| Stale counts anywhere in the workbook | `qa_selfaudit` A13a |
| Superseded / corrected text reappearing | `html_qa` CH02 |
| Empty field / unresolved placeholder | `html_qa` DOM05 |
| Stray ruby / furigana | `html_qa` H06 |
| Images (visual answer clues) in practice cards | `html_qa` H13 |
| Mojibake, escapes, unicode fidelity | `html_qa` EH07 + MOJI, `render` PDF-MOJI |
| Out-of-scope kanji for the world's cumulative pool | `practice_validate` |
| Lesson-page fidelity vs `kanji.json` | `html_kanji_qa` (13 checks) |
| Trim geometry, page count | `render` PDF-GEOM, PDF-PAGES |
| Font embedding, missing glyphs, tofu | `render` PDF-FONTS, PDF-GLYPH, PDF-TOFU |
| Kinsoku (forbidden line-start/line-end) | `render` PDF-KINSOKU |
| Reading/label split across a line | `render` PDF-INTEGRITY |
| Blank pages, unextractable text | `render` PDF-NONEMPTY |
| Clustered whitespace voids | `render` PDF-BALANCE |
| Sparse-page pagination regression | `render` PDF-PAGINATION |
| Offline-font degradation | `render` PDF-FONTDIFF |
| Pairing / build identity (book == workbook == banks) | `html_qa` CH05/CH06 + R07 `in_html_sha` |
| Unevidenced PASS claims in prose | `qa_selfaudit` A7/A7b/A7c/A10 |
| Detectors that cannot fail | negative tests (mutation) |

---

## Part B - MANUAL, sweep these template-wide EVERY round

This is where rounds actually get lost. The failure mode is checking only the page
that was pointed at. **Sweep by TEMPLATE, not by page:** the book has ~14 page
templates (cover, credits, how_to, world_divider, kanji_lesson, review_opener,
world_practice, mock_opener, mock_practice, answer_key_divider, answer_key,
mock_answer_key, appendix, back_cover). A finding on one instance applies to every
instance. `visual_qa_package/visual_sample_manifest.csv` already maps each sampled
page to its full horizontal-deployment scope; use that column as the sweep list.

### B1. Layout and negative space
- Undirected whitespace: a short block floating in the vertical middle with a large empty band.
- Top-heavy / bottom-heavy: an over-corrected "lift" that just moves the empty band.
- Empty lower half on a top-loading page (appendix/credits pattern).
- Mascot marooned at the trim edge instead of inside the content group.
- Sparse tails without a deliberate section-ender.

### B2. Repetition and scannability
- The same orientation sentence printed on every instance of a template. Say it ONCE, on the section divider.
- A heading and the note directly beneath it saying the same thing.
- Dense reference pages (answer keys) with no lookup gutter: can the eye run down the item numbers?
- Duplicated content across sections (search the whole output before adding).

### B3. Typography and micro-alignment
- Captioned values floating centered instead of chip-anchored on their heading's baseline.
- Competing weights on one line: is the intended anchor (the kanji) the largest thing?
- Line-height and colour on dense legal/colophon text.
- Punctuation discipline: em dash, en dash, ellipsis, smart quotes. Marketing surfaces get scrutinized most and are the easiest to miss.

### B4. Visual elements
- Watermark legible as intentional, not a half-hidden blob.
- Shared watermark/decoration classes: tuning one page changes every page using the class.
- Decoration framing the hero vs scattered in corners.

### B5a. Gloss quality (English side; an AI review can narrow this, not clear it)
- **Does the gloss build the RIGHT mental model?** A dictionary-defensible gloss can still mislead:
  下町 glossed "downtown" reads as city-centre/CBD to an English speaker, when 下町 is the
  low-lying merchant-artisan quarter (vs 山の手). Judge by the picture it puts in the reader's head.
- **Do the two corpora agree?** Compare each shared word's gloss on the kanji card against the
  practice item. Where they differ, one is usually better; the book should not disagree with itself.
  Ignore harmless wording variants ("autumn; fall" vs "autumn") and look for real divergence:
  noun/adjective inversions, and pairs glossed as if identical when they are not (特急 "limited
  express" vs 急行 "express" are different service classes).
- **Writability, not just readability.** The readability policy (out-of-scope kanji carry a reading)
  only guarantees a learner can READ an example. Count how many of each card's examples are writable
  from the taught set and flag any card at ZERO: that undercuts a collect-and-write book. Card 可 had
  0 of 3 (可能/許可/可能性 need 能/許/性) until 可能性 was swapped for 不可.
- **Register for the age band.** Formal words are fine when the carrier justifies them (学業 in a
  校長先生's speech). Flag them only when the context does not earn the formality.
- **Homophone pairs where BOTH members appear in the book** deserve a cross-reference note. Highest
  value here: かじ 家事/火事 and きる 切る/着る.
- Two mechanical traps when rewording a gloss: it must not duplicate another gloss's FIRST WORD on
  the same card, and it should share vocabulary with a JMdict sense for that word. Both are checked.

### B5. Japanese naturalness (needs a native reviewer; do not self-clear)
- Predicates that do not collocate with the subject (天気があつい, adjectives that cannot predicate テストのこたえ).
- Impossible tense/aspect pairings (来年 + 来ました).
- Single-answer validity: is more than one option defensible?
- Carrier sentences whose modifier structure can be misread.

### B6. Print production (needs the physical/KDP layer)
- Barcode zone: KDP prints the ISBN barcode bottom-right on the back cover (~2 x 1.2 in). Nothing meaningful may live under it.
- Cover bleed and pixel dimensions (6x9 trim + 0.125 in bleed = 1875 x 2775 at 300 dpi).
- Spine width depends on final page count plus paper stock.
- Gutter/binding creep, paper opacity/show-through, printed colour: proof only.

---

## Part C - process guards, confirm before sending anything out

1. **Stale-copy guard (cost a full round once).** Before acting on any external review,
   confirm the reviewer's quoted values match the values on disk NOW. If they quote
   pre-fix values, the finding is a snapshot artifact, not a defect. Re-send the
   current stamped pair instead of "fixing" what is already fixed.
2. **Pairing guard.** The workbook's R07 `in_html_sha` must equal the sha256 of the
   HTML actually being shipped. A mismatched pair means the evidence describes a
   different binary.
3. **Sign-off voiding.** Visual sign-offs are bound to the book sha. Any book-changing
   fix re-renders to a new sha and voids the whole manual visual gate. Batch
   book-changing fixes into ONE re-render; never apply them piecemeal mid-audit.
4. **QA-checked markup.** If a restyle changes an element a QA regex greps
   (`<b>answer N</b>` -> `<span class="aans">`), update the selector in the SAME batch.
5. **Horizontal deployment.** After fixing one instance, grep for the same pattern
   everywhere and fix all of them in the same pass. This includes SIBLING CSS CLASSES: a
   legibility or spacing change applied to one watermark/decoration class must be applied to
   all of them, or some pages keep the old treatment.
6. **Re-measure with the project's own scan, and snapshot it FIRST.** Copy
   `visual_qa_package/full_book_layout_scan.csv` aside before the batch, re-run
   `build_visual_qa_package.py` after, and diff per template. Ad-hoc metrics disagree with it
   (watermark ink thresholds differ) and will give false comfort. `largest_blank_ratio` is a
   MAX, so list the offending pages before declaring a template regressed.
7. **A height-changing CSS fix invalidates the pagination weights.** If a fix compacts or
   expands a question type, rescale the packer's `QW` weight for that type in the SAME batch,
   biased HIGH. Then confirm `PDF-FONTDIFF` still passes: `text_eq=False` means the layout has
   no slack for font-metric variation and content is being clipped by `overflow:hidden` in the
   offline-fallback render. Over-filling loses content; under-filling is only whitespace.
8. **Verify the fix did not just move the problem.** Lowering content to fill a bottom void can
   create a top void; a shared value tuned for one template can break its sibling. Re-measure
   both the target AND the pages that were already fine.

---

## Sweep log

Record each full sweep so the next one starts from real history rather than guesses.

| Date | Build | Templates swept | Findings | Notes |
|---|---|---|---|---|
| 2026-08-24 | `9306c4fe` -> `e0643819` | 14 / 303 pages | 8 fixed, 1 self-inflicted regression caught and fixed in-batch | Top finds were invisible to page-pointed review: paraphrase option numbers orphaned on 18 pages (descendant-selector bug), and a watermark legibility bump that had reached only 1 of 3 classes (17 pages behind). Punctuation scan clean on all 301 text pages but BLIND to both rasterised covers. |
| 2026-08-26 | `8955ea09` -> `cb6429f8` | kanji lesson (170 pages) | 4 fixed; 2 rejected approaches recorded | User-reported, not sweep-found: meaning crowding the glyph, reading tiles left-hugging in a flex:1 box, and one example-row leader with a different dot pitch. Leader fix took THREE attempts: `dotted` border (browser redistributes dots to fit width), `repeating-linear-gradient` (Chrome rasterises gradients into the PDF -> clumpy dashes, WORSE), then an SVG `<pattern>` (vector, fixed 5px period, correct). h3 rules left on `dotted` as accepted variance. |
| 2026-08-24 | `a7088975` -> `8955ea09` | build identity | HASH-SCOPE-01 fixed | `content_hash()` omitted `meaning`, so a printed answer-key gloss could change without the hash moving. Added it (9 -> 10 fields); regression-tested that a meaning-only mutation now moves the digest. Hash re-based `fd0054134f9f` -> `58a1ec5f901c`; CH06 three-way match re-verified. Historical hashes labelled, not overwritten. |
| 2026-08-24 | `e0643819` -> `a7088975` | content (170 kanji + 245 items) | 4 fixed; 1 evidence gap logged | Content-accuracy sweep (B5a). Fixed: 可 had 0 of 3 writable examples (可能性 -> 不可); 下町 "downtown" built the wrong mental model; 特急 glossed as 急行 in the items; 勉学 vague. WARN-7 ungrounded glosses 6 -> 4. Editing a passed item sent MOCK-M1-Q08 back to REVISE, so the manual-language gate is no longer clean. Logged HASH-SCOPE-01: `content_hash()` omits `meaning`, so an answer-key gloss can change without the hash moving. |
