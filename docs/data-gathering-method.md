# Data-Gathering Method

**Status:** Method of record for populating the history data model with **reliable, well-sourced** entries. Companion to [`history-model.md`](./history-model.md). Ikwerre is the worked example, but the method is **reusable for any people group**.

**Where things live:** this document is the *method* (cross-project). The blank capture templates and the actual collected data live in the backend repo (`Backend-Imu-Asusu`, `data/collection/`), next to the database they feed.

---

## 1. Principle: data-informed, not data-determined

We do **not** finalize the SQL schema first, and we do **not** gather a shapeless pile and reverse-engineer it later. Instead:

1. The **conceptual model is already locked** (see `history-model.md`) — it gives us the vocabulary to collect against.
2. We gather a small **representative sample** structured by that vocabulary.
3. The sample **validates or reshapes** the model.
4. *Then* we write migrations.

Two rules hold from the very first keystroke:

- **No claim without a source.** Every entry carries at least one source and a confidence level.
- **Contested history is captured, not flattened.** Where accounts disagree, log the disagreement with each side's sources.

---

## 2. The representative sample

Collect a deliberately small set chosen so that, between them, they **exercise every part of the model** — types, fuzzy time, entry-to-entry relationships, figures, language, provenance, and the `disputed` path. If the model holds all of these cleanly, it will hold thousands.

Worked example — **Ikwerre first sample (~15 entries):**

| entry_type | Ikwerre example | What it stress-tests |
| --- | --- | --- |
| `origin_tradition` ×2 | Akalaka descent account **+ a competing variant** | The disputed / multi-perspective path |
| `migration` | Movement into the Rivers forest zone | Fuzzy / relative dating |
| `settlement_founding` | One community (e.g. Aluu or Elele) | `place` link + `founded_by` relationship |
| `institution` | The Eze system / age-grades | Governance modelling |
| `figure` | A named Eze or founder | `figures` + `founded_by` / `succeeded_by` |
| `festival` | A named festival / new-yam rite | `commemorates` relationship back to origin |
| `proverb` ×2 | With literal + figurative meaning | Oral-lit + language capture |
| `folktale` | One narrative | Long `body`, oral source |
| `language` / lexicon | 15–20 words with pronunciation | Language sub-model + **audio** |
| `craft` or `cuisine` | e.g. banga cuisine / a craft | Material culture + `is_endangered` |
| `colonial_encounter` | **Port Harcourt founded on Ikwerre/Diobu land** | Dated event linking origins → land → modern |
| `modern_identity` | The Ikwerre–Igbo identity question | Disputed, multi-source, sensitive |

---

## 3. Collection templates

Four blank templates (living in `Backend-Imu-Asusu/data/collection/`), each mapping 1:1 to the future tables so import is mechanical.

**1. Entry capture** — one per fact/topic:
`entry_type` · `title` · `summary` · `body` · `significance` · `period_start` · `period_end` · `date_precision` · `is_approximate` · `era` · `place` · `people` · `is_endangered` · `is_written` · `relates_to` (other entries + relation_type) · `status`

**2. Source capture** — one per source, linked to entries:
`source_type` (oral_tradition/book/journal/archival/interview/museum/web) · `author_or_informant` · `title` · `year` · `citation_or_url` · `reliability_tier` — and for **oral** sources: `informant_name` · `role_standing` · `community` · `interview_date` · `location` · `language` · `consent_given`

**3. Interview guide** — themed open-question sets (origins, governance, belief, language, economy, festivals), with source-capture metadata filled at the top. Reused across informants so the same questions can be **triangulated**.

**4. Lexicon capture** — `word` · `pronunciation` (IPA or audio ref) · `meaning` · `example_sentence` · `dialect` · `audio_file`.

---

## 4. Provenance & triangulation workflow

1. **No claim without a source row.** Every entry links to ≥1 source from day one.
2. **`verified` only when ≥2 independent source *types* agree** (e.g. a scholarly text *and* an elder). A single source stays `unverified`.
3. **Conflicts are not dropped** — logged as separate entries/notes with `verification_status: disputed` and each side's sources.
4. **Reviewer pass** before an entry flips to `published`.

---

## 5. Ethics & sensitivity guardrails (non-negotiable)

- **Consent + attribution** for every oral informant. Recording an elder is a relationship, not extraction.
- **Sacred / restricted knowledge** — some shrine, masquerade, or ritual detail is deliberately not for public record. Flag it `restricted`; when in doubt, do not publish.
- **Charged topics** (for Ikwerre: the Ikwerre–Igbo identity question and the Port Harcourt land history) — collect *all* sourced perspectives, present as `disputed`, take no editorial side. Note the Educate-Org framing "Ikwere-Igbo" is itself one position; the provenance model lets us show the evidence rather than assert.

---

## 6. Source targets (Ikwerre worked example)

- **Oral (primary):** Rivers State council of traditional rulers; Ikwerre community / town unions; named elders in specific communities.
- **Scholarly:** University of Port Harcourt history & linguistics departments; the Kay Williamson / Roger Blench body of work on Rivers State languages.
- **Reference:** Ethnologue and Glottolog (Ikwere language entry).
- **Archival:** Nigerian National Archives (Enugu / Ibadan) — colonial-era Ikwerre records and early Port Harcourt records.

Every one becomes a row in the source template with a real, checkable citation — **never asserted from memory**. A fabricated citation is worse than none; it poisons the reliability the platform exists to provide.

---

## 7. Where the sample lives before there is a database

There is no live schema yet, so collect into the **structured intake in `Backend-Imu-Asusu/data/collection/`** — the templates map 1:1 to the future tables. When the schema is finalized, import is mechanical.

---

## 8. Feedback loop: how the sample reshapes the schema

After the ~15 are collected, run a validation checklist:

- Did every entry fit an `entry_type`? (missing types → add to the vocabulary)
- Did the time fields capture the fuzziness **honestly**? (if not → adjust `date_precision`)
- Did `relates_to` express the real connections? (missing relation types → extend)
- Did provenance capture everything we needed? (gaps → add source/confidence fields)

**The gaps become the schema refinements — and only then do we write SQL.**
