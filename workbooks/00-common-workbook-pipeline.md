# JLPT "Adventure" Workbooks — Common Build Pipeline (Level- & Type-Agnostic)

The shared foundation for every printable JLPT **Adventure** workbook — **grammar**, **vocabulary**,
and **kanji** — at **any level (N5 → N1)**. The three content-type manuals in this folder
(`grammar-workbook-build-manual.md`, `vocabulary-workbook-build-manual.md`,
`kanji-workbook-build-manual.md`) cover only what is *unique* to their content unit; everything
common — page geometry, design system, render pipeline, QA method, environment gotchas — lives here.

> Reference implementations: *My N4 Grammar Adventure* (128 gems / 18 worlds) and *My N4 Vocabulary
> Adventure* (682 words / 19 worlds). Every level- or count-specific number is a **parameter**, not a
> constant — see §10.

---

## 1. The product & house style

A single body of content, shipped as a **print-ready PDF** plus its build sources. Tone: warm,
child-friendly but not babyish (pitch to the level's typical reader — see §10); a **Shiba mascot
("Bunpo-chan")** gives encouragement. Each learning unit is a collectible **"gem."** Gems are grouped
into themed **"worlds"** with playful adventure names.

**Book spine (same order for all three types):**
- **Front matter:** cover · copyright/credits · How to Use This Book · Table of Contents · Adventure Map.
- **Per world:** world opener (a "collect the gems" checklist) → content pages (one-per-page for grammar,
  card-grids for vocab/kanji) → Practice → Review.
- **Back matter:** Progress Tracker · Answer Key · reference list/index · certificate · back cover.

**House style (hold these across the whole series):**
- **British English** (colour/favour/realise/programme/holidays) — pick once, never drift.
- **ASCII-clean punctuation** in learner text (no smart-quotes, em-dashes, ellipses) for uniformity.
- **No added gender** the Japanese doesn't specify. `あの人`/omitted subject → "they"/neutral, never
  "he/she"; `おばあさん→her`, `弟→him` are fine because the JP specifies.
- **Co-authors / branding** are constants set once in the builder.

---

## 2. Format & page geometry (CURRENT: 6"×9" KDP)

The series ships as **Amazon KDP trade paperback, 6" × 9" (152.4 × 228.6 mm)**.

- `@page{ size:6in 9in; margin:0; }` and `.page{ width:6in; height:9in; overflow:hidden;
  padding:13mm 12mm 15mm; display:flex; flex-direction:column; }`
- **Full-bleed cream background** so trimming leaves no white edge.
- **`overflow:hidden` on every `.page`** is the backstop against any element overflowing the trim.
- `print-color-adjust:exact` on `html` so background tints/chips print.

> **History (do not reintroduce):** the first drafts were A5 with a 2-up A4 imposition step
> (`_wb_impose_2up.py`) and an even-page-count rule. The series was re-trimmed to **6"×9" KDP**; the
> user rejected A5. **KDP prints single pages — there is no 2-up step and no even-count requirement.**
> If you find references to A5 / 2-up / `impose`, they are superseded.

---

## 3. Content source = one canonical JSON (the source of truth)

Each workbook's content lives in **one JSON file**, one record per gem, keyed by a stable headword/id.
The builder loads it directly and is the *only* place content is authored or corrected.

- Grammar: `_wb_entries.json` · Vocabulary: `_n4_vocab_full.json` · Kanji: `_<lvl>_kanji_full.json`.
- **Edit content in the JSON only** — never in the generated HTML or the PDF (they are derived and
  overwritten on every build). Edit **keyed by headword/id** via a tiny script (avoids the ambiguity
  of duplicate gloss strings), and **back up first** (`cp file.json file.json.bak_<reason>`).
- **Reviewer-facing DOCX (optional).** Early editions used a DOCX as a "fidelity anchor" (content
  corrected only in the docx, extracted verbatim, then a gate proved every field appeared in the PDF).
  That discipline is sound for a heavy external-review cycle, but the series has since moved to
  **JSON-direct** for agility. If you run a formal external review, generate a read-only DOCX *from*
  the JSON for the reviewer — but the JSON stays canonical; fold their fixes back into the JSON.

---

## 4. Render pipeline — HTML → Edge → fitz (600 DPI)

```
[1] JSON source        _<type>_full.json / _wb_entries.json
       v   (python _<builder>.py  — emits one big self-contained HTML; re-embeds mascot art as base64)
[2] HTML               N<lvl>_<Type>_Workbook/<book>.html
       v   (python _print2x.py <html> <out_hq.pdf> <edge-profile>)
[3] 600-DPI PDF        _hq2_<type>.pdf
       v   (stamp: copy to a datetime-stamped deliverable; keep only the latest)
[4] Deliverable        <book>_<YYYYMMDD_HHMMSS>_screen.pdf
```

**The 600-DPI technique (`_print2x.py`) — the core print-quality trick:**
1. Take the built HTML, rewrite the `@page` rule to **double size (12in × 18in)** and add `html{zoom:2}`.
2. Render that 2× page with **Edge headless** to PDF (so Edge rasterises embedded images — the mascot
   PNGs — at 300 DPI *of the doubled page* = **~600 DPI of the real 6×9**, while text stays vector).
3. **Downscale back to 6×9** with **PyMuPDF (`fitz`)**: `new_page(width=432,height=648)` (6×9 in points)
   + `show_pdf_page(rect, src, i)`; save with `garbage=4, deflate=True, deflate_images=True, clean=True`.

**Edge headless flags (exact):**
```
msedge --headless=new --disable-gpu --no-pdf-header-footer --virtual-time-budget=120000 \
  --run-all-compositor-stages-before-draw --user-data-dir="<temp-profile>" \
  "--print-to-pdf=<out>.pdf" "file:///<html, spaces as %20>"
```
Edge path (Windows): `C:\Program Files (x86)\Microsoft\Edge\Application\msedge.exe`. Use a **fresh temp
`--user-data-dir` per run** to avoid profile locks. Google Fonts load via `@import` (network) — keep a
generous `virtual-time-budget`. `fallback_task_provider`/SmartScreen lines in stderr are benign noise.

---

## 5. Design system

- **Fonts (Google, embedded via `@import`):** `--display:"Baloo 2"` (headings) · `--jp:"Zen Maru
  Gothic"` (Japanese) · `--en:"Nunito"` (English) · `--hand:"Klee One"` (mascot/handwritten).
- **Palette:** a fixed set of ribbon themes (`matcha / coral / sky / lav / sakura / gold` + each
  `-soft`), one `--var` per. Each world is assigned a theme tuple; assign so adjacent worlds differ.
  Per-world page tints are pale versions of the theme applied as the section background.
- **Charm layer:** washi-tape strips, a dotted-paper background (`.page.dot`), the Shiba mascot in
  speech-bubble callouts, gem/star/blossom SVG symbols. Defined once, reused everywhere.
- **Mascot art** is embedded as **base64** in the HTML (so the file is self-contained for Edge);
  source PNGs live in `mascot_assets/`. Keep raster art ≤ ~800 px native — the 2× render already
  doubles it; oversized embeds make Edge's print pass time out.
- **Series uniformity is mandatory from build #1 — not a retrofit.** Every type-edition (grammar /
  vocab / kanji) ports the SAME charm layer: `SVG_DEFS` (the base64 Bunpo-chan mascot from
  `mascot_assets/`), washi tape, the "notebook" cover, and the opener mascot speech-bubble callout.
  **Never ship emoji placeholders** for the mascot — a v1 that renders but uses emoji is *not*
  series-uniform. (The N4 kanji v1 shipped with emoji and needed a full mascot/cover/opener retrofit;
  port the shared layer upfront instead.)
