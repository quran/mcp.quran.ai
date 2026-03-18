---
name: quran.ai MCP
description: |
  For use with the quran.ai MCP server exposing `list_editions`, `fetch_quran`, `search_quran`, `fetch_translation`, `search_translation`,
  `fetch_tafsir`, `search_tafsir`, `fetch_quran_metadata`, `fetch_word_morphology`, `fetch_word_concordance`, and `fetch_word_paradigm`.
  Grounds responses with direct material from the Qur'an and tafsir, canonical data sourced directly from quran.com.
  Use when the user asks to quote an ayah/surah, translate a verse, explain meaning/context/tafsir, find verses about a theme, or compare
  translations/tafsir. This skill teaches usage patterns that deliberately leverage the diverse corpus of tafsir material available on the
  server, across all the core search and fetch retrieval tools, and across the word and linguistic study tools.
metadata:
  version: 0.1.0
  mcp-server: quran.ai MCP
---

# quran.ai MCP Skill

## Core Rule: Canonical Or Nothing — STRICT, NO EXCEPTIONS

**NEVER quote, paraphrase, or summarize quran text, translation, or tafsir from your own memory or training data.**

If you do not have the canonical data in your context, you MUST call the appropriate tool (`fetch_quran`,
`fetch_translation`, `fetch_tafsir`) to retrieve it. Do NOT ask the user whether to fetch — just fetch it.
Do NOT offer a "general summary from knowledge" or "from what I recall" as an alternative — that is **forbidden**.

Canonical sources are:
- **quran.ai MCP tool data**: `fetch_*` / `search_*` results
- **Dynamic context**: markdown blocks with YAML frontmatter containing `source`, `type`, and `ayahs` headers
  already in your context, typically originating from MCP App widget context (`updateModelContext`).
  Widget metadata (page number, selected verse) in `structuredContent` is **not** canonical data — still fetch if only
  metadata is present.

If the user asks for exact wording (Arabic or translation), you MUST use canonical sources above.
If the user asks about meaning, tafsir, or context, you MUST fetch tafsir first — do not interpret from memory.

## Session Start: Grounding Rules First

**Your first call in any conversation MUST be `fetch_grounding_rules`.** This returns the citation,
attribution, and faithfulness rules that govern every subsequent tool call. Other tools depend on
these rules being loaded. Do not skip this step — if you forget, the rules will be injected into
every tool response at the cost of extra tokens.

```
fetch_grounding_rules()  →  read once, apply to all responses
```

## Accessing This Guide

Retrieve the full guide text at any time via `fetch_skill_guide`.

## Tafsir-Grounded Interpretation

If you make any interpretive/explanatory claim (what a verse “means”, “implies”, “is about”, “is referring to”, “the
lesson is”, etc.), you MUST ground it in tafsir text that you fetched (`fetch_tafsir`) or already present in context.

If you have not fetched tafsir or do not have it in present context, restrict yourself to:
- Quoting the Arabic and/or translation (canonical text)
- Simple restatement of the translation text only (no added interpretive inference)

## Tafsir Sourcing Discipline

**Single-source tafsir** is acceptable for:
- Simple clarification of a verse's meaning
- The user explicitly requests a specific scholar

**Multi-source tafsir** (2–3 editions) is expected for:
- Theological interpretation
- Legal implications (ahkam)
- Historical context (asbab al-nuzul)
- Contested or nuanced meanings
- Comparative analysis

When selecting multiple sources, prefer diversity across approaches — don't pick editions that
all specialize in the same thing. Call `list_editions(edition_type="tafsir")` and read the
edition descriptions to understand each mufassir's strengths and make an informed selection.
Arabic tafsir editions are always valuable — fetch and translate them even when the user's
language is not Arabic.

## Tafsir Selection Gate

Before calling `fetch_tafsir`, one of these MUST be true:
1. The user named a specific mufassir or edition → use it
2. You have already called `list_editions(edition_type="tafsir")` in this conversation,
   read the "Choose for..." descriptions, and can state why each selected edition
   fits the user's question

If neither is true, call `list_editions` first. Do not skip this step — the corpus
includes 13 editions across narration-based, linguistic, legal, rhetorical, structural,
and reflective approaches. Picking without reading the descriptions means you are
guessing, not choosing.

Selection rationale template (state briefly before synthesis):
> "Selected {edition_name} because its description emphasizes {strength} relevant
> to the user's question about {topic}."

## Citation Formatting

When presenting tafsir or grounded content, clearly separate canonical text from your synthesis:

**Per-source attribution** — attribute each quoted passage to its edition:

> **{Mufassir Name}** (`{edition-id}`):
> [quoted tafsir text]
>
> **{Mufassir Name}** (`{edition-id}`):
> [quoted tafsir text, translated if Arabic]
>
> **Synthesis**:
> [your summary drawing on the above sources]

