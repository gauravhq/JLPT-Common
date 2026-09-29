# JLPT Kanji "Adventure" Workbook — Build Manual (Level-Agnostic)

How to build the **kanji** edition for any level (N5 → N1). Read
**`00-common-workbook-pipeline.md`** first (shared 6"×9" KDP pipeline, render, openers, typography,
extraction-artifact discipline, QA, environment). This manual covers what is **kanji-specific**.

> ✅ **Build #1 shipped: *My N4 Kanji Adventure* (143 kanji, 12 worlds, 53 pages, 2026-06).** Values it
> confirmed are tagged **[v1-confirmed]** below, and the v0 open questions (§8) are now resolved. The
> data + assets came entirely from the existing N4 app (no network). The grammar and vocabulary editions
> remain the references for shared mechanics.

---

## 1. The content unit — a kanji card

One record per kanji in `_<lvl>_kanji_full.json`, keyed by `char`:

```json
{
  "id": "...", "char": "持", "categoryOrder": 4,
  "meaning": "hold; have",                       // English keyword(s), keyword first
  "on":  ["ジ"],                                  // on'yomi in KATAKANA
  "kun": ["も.つ"],                                // kun'yomi in HIRAGANA, '.' marks the okurigana split
  "strokes": 9,
  "radical": "扌 (て・hand)",                       // radical + its name/meaning
  "components": ["扌", "寺"],                       // optional: mnemonic component breakdown
  "examples": [                                   // 2–3 compound words, at/under level
    { "word": "持つ",   "reading": "もつ",   "en": "to hold" },
    { "word": "気持ち", "reading": "きもち", "en": "feeling" }
  ],
  "joyo_serial": 327, "joyo_grade": 3,             // [Build #3] fetched: Jōyō serial (grade-grouped order) + grade
  "compounds": [                                   // [Build #3] up to 4 = curated example(s) first, jisho/JMdict fills
    { "word": "持つ", "reading": "もつ", "en": "to hold" }
  ],
  "strokeOrderSvg": "kanjivg/06301.svg"            // path to the numbered stroke-order asset (§3)
}
```

**Card anatomy:** the **kanji** (very large) · **on'yomi** (katakana) and **kun'yomi** (hiragana, with
okurigana shown after `・` or a dot) · **English keyword(s)** · **radical + components** · **stroke
count** · a **stroke-order diagram** (numbered) · **2–3 example compounds** (word + reading + EN) · a
**writing-practice grid**.