- **Print darkness:** trace glyphs and guide-grid lines tuned to look right *on screen* are **too faint
  on paper**. Darken them (trace ~`.28` alpha, grid lines ~`.34`) and confirm on a **rendered raster**,
  not the HTML. Likewise, a review/exercise page must test **every** item in its section (not a subset)
  and **fill** the page; paginate any reference index that would overflow a single page.

---

## 6. The world opener (the "treasure board") — shared component

Each world starts with a single-column checklist of its gems ("Tick each gem as you collect it!").
**Single column on purpose:** a 2-column layout fuses adjacent items in content-order PDF extractors.

- Layout is a 3-column **CSS grid**: `grid-template-columns:auto max-content 1fr` =
  **[checkbox] [Japanese] [English]**. `max-content` locks the English column to a fixed start per page;
  the English is left-aligned right after it.
- **Balance the vertical rhythm:** the gap *above* the white checklist box (pill→box) should ≈ the gap
  *below* it (box→callout). These are set by the box's top/bottom margin + the callout's top margin.
- **English column readability:** keep the English a notch darker than a mid-grey (≈ `#5e5e76`) so the
  right side doesn't read as empty; keep the JP→EN column-gap tight (~`.2–.3em`).
- **Inherent trade-off:** a *fixed* (aligned) English column means short headwords have visible lead-in
  space before the English. That is the cost of alignment; do not "fix" it by right-aligning the JP
  (that flings the checkbox far from short words). If you want zero gap, you must drop alignment
  (ragged) — that was the deliberate choice for the back-of-book index, not the openers.