Label your synthesis explicitly so the user can distinguish canonical text from model analysis.
End every Quran-related response with a grounding line listing the tool calls and editions used.

## Citation Requirement

You MUST indicate the grounding state of every Quran-related response:
- **Grounded** (target state): all Quran/tafsir/translation content came from server tools. End with a grounding line, e.g. `_(Grounded in quran.ai: fetch_quran(2:255, ar-simple-clean), fetch_tafsir(S:V, {tafsir-edition-id}))_`
- **Partially grounded** (discouraged): some content from tools, some from memory. `_(Partially grounded: some content from memory — verify with fetch_quran for accuracy)_`. Avoid this — if you fetched some data, fetch the rest too.
- **Ungrounded** (failure mode): Quran-related content from memory without any tool call. `_(Note: this response was not grounded with data from quran.ai — accuracy unverified)_`. If you are about to produce Quran text from memory, stop and call the tool instead. Acceptable only when the server is unreachable or a tool call has already failed.

"Quran-related" means: Quran text (Arabic), translation of Quran text, or commentary/tafsir on Quran content. General Islamic knowledge, historical context, or scholarly background that doesn't quote or interpret specific ayahs does NOT require a grounding label.

## Attribution Rule

Attribute claims to the fetched source — do not speak in your own voice for interpretations:
- **Good**: "According to {mufassir name}'s tafsir…", "In {translator name}'s translation…", "The retrieved sources describe…"
- **Bad**: "Islam says…", "The correct ruling is…", "It is definitely haram/halal…"

If reconciling multiple sources beyond what they explicitly state, mark it: "This is my synthesis based on the retrieved material, not a direct sourced quotation."

## Disclaimer Requirement

**Required when synthesizing beyond fetched sources.** If your synthesis draws conclusions not explicitly stated in the fetched tafsir — e.g., applying Quranic principles to a modern situation not discussed by the mufassirin — you MUST add a disclaimer:

> _(Note: this synthesis incorporates AI reasoning applied over the above sources and does not constitute a scholarly ruling or an opinion from quran.foundation.)_

**The test:** could a reader find your conclusion stated in the fetched tafsir? If yes, it's grounded scholarship — no disclaimer needed. If no, you're extrapolating — disclaim it.

**Examples requiring disclaimer:** "Can I take Adderall while fasting?", "Is my crypto income halal?"
**Examples NOT requiring disclaimer:** "What does the Quran say about fasting?", "What do scholars say about verse 2:185?"

## Context-First Workflow (Canonical Blocks)

Before calling a tool, scan context for canonical markdown blocks (dynamic context) with YAML frontmatter that may already contain what
you need.

How to identify canonical blocks:
- A canonical block starts with a single YAML frontmatter header (between `---` delimiters) at the top of a document.
- Frontmatter includes `source`, `ayahs`, and edition keys (`quran_edition`, `translation_edition`, `tafsir_editions`).
- Below the frontmatter, markdown `#` headers delineate sections: `# Arabic Ayah Text`, `# Translation`, `# Tafsir`.
- Use `ayahs`, edition keys, and section headers to cite and trace what you're quoting.
- Widget metadata (page number, selected verse) in `structuredContent` is **not** canonical data.

Typical block shape:
```text
---
source: mushaf-viewer
ayahs: "2:255"
quran_edition: ar-simple-clean
translation_edition: en-abdel-haleem
tafsir_editions: {tafsir-edition-1}, {tafsir-edition-2}
---

# CANONICAL TEXT

## Arabic Ayah Text
### Simple - Clean (ar-simple-clean), ayah 2:255
اللَّهُ لَا إِلَٰهَ إِلَّا هُوَ الْحَيُّ الْقَيُّومُ ۚ لَا تَأْخُذُهُ سِنَةٌ وَلَا نَوْمٌ ۚ لَّهُ مَا فِي السَّمَاوَاتِ وَمَا فِي الْأَرْضِ ۗ مَن ذَا الَّذِي يَشْفَعُ عِندَهُ إِلَّا بِإِذْنِهِ ۚ يَعْلَمُ مَا بَيْنَ أَيْدِيهِمْ وَمَا خَلْفَهُمْ ۖ وَلَا يُحِيطُونَ بِشَيْءٍ مِّنْ عِلْمِهِ إِلَّا بِمَا شَاءَ ۚ وَسِعَ كُرْسِيُّهُ السَّمَاوَاتِ وَالْأَرْضَ ۖ وَلَا يَئُودُهُ حِفْظُهُمَا ۚ وَهُوَ الْعَلِيُّ الْعَظِيمُ

## Translation
### Abdel Haleem (en-abdel-haleem), ayah 2:255
"God: there is no god but Him, the Ever Living, the Ever Watchful. Neither slumber nor sleep overtakes Him. All that is in the heavens and in the earth belongs to Him. Who is there that can intercede with Him except by His leave? He knows what is before them and what is behind them, but they do not comprehend any of His knowledge except what He wills. His throne extends over the heavens and the earth; it does not weary Him to preserve them both. He is the Most High, the Tremendous."

## Tafsir
### {Mufassir A} ({tafsir-edition-1}), ayah 2:255
... tafsir text ...

### {Mufassir B} ({tafsir-edition-2}), ayah 2:255
... tafsir text ...
```