**Density — choose per goal; the series has shipped all three (Build #2–#3): 6/page compact reference · 4/page with the build-up strip · 1/page full teaching page.** Build #1 first tried 4/page and each card had a
large empty middle; the fix was to let the **writing-practice grid fill the card body**
(`justify-content:space-evenly` over multiple rows) and pack 6/page. A kanji book is mostly writing
practice — don't leave a void. (143 kanji → 53 pages incl. front/back matter.)

**Reading discipline (kanji's #1 risk, mirrors vocab readings):** on'yomi in **katakana**, kun'yomi in
**hiragana**, okurigana split marked (`も.つ`, `あ.がる`). Show only the level-relevant readings; a kanji
can have many — curating to the readings the learner needs is an editorial decision, not a dump.

---

## 2. Writing-practice grid (the kanji-only component)

The feature a grammar/vocab book never has. Per kanji, a row of **square cells with a faint cross/quad
guide** (the 田-style grid that teaches proportion):
- **Cell 1–2:** the kanji **traced** in a pale grey (the learner traces over it).
- **Remaining cells:** empty guide squares for free writing.
- Optionally a first cell showing the **numbered stroke order** at writing size.

Build the grid in CSS (a flex/grid row of fixed-size bordered squares with a `::before` cross-hair) so
it scales cleanly and prints crisp — do **not** rasterise it. Keep the guide lines very light so they
don't dominate the printed page. Verify the grid doesn't overflow the page bottom (common §9,
`overflow:hidden` backstop).

---

## 3. Stroke-order diagrams — the main new production task

**[v1-confirmed] — KanjiVG gives three things for free; don't recompute any of them.** The level app
already ships KanjiVG SVGs at `<lvl>/svg/kanji/<glyph>.svg` (CC-BY-SA — credit on the copyright page):

- **The numbered diagram itself.** Each SVG already contains a `<g id="kvg:StrokeNumbers_…">` group of
  positioned `<text>` numbers, so you **inline the SVG verbatim** and get a numbered, vector, any-DPI
  diagram with **zero generation**. Only work: recolour the stroke paths (`stroke:#000000` → ink), size
  to fit the card. Internal IDs are per-glyph hex (`kvg:0540c…`), so inlining 100+ causes no ID clashes.
- **Stroke count** = `len(re.findall(r"<path[^>]*kvg:type", svg))` (one path per stroke) — derive it.
- **Radical** = the `<g>` carrying a `kvg:radical` attribute → its `kvg:element` (prefer `tradit` →
  `general` → `nelson`). E.g. 同 → 口.

So the v0 "build a numbered-diagram helper" task evaporates — KanjiVG is pre-numbered. Spot-check one
**high-stroke** page (11–13 strokes) to confirm legibility at print size. (No font-glyph or raster
"stroke order" hacks — they look poor at print size.)

---

## 4. World taxonomy (kanji)

~N kanji ÷ ~12–18 worlds. Two viable groupings (pick one, keep it uniform):
- **By radical / component family** (teaches the system: water 氵 world, hand 扌 world, person 亻 world…).
- **By theme / frequency tier** (Numbers & Time, Nature, People & Body, School, …), which aligns with
  the vocab worlds and lets a series share world names.
Same tuple structure as grammar/vocab; the world name must fit **all** its kanji. Stamp `categoryOrder`
from the taxonomy. Per-level **kanji list** comes from the official-ish source (Tanos/JLPT lists);
de-dupe against lower levels (a kanji is taught once, at its lowest level).

---

## 5. Kanji exercises

Per world, after its kanji cards:
- **Reading** — given the kanji/compound, write the **kana reading** (the core skill).
- **Writing** — given the meaning + reading (or an English keyword), write the **kanji**; a guide square
  is provided.
- **Compound building / matching** — match a kanji to a compound it forms, or assemble a compound from
  two taught kanji.
- **Radical/meaning matching ("draw a line")** — reuse the shared deranged dot-matching component
  (grammar manual §5): one line per item + its dot, uniquely matchable, deranged shuffle.
- **Answer key** per world (back matter). Build banks from the world's kanji so the "answer-in-bank"
  check holds.

---

## 6. Build procedure

1. **Assemble the level's kanji list** → `_<lvl>_kanji_full.json`; de-dupe against lower levels.
2. **Populate each record:** meaning keyword(s), curated on/kun readings, stroke count, radical +
   components, 2–3 at-level example compounds (with readings + EN), and the stroke-order asset path.
   Hand-verify readings, stroke counts, and that example compounds are at/under level.
3. **Wire stroke-order SVGs** (§3) + the writing grid (§2).
4. **Define the taxonomy** (§4).
5. **Build the HTML** (a `_<lvl>_kanji_build.py`, modelled on `_wbv_workbook.py`); confirm kanji count =
   list size.
6. **Render** the 6×9 600-DPI PDF (common §4) — verify stroke diagrams stay vector/crisp.
7. **Verify** (common §9): montage **all** card pages (stroke diagram present + correct count, grid not
   clipped) and **all** exercises; spot-check the densest pages.
8. **Review loop** (§7) to zero; **stamp + keep-latest** (common §11).

---

## 7. Review loop & recurring kanji defect classes

Keep an `<Level> kanji Book review Prompt.txt`. Pass order: programmatic scans (stroke-count vs diagram;
on=katakana / kun=hiragana; example readings present; level-scope of example compounds) → content read
of every kanji (readings, meaning, radical, examples) → layout montage of all card + exercise pages.

**Recurring kanji defect classes (re-check the whole book each pass):**
1. **Wrong/extra/missing reading**; on'yomi written in hiragana or kun'yomi in katakana; okurigana split
   wrong (も.つ).
2. **Stroke count ≠ diagram**, or wrong **stroke order** in the asset.
3. **Wrong radical / component** breakdown.
4. **Circular example** — the "example word" is just the bare kanji (a kun-fallback where the kun has no
   okurigana, e.g. 京→京, 心→心) — or a **missing example**; also a compound above level, wrong reading,
   or one that doesn't use the kanji. *(Build #1's dominant finding: 30 circular + 8 missing; the review
   scan flags `example.word == char`, fixed with hand-authored compounds — language-review those.)*
5. **Writing grid** overflow / guide lines too heavy / traced glyph misaligned in the square.
6. **Stroke-order numbering** illegible or overlapping the kanji at print size.
7. **Matching** not uniquely solvable.
8. **Licence/attribution** for stroke-order data missing on the copyright page.
9. **Extraction artifacts mistaken for defects** (common §8).
10. **Review/exercise page omits some of the world's kanji** (tests a subset, not all) or is left badly
    **underfilled** with a large void; **reference index clipped** (overflows one page — must paginate);
    **trace glyphs / grid lines too faint for print**; **emoji used where the series uses the Bunpo-chan
    mascot** (breaks uniformity). *(All four surfaced in the N4 v1 PDF audit — see §8.)*
11. Process: fixed one kanji but not its siblings; declared done with a check failing.

**Recurring PRACTICE-QUESTION defect classes (Build #4 exam layer - re-check every review + the mock):**
12. **Muddy distractor** - a 用法/文脈 wrong option a fluent speaker could accept as valid (時間を使う, 川が強い =
    "strong current", 水/風が少ない). Apply the "could a fluent speaker accept this?" test to EVERY wrong option;
    yes = no single answer = defect. (The hardest class to catch on a casual read.) **Sharp sub-trap for
    color / concrete-attribute adjectives in 用法:** misapplying the word to an object that CAN legitimately
    have that attribute yields a *valid* sentence, so it becomes a second correct answer. "この にもつは
    赤いです" (a bag can be red) is NOT a wrong-usage of 赤い. Fix by using the word only where it is clearly
    wrong - color-vs-color ("はれた空が赤い" -> 青い; "雪で山が赤い" -> 白い), or on an object that cannot
    have the attribute at all.
13. **Mis-calibrated 用法 distractor** - either **absurd/ungrammatical** (漢字を食べる, 冬い) or accidentally
    valid (#12). Each wrong option must be a real, grammatical sentence failing for a *nameable* reason.
    Two recurring accidental-valid sub-traps (both surfaced repeatedly in the N4 build, incl. in "fixed"
    items): (a) **attribute-trap** - a color/size/quality adjective applied to an object that *legitimately
    has* that attribute reads as correct (赤い荷物 "a red bag", 広いビル "a spacious building", 明るいケーキ
    "a bright-coloured cake"); use the adjective only where it is clearly wrong (misapply where a *different*
    attribute-word belongs, e.g. 広い->大きい on an apple). (b) **correct-usage + unrelated-logic-slip** - the
    target word is used perfectly (明るくなる, 帰っていない) and the sentence only fails on real-world
    reasoning elsewhere (clouds->should be dark; "arrived home" ~ "returned home"); this tests reasoning,
    not the word - rewrite so the *word itself* is misused. (c) **directional-verb reversal** - for a
    give/receive/return verb (借りる/貸す/返す/もらう/あげる), do NOT build a lending/returning scenario and
    swap the target in: 借りる (incoming) dropped into a "give the book back / lend to a friend" frame comes
    out *backwards*, not cleanly wrong. Instead use an **acquisition/consumption** frame where a
    non-directional verb (食べる/買う/使う) is correct (e.g. 借りる vs 食べる/買う/もらう), so the target is a
    simple wrong verb, not a reversed one. (d) **motion-verb "at a place" trap** - 走る/泳ぐ/飛ぶ contrasts
    must force the medium *inside* the sentence (水の中を走る -> 泳ぐ; そらを走る -> 飛ぶ); a mere location
    (海で走る, プールで走る) is accidentally valid (beach-running, aqua-jogging).
17. **Grab-bag vs one-class distractors (用法 quality, not a defect).** The strongest 用法 items teach ONE
    confusion class across all three wrong options (味 vs におい/意味/趣味 = "words 味 is confused with"; 音 vs
    声/発音/音楽; 早い/速い; 建てる/立てる) - the counterpart words differ but the class is unified. A "grab-bag"
    (three unrelated fixes) is weaker though still valid if each option is individually excludable. Prefer a
    same-reading homophone-kanji counterpart as a distractor where one exists (建/立, 早/速, 帰/返). Targets
    that RESIST a clean one-class set: **directional verbs (貸す/借りる)** and **broad/abstract words (運ぶ/
    動く/漢字/考える)**, and **determiners / intransitive-only verbs (同じ, 帰る)** - the last go
    *ungrammatical* when misused (同じ+adjective, 帰る+object), not natural-but-wrong, so test them via
    問題1/2/3, rebuild as a clean contrast that stays grammatical (帰る: go-home vs 戻る/行く, all four keep
    the 帰る form per TC-J05), or swap the 用法 slot to a clean 自他 pair (同じ -> 立てる/立つ). Accept a
    grab-bag rather than force an unnatural set. Also watch **distractor-
    pool hygiene**: a small level pool means a few words (意見, 通る, 教える) recur as distractors many times;
    don't let one become a pattern-eliminable giveaway.
    **用法 target-selection under a strict native bar (learned N4, 2026-07-22, after a reviewer rejected an
    entire 用法 layer 3x):** to a strict native reviewer a 用法 distractor must be a *fully natural sentence*
    that is wrong only by a nameable lexical swap. In practice only two target classes reliably clear that
    bar: (i) **homophone-kanji** (建てる/立てる, 早い/速い) and (ii) **tight verb near-synonyms** whose wrong
    sentences read completely naturally (通う/通る, 開ける/つける). Everything else tends to fail: transitivity
    / 自他 pairs (集める/始める/止める/出す/入れる/動く) read as *grammatically broken*; bare adjectives and
    factual nouns (強い/元気/赤い/冬/漢字) test world-knowledge not usage; loosely-related synonyms (味/音/思う)
    yield forced sentences. **The durable fix is reallocation, not endless rewriting: a 用法 slot whose target
    kanji cannot host a clean pure-lexical item is moved to 問題1 reading** (distractors become misreadings -
    no naturalness surface, near-zero review risk), reusing the item's already-gated natural sentence and
    keeping its answer position so the key distribution is unchanged. Relax the per-chapter split to let 問題1
    absorb the freed slots (keep 問題2 / 問題4 fixed; 用法 may drop to 0-2/chapter); paginate the now-larger
    reading block and skip any type-block left empty. Reserve 用法 for the handful of genuinely-capable
    targets rather than forcing a 用法 item onto every kanji.
    **Shared-te-form homophone trap (learned N4, 2026-07-23):** even a "tight near-synonym" verb pair is
    UNSAFE for 用法 when the two members share a written form via inflection - 通う (かよう) and 通る (とおる)
    both surface as 通って, so a distractor written with 通って is genuinely valid Japanese (read as 通る),
    not a clean error. Before using a verb as a 用法 target, inflect it and its counterpart; if any surface
    form collides in kanji, drop it to 問題1 reading instead.
15b. **Reading-collision list must include specialized / literary / rare readings (learned N4, 2026-07-23).**
    The 問題1 "distractor is a real reading" check can miss low-frequency legitimate readings, so a distractor
    that looks invented is actually attested: 安心 has a Buddhist reading あんじ; 兄弟 has a formal reading
    けいてい. Any genuine reading of the same written word = two answers (TC-R02) - reject it, and never let
    such a reading appear even inside a distractor explanation. Curate an explicit per-level block-list of
    these rare readings; do not trust the automated collision check alone.
15c. **Wakachigaki must not split a compound (learned N4, 2026-07-23).** In a space-segmented (分かち書き)
    sentence a multi-kanji compound stays one token even when only part is the underlined target: 日本料理
    must render 日本料理, not 日本 料理, when 料理 is tested. Underline the sub-span inline (substring wrap),
    never with a visible space; grep rendered output for a space inside known compounds.
15d. **A guide's own worked SAMPLE must pass the current gate (learned N4, 2026-07-23).** When the QA bar is
    tightened, re-audit every illustrative example embedded in the authoring/QA docs - a stale sample (e.g. a
    用法 sample with mechanically-substituted distractors) reproduces the very defect the new rule bans.
    Treat doc samples as first-class content subject to the same acceptance gate.
15e. **Build-identity + Excel<->HTML sync tooling (learned N4, 2026-07-23) - the durable fix for the
    stale-copy-audit loop.** When the same source (the item banks) produces sibling artefacts (a review
    workbook + a rendered book), reviewers repeatedly audit a stale one. Fix it structurally: (a) compute a
    single SHA-256 over ALL item content (`_content_id.py`) and embed the SAME hash in every artefact - a
    `<meta name=content-hash>` in the HTML head plus a human-visible short 'build ...' line, and a 'Build
    identity' line in the workbook; (b) put stable machine IDs on the rendered questions (`id` + `data-bank`
    + `data-type` on each question element, `data-qid` on each answer-key entry); (c) write a cross-check
    tool (`html_qa.py`) that fails unless book-hash == workbook-hash == current-bank-hash (TC-CH06) and that
    mechanically verifies inventory (215, no missing/extra/dupe), per-item type allocation vs the banks
    (grouping follows the type field, NOT the historical M-number in the ID), verbatim sentence+option
    equality (which also catches compound-space splits), answer-key answer==key, section-heading==item-type,
    furigana-free, no answer-leak attributes/classes, and unicode fidelity (no U+FFFD / no literal `&#39;`
    entity leaking from a bank field into the answer key). Run it beside the bank gate on every rebuild. A
    hash covering only the graded fields misses explanation-only edits - hash the explanations + distractor
    reasons too so ANY content change flips it. Two follow-on lessons: (i) make the cross-check tool's
    HTML parser ATTRIBUTE-ORDER-INDEPENDENT (regex per-attribute, not one rigid opening-tag pattern) - adding
    a later attribute (e.g. data-question-id) otherwise silently breaks a fixed `id="..." data-bank="..."`
    regex; (ii) the hash only prevents stale-copy audits if the reviewer actually checks it - reviewers will
    still audit an old saved copy and even compute a raw zip-file SHA of it, so surface the content-hash
    prominently (visible line in the artefact + a Read-Me instruction to compare it BEFORE reviewing).
15f. **The 用法 acceptance criterion must not be self-contradictory (learned N4, 2026-07-23 - the root
    cause of a multi-round 用法 churn).** An early gate demanded each wrong option be BOTH "a sentence a
    fluent speaker would actually say" AND "a sentence where the target word is wrong". Those cannot both
    hold - a genuine misuse is by definition not something a competent speaker utters - so every 用法 item
    eventually failed and got reallocated. Replace it with a coherent five-part matrix per wrong option:
    (A) syntactically well-formed; (B) the SCENARIO/frame is realistic (not the whole sentence-with-wrong-
    word); (C) the tested word is unacceptable in that scenario; (D) exactly one replacement is clearly
    required; (E) no ordinary reinterpretation rescues it. Correct option = A+B yes, C no. This separates a
    *controlled misuse* (keep) from *broken grammar / absurd content* (reject: fails A/B) and *accidental
    validity* (reject: fails C/E). Under it, controlled LEXICAL misuse (different words, 開ける/つける) and
    COLLOCATIONAL misuse (建てる=build vs 立てる=stand-up) are valid 用法; only genuinely ambiguous pairs
    (早い/速い) still fail. Word the gate as "realistic FRAME + wrong word", never "a sentence a speaker
    would say".
15g. **Kana-first answer keys, mechanically enforced (learned N4, 2026-07-23).** Answer-key explanations
    drift into romaji readings (自分 = jibun, 銀色 = gin'iro) over many authoring passes, which a Japanese-
    first reviewer rejects. Enforce one format - "[form] = [kana reading], 'concise English meaning'; why it
    fails" - and MECHANIZE it: a Hepburn->hiragana converter that returns None on a non-parse cleanly
    separates romaji readings (kibun, sakanaya - they parse) from English glosses (mood, half - they do not),
    so a QA check can assert "romanised-Japanese-reading count = 0" without an English allow-list. Convert in
    detect mode first and eyeball every token (the converter re-scripts existing readings, so it never
    fabricates - but verify long vowels / geminates / ん). Keep the same explanation format across all items
    (TC-D03/D05): when a rule is superseded, delete or mark the old wording so guide + checklist + QA sheets
    describe ONE acceptance model.
15h. **A narrow QA detector is a FALSE PASS that hides a real defect for rounds (learned N4, 2026-07-23).**
    The first romaji cleanup and its AK02 check matched only `KANJI(romaji)` (half-width parens) and missed
    the dominant distractor-reason form `KANJI = romaji 'gloss'`, so the tool reported "0 romaji" while ~146
    remained and the reviewer kept (correctly) flagging them across three rounds. Two rules: (1) when a
    reviewer keeps flagging something the tool calls clean, SUSPECT THE TOOL - broaden the pattern and test
    it against the reviewer's ACTUAL cited examples before trusting the pass; (2) distinguish romaji from
    English by an authoritative READING MATCH (the token's r2k-kana is a real JMdict reading of the
    preceding word), never by parse-alone - "run/fun/tea/same" all parse as kana but are English, and only
    the reading-match rejects them without a hand-maintained allow-list. (3) UNIT-TEST the converter/
    detector on edge cases before trusting it: a romaji->kana converter that mishandled Hepburn `n'`
    (turning gin'iro into ぎんんいろ, double ん) silently failed on exactly the apostrophe subset while
    passing everything else - so gin'iro/kin'iro slipped a second round. Cover n' (=one ん), geminate っ
    (kk/tt/ss), and long vowels (ou/uu). And when cross-checking a reviewer's flagged list, DUMP the raw
    current cells - it separates genuinely-remaining defects from stale quotes (the reviewer's "~80 rows
    still romaji" was stale; only the 2 apostrophe cells were real). The same romaji detector was too
    narrow THREE times running (parens-only -> missed `KANJI = romaji`; then -> missed the `n'` apostrophe;
    then -> missed SPACE-delimited `KANJI romaji`), each a fresh silent false-pass. A format detector must
    enumerate EVERY delimiter the data actually uses (`(` `（` `=` and whitespace) - grep the raw field for
    the token in all forms, not one.
15i. **One 問題-type per rendered card (learned N4, 2026-07-23).** An earlier page-packing optimization put
    the lone 問題4 (言い換え) item and the lone 問題5 (用法) item on the SAME card to avoid a near-empty
    page; that produced a card carrying two 問題-types (and a reviewer reading it as "用法 under a 問題4
    heading"). Give each 問題 block its own card, and enforce it mechanically: every practice `.pcard` must
    contain exactly ONE `data-type` and its もんだい heading must match that type. A near-empty single-type
    page is fine; a mixed-type card is a JLPT-format defect.
15j. **Mechanize EVERY mechanizable checklist item; a documented-but-unrun check is not coverage
    (learned N4, 2026-07-23).** A reviewer catching mechanical defects round after round means the checks
    exist only as prose. Wire every structural / cross-artifact-sync / format / orthography-script /
    print-integrity case to a script (bank gate + item checker + an HTML/Excel-sync checker), and keep an
    explicit COVERAGE MAP (a sheet) that ties each test case to the script that enforces it and honestly
    lists the ones that remain MANUAL - the genuinely semantic (naturalness, single-answer, register,
    usage validity) and rendered-visual (overflow, pagination, browser) judgments a machine cannot make.
    Run the whole suite before declaring done, so a reviewer only ever finds semantic/visual judgments,
    never a mechanical defect the tooling should have caught. Examples now auto-enforced that had been
    "visual only": schema well-formedness, option-script per type, no-English-in-stems, footer sequence,
    keys-only-at-back, no branding, text-only, lang marking, timing consistency, retired-decision drift.
15k. **A 表記 sentence must force the target's MEANING, not merely contain the word (learned N4,
    2026-07-23).** For a kana target with homophone-adjacent kanji spellings (はやい -> 早い early / 速い
    fast), the stem context must make exactly one spelling correct. 'けさは でんしゃが はやいです' fails -
    a train can be 早い (early) OR 速い (fast), so both spellings are defensible. Anchor the meaning with a
    disambiguating word: 'はやい 時間に' (an early TIME) forces 早い. Check every 表記 item whose target has
    a same-reading alternative kanji (早い/速い, 開く/空く, 明ける/開ける, 熱い/暑い) that the sentence rules
    the alternative out.
15l. **The review workbook is itself an audited artifact - keep it internally consistent and honestly
    scoped (learned N4, 2026-07-23).** A reviewer audits the workbook's own prose the way they audit items.
    Three recurring self-inflicted flags: (a) **One publication rule, stated once.** Do not let the Read Me
    say 'publication = recheck-passed' while the Authoring Guide says 'may ship automated-check-passed with a
    caveat'; pick the strict rule (publication-ready = recheck-passed ONLY; automated-check-passed /
    language-review-pending items circulate only as draft/review material) and repeat it verbatim in every
    sheet that mentions it. (b) **A print master's rendered-page checks are REQUIRED, not out-of-scope.** Only
    browser-app behaviour (navigation, responsive/mobile UI, keyboard nav, question<->answer hyperlinks) is out
    of scope; PDF rendering at trim size, overflow/clipping, page breaks, footer collisions, font fallback,
    underline integrity and JP glyph forms are the print master's core QA and must be listed as a required
    manual/tooling gate. (c) **No workflow column may be 'permanently empty'.** Auto-populate a `revision_state`
    column = 'automated-check-passed' on every row (the honest current state after the mechanical gate), and
    document that the human-judgment columns (review_result / confidence) being blank means 'not yet reviewed',
    not 'passed'. Also keep the sheet-count sentence in sync (adding an 'Automated Coverage' sheet made 'Five
    sheets' wrong). **MECHANIZED (do not rely on a human noticing):** (a) and (b) are now regression-tested -
    `html_qa.py` CH10 asserts the canonical publication-rule phrase appears in BOTH Read Me and Authoring Guide
    and the banned 'may ship automated-check-passed' framing appears in neither; CH11 asserts the Automated
    Coverage sheet contains 'REQUIRED for the print master' + every render-dimension keyword (overflow,
    clipping, page breaks, footer, font fallback, underline, glyph forms) and does NOT contain the lumping
    phrase 'out of scope for the print'. Both are wired to Test-Checklist TC-CH10 / TC-CH11 and listed in the
    Automated Coverage enforced map. When you introduce a workbook-prose invariant, add the matching html_qa
    check the same session - a self-inflicted prose inconsistency is exactly as mechanizable as a data one,
    and a negative test (reintroduce the bad string, confirm the check FAILS) proves the guard has teeth.
15m. **QA scripts shipped as a portable package must be portable AND fail loudly (learned N4,
    2026-07-23).** Reviewer execution of the embedded QA package surfaced defects that never bite on the
    author's machine: (a) **No hard-coded absolute paths** - derive the root from the script's own location
    (`BASE = os.path.abspath(os.path.join(os.path.dirname(os.path.abspath(__file__)), ".."))`) and build data
    paths with `os.path.join`, never `BASE + r"\data\..."`. (b) **Preflight every hard external dependency**
    (JMdict cache) with an existence check that prints `BLOCKED: missing ...` and `sys.exit(2)` - never a raw
    traceback. Standardise return codes: 0=PASS, 1=FAIL, 2=BLOCKED/missing-dependency, 3=config/exec error.
    (c) **Prefer explicit CLI args over newest-glob** (`--workbook`/`--html`): "newest match" silently picks a
    stale file, and a too-narrow glob won't match the delivered package name; make the fallback raise a clear
    `FileNotFoundError`, not `IndexError`. (d) **Scope a content check to the region it claims** - a
    whole-document `"<img" not in H` "text-only" test is a false pass because the book legitimately carries
    `<image>`/`data:image` mascot + stroke-order assets on lesson/cover pages; scope the practice-text-only
    check to the `.q` practice cards (balanced-div extractor) and scan every image form (`<img`, `<image`,
    `data:image`, `background-image:url`). (e) **State reproduction honestly** - an automated layer that passed
    in the author's environment is NOT reproduced from the package until the package's own execution log shows
    real return codes; the coverage sheet must say so. And a rule must match its documented TC: the mock-retest
    check compares target_word + option SET (not target_word alone - a cumulative mock legitimately re-tests a
    word with fresh options).
15n. **Equality across sources is NOT correctness - add integrity + item-fit checks (learned N4,
    2026-07-24).** The Excel<->HTML<->bank sync checks (CH06 hash, EH02 field-equality, EH09 explanation-equality)
    prove the three copies are IDENTICAL; they do NOT prove the content is right. Two defect classes slipped
    through byte-identical: (a) **encoding integrity** - a field mis-serialised as visible bytes (e.g. a
    reviewer's tool showing `\xe3\x81\x84` for `い`) passes every equality check; add a detector rejecting
    literal `\xNN` / `\uXXXX` escape sequences and the U+FFFD replacement char across all fields + the rendered
    HTML (html_qa MOJI). (b) **explanation fits its own item** - a 問題1 answer-key explanation copy-pasted from
    a different word passes EH09 because it is identical in all sources; add a check that each reading-item
    explanation contains its OWN reading or every target kanji (html_qa EXPL-TGT). LESSON WITHIN THE LESSON:
    when a reviewer reports N instances (here 4: MOCK-M1-Q02..Q05), MECHANIZE the check and run it over the
    whole corpus rather than fixing only the N - the EXPL-TGT check immediately surfaced 2 MORE the reviewer
    missed (MOCK-M1-Q08 特急 explained as 発音, Q09 親切 as 集中). A human report is a sample, not the full set.
14. **Non-original content** - mock item copied/near-copied from a chapter, or れい copied from the reference
    paper. Enforce with a **0-cross-copy scan** (mock vs chapters) + a cross-bank duplicate-sentence scan.
15. **Variety** - two items share an identical 4-option set (the 3-color 表記 clone), or two chapters share
    an answer sequence, or heavy scenario repetition.
16. **Register mismatch for the age band** - scope-forced adult/literary words (借金, 肉体, 学業); and any
    **author-introduced reading not verified** against JMdict/KANJIDIC2 (anti-fabrication).

**Added after the N4 build's four-round external PDF audit (2026-09-06). These six survived every
automated gate the project had, across many builds, and were only found by reading rendered pages.**

17. **Kun printed as a BARE STEM.** The card shows `か` where the reading is `か(う)`, `の` for `の(む)`,
    `ふる` for `ふる(い)`. The card then contradicts the example directly beneath it (言 showed `い` above
    `言う いう`). N4 build #1 shipped **9 such cards**, all in the late-added batch, while 116 of its 203
    kun readings used correct `よ(む)` notation. **Detection:** a kun with no parentheses is a defect when
    `glyph + tail` is a JMdict entry read `kun + tail` for some inflectional tail, AND the bare kun is not
    itself a standalone JMdict word. That second clause is essential: 心 こころ, 元 もと, 歌 うた, 黒 くろ,
    夏 なつ, 兄 あに are all real nouns and must NOT be flagged. Judgement remains for readings attested
    only as a name or prefix (安 やす, 古 ふる): treat them as stems, because the card teaches 安い / 古い.
18. **An example's okurigana matches no listed kun.** 楽 listed only `たの(しい)` but printed the example
    楽しむ たのしむ. 楽しい is `adj-i`, 楽しむ is `v5m,vt`: separate words, and KANJIDIC2 lists both
    `たの.しい` and `たの.しむ`. **This is the gap class 17's check does NOT cover** - a parenthesised kun
    is skipped by that scan, and an attribution check that accepts a substring passes `たの` inside
    `たのしむ`. Needs its own check: for each example, does its okurigana match SOME listed kun form.
19. **The same written form carrying two readings on two cards.** 身体 appeared as からだ on the 体 card
    and しんたい on the 身 card, 151 cards apart, with no cross-reference. Both are attested, so no
    dictionary check fires. Worse, the からだ example was the 体 card's ONLY demonstration of its own kun,
    and it used 身, untaught until card #169. Fix by using the JMdict lead form for the reading (体 for
    からだ) so each written form carries one reading per book.
20. **A non-Joyo or untaught kanji in an example where a taught one exists.** 仔犬 for 子犬: both are
    JMdict surface forms and neither is tagged rare, so a tag-based scan passes it. The decisive test is
    the TAUGHT SET, not the dictionary: 子 is an N5 prerequisite, 仔 is in neither the level's set nor N5
    and is not Joyo. Same class in reverse: a taught kanji spelled in kana on a CARD (力もち, 買いもの,
    飲みもの, 言いかた) when 持/物/方 were taught 70-150 cards earlier. Note the asymmetry - the
    cumulative-scope rule that forces kana in EXERCISE items does not apply to lesson cards, so
    `飲みもの` is correct in a World 2 item and wrong on the World 14 飲 card.
21. **A card meaning that licenses the wrong homophone kanji.** 早 glossed `early; fast` with the example
    早い glossed `fast; quick`. JMdict's own entry flags the split: sense 1 "fast; quick; rapid" carries
    the note **"esp. 速い"**, sense 2 is "early; soon". Unqualified "fast" on the 早 card teaches 早い for
    physical speed, which is 速い's job, and it contradicted the book's own items, which used はやい only
    for time. Beware the over-correction: retitling the card "early" alone strands 早口 "fast-talking" and
    早々 on the same card. Fix the EXAMPLE gloss to the kanji-specific sense and keep the card heading
    covering both, e.g. "early; ahead of time". Audit the whole homophone family this way: 早い/速い,
    開く/空く, 立てる/建てる, 会う/合う, 帰る/返る, 直す/治す.
22. **Readings live in THREE files, not two.** The N4 cross-file gate compares `data/kanji.json`,
    `data/<level>_kanji_readings.json` AND the card-prompts `.xlsx`. A kun edit failed the gate twice
    before all three agreed. The trap: **the xlsx covered only the ORIGINAL 143 kanji**, so an earlier
    9-card batch (all at lesson_order 146+) never tripped it and gave false confidence that two files
    were enough. Any reading edit at or below the xlsx's row count must go through the Office
    snapshot/check protocol on that workbook too.
    **The three need not share a folder** (2026-09-20): after the N4 consolidation the two JSONs stay
    under `<level>/data/` because the live site reads them, while the card-prompts xlsx sits in the
    workbook folder. Resolve each by its own path and never infer the other two from one. Examples
    are NOT carried in the readings ref or the xlsx, so an example-only edit touches `kanji.json`
    alone; a READING edit touches all three.
23. **After moving anything, re-run the gates and compare FINDINGS, not the exit code.** A gate that
    auto-downloads a missing reference cache will re-fetch a *newer* dictionary and still print its
    usual pass, leaving sibling gates judging against different snapshots. See
    `00-common-workbook-pipeline.md` §16 for the full relocation checklist.

---

## 8. Build-#1 decisions [v1-confirmed]

The v0 open questions, resolved by *My N4 Kanji Adventure*:
- **Cards-per-page: 6 (2×3).** Stroke diagram sits **inline on the card** (top-right); the writing grid
  fills the card body (§1–2).
- **Taxonomy axis: learning-order "journey" stages** for v1 — chunk `lesson_order`/`frequency_rank` into
  ~12 named worlds. Radical-family / semantic-theme grouping is a richer v2 axis but needs more
  authoring — deferred (§4).
- **Stroke-order source: KanjiVG** from the app repo; diagrams are pre-numbered (§3); credit CC-BY-SA.
- **Readings: the curated app `kanji.json` set** (on=katakana, kun=hiragana) — not a comprehensive dump.
- **Cross-link kanji ↔ vocabulary: YES** — examples mined from the reviewed vocab (~94% at N4); the gap
  filled with hand-authored compounds (language-review them) (§5).
- **Data pipeline:** `_kanji_assemble.py` merges app `kanji.json` + KanjiVG + vocab → `_kanji_full.json`;
  a `_kanji_review_scan.py` (objective flags) + `_kanji_review_export.py` (DOCX/CSV/MD for reviewers)
  complete the loop. Render with **absolute paths** (relative paths break `_print2x`'s file URI).

**Layout & print lessons (N4 v1 PDF-audit round):**
- A per-world **review page must test EVERY kanji in that world** (not the first N) and **fill the page**
  — expand the panel to `flex:1` and space the rows evenly; don't leave a void.
- **Paginate the back-of-book index** — 143 entries clip a single 3-column page (silent `overflow:hidden`
  loss); split into 2 pages.
- **Fill an underused final card page** (e.g. the 11-kanji Summit's 5-card page) with a completion
  callout in the empty grid slot rather than leaving a hole.
- **Print darkness:** trace glyphs + writing-grid lines that look right on screen are **too faint on
  paper** — darken (trace ~`.28` alpha, grid ~`.34`) and verify on a rendered raster, not the HTML.

**Design uniformity (do it from build #1, not as a retrofit):** every type-edition MUST share the series
identity — **port `SVG_DEFS` (the Bunpo-chan base64 mascot from `mascot_assets/`) + washi tape + the
"notebook" cover + the opener mascot speech-bubble callout** from the grammar/vocab master. **Never ship
emoji placeholders** for the mascot: a v1 that renders but uses emoji is *not* series-uniform — the N4
kanji v1 needed a full mascot/cover/opener retrofit, avoidable by porting the shared layer upfront.

## 9. Kanji-specific automated gates (added 2026-07-26)

Beyond the practice-item gates, the lesson layer got its own automated checks (all offline, wired into
the build's execution log). Reuse these for the next level:

- **`html_kanji_qa.py` (lesson-page fidelity, the EH02 equivalent for lesson cards).** `verify_kanji_data`
  proves the *data* (kanji.json) is right and `html_qa` proves the *practice* renders match the banks —
  neither checks that the LESSON pages faithfully render the verified data. This parses every lesson card
  from the compiled HTML and asserts glyph / on'yomi / kun'yomi / meaning (uses `display_meaning` when
  present) / every example (form+reading+gloss) / memory-note / stroke-count **==** kanji.json, plus
  completeness (all N cards, one per glyph, in `lesson_order`), the ribbon "Kanji i / N" sequence, and the
  reading-script rule (on=katakana, kun=hiragana). A `build_kanji_book` render regression that the data
  checks can't see is caught here.
- **`lang_morph_qa.py` (deterministic MeCab checks).** LM-TEMPORAL (a future adverb + past predicate =
  contradiction, but exclude adnominal `来年の…`), LM-OKURIGANA (written okurigana tail == reading tail),
  LM-READSPACE (readings have no internal space, so kanji↔reading never splits in print). **Do NOT ship a
  fugashi surface-POS parallelism or a bare JMdict-rarity check** — both are ~100 % false-positive on N4
  kana/compositional words (surface POS mis-tags kana adjectives/homographs; rarity flags natural
  compositional forms 何曜日 / 妹さん / 安く that aren't single JMdict entries). Reliable versions need
  dictionary POS + morphological normalisation; until then leave them HYBRID/manual.
- **`render_print_qa.py` PDF-TOFU (wrong-glyph / Chinese-font fallback).** Every CJK span must render in an
  embedded JP font (accept Type3 = Chrome's embedded subset); a CJK char in a Chinese system font (SimSun)
  = wrong glyph. **This caught a real defect:** a stray CJK-extension **亲** in 親's memory note (describing
  its phonetic component) fell back to SimSun. **Lesson: never put a bare non-Jōyō CJK-extension character
  (亲, 昜, etc.) in note/mnemonic text** — it isn't in the JP production font and falls back to a Chinese
  face. Describe the component in words or with in-font parts (「left side = 立 over 木」), not the raw glyph.
- **Jōyō → "Joyo" (ASCII).** Chrome print-to-pdf corrupts precomposed macron vowels in subsetted fonts
  (`Jōyō` → extracted `Joōyoō`); glyphs print fine but search/copy break. Romanise with ASCII for the
  searchable layer (see common manual §13).

**Stroke-order teaching — the progressive build-up strip (build #2):** a single small numbered diagram is
NOT enough (hard to follow on complex kanji). Teach 書き順 with a **progressive build-up strip** — one
mini-glyph per stroke showing strokes 1..k cumulatively, the **new stroke k in an accent colour** (coral,
bolder width) and earlier strokes grey — derived straight from KanjiVG's ordered stroke `<path>`s
(`re.findall(r'<path[^>]*kvg:type="[^"]*"[^>]*\sd="([^"]+)"', svg)`). Use **≥26 px** boxes + a bold
new-stroke so even an **18-stroke** kanji reads (it wraps to ~3 rows — verify the worst case fits, no
clip). The strip needs room: drop to **4 cards/page** (the 6-up grid is too tight). It replaces the
corner diagram.

**Reading-token wrapping:** wrap each on/kun reading token (e.g. `あ(ける)`) in `white-space:nowrap` so a
kana stem and its parenthesised okurigana never split when a multi-reading line wraps — the line then
breaks only between tokens (at the `,` / `/` separators). *(A space the reviewer sees as `あ (ける)` may
instead be a PDF-extraction artifact — confirm visually before "fixing".)*

**Build #3 — one-kanji-per-page + API-sourced enrichment (the richest layout):** the natural endpoint of
the density progression **6/page → 4/page → 1/page**. Give each kanji a **full page** when the goal is to
*teach*, not just list: header (book # + Jōyō number) · hero (large glyph + readings + keyword +
strokes/radical) · the full stroke-order build-up strip (≥38 px boxes, unclipped even at 18 strokes,
wraps to 2 rows) · **4 compound words** (2×2) · a generous **3-row trace+write grid** (`flex:1` +
`space-evenly` so it fills) · a **mascot on every page** (rotate expression + a writing tip by index). The
front/back matter also gets a mascot (toc, index, review, completion). N4: 143 kanji → **175 pages** incl.
a completion page + a 3-page index. Pick density per goal: **6/page** compact reference · **4/page**
build-up strip fits · **1/page** full practice/teaching page.

**Sourcing data the local corpus can't supply — fetch from free public kanji APIs, don't mass-author.**
When a spec field isn't in the app data (the N4 set had **no Jōyō number**, and only **15/143 kanji had 4
at-level compounds** in the reviewed vocab — 84 had ≤1), prefer fetching **authoritative** values over
authoring hundreds of unreviewed entries:
- **Compounds → `https://jisho.org/api/v1/search/words?keyword=<kanji>`** (JMdict). Sort `is_common` first,
  keep multi-char words containing the kanji, and **merge after the existing reviewed example** (curated one
  first, fill to 4). ~143 calls @ ~0.45 s, with retry/back-off. One bad pick still slipped through (者 →
  者ども "you", archaic) — **spot-check the fetched words for register/level**; replace archaic/rough/over-
  level picks by hand.
- **Jōyō serial → kanjiapi.dev.** Its `/v1/kanji/joyo` list is **Unicode-codepoint-ordered, NOT the official
  order** — don't use that index as a serial. Build the order by concatenating the **grade lists**
  `/v1/kanji/grade-1..6` then `grade-8` (the official 学年 grouping), take the 1-based position, and also
  store the grade. Verify every kanji resolves (143/143).
- Probe both APIs on one kanji before the full run; keep the console ASCII-only (cp932) — write Japanese to
  the UTF-8 JSON, print codepoints/counts.

**Two layout pitfalls from build #3:**
- **Macron in a display font.** "Jōyō" rendered as "Joȳo" in Baloo 2 (no precomposed ō at that weight) yet
  was fine in the body font — a per-font glyph-coverage issue. Either set the chip in a Latin-Extended-A
  font or just use ASCII **"Joyo"** (how most users type it). Check macrons on the actual chip, not only the
  paragraph.
- **A fixed element can tip a near-full flow page into overflow.** Adding the per-page mascot strip to the
  **index** pushed the already-tight 2-page (72/page) index past the page bottom. Fix by **re-paginating**
  (→ 3 pages, ~48/page), not by shrinking rows — keep reference pages readable. (Generalises §10's "paginate
  the index": re-check pagination whenever you add a fixed-height element to a `flex:1` flow page.)

**Build #4 — JLPT-style 文字・語彙 practice layer + interleaved full book (N4, 2026-07).** A richer
exercise track than §5's reading/writing/matching: authentic **文字・語彙 (Moji-Goi) exam questions** with
a per-chapter *review* and a cumulative *mock*, then merged into one book with lessons.

- **The five question types (verbatim 問題 instruction wording taken from a real level paper):** 問題1
  **漢字読み** (kanji→kana reading), 問題2 **表記** (kana→kanji spelling), 問題3 **文脈規定** (context
  gap-fill), 問題4 **言い換え類義** (sentence paraphrase / near-synonym), 問題5 **用法** (correct-usage
  sentence). **Per-chapter review = 15 items (5/4/3/1/2)**; **cumulative mock = 35 (9/6/10/5/5)**, framed as
  ~30 min "extended practice" (say so explicitly — it is NOT the official 25-min length).
- **Cumulative-scope rule (the #1 correctness risk).** A World-N review may use ONLY kanji from Worlds
  1..N **∪ the level's prerequisite (lower-level) kanji set** (for N4, the 106 N5 kanji, carried in the
  readings JSON with a `tier` field). Later-world kanji slip in constantly while authoring at speed — an
  automated **OUT-OF-SCOPE gate is the non-negotiable safety net** (it scans every displayed sentence +
  option; distractor-explanation prose is exempt). Anything out of pool → kana.
- **表記 real-word gate (dictionary-enforced).** EVERY option of EVERY 表記 item must be a real dictionary
  headword — gate all options against the **JMdict `keb` set** (`_refcache/_jmdict_e.gz`, ~228k), don't
  eyeball it. Best distractors are real words that **share a component** with the target (自分/気分/半分/
  十分; 問題/話題/主題/題名) for genuine confusion. **Verb targets must be dictionary form** (JMdict has
  知る, not 知って — te-forms fail the gate).
- **用法 distractor design (the subtle trap).** Each wrong option must be a **real, grammatical sentence
  that fails for a nameable, learnable reason** — a word the learner could plausibly confuse. Two failure
  modes to avoid: (a) **absurd/ungrammatical** options (漢字を食べる, 冬い) — worthless; (b) **accidentally
  valid** options (時間を使う, 川が強い as "strong current") — muddy, no single answer. Concrete-noun kanji
  are where authors default to absurd; instead misuse the noun where a *near-synonym* belongs (味↔におい/
  意味/趣味; 音↔声/発音/音楽).
- **Answer distribution + originality.** Balance the key (per 15-item review 3/4/4/4, **no run of 3**) and
  give **each chapter a unique answer sequence**. The れい (worked example) and the **cumulative mock must
  be ORIGINAL** — never copy the reference paper or reshuffle chapter items; run a **0-cross-copy scan**
  (mock sentences + usage options vs every chapter) and a cross-bank duplicate-sentence scan.
- **Machine gate ≠ language review.** A reusable `practice_validate.py <bank.json>` certifies **mechanics
  only**: scope, type split, answer balance, coverage, dup ids/sentences, **answer-key completeness**
  (explanation + one reason per option), opener variety per 問題 block, register watchword, and the JMdict
  real-word gate. It does **not** certify naturalness — always flag authored/reworked items
  **automated-check-passed / language-review-pending** until a language-review pass.
- **Interleaved full-book merge.** Order = for World N=1..N `[divider → lesson pages → World-N review]`,
  then the cumulative mock, then a back-of-book "Answer Keys" divider with **ALL keys collected at the very
  back** (chapters + mock), then the appendix — one combined file, whole-book sequential page numbers.
  **Implementation trick that matters:** pages number in **creation order** via a shared counter, so
  *create pages in final document order* and have the review renderer return its **answer-key renderer
  UN-CALLED** (a closure), invoking them all last so the keys page-number at the back. Reuse the lesson
  builder's page renderers/CSS/mascot defs (import it), and **guard any importable builder's top-level
  output emission under `if __name__ == "__main__"`** or importing it fires a stray build + resets the
  page counter.

**Full-book visual-QA reviewer pass — layout defect classes (N4, 2026-07).** A human reviewer looking at
rendered PNGs/contact sheets (not extracted text) caught page-composition defects the mechanical gates
missed. Bake these into the builder and the automated package for the next level:

- **Sparse continuation pages: never `space-between` a 1-2-item card.** `justify-content:space-between`
  fills a *full* question/answer page nicely but on a **continuation page** with one leftover item it
  strands that item at the page bottom (huge middle void, looks unfinished). Fix: auto-detect item count
  and switch sparse cards (`<3` scored items or answer entries) to **`flex-start` + a top margin** so the
  content sits in reading flow just under the heading, void at the bottom. Keep `space-between` for full
  pages only. (The PDF-BALANCE gate does NOT catch this — it excludes low-inked pages, so a stranded lone
  item reads as "acceptably sparse". This is a genuinely visual defect; a reviewer eye is still required.)
- **Balanced pagination: never end a section on a lone-entry page.** A fixed `items[i:i+N]` chunker leaves
  a `[7,1]` tail. Rebalance the **last two** pages when the final chunk is tiny (`[7,1] → [4,4]`); uniform
  entry heights make an even split fit. Applied to the answer-key: every world went `[7,7,1] → [7,4,4]`,
  0 lone pages (was 14).
- **Line-break protection on atomic request verbs.** A wrapping instruction split `えらんでください` as
  `えら / んでください` (visible in the render). Wrap the polite-request verb+auxiliary units
  (`えらんでください`, `かいてください`, …) and short bind phrases (`いちばん いい ものを`) in
  `white-space:nowrap` spans. Don't nowrap the whole paragraph (overflow). Same class as the reading-token
  wrapping above — Japanese lexical units must not split mid-word.
- **Classify by an authoritative attribute, never by shared CSS classes.** The visual-QA packager inferred
  page type from `.pcard`/`もんだい` heuristics and mislabeled practice pages as answer keys. Fix: have the
  builder stamp **`data-page-type` / `data-template`** on every `<section>`, and the classifier/manifest
  read that attribute (keep the heuristic only as a fallback). Cheap, exact, and self-documenting.
- **A QA script that splits on an exact class string breaks when you add a modifier class.** Adding
  `class="pcard sparse"` broke html_qa **H14** (it split on the literal `<div class="pcard">`), lumping
  cards → false "mixed 問題-type". Use a token/prefix-tolerant match (`<div class="pcard[^"]*">`). Grep every
  QA script for exact `class="X">` matchers before introducing a variant class.
- **Keep developer metadata off learner-facing pages; keep it in `<meta>`.** A visible "build sha256:…" line
  on the Answer-Keys divider weakened the page. Remove it from print; the machine-readable identity lives in
  `<meta name=content-hash>` + the workbook Read Me (that's what the automated CH06 pairing check reads).
  Center the divider composition + a faint watermark instead of leaving three-quarters blank.
- **Retention sweeps must cover every deliverable extension.** The archive script globbed only `*.html` /
  `*.xlsx` and left stray superseded `*.pdf` in the main folder — add every shipped extension to the
  deliverable globs.

**Sparse practice pages: flex-start was a band-aid; a greedy pagination packer is the real fix.** The
first VQA-01 attempt (top-align a lone item with `.pcard.sparse`) stopped the *stranding* but the reviewer
re-flagged the pages as still dramatically underfilled. Fixed-cap pagination (5 questions/page, then each
of 問題4/問題5 on its own page) inevitably produces near-empty pages because a review's tail sections have
1-2 items. The durable fix is a **greedy line-budget packer**: give each question type a line-weight
(`reading≈2, orthography≈2, context≈2.5, paraphrase≈4, usage≈5.5`), a section overhead (heading +
instruction + example ≈ 4.5) and a card budget (~20 "lines" for a 6×9 card); flow consecutive 問題 sections
onto the same card, starting a new page only when the next section's heading+example+first item won't fit,
splitting a long section with a （つづき） heading, and never splitting one question. This cut a typical
world from ~5 pages to 3 (each full) and dropped near-empty pages from 33 to 4 (only the genuinely
unmergeable usage-only tails). **Two prerequisites:** (1) relax the "one 問題-type per card" invariant to
"every scored item sits under its correct 問題 heading" — walk the card's headings + `data-type` in
document order and match each item to its nearest preceding heading (multiple headed sections per card is
now valid *and* more exam-authentic); (2) don't hard-assert a page count anywhere — the PDF-PAGES gate
must compare `pdf.page_count == len(HTML sections)` dynamically, so re-pagination never trips it. Verify no
overflow with the render gate after tuning the budget; if it flags, lower the budget a line or two.

**Removing a rendered feature is a cross-artifact ripple, not a one-line delete.** Dropping the "Remember
it" memory-hook section from every lesson page touched **seven** files, and missing any one fails a gate or
ships stale copy: the renderer (remove the block + now-dead vars + its CSS), the how-to page (remove the
feature's explanation row), the credits/rights page (remove it from the "original content" list), the
lesson-fidelity QA (remove the note + mnemonic-label checks and their count), the workbook test-checklist
(remove the obsolete TCs — and update the self-audit's exact-TC-count assertion to match), the QA-package
sheet map (remove the feature from a script's description), and the summary/Read-Me prose. Keep the *source
data* (the `notes` field stayed in kanji.json, just unrendered) so the decision is reversible, but sync
every *reference* to the feature in the same batch. A QA script that split HTML on an exact class string
(`<div class="pcard">`) also broke when a modifier class (`pcard sparse`) was introduced — grep every QA
splitter for exact `class="X">` matchers before adding a variant class, and prefer `class="X[^"]*"`.

**A mechanical render PASS is not a visual sign-off — split the render gate so the workbook never conflates them.** A single "R11 = PASS" execution-log row (derived from the headless-render script) reads as "the whole visual gate is cleared," when it only covers the *mechanical* layer (page count, 6×9 geometry, font embedding, glyph presence). The honest structure is a **split gate**: mechanical sub-gates (R11A page-count+geometry, R11B font-embed+glyph) carry the automated evidence (result artifact + sha + rc0) and read PASS; the human visual sub-gates (clipping/overflow, negative-space/balance, kinsoku, underline geometry, Q/A grouping, offline-font, author/footer/frame, final recheck) read **PENDING** until a person signs them off — even where an automated *proxy* exists (overflow scan, balance void-scan, kinsoku check, `html_qa` heading placement, offline fontdiff). Record the proxy result in the row's note, but do not let it flip the visual sub-gate to PASS. Mirror the same split in (a) the HTML-coverage matrix — point each learner-visible component at its specific sub-gate, and never label a pending cell with a stale "BLOCKED" when the mechanical layer actually passed; and (b) a **publication-gate summary** at the top of the review workbook that states each gate's status explicitly and ends with `Final publication visual sign-off: NOT CLEARED` until the manual pass is done. Two guardrails keep this honest automatically: a prose-honesty check that bans unevidenced "all pass / all green" claims in the Read Me, and an over-claim grep that bans "review complete / approved / publication-ready" — write the summary in gate-status form (`gate: PASS` / `PENDING` / `NOT CLEARED`) so it passes both. When splitting one execution-log row into many, verify the self-audit first: keep every configured `*_command` key present in at least one row's name (the config↔log coverage check matches on the key string), and confirm the PASS-row-evidence check only fires on `PASS` rows so `PENDING` sub-rows need no artifact. Org sensitivity labels are a manual application-layer step — the build cannot apply them; state that as an explicit manual action in the gate summary rather than silently omitting it.

**Unavoidable sparse pages: a compact panel + section-ending motif beats a half-empty full-height card.** After the greedy packer, a few section tails (a 1-2 item 用法/言い換え remainder, a mock tail) genuinely cannot merge onto the previous full page. Top-aligning them in a full-height card still looks like a defect. The fix a reviewer accepts: shrink the card to its content (`flex:0 0 auto`, top-aligned, text NOT enlarged, page NOT vertically centred) and fill the leftover with a restrained section-ending motif (a couple of low-opacity decorative glyphs) so the negative space reads as an intentional wrap-up. Section-start rhythm: if the card uses `justify-content:space-between` to fill, it also spreads heading/instruction/example apart; group those three into ONE flex item (a `.sechead` wrapper) so space-between distributes only between [header, q1, q2, ...] - tight header, still-filled page. And on pagination "avoidable carryover" checks: ink-fraction is NOT a proxy for "the previous page had room" (a practice page is legitimately ~35% inked while budget-full), so an ink-based test false-positives every sparse tail; the packer already guarantees no avoidable carryover by construction (breaks only when the next item does not fit), so the useful automated guard is a REGRESSION cap on the count of very-sparse practice pages, not a per-page "had room" heuristic.

**A book change voids every visual sign-off - reset the ledger and re-audit.** Because sign-offs are bound to the book sha, any book-changing fix (even a one-word callout) re-renders to a new sha and invalidates the whole manual visual gate. Batch all book-changing fixes into ONE re-render, then reset the ledger: set `book_sha256` to the new sha, empty `signoffs`, move every sub-gate to `pending`, and archive the prior build's verdicts under a `prior_builds` key (they do NOT carry). The reviewer then re-runs a TARGETED re-audit (affected templates + known-risk pages + new outliers), not the full pass. Corollary: never apply book-changing fixes piecemeal mid-audit - it thrashes the reviewer's in-progress sign-offs.

**Offline-font gate: the deliverable PDF is self-contained even when the HTML uses web fonts - verify, don't assume.** A headless-Chrome print-to-PDF embeds/subsets every font actually used at render time, including `@import`ed Google Fonts (Latin display faces often show up as nameless Type3 subsets, so a font-NAME scan can wrongly conclude "not embedded" - check for Type3 too, and confirm with an offline-vs-online pixel compare). So the shipped PDF renders identically anywhere regardless of network. The residual is only BUILD reproducibility: an offline HTML re-render falls back on the web fonts (headings differ). Fully self-hosting is impractical for the CJK body font (base64-inlining a full JP font bloats the HTML; Chrome's subset-on-render is why the PDF stays small), and self-hosting only the Latin display font doesn't make the whole build offline-reproducible - so for a print master whose deliverable IS the PDF, record the offline-font gate as PASS on the self-contained-PDF evidence + a documented build-network note, rather than chasing a marginal font-embed.

**A builder can close the visual sub-gates with an AI-assisted rendered-visual review - if it is genuine and transparently labelled.** When no independent human reviewer is available to re-sign after a fix batch, the builder may render + actually inspect the changed templates + a representative set, and record each R11 visual sub-gate as PASS with the reviewer field naming it honestly (e.g. "AI-assisted rendered-visual review") and the evidence citing both the inspected pages and the automated proxy that backs it (PDF-BALANCE, PDF-KINSOKU, PDF-PAGINATION, html_qa H14/H06, overflow, font-embedding). Keep the gate summary honest: mark it "AI-reviewed PASS - independent human final glance recommended", never a bare human "CLEARED". Two guards make this safe: unchanged templates are byte-identical to a previously human-approved build (so their prior sign-off substantively carries even though the sha-bound ledger entry does not), and the A7c self-audit check still requires the sign-off to cite the ledger + sha + a named reviewer - so the record stays evidenced and attributable.

**Design-uplift patterns for the "airy" front-matter/transitional pages.** The content pages (lessons, practice) tend to come out strong; the weakness clusters in the front-matter and interstitials (colophon, how-to, world dividers, review openers), where a short content block floats in the vertical middle with large *undirected* whitespace above it and the mascot marooned at the bottom. Fixes that lift these to the content pages' level, all inside the existing identity: (a) **turn an orientation list into numbered step-cards** - wrap each row in the same rounded panel used elsewhere, add a small numbered circle and a larger icon tile, and *add the missing step* (the practice/review section was never explained on the how-to page); four filled panels distributed with `space-between`/`space-evenly` read as a game-tutorial screen instead of labels adrift. (b) **Make a divider's ghost-kanji watermark legible** (bump opacity ~2x, enlarge) so it's a themed backdrop, not a blob, and reposition scattered washi/decoration from the top corners to *frame the hero*. (c) **Anchor a captioned value as a chip on its heading's baseline** (e.g. "N strokes" right-aligned on the "Stroke order" line via a `justify-content:space-between` wrapper) rather than floating it centered - micro-alignment is what separates "nice" from "crafted". (d) **Give an unavoidable sparse tail a deliberate section-ender** (a short rule + a single motif glyph + a short rule) so leftover space reads as a designed pause. (e) **Warm dense legal/colophon text** with +10% line-height and one darker step. **Gotcha:** "lift the centered content up so it anchors near the top" *overshoots* on a short-content page - it just moves the empty band from the top to the bottom (top-heavy). A short block is often best left near-centered with a *gentle* upward nudge (~10-15mm), not a hard top-anchor; verify by rendering, because the right amount is page-content-dependent. **Regression guard:** if a QA script greps rendered HTML for an exact element (e.g. `html_kanji_qa` matches `<div class="facts"><b>(\\d+) strokes`), keep that element's tag/class when restyling it (wrap it, don't convert `<div>`→`<span>`), or the fidelity check silently breaks.

**Make the collected answer-key section scannable, not a wall of near-identical blocks.** The back-of-book answer key is the most utilitarian surface, and it tends to render as N uniform rounded blocks stacked in an alternating stripe - a data dump whose #1 job (jump to item 33 fast) it does poorly. The upgrade, all in-identity: (a) **hang the item number as a chip in a fixed left gutter** (`.akitem{display:flex;gap}` + a fixed-width coloured number chip as the first child, the rest wrapped in an `.akbody` column) so the eye runs straight down the numbers - inline digits fused to a task label do not scan. (b) **Replace the alternating white/tint stripe with ONE uniform light fill + a thin accent rule down the left edge** (`border-left:3px solid`, `border-radius:0 9px 9px 0`) and cut internal padding ~25%; the alternating stripe reads busier than a single calm treatment. (c) **Let the kanji/target word anchor the entry** - promote it to the largest element on line 1, demote the "M1/M2" task label to a tiny pill, and push "answer N" to a right-aligned coloured chip (`margin-left:auto` inside a `flex-wrap` head). (d) **Say each orientation sentence ONCE** - the per-world first-page note that repeated "World N - Answer Key. All keys are collected here at the back..." duplicated both the section-heading directly above it and the answer-key *divider*; move the one-time "all keys gathered here" line onto the divider and leave a single slim "each entry shows the answer + why every option is right or wrong" note. **Regression guard (critical):** an answer-key entry is read back by QA regexes (`html_qa` EH03 pulls the answer number, EH09 pulls the explanation, H08 checks keys only appear at the back) that key on exact markup - EH03 matched `<b>answer (\\d+)</b>`. Restyling the answer number to `<span class="aans">answer N</span>` silently broke EH03 until its selector was updated. **A QA-checked element's markup change must update its QA regex in the same batch** - after any answer-key restyle, keep `<div class="akitem" data-qid="X">` and `<div class="akx">` intact (the id + explanation extractors depend on them) and re-point any selector you moved.

**CSS gradients RASTERISE into the print PDF; borders and SVG stay vector. Choose accordingly for fine detail.** The example-row leader dots took three attempts and the middle one was a regression, which is the lesson worth keeping. (1) `border-bottom:1.5px dotted` on a `flex:1` element: the browser distributes dots to FIT the box, so a leader shortened by a long gloss rendered a visibly tighter pitch than its two siblings on the same card. (2) `repeating-linear-gradient` with a fixed 5px period: correct in CSS, but Chrome rasterises gradient backgrounds into the PDF, and the fine period beat against the device pixel grid, producing clumpy irregular dashes that looked far worse than the original defect. (3) An SVG `<pattern>` in the shared defs, referenced by a `<rect width="100%" height="100%">`: `patternUnits="userSpaceOnUse"` with NO viewBox on the consuming `<svg>` means 1 user unit == 1 CSS px, so the period is a fixed 5px at any width, it stays vector in the PDF, and it adds nothing to the text layer. **Rules of thumb:** for a repeating fine-detail motif in print, reach for an SVG pattern, not a CSS gradient and not a `dotted`/`dashed` border whose pitch you cannot control. Avoid the "repeat a `·` character with `overflow:hidden`" trick too: it works visually but injects hundreds of dot glyphs into the PDF text layer. And judge the result at REAL print scale (~150dpi) before iterating: at 450dpi the 1.8px dots looked square, which was pure magnification artifact, and "fixing" it would only have made a subordinate connector louder.

**Not every inconsistency is worth removing. Ask whether the two instances are ever seen together.** The same `dotted` pitch problem exists on the h3 section rules, whose width follows the heading text (short "Stroke order" vs full-width "Example words"). It was deliberately left alone and recorded in the ledger under `accepted_variance`: stacked example-row leaders sit three-in-a-row and invite direct comparison, which is exactly why that mismatch jumped out, whereas headings are separated by whole sections and never compared. Converting them would mean injecting an inline `<svg>` into every heading across templates (markup churn plus fresh QA-regex coupling) to fix something imperceptible at print scale, and the CSS-only shortcut would rasterise exactly like the rejected gradient. Record the decision and the reasoning where the next person will find it, rather than silently leaving a known-odd thing unexplained.

**Before changing the ELEMENT TYPE of anything inside a QA-parsed row, re-read the regex.** `html_kanji_qa` verifies all 170 lesson pages' examples with a single pattern keyed on the `exw` / `exr` / ... / `exg` span sequence, and it only tolerates the leader element sitting between `exr` and `exg` because `_txt()` strips tags out of the captured group. Swapping `<span class="exlead">` for `<svg class="exlead">` would have turned the `</span>` the regex expects into `</svg>` and failed every lesson page. The fix was to NEST the new `<svg>` inside the existing span so the tag sequence is untouched. Generalise: adding a child is safe, changing a tag or class that a checker greps is not, and the cheap way to know which you are doing is to grep the QA scripts for the class name BEFORE the edit rather than after a red gate.

**A build-identity digest must cover EVERY learner-visible field, and its docstring is a contract to TEST, not to trust.** The content hash exists so a reviewer can prove book == workbook == banks are the same content state. Ours digested id / type / target_word / reading / sentence / options / correct / explanation / distractors, and its docstring claimed "any content edit flips the hash" - but it omitted `meaning`, which renders on every answer-key line. Correcting one item's gloss therefore changed the printed book while the hash sat unchanged, so two builds could share an identity while an answer-key line differed. **Audit the digest against what is actually PRINTED, field by field, rather than against the field list someone wrote down.** The regression test is the real deliverable: mutate ONLY the suspect field and assert the digest moves, then re-assert the properties you must not have broken (order-independence by id, sensitivity to stem changes, unchanged item count). A hash whose coverage you have not tested is worse than no hash, because it is quoted as evidence. **When the digest SHAPE changes, the hash re-bases for every artefact** (ours went fd0054134f9f -> 58a1ec5f901c at 9 -> 10 fields): do not silently overwrite the old value wherever it appears, LABEL each historical value with the build and digest shape it describes, and note explicitly that pre-change hashes are not comparable. Also treat this as a cross-artifact change: re-run the three-way pairing check (CH06) and confirm the book meta, the workbook Read Me and the live banks all agree under the new definition before believing it.

**When you script-edit a JSON content file, match its ORIGINAL serialisation or you destroy your own audit trail.** Re-writing an `indent=1` file with `json.dump(..., indent=2)` reformatted every line: the Change Guard reported **8169 CHANGED** lines for what were 4 one-line edits, and the diff became unreviewable. Read the file's existing indent first (or re-dump and compare) and write it back the same way. Then the guard shows exactly the intended lines and nothing else.

**The Change Guard's line diff is POSITIONAL, so an INSERTION makes every following line look CHANGED.** After appending a `_meta.history` entry (4 lines) and three fields to one item (3 lines), the guard reported `CHANGED 8041 | ADDED 4` and `CHANGED 782 | ADDED 3`. Nothing after the insertion point had actually changed; the indices had shifted. **`REMOVED` and `ADDED` are the trustworthy numbers; a large `CHANGED` alongside a small `ADDED` and `REMOVED 0` is the signature of a pure insertion.** When you need a definitive proof after inserting, do an **isolation check**: programmatically revert exactly the edits you claim to have made, run `change_guard check` (it must report NO CHANGES), then restore the edited file and assert the restore by sha256. That proves your edits are the only difference without needing the original text. Do NOT reach for `git diff` to settle it unless the file is actually committed up to date: in this project `N4/data/kanji.json` sat months behind HEAD (8172 insertions vs 6827 deletions), so git mixed four small edits with an entire prior expansion.

**An example-word gloss has to satisfy two independent checks at once.** `verify_kanji_data` check 4 fails on a duplicate **first word** among the glosses *within one card*, and check 7 warns when a gloss shares no vocabulary with any JMdict sense for that word. Rewriting 勉学 from "learning" therefore could not lead with "study" (勉強 on the same card already does) but still needed a word JMdict actually uses ("study; pursuit of knowledge"). "pursuit of learning; studies" satisfies both. Same trick cleared 下町: JMdict says "low-lying **part** of a **city**", so "old-town district; the traditional part of a city" grounds while staying readable. Suppressing WARN 7 is not the goal, but when two wordings are equally accurate, prefer the one that grounds.

**A gloss can be dictionary-defensible and still build the wrong mental model.** 下町 was glossed "downtown". For an English reader "downtown" means the city centre / CBD; 下町 is the older low-lying merchant-artisan quarter, the counterpart of 山の手. Judge a gloss by the picture it puts in the target reader's head, not only by whether a dictionary would allow it. Cross-check the same word's gloss in the *other* corpus too: the practice items already said "the old-town (shitamachi) district", so the card was the outlier and the book disagreed with itself.

**Writability is a distinct pedagogy check from readability.** The readability policy (every example carrying an out-of-scope kanji also carries a reading) lets a learner READ any example, and it passes. It says nothing about whether they can WRITE it. Card 可 had 可能 / 許可 / 可能性, needing 能, 許 and 性 - so **0 of 3 examples were writable** in a book whose promise is collecting and writing kanji, while 的 and 身 had 2 of 3 and 3 of 3. The fix was to swap the heaviest and most redundant example (可能性, needing two unknown kanji and largely duplicating 可能) for 不可, whose only other kanji IS taught. Worth adding as an explicit per-card metric: count examples writable from the taught set, and flag any card at zero.

**Editing an item that a reviewer already passed sends it BACK to REVISE, and the gate prose must move with it.** Correcting MOCK-M1-Q08's English gloss left the Japanese stem, options, keyed answer and explanation untouched, but the reviewer had passed the *old* gloss, so the item returns to `REVISE / recheck PENDING`; the builder does not get to re-pass what the builder edited. Crucially, the Read Me gate-summary string is HARDCODED while the R12 execution-log row is DERIVED from the sheet - change one without the other and the workbook contradicts itself, which is the exact failure the R12 derivation was introduced to prevent. Update both in the same edit, and state the residual plainly ("244 of 245 PASS; 1 awaiting recheck") rather than rounding it back up to a clean number.

**A CSS fix that changes rendered HEIGHT invalidates the pagination packer's weights - rescale them in the same batch.** The greedy line-budget packer assigns each question type a weight (`QW`) against a page `BUDGET`. Those weights encode the type's *rendered height*, so any CSS change that compacts or expands a question type silently desynchronises them. Concretely: `.qo.stack span{display:block}` was a DESCENDANT selector, so it also matched the nested `<span class="on">` option number and turned it into a block, orphaning every option number onto the line above its sentence (152 numbers across 18 pages). Scoping it to `.qo.stack > span` fixed the defect but cut paraphrase/usage questions to ~0.6x their old height (measured 54->38, 46->35, 28->20 lines on affected pages). Left at the old weights the packer kept breaking pages at the old rhythm and practice sparseness REGRESSED (`world_practice` largest-blank 0.22->0.37, top/bottom ink 3.3->11.4). **Two further traps in that same fix:** (a) CSS specificity - `.qo.inline span` (0,2,1) outranked `.qo .on` (0,2,0), so the old descendant rule had also been giving every inline option number a 22px right margin; scoping to `>` dropped it to 4px and re-flowed every inline option row in the book, a change that was never intended. Diff the *rendered* result of a selector-scoping fix, not just the target case. (b) Bias the rescaled weights **HIGH**. An initial paraphrase 4.0->2.5 / usage 5.5->3.5 packed pages to the edge and `PDF-FONTDIFF` failed with `text_eq=False`: with no slack for font-metric variation, the wider offline-fallback render pushed content past `.page{overflow:hidden}` and it was silently CLIPPED. Over-filling loses content (hard defect); under-filling is only whitespace (cosmetic). 3.5/5.0 passed with slack and still recovered a page. **Treat `PDF-FONTDIFF text_eq=False` as the over-fill canary** - it is the only gate that catches "this layout has zero margin for a Chrome or font-version shift." Note the gate is `same_pages and text_eq`; `wrap_drift_pages` is surfaced but deliberately NOT gated, so a large drift number alone is not the failure.

**One shared CSS value cannot serve two templates with different content heights - use a modifier class.** `.opener{padding-bottom:42mm}` was shared by the world dividers (tall hero: badge+title+tag+count+mascot) and the answer-key divider (short hero: title+sub+mascot). At 42mm the ak_divider was the book's worst-balanced page (top/bottom ink 11.33); dropping the shared value to 20mm fixed it (->0.94) but made the world dividers bottom-heavy (0.69->0.22). The fix is `.opener{42mm}` + `.opener.akd{20mm}`, not a compromise value. **Corollary for page-absolute decorations:** washi/star/sakura tops are positioned against the PAGE, so they only frame the hero at one padding value. If that padding changes they must move with it, or silently stop framing anything - and if the padding change is later reverted, revert the decoration tops too.

**Measure layout regressions with the project's OWN scan, not an ad-hoc metric.** An ad-hoc pixel-ink profile disagreed sharply with `full_book_layout_scan.csv` on the divider pages (ad-hoc largest-blank 0.02 vs project 0.27/0.40) because the ad-hoc threshold counted the faint `.10`-opacity watermark as ink while the project scan treats it as blank. Since the finding came from the project scan, the verification must too: snapshot `full_book_layout_scan.csv` BEFORE the batch, re-run `build_visual_qa_package.py` after, and diff per template. Also note the scan's per-template `largest_blank_ratio` is a MAX, so a single sparse tail moves it while mean fill barely budges - always list the offending pages before concluding the template regressed.

**Scan for repeated boilerplate with word n-grams, not sentence splitting.** A sentence-split repetition scan reported only 1 repeated string and MISSED a note that sat on 15 pages, because the split merged a varying token (the world number in the heading above it) into each fragment and made every instance unique. Word n-gram shingles (n=9) with page-set grouping found it immediately. When interpreting results, remember that JLPT 問題 instruction lines legitimately repeat 15-30 times and must NOT be "fixed" - repetition is only a defect when it duplicates something already said on the section divider or the heading directly above it.

**Keep every watermark class in sync, and anchor a section-ender rather than centring it.** A legibility bump applied to `.obig` (opacity .05 -> .10) left `.crwm` and `.rvwm` behind at .05, so 17 pages kept the old faint blob while 15 got the intended backdrop - and a later review that said "match the appendix watermark to the credits one" matched them at the OLD value, propagating the miss. Grep all watermark classes together. Separately, the sparse-tail section-ender (rule + motif + rule) was centred in the leftover with `flex:1;align-items:center`; once the option compaction grew a tail's leftover to ~45%, centring stranded the motif mid-void with an empty band above AND below it. Anchor it just below the card (`align-items:flex-start;padding-top:12mm`) so the slack becomes one clean bottom margin.

**Fill the appendix/credits lower half with a bottom-anchored sign-off, not empty space.** A Sources & Licenses appendix naturally top-loads (title + license bullets) and leaves the lower ~40% blank, which reads as unfinished next to every other filled page. Make the content wrapper a full-height flex column (`flex:1;display:flex;flex-direction:column`) and push a closing block to the foot with `margin-top:auto`: the same section-ender motif used on sparse practice tails (rule + gem + rule) + a warm mascot sign-off + a one-line colophon (title · edition · year · "made with care..."). This also makes a centered page-watermark read as intentional (it sits behind the now-filled column instead of floating in a void). Secondary: a long "how it was made" paragraph sitting under a header styled identically to the bulleted "credits" list above it should become a **parallel bullet list** for scannability. **Shared-class gotcha:** a page watermark class reused across two pages (credits `漢` / appendix `典`) must not have its opacity tuned on one page - editing the shared `.crwm` changes both; fill the page instead of bumping the watermark. And `z-index:-1` on a full-page watermark paints it *behind* the page background (invisible) - use watermark `z-index:0` + a `position:relative;z-index:1` content wrapper.

---

*Kanji-specific addendum to the common pipeline — extrapolated from the grammar/vocab editions and
standard kanji pedagogy. Replace the v0 assumptions with confirmed values after the first real build,
and add new kanji defect classes here as they surface.*