---

## 7. Japanese typography rules (critical)

- **Deliberate word-spacing:** author Japanese example/phrase text with **spaces between bunsetsu**
  (`へやに 入って ください。`). It aids the young reader and gives the line clean break opportunities.
- **`word-break:keep-all` on every Japanese sentence element** (`.exj` example, `.uc` use-phrase, etc.).
  Without it, CJK wraps between *any* two characters and orphans a trailing kana on the next line
  (`…教えて もらいま` / `す。`). With `keep-all`, lines break **only at the inter-bunsetsu spaces**, so
  conjugations (`もらいます。`, `しました。`) stay whole. This is mandatory for print polish.
- **Furigana/readings:** a vocab/kanji book *shows* the reading (it teaches it); a grammar book omits
  furigana on purpose. Never mix kanji+kana inside one word (`京と`, `問だい`) — run a kanji→kana scan.

---

## 8. ⚠️ PDF text-EXTRACTION artifacts vs visual defects (the #1 review trap)

A reviewer who **copies/extracts text** from the PDF (or uses a screen reader / automated text check)
will see "bugs" that **are not on the printed page**. Before changing source for any reported text
defect, **verify visually** (render the page to PNG) *and* with extraction (`page.get_text()`) side by
side. **Never edit source to fix an extraction artifact — it damages the visual or is a no-op.**

Known artifact classes (all confirmed on this series):
| Extraction shows | Reality on the page | Cause |
|---|---|---|
| `People & Relationshipsch.1` | "People & Relationships … ch.1" (chip far right) | two adjacent flex `<span>`s, no text node between |
| `V O C A B U L A R Y`, `S R I VAS TAVA` | clean tracked display type | CSS `letter-spacing`; extractor inserts spaces between glyphs |
| `挨拶 section managerちゃん…` | clean two-column "draw a line" exercise | extractor reads across the two columns row-wise |
| `お見舞いvisiting`, `コンピュータcomputer` | form and gloss in separate grid/block cells | no space node between separate elements |
| `USE**_**…_**` | bold label + italic text | bold/italic styling rendered as markdown-ish markers |
| `しまし た`, `もらいま す` | a sentence that *wrapped* mid-bunsetsu | line-wrap (fix with §7 `keep-all`), **not** a source space |

**Real** text defects (incomplete sentences, wrong meaning, mismatched phrase↔example) are fixed in the
JSON. **Artifacts** are left alone. Forcing spaces into the opener grid would break its
`auto/max-content/1fr` columns; removing letter-spacing would degrade the headers.

---

## 9. QA & verification method (verify, don't assume)

File edits are invisible until rendered. **Always render → read the PNG/PDF → only then report.**

- **Rasterise specific pages** with fitz: `doc[p].get_pixmap(matrix=fitz.Matrix(200/72,200/72))` (~200
  DPI) → save PNG → look at it. To find a gem's card page, `page.search_for(term)` and skip the opener
  (contains "Gems to collect") and the back-matter index (`page_index >= index_start`).