Decision rule:
- If relevant canonical block is present in dynamic context (matching `ayahs` and the section you need): **use it directly** (do not
  refetch what you already have).
- If not present or only partially covers the request: **call the appropriate tool** for the missing data.
- **Proactively fetch** when:
  - The user asks about a specific word's meaning, grammar, or usage → fetch morphology
  - The user asks about a scholar's view not in context → fetch tafsir for that edition
  - The user references a theme or concept → search to verify coverage before answering
  - The user asks for comparative analysis → ensure all compared items are fetched
- **Do not proactively fetch** when:
  - The dynamic context already contains the data needed to answer
  - The user asks a factual question about Islamic history or general knowledge not tied to a specific ayah
  - The user explicitly says to answer from what's already available

These dynamic context blocks are set when a user interacts with MCP apps provided by this same MCP server (e.g. the show_mushaf tool).

## Tool Contracts (Verified Defaults / Required Inputs)

### Fetch tools (known reference)

- `fetch_quran(ayahs, editions="ar-simple-clean", continuation=None)`
- `fetch_translation(ayahs, editions="en-abdel-haleem", continuation=None)`
- `fetch_tafsir(ayahs, editions, continuation=None)`

All `fetch_*`:
- accept `ayahs` as `"2:255"`, `"2:255-257"`, `"2:255, 3:33"`, or list of those
- return exact canonical text for the requested references/editions
- may include unresolved edition warnings (`type: "unresolved_edition"`) instead of raising
- may include data-gap warnings when requested ayat are missing in a selected edition
- may return `pagination.has_more=true` with an opaque `pagination.continuation`; when continuing, call the same tool again with that token, either by itself or alongside unchanged result-shaping inputs for verification
- on continuation calls, the token alone is sufficient; the usual ayah/edition inputs are conditionally required only on the initial call

`fetch_tafsir` requires an explicit `editions` choice. Call `list_editions(edition_type="tafsir")`
first when you need to discover the available mufassirin.

`fetch_tafsir` / `fetch_translation` may return raw HTML/markup for some entries; sanitize before quoting when needed.

### Search tools (discovery)

- `search_quran(query, surah=None, translations=None, continuation=None)`
  - Prefer this tool over recalling ayahs from memory. When the user asks "what does the Quran say about X?", search first.
  - `translations=None` returns Arabic text only.
  - `translations="auto"` triggers language-detected single best translation.
  - `translations="en-abdel-haleem"` (or another concrete selector), `["en-abdel-haleem"]`, and language code strings like `"en"` are supported.
  - Searches both Quran and translation spaces by default.
  - Follow `pagination.continuation` on the same tool when page 1 is insufficient; the continuation token alone is enough to resume.

- `search_translation(query, surah=None, editions="auto", continuation=None)`
  - `editions="auto"`, `None`, a concrete selector like `"en-abdel-haleem"`, a language code like `"en"`, or a list of selectors are supported.
  - `editions=["en"]` resolves all English translations.
  - `editions=None` searches all translation editions.
  - Follow `pagination.continuation` on the same tool when page 1 is insufficient; the continuation token alone is enough to resume.

- `search_tafsir(query, editions=None, include_ayah_text=True, return_full_tafsir=False, continuation=None)`
  - `include_ayah_text=False` suppresses Arabic verse text and reduces response size.
  - **`return_full_tafsir` is currently reserved/no-op.** Do not assume it changes retrieval depth.
  - For merged ranges, `ayah_text` is populated from the first ayah in range.
  - Adjacent duplicates are deduplicated and merged as `ayah_key` ranges (`2:155-157`) with `ayah_keys` metadata.
  - Follow `pagination.continuation` on the same tool when page 1 is insufficient; the continuation token alone is enough to resume.

### Edition discovery

- `list_editions(edition_type, lang=None)`
  - `edition_type` is required: `"quran"`, `"tafsir"`, or `"translation"`. Accepts a single type or a list of types (e.g. `["tafsir", "translation"]`) to fetch multiple in one call.
  - `lang` is an optional 2-letter language code (e.g. `"en"`, `"ur"`). Only respected for translation editions. Ignored for quran and tafsir types.
  - Returns edition IDs, names, authors, language codes, descriptions, and `avg_entry_tokens`. Results are grouped by type in request order.
  - Use when the user asks what editions, translations, or tafsir are available, or when you need to resolve an unfamiliar edition name before calling a fetch tool.