- **Check the worst cases, not a sample:** the **densest** content page (most items) for clipping, the
  **sparsest** for emptiness, plus the longest gloss/example. The card grid is **height-constrained**;
  long content clips silently behind `overflow:hidden`.
- **Measure when "pixel-perfect" matters** (opener gap balance, callout-vs-box alignment): scan pixel
  rows/columns for the white-box and callout bands; treat soft drop-shadows as ±2–3 px noise.
- **Programmatic scans** (cheap, run every review pass): kanji→kana mixing; provenance-tag leakage
  (must be 0); British/US spelling drift; gendered-pronoun candidates (verify each against the JP);
  polite-form consistency (examples end です/ます…); for vocab, reading-present + POS-in-controlled-set.

---

## 10. Per-level parameters (what changes between N5…N1)

| Parameter | N5 | N4 (ref) | N3 | N2 | N1 |
|-----------|----|----------|----|----|----|
| Typical reader | younger child | ~12 yr | teen | teen/adult | adult |
| Tone | most playful | playful | playful-lite | neutral-warm | neutral |
| Kanji policy | mostly kana | N5–N4 whitelist | + N3 | + N2 | + N1 |
| Example complexity | very simple | simple | moderate | longer | nuanced |
| Grammar gems | ~80 | **128** | ~180 | ~210 | ~250 |
| Vocab words | ~650 | **682** | ~1.5k | ~1.8k | ~2k+ |
| Kanji | ~80 | ~170 | ~370 | ~1k | ~2k |

**Level-invariant (leave alone):** the 6×9 geometry, the render flags, the `_print2x` technique, the
design system, the QA method, the environment workarounds. **Per-level (change):** the source corpus
JSON, the world taxonomy (`_wb_cats.py`-style tuples), the kanji whitelist, the `N<lvl>_…/` paths,
co-author/branding constants, and the per-level review prompt.

---

## 11. Deliverables & cleanup discipline

- **Datetime-stamp every generated deliverable:** `<book>_<YYYYMMDD_HHMMSS>_screen.pdf`. Working files
  (scripts, template, taxonomy, the source JSON) keep stable names.
- **Keep only the latest** stamped PDF in the deliverable folder; delete superseded stamps. The
  intermediate `_hq2_*.pdf` is a duplicate of the latest deliverable — remove it after stamping.
- **Keep content backups** (`*.json.bak_<reason>`) so any single edit is revertible.
- **Remove scratch** (one-shot verify/dump scripts, `_verify/`, `_diag/` PNG folders) when done.
- Point the user at the newest stamped file; flag older copies as superseded.
- **Never run an age-based sweep on a folder that also holds SOURCE** (added 2026-09-20, N4 kanji).
  The retention rule above is safe only while `N<lvl>_<Type>_Workbook/` contains deliverables plus
  genuine inputs. Once the build modules, bank JSONs and the pending-items register live in the same
  folder (which is where a "put everything for this workbook in one place" consolidation lands), an
  "older than N days" rule matches them too: stable-named source carries no datetime stamp, so it is
  *always* old. Scope a sweep to `old/` and exclude `*.py`, `*.json`, `*.md` and any `_refcache/`
  even there. Check whether the folder is gitignored first: if it is, a wrong delete has no undo.
  When a folder's contents change class, update the retention note that governs it in the same pass.

---

## 12. Environment & tooling pitfalls (Windows)

- **cp932 console mangles Japanese** in `python -c "…日本語…"` (throws / corrupts). **Write Japanese to
  UTF-8 files and read them back; print only ASCII** summaries/counts. Source `.py/.html/.json` are
  UTF-8 — Japanese inside files is fine.
- **Open PDF viewers lock the file** ("being used by another process" on delete/overwrite). The new
  stamped render is unaffected; ask the user to close the old one, then delete it.
- **`fitz` (PyMuPDF)** does both the downscale and all verification rasterisation — it is the one
  hard dependency beyond Edge.