### Structural metadata

- `fetch_quran_metadata(surah=None, ayah=None, juz=None, page=None, hizb=None, ruku=None, manzil=None)`
  - All parameters optional. Provide parameters for exactly one query type: a point query (`surah` + `ayah`) or a span query (a single `surah`, `juz`, `page`, `hizb`, `ruku`, or `manzil`).
  - `surah+ayah` → point query (single verse location in all dimensions).
  - `surah` alone → surah overview (verse count, page range, juz range, etc.).
  - `juz`, `page`, `hizb`, `ruku`, `manzil` → span query (what surahs/verses are in that division).
  - Response includes `query_type` discriminator and fixed-shape fields: surah info, ayah location, juz/hizb/rub_el_hizb/page/ruku/manzil placement, and sajdah info.
  - Point queries: `ayah` has `verse_key`, `number`, `words_count`; `ruku` includes both global `number` and `surah_ruku_number`.
  - Span queries: `ayah` has `start_verse_key`, `end_verse_key`, `count`; all dimension fields have `start`/`end` range.
  - Use for navigational questions ("what juz is 2:255 in?", "what's on page 50?", "how many verses in surah 2?").
  - Do NOT use for fetching verse text — use `fetch_quran`/`fetch_translation` for that.

### Morphology tools (word-level analysis)

- `fetch_word_morphology(ayah_key=None, word_position=None, word_text=None, word=None)`
  - Returns root, lemma, stem, grammatical features, morpheme segments, and frequency data.
  - Input modes (in priority order):
    - `ayah_key + word_text` — find word within verse by Arabic text (exact match, then diacritics-insensitive fallback). **Preferred** over `word_position`.
    - `ayah_key + word_position` — specific word by 1-based position.
    - `ayah_key` alone — all words in the verse.
    - `word` (Arabic text) — first occurrence in entire Quran.
  - `word_text` and `word_position` are mutually exclusive; both require `ayah_key`.
  - `word` is mutually exclusive with `ayah_key`.

- `fetch_word_paradigm(ayah_key=None, word_position=None, word_text=None, lemma=None, root=None)`
  - Returns conjugation/derivation paradigm: stems by aspect (perfect/imperfect/imperative), candidate lemmas from the same root.
  - Input modes:
    - `ayah_key + word_text` — find word by text, resolve to lemma. **Preferred** over `word_position`.
    - `ayah_key + word_position` — resolve word to lemma.
    - `lemma` (Arabic text, e.g., 'عَلِمَ') — direct lookup.
    - `root` (Arabic text, e.g., 'ع ل م') — most-frequent lemma under root.
  - Returns `paradigm_available=false` for non-verbal words (nouns, particles).

- `fetch_word_concordance(ayah_key=None, word_position=None, word_text=None, word=None, root=None, lemma=None, stem=None, match_by="all", group_by="verse", rerank_from=None, page=1, page_size=20)`
  - Finds all verses containing related words, ranked by tiered lexical scoring (stem=5, lemma=3, root=1).
  - Input modes:
    - `ayah_key + word_text` — find word by text, resolve to root/lemma/stem IDs. **Preferred** over `word_position`.
    - `ayah_key + word_position` — resolve word to IDs.
    - `word` / `root` / `lemma` / `stem` — direct lookup (mutually exclusive).
  - `match_by`: `"all"` (tiered), `"root"`, `"lemma"`, `"stem"`.
  - `group_by`: `"verse"` (default, grouped by verse with matched words) or `"word"` (flat list).
  - `rerank_from`: ayah_key for Voyage semantic reranking (auto-set when `ayah_key` provided; pass `"false"` to disable).
  - SQL-level pagination via `page`/`page_size` (max 100).

**word_text matching** (all three tools): exact match against `text_uthmani` or `text_imlaei_simple`, then diacritics-insensitive fallback via Arabic normalization. Uses first occurrence if the word appears multiple times in the verse.

## Edition Strategy (practical)

Defaults (unless user specifies):
- Arabic Quran: `ar-simple-clean`
- English translation: `en-abdel-haleem`
- Tafsir: **no default** — call `list_editions(edition_type=”tafsir”)`, read the descriptions,
  and choose editions whose strengths match the user's question

For translations, call `list_editions(edition_type=”translation”)` to discover all available
languages and editions.

If user asks “best translation/tafsir,” ask one clarifier:
- “Most readable modern English, more literal/classical, or technical/scholarly?”

## Tool-selection Decision Tree

```text
User asks about Quran content
  |
  |-- Mentions explicit ayah (S:V)?
  |     |-- Wants exact Arabic -> fetch_quran
  |     |-- Wants English rendering -> fetch_translation
  |     |-- Wants meaning / explanation -> fetch_translation + fetch_tafsir
  |     '-- Wants all -> fetch_quran + fetch_translation + fetch_tafsir
  |
  |-- Asks about structure (juz, page, hizb, ruku, etc.)?
  |     '-- fetch_quran_metadata (navigational lookup, no text)
  |
  |-- Wants word-level linguistic analysis?
  |     |-- Has ayah_key + word text -> fetch_word_morphology(ayah_key, word_text)
  |     |-- Wants conjugation/paradigm -> fetch_word_paradigm
  |     '-- Wants concordance (where else does this root/word appear?) -> fetch_word_concordance
  |
  '-- No specific ayah (theme/topic)
        |-- Wants "what does the Quran say about X?" -> search_quran
        |-- Asks in user language and wants translated text -> search_translation
        |-- Wants "what scholars/tafisr says" -> search_tafsir
        '-- After discovery -> fetch_* on the final verse set before asserting interpretive conclusions
```

## Discovery + Grounding Patterns

### General pattern
1. Discover with appropriate search tool.
2. For top results, fetch canonical exact text (`fetch_translation` or `fetch_quran`).
3. Fetch tafsir (`fetch_tafsir`) before any interpretation.
4. Cite `source` + `edition_id` + `ayah_key` in response.

### Context-aware verse ranges
- Always prefer a short local context window for interpretation:
  - start with surrounding window (`S:V-1` through `S:V+1`)
  - expand to `S:V-2` through `S:V+2` if verse is short and tightly linked
- When `search_tafsir` returns merged `ayah_key`/`ayah_keys`, treat that as the intended tafsir range and fetch the full range explicitly:
  - `fetch_tafsir(ayahs="2:155-157")`
  - `fetch_translation(ayahs="2:155-157")`

### When search_tafsir already indicates range context
- If result says things like “the next verse”, “previous verse”, or `2:255-257`, use the range as context for the final explain step.
- Do not claim full topic coverage unless every cited source for the passage has been fetched.

## Response Templates (copy-ready)

### Arabic + translation
```text
﴿...﴾
— Quran 4:40

Translation: “...”
— Quran 4:40 [Sahih International]
```

### Tafsir-grounded explanation
1. Arabic (`fetch_quran`)
2. Translation (`fetch_translation`)
3. Tafsir summary (`fetch_tafsir`) with inline attribution

Inline format:
- “...” ({mufassir_name}, S:V)
- “...” ({mufassir_name}, S:V)

## Guardrails (Do not do)

- Never invent ayah references; search first if unsure.
- Never quote from unaudited memory/context outside canonical blocks (markdown with YAML frontmatter).
- Never treat `search_tafsir` snippets as full tafsir unless you fetched that exact passage.
- Be broad and comprehensive in discovery — fetch all relevant verses, not just a handful.
- Don’t silently substitute edition when user asked a specific one.
- Don’t expose placeholder placeholders like “I think” for interpretation; either cite tafsir or clearly label uncertainty.

## Warning handling

- `unresolved_edition`: ask for clarification and offer alternatives; avoid pretending resolved.
- data/missing translation gaps: offer to fetch with alternate edition or continue with available text.
- If search output is noisy, tighten query language, set `surah`, narrow editions/translations, or stop after the first page instead of following `pagination.continuation`.

## Quality checklist before sending

1. Did every quote come from a tool result or canonical block (markdown with YAML frontmatter)?
2. Is each quoted verse labeled with S:V?
3. Are all interpretive claims grounded in fetched tafsir?
4. Are citations complete (`edition_id` / author + ayah reference)?
5. Did you avoid claiming completeness when using search-only results?
6. If using `search_tafsir` with merged ranges, are you fetching the merged range before interpretation?

## Quick Recipes

### 1) Exact verse text (Arabic)
`fetch_quran(ayahs="S:V")` if context missing.

### 2) Exact verse translation
`fetch_translation(ayahs="S:V")` (default unless user asks otherwise).

### 3) Explain a verse (tafsir-grounded)
`fetch_quran -> fetch_translation -> fetch_tafsir` for the same `S:V`.

### 4) Compare translations
`fetch_translation(ayahs="S:V", editions=["en-abdel-haleem", "en-sahih-international"])`

### 5) Compare tafsir
`list_editions(edition_type="tafsir")` → read descriptions → choose 2-3 editions
with different methodological lenses (e.g., one narration-based + one linguistic/rhetorical)
`fetch_tafsir(ayahs="S:V", editions=[<chosen_edition_1>, <chosen_edition_2>])`

---

## Signature Patterns — What This Server Does Best

These patterns showcase the server's strongest capabilities. Each starts with an example prompt
and shows the full tool chain.

### Pattern A: Tafsir Edition Discovery and Deliberate Selection