- **EFS-encrypted working tree (env-specific):** on some corporate machines the workbook folder is
  EFS-encrypted; if the user's EFS key is unavailable, file reads fail with "Access denied" even though
  they own the files. That is an environment issue, not a build bug.
- **Verify by rendering, never by assuming.** "It should work" is not evidence.

## 13. Automated design/print-QA gates (added 2026-07-26, N4 kanji GD pass)

A reviewer's 20-category book-design spec (negative space, margins, density, overflow, concatenation,
line-break, typography, hierarchy, cards, pagination, PDF-technical, full-book consistency) was largely
mechanised. Reusable checks (all in a `render_print_qa.py`-style renderer, PyMuPDF over the rendered PDF):

- **PDF-BALANCE (negative space, the biggest previously-manual gate).** Rasterise each page (~48 DPI),
  count dark rows in the central column band, and flag an *ordinary* page (exclude covers, world openers,
  colophon, short blocks) whose longest ink-free band > ~30 % of the content height **AND** whose inked
  fraction ≥ ~35 % (i.e. enough content to fill but clustered). This catches "content bunched at top,
  big void below" without false-flagging genuinely sparse pages (design rule: sparse is fine for
  openers/pauses/short blocks). On the N4 kanji book it flagged 84 pages and forced the fix below.
- **The fix it forces = distribute, don't dump the void.** `.page`/`.kcard`/`.pcard` are flex columns;
  top-aligning content + `margin-top:auto` on the footer strip collapses all slack into one dead gap.
  Instead: `justify-content:space-between` (practice/answer cards → items spread to fill) or, for a card
  that must read as evenly-spaced, `justify-content:space-evenly` + wrap each heading-with-its-content in
  a `.sec` group (so headings hug content while the *inter-section* gaps become uniform = equal top,
  bottom and between). This is how you get "equal top/bottom margin + uniform section spacing."
- **Adaptive vs uniform fill (a real tension to decide per page type).** A flex-`1` grid *fills* the
  leftover space (good for a writing/trace grid), but that conflicts with "uniform rhythm." If the ask is
  uniform spacing, make the grid a normal fixed block and let `space-evenly` distribute; if the ask is
  "fill the bottom," make it flex-grow. Don't do both.
- **PDF text-layer fidelity (searchable/accessible layer, distinct from visible glyphs):**
  - **PDF-NONEMPTY** — every content page has extractable text (only full-bleed image covers may be
    text-less); catches an unexpected blank page.
  - **PDF-TEXTCLEAN** — sampled Latin strings round-trip AND *no macron characters* appear in the
    extracted text. **Chrome/Edge print-to-pdf corrupts precomposed macron vowels in subsetted fonts**
    (ToUnicode bug: `Jōyō` → `Joōyoō`); the glyphs *print* correctly but copy/paste + search break.
    **Fix: use ASCII romanisation (`Joyo`) for searchability** — do not chase the font-subset ToUnicode.
  - **PDF-KINSOKU** — hard 行頭/行末 禁則 over every real wrap point (consecutive JP lines that drop to a
    lower row in the same block; standalone labels like （れい） are their own block, not a wrap). Small-
    kana line-starts are a *soft* rule → surface as a warning, don't gate.
  - **Reading-integrity / split readings** — see §8: `あ (く)` in extracted text is an EXTRACTION
    artifact, not a visual space, as long as the *source* reading is contiguous. Prove it with a source
    check (no internal space in any reading) rather than trusting/eyeballing the PDF text layer.
- **Offline-font parity (PDF-FONTDIFF).** Render a 2nd time with the web-font CDN blocked
  (`--host-resolver-rules="MAP fonts.gstatic.com 127.0.0.1,MAP fonts.googleapis.com 127.0.0.1"`); the
  fixed-page structure must keep the same page count + extracted text (font-independent). Per-page line
  reflow will differ (font metrics) — surface it, don't gate. Confirms graceful offline degradation.
- **Portability of embedded QA scripts:** discover the browser from env vars + PATH, never a hard-coded
  `C:\…` literal (an embedded-script portability audit flags drive-letter paths).