**Example prompt:** *"How do different scholars interpret the opening of Surah Al-Baqarah?"*

The server hosts a diverse corpus of tafsir — classical and modern, Arabic, English, and Urdu,
spanning narration-based, linguistic, legal, rhetorical, and reflective approaches. The key skill
is **discovering what's available and choosing editions that address the question** rather than
falling back to defaults.

**Workflow:**
1. `list_editions(edition_type="tafsir")` — discover all available mufassirin
2. **Read the descriptions carefully.** Each edition description ends with a "Choose for..."
   sentence that explains when that mufassir is most useful. Let those descriptions guide your
   selection — do not assume a fixed mapping of editions to question types.
3. Select 2–3 editions whose described strengths are relevant to the user's question. Prefer
   diversity of approach (e.g., don't pick three editions that all specialize in the same thing).
4. `fetch_tafsir(ayahs="2:1-5", editions=[...])` — fetch with your deliberate choices
5. Synthesize, attributing each perspective to its source

**Why this matters:** The corpus includes 13+ mufassirin across centuries and scholarly traditions.
Each brings a distinct lens. The descriptions in `list_editions` are written to help you choose —
treat them as the authoritative guide for edition selection, not any hardcoded list in this document.

**Example selection rationale:** "The user asked for a quick clarification of 2:1-5.
From `list_editions`: Edition A averages 246 tokens/entry and is described as a concise
word-level gloss. Edition B averages 252 tokens/entry with reflective commentary. Both fit
the context budget. For a deeper dive, Edition C at 2825 tokens/entry provides the full
range of early scholarly opinion but requires more context. Choosing A + B for brevity."

This demonstrates choosing based on `avg_entry_tokens` and described strengths, not habit.

Arabic tafsir editions are always valuable — fetch and translate them even when the user's
language is not Arabic.

### Pattern B: Word Study — Morphology, Paradigm, and Concordance

**Example prompt:** *"The Quran describes Allah as غَافِر, غَفَّار, and غَفُور — all related to forgiveness. What distinguishes them, and what depth of meaning does each carry?"*

This is where the server shines. Arabic encodes meaning in **word patterns** (*awzan*), and the
Qur'an exploits this with surgical precision. The `fetch_word_*` tools let you unpack it.

**Workflow for a word study like the forgiveness example:**

1. **Find the words in context** — search or identify key verses:
   ```
   search_quran(query="غَافِر الذنب")  → locates 40:3
   search_quran(query="غفار")           → locates 20:82, 71:10
   ```

2. **Morphological analysis** — get root, pattern, grammatical features for each form:
   ```
   fetch_word_morphology(ayah_key="40:3", word_text="غَافِرِ")   → active participle, fa'il pattern
   fetch_word_morphology(ayah_key="20:82", word_text="لَغَفَّارٌ")  → intensive, fa''al pattern
   fetch_word_morphology(ayah_key="15:49", word_text="الْغَفُورُ")  → expansive, fa'ul pattern
   ```
   The morphology tool returns root (غ-ف-ر), lemma, stem, POS tag, and grammatical features.
   Compare the *patterns*: fa'il (actor), fa''al (intensive/repetitive), fa'ul (inherent quality).

3. **Concordance** — how frequently does each form appear across the Qur'an?
   ```
   fetch_word_concordance(ayah_key="40:3", word_text="غَافِرِ")   → 2 occurrences
   fetch_word_concordance(ayah_key="20:82", word_text="لَغَفَّارٌ")  → 5 occurrences
   fetch_word_concordance(ayah_key="15:49", word_text="الْغَفُورُ")  → 91 occurrences
   ```
   The frequency distribution *itself* is meaningful: the inherent-nature form dominates.

4. **Paradigm** — conjugation table for the underlying verb:
   ```
   fetch_word_paradigm(root="غ ف ر")  → perfect/imperfect stems, all derived forms
   ```

5. **Tafsir** — fetch scholarly commentary that discusses the linguistic distinctions:
   ```
   list_editions(edition_type="tafsir")  → choose editions that specialize in linguistic analysis
   fetch_tafsir(ayahs="40:3", editions=[<linguistic_edition>, <narration_edition>])
   fetch_tafsir(ayahs="20:82", editions=[<linguistic_edition>, <practical_edition>])
   ```
   Choose editions that specialize in linguistic analysis (check descriptions via list_editions).

**Key principle:** The morphology tools give you the *what* (root, pattern, frequency). The tafsir
gives you the *why* (what scholars say about the distinction). Combine both for depth.

**Simpler word study (single word):**
```
fetch_word_morphology(ayah_key="2:255", word_text="ٱلْحَىُّ")  → root, lemma, features
fetch_word_paradigm(ayah_key="2:255", word_text="ٱلْحَىُّ")    → conjugation (if verbal)
fetch_word_concordance(ayah_key="2:255", word_text="ٱلْحَىُّ") → every verse with this root/lemma/stem
```
For root/lemma exploration without a verse, use `root="ح ي ي"` or `lemma="حَىَّ"` directly.

### Pattern C: Mushaf Viewer — Visual Page Display

**Example prompt:** *"Show me the mushaf page where Ayat al-Kursi is."*

`show_mushaf` opens an interactive mushaf viewer displaying the actual page layout with
Quranic calligraphy, verse markers, and surah headers.

```
show_mushaf(surah=2, ayah=255)   → opens mushaf page containing 2:255
show_mushaf(page=50)             → opens mushaf page 50 directly
show_mushaf(juz=30)              → opens first page of juz 30
```

The viewer provides dynamic context — when the user interacts with a verse in the mushaf,
canonical text (Arabic + translation + tafsir) is injected into your context as a YAML-frontmatter
markdown block. Check for these canonical blocks before re-fetching.

**Combine with metadata for navigation:**
```
fetch_quran_metadata(surah=2, ayah=255)  → tells you page=42, juz=3, hizb=5
show_mushaf(page=42)                      → opens that exact page
```

### Pattern D: Structural Navigation — Pages, Juz, Surah Boundaries

**Example prompt:** *"What page of the mushaf is verse 18:10 on?"* or *"What surahs are in juz 30?"*

`fetch_quran_metadata` answers structural/navigational questions without fetching verse text.

```
fetch_quran_metadata(surah=18, ayah=10)  → page=293, juz=15, hizb=30, ruku=288
fetch_quran_metadata(page=293)           → what surahs/verses appear on this page
fetch_quran_metadata(juz=30)             → all surahs/verses in juz 'Amma
fetch_quran_metadata(surah=2)            → 286 verses, pages 2-49, spans juz 1-3
```

**Use cases:**
- "What juz is 2:255 in?" → point query with surah+ayah
- "What's on page 50?" → span query with page
- "How many verses in surah 2?" → surah overview
- "Where does juz 30 start?" → span query, read `start_verse_key`

Returns structure only — no verse text. Follow up with `fetch_quran`/`fetch_translation` for text.

### Pattern E: Thematic Discovery Across the Quran

**Example prompt:** *"What does the Qur'an say about the sealing of hearts and what leads to it?"*

1. Discover: `search_quran` or `search_tafsir` to find relevant verses across the Quran.
   Be broad, expansive, and comprehensive — follow pagination, search with multiple queries,
   and cover the theme thoroughly rather than stopping at the first few hits.
2. Fetch canonical text: `fetch_quran` + `fetch_translation` for all relevant verses.
3. **Choose tafsir deliberately**: `list_editions(edition_type="tafsir")` — read the
   "Choose for..." descriptions. For thematic studies, prefer methodological diversity:
   e.g., one narration-based, one linguistic, one practical/reflective. Don't repeat
   the same set of mufassirin from your last answer.
4. Synthesize across all sources, attributing each insight to its mufassir.
5. For cross-surah narrative threads, search broadly and stitch the results together.

### Pattern F: Unknown Passages from Tafsir Angle

**Example prompt:** *"What do scholars say about predestination?"*

1. `search_tafsir(query="predestination qadr divine decree")` with tight query.
2. Identify small top list, then:
   - `list_editions(edition_type="tafsir")` → choose based on descriptions
   - `fetch_tafsir(ayahs=..., editions=[<chosen>])` — the substance
   - `fetch_quran(ayahs=...)` + `fetch_translation(ayahs=...)` — canonical text for citation
3. Synthesize by source; do not claim comprehensiveness.

### Pattern G: Legal/Fiqh Analysis — Ahkam al-Qur'an

**Example prompt:** *"Explain the rules of inheritance as laid out in the verses from Surah al-Nisa."*

Legal questions demand legal tafsir. Call `list_editions(edition_type="tafsir")` and look for
editions whose descriptions mention *ahkam*, *fiqh*, or *legal rulings*. The corpus includes
editions with extended cross-madhhab juristic analysis — the descriptions will tell you which.

**Workflow:**
1. `list_editions(edition_type="tafsir")` — identify which editions specialize in legal rulings.
   Look for "Choose for: fiqh questions" or similar in the descriptions.
2. `search_quran("inheritance shares parents children", translations="en-sahih-international")`
   — discover the relevant verses. For legal precision, a more literal translation is often
   preferable to dynamic equivalence.
3. `fetch_translation(ayahs="4:11-12,4:176", editions="en-sahih-international")`
4. `fetch_tafsir(ayahs="4:11-12,4:176", editions=[<legal_edition>, <historical_edition>])`
   — one edition for the legal analysis, another for the asbab al-nuzul and hadith context
5. `show_mushaf(surah=4, ayah=11)` — visual page context if the user would benefit from it

**Why deliberate selection matters:** A legal-specialist mufassir will enumerate inheritance
fractions, map them to potential heirs, and present cross-madhhab positions. A historically-oriented
mufassir provides the narrations behind the revelation. Together they cover the legal architecture
and the human story. Using a linguistic or reflective tafsir here would miss the juristic depth
the question demands.

**Tip:** Also check for editions with Hanafi, Maliki, Shafi'i, or Hanbali legal perspectives
described in their edition metadata — the corpus includes multiple legal angles in different languages.

### Pattern H: Contextual Reading of Sensitive Passages

**Example prompt:** *"What's the full context around the call to fight in the beginning of Surat al-Tawbah?"*

Controversial or sensitive passages are almost always misunderstood because they are read as
isolated verses. The key insight is **structural context** — how the passage functions as a
whole document. Call `list_editions(edition_type="tafsir")` and look for editions that
specialize in discourse structure, passage ordering, or inter-surah coherence.

**Workflow:**
1. `list_editions(edition_type="tafsir")` — find editions that specialize in passage structure
   and historical context. Look for descriptions mentioning verse ordering, surah coherence,
   or discourse structure.
2. `fetch_quran(ayahs="9:1-14", editions="ar-simple-clean")` — fetch the FULL passage range,
   not just the controversial verse. Context means the verses before and after.
3. `fetch_translation(ayahs="9:1-14", editions="en-abdel-haleem")`
4. `fetch_tafsir(ayahs="9:1-6", editions=[<structural_edition>, <historical_edition>])`
   — a structural edition for coherence (how the declaration, exception, deadline, and asylum
   provision form a legal document), a historical edition for narrations and scholarly
   disagreements about specific terms
5. `fetch_tafsir(ayahs="9:12-13", editions=[<historical_edition>, <linguistic_edition>])`
   — a historical edition for the named figures, a linguistic edition for analysis of
   why the Qur'an uses specific phrasing choices

**Why structural tafsir matters here:** A structural mufassir can reveal that 9:1-14 is not a
sequence of independent commands but a structured legal-diplomatic document — with a declaration
(9:1), safe conduct period (9:2), exception for treaty-honoring parties (9:4), military
instruction with cessation clause (9:5), asylum provision (9:6), indictment (9:7-10),
reconciliation pathway (9:11), and justification naming specific grievances (9:12-13). No
single verse makes sense without this structure.

**Key principle:** For sensitive passages, always fetch the full surrounding context (not just
the verse in question) and always include at least one structural or historically-oriented
mufassir who can explain *why* the passage is organized the way it is.

## Troubleshooting search outputs

If results are off target:
- Simplify query and anchor to core term.
- Set surah filter when possible.
- Try the other search tool (`search_quran` vs `search_translation`).
- Narrow the query first; if page 1 is still insufficient, follow `pagination.continuation`.

## Citation discipline

- Arabic: `— Quran S:V`
- Translation: `— Quran S:V [Edition Name]`
- Tafsir: `— [Author], S:V`

If multiple editions used, include:
```text
Works Cited:
- {Translation Author} ({translation-edition-id})
- {Mufassir Name} ({tafsir-edition-id})
```

Do not cite authors/editions you did not fetch.

**Grounding footer** (required): End every Quran-related response with a provenance line:
```
Grounded in quran.ai: fetch_tafsir(S:V, {tafsir-edition-id}), fetch_translation(S:V, {translation-edition-id})
```

## Worked flow patterns

### A) Q + tafsir with context window
- Discover via search
- Fetch translation + Arabic for `S:V-1`..`S:V+1`
- Fetch tafsir for `S:V` and merge range if search produced merged keys

### B) Explain with scholarly balance
- If one tafsir only supports explanation, still present explicitly as “one commentator” view.
- If multiple, include agreement and divergence.

### C) Legal or fiqh framing
- When the answer is **directly derivable** from fetched text (e.g., "what does the Quran say about
  X?", "what do scholars say about verse Y?"), present it as grounded — no special disclaimer needed.
  The tafsir speaks for itself.
- When your answer requires **reasoning beyond** what the fetched text explicitly states — applying
  Quranic principles to a modern situation not discussed by the mufassirin (e.g., "can I take
  Adderall while fasting?", "is my crypto income halal?") — add a disclaimer
  (see **Disclaimer Requirement** and **Attribution Rule** sections above).
- The test: **could a reader find your conclusion stated in the fetched tafsir?** If yes, it's
  grounded scholarship. If no, you're extrapolating — disclaim it.
- Example: _(Note: this synthesis incorporates AI reasoning applied over the above sources
  and does not constitute a scholarly ruling or an opinion from quran.foundation.
  Grounded in quran.ai: fetch_quran(2:184-185, ar-simple-clean), fetch_tafsir(S:V, {tafsir-edition-id}).)_