- **Morphology gates (MeCab via `fugashi`+`unidic-lite`), and their limits.** Reliable *deterministic*
  checks: a FUTURE adverb + PAST predicate = temporal contradiction (`来年…来ました`) — but exclude the
  *adnominal* `来年の…` (modifies a noun, doesn't govern tense); okurigana = the written kana tail must
  equal the reading tail; readings contain no internal space. **DON'T ship a `fugashi` *surface*-POS
  option-parallelism check** — surface POS is unreliable on the kana-written words common in N4 options
  (na-adjectives tag 名詞, i-adjectives like つまらない tag 動詞, homographs みせ→見せ/動詞, counters vary),
  giving ~100 % false positives. Reliable option-parallelism needs *dictionary* POS (JMdict adj-i/adj-na
  /n), or leave it a human/HYBRID check.
- **TC honesty for design rules:** aesthetic judgments (illustration "feel", card balance, hierarchy
  nuance) get an automated *proxy* (BALANCE/geometry) but keep a MANUAL_RENDERED flag — never label a
  pixel-aesthetic call a deterministic pass.

---

## 14. Answer-key integrity (added 2026-09-06, N4 kanji build; applies to every workbook type)

**The single worst defect this project has shipped**, and every automated gate stayed green through it:
**164 of 245 answer-key entries (67 %) pointed the learner at the wrong answer.**

**Root cause: two independent numbering schemes that were free to drift.** The question pages numbered
items with a running counter over a `type-group` render order (all 漢字読み first, then 表記, then
文脈規定 ...). The answer key derived its number from the item ID (`int(id.split("Q")[1])`) and iterated
the bank in FILE order. The two agree only when the bank file order happens to match the render order.
It did not, because the "mixed review" slot (M5) holds items whose TYPE sends them back up into もんだい1.
World 1 printed 立てました as question 6; the key filed it at 14 and put じぶん at 6.

**Why nothing caught it.** Every existing check validated items in isolation: correct-option position,
CORRECT marker placement, explanation-names-the-answer, per-bank counts. None compared the number
PRINTED BESIDE THE QUESTION with the number PRINTED ON ITS KEY ENTRY. Bank validation passed 15/15
throughout.

**The invariant to build in, not to check for:**

- Compute the display numbering **once**, from a single named constant for the section order, and have
  both the page renderer and the answer key read that same map. Do not let the key re-derive a number
  from an ID, a file position, or anything else.
- Emit answer-key entries **sorted by that display number**, so entries run 1..N down the page.
- **Assert equality at render time** so a future drift fails the build instead of shipping:
  `assert DISPNUM[q_id] == printed_n`. A gate that runs after the fact is weaker than a build that
  cannot produce the defect.

**Regression check to keep:** parse the rendered HTML for every `question id -> printed number` and every
`answer-key qid -> entry number`, and assert the two maps are identical for all N items. This runs in
seconds and is the only check that would have caught the original bug.

**Related trap - a stale checker outliving the bug.** The throwaway script written to MEASURE this
defect replicated the old key logic. After the fix it kept reporting 164 mismatches against a correct
build. Verify against the RENDERED artifact, and delete or update measurement scripts once the thing
they model has changed.

## 15. False-positive control for language audits (added 2026-09-06)

An external reviewer with only the rendered PDF produced 26 findings over four rounds; 23 were real.
Auditing without the source is legitimate and it found things the source-side scans missed, but these
classes recur and must be triaged before reporting:

- **Kana-headword collision.** Looking up a kana surface returns every homograph: あく gives a spring
  wind, おおい gives "ahoy!", ことり gives "click", はな gives an emphasis particle. Look up the KANJI
  form, or constrain by reading.
- **English stemming.** "sells" vs "sell", "characters" vs "character", "uneasy" vs "uneasiness" are not
  defects. Compare stems or accept substring matches in either direction.
- **Naive de-conjugation.** 通って stems to 通る, missing 通う.
- **One JMdict sense read as the whole entry.** A `uk` tag on a minor sense is not a ban on the kanji
  (犬, 目, 会う, 味噌 are properly written in kanji). Conversely a gloss can be verbatim-attested and still
  be the wrong sense to teach (see kanji manual class 21).
- **PDF text-layer artifacts.** Extraction inserts spaces inside words and concatenates blocks. Confirm
  every spacing or punctuation candidate against RENDERED PIXELS, not the text layer. In this cycle a
  reviewer twice reported a missing hyphen (`かざ` for `かざ-`) that a 400 dpi crop showed present; it
  renders as a short low-set dash.
- **An intentionally wrong option in a usage question** is not a defect; being wrong is its job.
- **Inverted threshold checks.** An aspect check that flags "habitual context + bare non-past" must fire
  when NEITHER a habitual adverb NOR `ている` is present. After a fix ADDED まいあさ, the check kept firing
  on the now-correct sentence. Re-read the check's polarity whenever it fires on something you just fixed.

**Two coupling traps worth pre-empting on any gloss or reading edit:**

1. **A same-card uniqueness rule can be tripped by fixing one entry.** Correcting 早い to "early; soon"
   made three of four examples on the 早 card lead with "early", failing a "first sense unique per card"
   gate. Budget for a second, compensating edit sourced from the same dictionary entry.
2. **Never read a threshold gate flipping green as a fix.** A balance gate that had failed for three
   builds went green in a pass that did not target it: the measured gap was unchanged (116 -> 117 rows
   against a 103 limit) and only the ink fraction drifted across a 0.350 cut because an unrelated padding
   change reflowed the page. When a long-standing failure disappears in an unrelated change, measure the
   underlying quantities before recording it as resolved.

---

## 16. Relocating a build: what breaks that no gate reports (added 2026-09-20, N4 kanji)

Consolidating a workbook's scripts, banks and state into its own folder is a good end-state, but the
move itself has a failure mode that every green gate hides.

**Do the dependency walk with the import graph, not from memory.** The first claim recorded on the
N4 move ("the vocabulary build imports the kanji core") was wrong; a real import-graph walk showed
the two closures share nothing. State the graph you actually walked, not the one you remember.

**Classify every file by CONSUMER before moving it, not by topic.** A file that looks like workbook
data can be read by a shipped website, a sibling level, or a downstream folder. On the N4 move
`data/kanji.json` and `data/<level>_kanji_readings.json` had to stay put for exactly this reason
(live site JS, a service worker, and 15 scripts in `tools/`), while a `mascot_assets/` folder assumed
to be kanji-only turned out to be shared with the vocabulary build. **The test is "who reads it",
not "what is it about".**

**A scripted path rewrite misses at least three shapes.** Budget for hand fixes: (a) an alias, where
a module reaches the parent through its own variable (`N4 = os.path.dirname(WB)`) and the pattern
only matched the literal form; (b) a call whose `os.path.join(...)` **spans two source lines**, which
a line-oriented regex cannot see; (c) any consumer OUTSIDE the moved set. Each of these surfaced only
as a concrete runtime failure.

**The dangerous one: an auto-downloading cache converts a broken path into a silent pass.** The N4
`verify_kanji_data.py` lives in `tools/`, was not in the moved set, and still pointed at the old
`_refcache/`. Because it auto-fetches a missing cache it re-downloaded 12 MB from edrdg.org and
reported its usual `0 FAIL`. Nothing failed, so nothing surfaced. The real damage was a **split
dictionary baseline**: that gate judged against a September JMdict while three sibling gates read the
July archive, so two gates could legitimately disagree about the same word. Rules:

- After moving a shared cache, grep its name across the **whole repo**, not the moved file set.
- Prefer a resolver that checks the canonical location and **exits non-zero when absent**
  (`print("BLOCKED: missing ..."); sys.exit(2)`) over one that quietly downloads a fresh snapshot.
  An auto-download is convenient on a first run and a correctness hazard on every run after.
- Re-run the gate after repointing and confirm the **findings are identical**, not merely that it
  still passes. On the N4 fix: 0 FAIL / 13 with the same four advisory WARNs as before the move.
- Keep the pre-move backup and the orphaned old cache until the new layout is trusted; do not delete
  either as part of the move.

## 17. Gloss an INFLECTED target word in its dictionary sense (added 2026-09-20, N4 kanji)

A practice item whose `target_word` is inflected must carry the **dictionary** sense in its metadata
gloss, never the tense or polarity the carrier sentence happens to supply. N4 MOCK-M4-Q28 underlined
`帰って` inside `帰って いません` and glossed it **"returned home"**: the te-form has no tense of its
own, the sentence is a negative present perfect, and the item's own explanation already said "hasn't
returned". Corrected to "to go home; to return".

Why no gate caught it, which is the part that generalises: the `meaning` field is **metadata for the
review spreadsheet and is not rendered into the printed item**, so every render-fidelity check steps
over it, and the gloss-grounding checks run over lesson-card examples rather than bank items. Any
field that is neither printed nor gate-covered is reachable only by reading. When you add such a
field, either render it, gate it, or record it as read-only-by-human in the register. An unexamined
metadata field drifts silently. Cheap check to add: for every bank item, assert the gloss has no
past-tense or negative marker unless the `target_word` is in dictionary form.

---

## 18. Recording a review sign-off without corrupting the evidence (added 2026-09-20, N4 kanji)

When the reviewer's verdicts finally come in, the temptation is to type PASS into the status line.
Do not. A sign-off is data, and the gate should be a *function* of it.

**Derive the gate row from the per-item sheet; never write both.** The N4 execution-log row R12 is
computed by reading the Review Items columns (`review_result`, `recheck_result`) at log-build time,
so the log row physically cannot contradict the item verdicts. This was itself added after a
reviewer caught the log saying NOT_EXECUTED while item results were populated. Back it with a
self-audit assertion from the other direction: a manual PASS row is valid only if the sheet shows
0 REVISE and 0 blank verdict. One value, two independent readings.

**Flip membership in a named set, not the cells.** Keep the revised-then-passed items in a set whose
branch already emits PASS / recheck-passed / high confidence **while retaining `flag_reason` and
`suggested_fix`**. Moving an id between sets gets the whole verdict shape right and preserves the
defect-to-fix-to-recheck trail for free. Keep the now-empty REVISE set rather than deleting it, so
the next round has somewhere to land.

**Close an advisory warning with a PINNED exception list, never by editing content or loosening the
check.** The four N4 WARN-7 glosses were reviewed as accurate with no content change. The closure is
a set of exact `"form=gloss"` strings: edit either side and the acceptance lapses and the item warns
again; add a new ungrounded gloss and it warns as before. Also report pinned entries that have
STOPPED firing, so the list cannot silently rot. Deleting the check or raising its threshold would
have removed the signal for every future gloss. Comment the list with "do not add an entry here to
silence a gloss you have not had reviewed".

**Be explicit that a heuristic's WARN is not an accuracy verdict.** Check 7 warns when a gloss shares
no WORD with a JMdict sense; an accurate paraphrase can legitimately do that. Recording "reviewed,
accepted" is the correct resolution, and it is a different act from "fixed".

**Do not rebuild artifacts a status field never reached.** Prove it rather than assume it: grep the
built HTML for the field (0 occurrences), confirm no builder reads it, and confirm the content hash
covers only the item banks. Then run the render-fidelity gate against the EXISTING book with the
EDITED data: a pass there is the evidence that the artifact still matches its source. Rebuilding a
400-page print master to change a metadata flag orphans a signed-off binary for nothing.

**Say what the sign-off did NOT clear.** A sha-bound rendered-visual ledger correctly reverts to
PENDING when the book binary changes, and clearing the language gate does not clear it, the physical
print check, or an independent recheck. Update every stale status line in the register in the same
pass (strike through, do not delete: the record of what was open and why is worth keeping), and
re-run the gates to produce the evidence rather than reporting the edits you made.

---

*This common manual + the three type manuals encode the lessons of the N4 grammar and vocabulary first
editions (build + dozens of review/correction rounds). When a build surfaces a new defect class or
invariant, add it to the right manual in the same pass that fixes it.*
