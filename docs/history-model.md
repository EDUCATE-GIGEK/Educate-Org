# History Data Model — Conceptual Spec

**Status:** **Validated (2026-07-19)** against the first Ikwerre sample (`Backend-Imu-Asusu/data/collection/ikwerre/`). Sections below reflect the sample-driven refinements; the authoritative summary is the tiered **[final entity list](#9-validated-final-entity-list-tiered)** at the end. Ready for the Tier-1 migration (raw PostgreSQL in the backend's `SQL/`).

**Scope:** the shared data model behind Ịmụ-Asụsụ (and future Educate projects) for representing the history and culture of any people group. This document is the cross-project source of truth; the implementing migrations will live in the backend repo (the future self-hosted database + API), not the frontend.

---

## 1. Purpose & principles

The platform preserves and teaches the histories of underrepresented peoples in a way that is **accessible, engaging, and — above all — accurate**. The data model must therefore do three things the current schema cannot:

1. **Place knowledge in time** — history needs dates, periods, and eras (including honestly fuzzy ones for oral tradition).
2. **Connect knowledge to knowledge** — the value is in the *web* of relationships between aspects of a culture, not isolated facts.
3. **Carry its own provenance** — every claim knows where it came from and how confident we are. Reliability is a first-class feature, not an afterthought.

Design principles:

- **Generic structure, localized labels.** The schema must not hardcode one country's administrative or ethnic vocabulary.
- **Provenance from the first keystroke.** No fact enters without a source and a confidence level.
- **Contested history is modelled, not flattened.** Where accounts disagree, we store the disagreement with its sources rather than picking a winner silently.
- **Fewer, more general surfaces.** Fewer entity types → fewer page templates → the next people group (or country) needs little to no new code.

---

## 2. The two-axis taxonomy (replaces the Nigeria-specific tables)

The current tables — `states`, `local_governments`, `ethnic_groups`, `tribes` — hardcode Nigeria's structure and tangle **two distinct axes**:

- **Place** (*where*): every country subdivides differently (state / province / région-département / county / prefecture / emirate / chiefdom …).
- **People** (*who*): also varies (ethnic group / nation / tribe / clan / lineage / house …).

Generalize each axis into **one self-nesting tree** where the level is a **label on the node**, not a table.

### `places` — recursive geography
| column | notes |
| --- | --- |
| `id` | |
| `parent_id` | nullable, self-reference |
| `name` | |
| `designation` | the level label — see vocabulary §5. e.g. Country, State, Province, County, LGA, City, Village |
| `level_rank` | optional int, for ordering/depth checks |
| `iso_code` | nullable; for countries |
| `general_info` | jsonb |

Folds continents + countries + states + LGAs into a single tree of arbitrary depth.

### `peoples` — recursive people groups
| column | notes |
| --- | --- |
| `id` | |
| `parent_id` | nullable, self-reference |
| `name` | |
| `designation` | Ethnic Group, Nation, Tribe, Clan, Lineage, Community … (§5) |
| `general_info` | jsonb |

### `people_places` — many-to-many link
| column | notes |
| --- | --- |
| `people_id` | |
| `place_id` | |
| `relationship` | `homeland` \| `diaspora` \| `historical` |

A people is rarely 1:1 with a place. **Keep the two trees separate** (do not merge into one table): the Ikwerre span several LGAs (Ikwerre, Emohua, Obio/Akpor, Diobu); the Kurds span four countries. `people_places` captures exactly this — a merged tree cannot.

### Why this generalizes
```
Nigeria:   Continent → Country(Nigeria) → State(Rivers) → LGA(Ikwerre) → Village(Aluu)
France:    Continent → Country(France)  → Région → Département → Commune
Kingdom:   —          → Country          → Kingdom → Chiefdom → Village
Ikwerre (peoples tree): Ethnic Group(Ikwerre) → Clan → Community → Lineage
```

### Frontend payoff
Five page hooks today (`useStatePage`, `useCountryPage`, `useLocalGovernmentPage`, `useEthnicGroupPage`, `useTribePage`) collapse to **two** generic templates (`usePlacePage`, `usePeoplePage`) that render any level from data. Breadcrumbs and "contains / part-of" navigation fall out of `parent_id`.

### Trade-offs (eyes open)
- Ancestor/descendant lookups need **recursive CTEs** instead of flat joins (Postgres handles this natively).
- `designation` needs a **controlled vocabulary** or it degrades ("State" vs "state" vs "province"). See §5 open decision.
- Level integrity (a Country shouldn't nest under a Village) is **curation-enforced**, optionally checked via `level_rank`.

---

## 3. The knowledge model (evolves the current `history` table)

Today `history` is one flat table with a free-text `category` and a loose `entry` blob. Evolve it into a small **knowledge graph**.

### `entries` — typed units of knowledge (**replaces the flat `history` table**)
| column | notes |
| --- | --- |
| `id` | |
| `entry_type` | controlled enum, the "best-aspect" lenses — §5 |
| `title`, `summary`, `body`, `significance` | content (short teachable summary; full text in `body`) |
| `period_start`, `period_end` | may be null for oral tradition |
| `date_precision` | `year`\|`decade`\|`century`\|`era`\|`relative` |
| `is_approximate` | boolean |
| `period_note` | free text for relative dating, e.g. "≈4 generations before present" *(validation: `relative` precision lost its reference point without this)* |
| `era` | §5 |
| `place_id?`, `people_id?` | FKs into the two trees — **this is what collects a group's entries** (see §3.1) |
| `is_endangered`, `is_written` | boolean |
| `is_restricted` | boolean — sacred/secret knowledge, collected but **not for public display** *(validation gap)* |
| `verification_status` | `unverified`\|`verified`\|`disputed` — **epistemic** state of the claim |
| `workflow_status` | `draft`\|`in_review`\|`published` — **editorial** pipeline |

Two distinct status fields (a validation finding): whether a claim is *trusted* is separate from where it is in the *editorial* pipeline. Fuzzy time flags + `period_note` let "founded ~4 generations ago" sit honestly on a timeline.

### 3.1 Belonging vs connecting — two different links
- **Belonging** (assemble "all of Ikwerre's history"): filter `entries` by `people_id` (and its descendant clans in the `peoples` tree) and/or `place_id`. This is what gathers a group's whole set — it is *not* a relationship.
- **Connecting** (how those entries relate): the join tables below. This is what makes the set navigable (timeline threads, knowledge graph).

### `entry_relationships` — the entry↔entry graph (the key connective table)
`from_entry_id`, `to_entry_id`, `relation_type` (§5). Powers the timeline threads and graph view. Relationships that involve a **person** live in `entry_figures` / `figure_relationships`, not here.

### `figures` — people as their own entity (**not** an entry_type)
`figures` (`name`, `role`, `people_id`, `birth_note`, `death_note`, `is_restricted`) + `figure_relationships` (`parent_of` / `succeeded_by` / `married`) for genealogy & succession + `entry_figures` (`entry_id`, `figure_id`, `role`: `founded_by` / `about` / `led_by`) linking people to the entries they appear in. *(Validation: `figure` as an entry_type couldn't hold lifespan or figure↔figure links.)*

### Languages — their own entities (**not** an entry_type)
`languages` (`name`, `iso_code`, `classification`, `endangerment_status`, `people_id`) + `dialects` + `lexicon` (`word`, `pronunciation`, `meaning`, `example_sentence`, `audio_media_id`).

### `media`
`media` (`media_type`, `url`, `caption`, `source_id`, `is_restricted`) + an `entry_media` join.

---

## 4. Provenance — the reliability backbone (non-negotiable)

Because this history is under-documented in written records, **method matters more than any single fact**: triangulate, and record where everything came from.

### `sources`
| column | notes |
| --- | --- |
| `type` | `oral_tradition` \| `book` \| `journal` \| `archival` \| `interview` \| `museum` \| `web` |
| `author_or_informant` | person / elder / author |
| `title` | |
| `year` | |
| `citation` | full reference or URL |
| `reliability_tier` | editorial grading |
| `notes` | |

### `entry_sources` (join)
| column | notes |
| --- | --- |
| `entry_id`, `source_id` | |
| `confidence` | per claim |
| `verification_status` | `verified` \| `unverified` \| `disputed` |
| `reviewer` | |

**Provenance discipline:**
- A claim becomes `verified` only when **independent source types agree** (e.g. a scholarly text *and* an elder interview).
- Conflicting accounts are stored as **`disputed`** with each side's sources — never silently dropped.
- Oral testimony is captured with **informant + place + date** at collection time.

This model is also what a trustworthy AI fact-checker grounds against.

---

## 5. Controlled vocabularies

> **Open decision — how strict?** `designation` (and to a lesser extent `entry_type`) can be a fixed **enum** (safest; code edit to add "Emirate"), a **lookup table** (add rows, no code — *recommended*), or **free text** (max flexibility, least consistency). Recommendation: lookup table with an `other` escape.

**`place.designation`** (starter): Continent, Country, State, Province, Région, Département, County, LGA, District, Chiefdom, Emirate, City, Town, Village, Compound.

**`people.designation`** (starter): Ethnic Group, Nation, Tribe, Clan, Lineage, House, Community, Age-grade.

**`entry_type`** (the cultural lenses): origin_tradition, migration, settlement_founding, institution, deity_spirit, shrine_site, cosmology, festival, rite_of_passage, masquerade, proverb, folktale, praise_name, naming_custom, craft, architecture, attire, cuisine, music_dance, economy, agriculture, trade_route, marriage_custom, kinship, event, conflict, alliance, colonial_encounter, modern_identity, diaspora. *(Validation removed `figure`, `language`, and `lineage`: figures and languages are their own entities, and `Lineage`/`Clan` are `people.designation` values, not entry types.)*

**entry `relation_type`** (entry↔entry): caused, followed_by, part_of, commemorates, **contradicts** *(for disputed accounts)*, derived_from, related_to.
**figure `relation_type`** (figure↔figure): parent_of, succeeded_by, married, sibling_of.
**entry_figures `role`** (entry↔figure): founded_by, led_by, about, mentions, attributed_to.

**`verification_status`**: unverified, verified, disputed.  **`workflow_status`**: draft, in_review, published.

**`era`**: pre-colonial, colonial, post-independence, contemporary.

---

## 6. Migration path from the current schema

An **evolution, not a teardown**, built in the two tiers of §9:

1. **Tier 1 first** — create `places` + `peoples` + `people_places`; backfill from `continents`/`countries`/`states`/`local_governments` (→ `places`) and `ethnic_groups`/`tribes` (→ `peoples`) with a `designation` each. Reshape `history` → `entries` (add `entry_type`, time columns, `period_note`, `is_restricted`, `verification_status`, `workflow_status`; repoint the four geo/ethnic FKs to `place_id`/`people_id`; backfill the ~6 rows). Add `entry_relationships`, `sources`, `entry_sources`.
2. **Tier 2 next** — `figures` + `figure_relationships` + `entry_figures`; `languages`/`dialects`/`lexicon`; `media` + `entry_media`.

Written as raw PostgreSQL in the backend's `SQL/`. If also applied to the live Supabase project, layer RLS policies on top; the core DDL is portable and unaffected.

---

## 7. Open decisions

**Resolved by the 2026-07-19 validation pass:** figures & languages are their own entities (not entry_types); two status fields (`verification_status` + `workflow_status`); an `is_restricted` flag; explicit typed join tables (not one polymorphic relationships table); `contradicts` relation for disputed accounts.

Still open:
1. **`designation` strictness** — enum vs **lookup table (recommended)** vs free text.
2. **Table naming** — `places` / `peoples` acceptable? 
3. **Contested-history presentation** — **multi-perspective + sourced (recommended)** vs a single curated canonical view. Note: the Educate-Org README frames the group as the "Ikwere-Igbo ethnic group," which is *itself* a position on a contested question (see §8). The provenance model lets us hold that framing while still showing the evidence.

---

## 8. Ikwerre — starter facts and sourcing

> ⚠️ **Every factual line below is `unverified` until sourced.** It is scaffolding to test the model, **not** established fact. Origin traditions and clan genealogies in particular exist in *variants* and must be captured as competing `origin_tradition` entries, not merged.

- Endonym *Iwhuruọha*; a people of Rivers State in the Niger Delta, Nigeria.
- Present across the Ikwerre, Emohua, Obio/Akpor and Port Harcourt (Diobu) local government areas. *(to be sourced against Rivers State / census references)*
- Language classified within the Igboid group — a **contested** classification bound up with the Ikwerre–Igbo identity question. *(verify on Ethnologue / Glottolog; note the ISO 639-3 language entry there)*
- Origin traditions cite a progenitor often named **Akalaka**, with clan genealogies branching from him — **variants exist.** *(oral tradition; triangulate across communities)*
- Traditional governance led by **Eze** titles, with councils and age-grades. *(to be sourced)*
- Historically an Ọdịnala-type indigenous cosmology with shrines; predominantly Christian today. *(to be sourced)*
- Economy: yam and cassava farming, fishing, palm produce. *(to be sourced)*
- **Port Harcourt was founded by the British (c. 1912–13) on Ikwerre / Diobu land** — a load-bearing `colonial_encounter` entry linking origins, land, and modern identity. *(verify founding date and land history against archival + scholarly sources)*

### Sourcing method
Triangulate three source types, each recorded in `sources`:
1. **Scholarly / linguistic** — e.g. the body of work by **Kay Williamson** (and collaborators such as **Roger Blench**) on Rivers State / Niger Delta and Igboid languages; university (e.g. University of Port Harcourt) scholarship on Ikwerre history. *(look up specific titles; do not cite from memory.)*
2. **Archival** — the **Nigerian National Archives** (Ibadan / Enugu): colonial intelligence reports, early Port Harcourt records.
3. **Oral history** — structured interviews with **elders, traditional rulers' councils, and town/community unions**, recorded with informant, place, and date.

### References & further reading (starting points — verify before citing in content)
- **Ethnologue** (ethnologue.com) and **Glottolog** (glottolog.org) — language classification entry for Ikwere.
- **Kay Williamson** — foundational documentation of Rivers State languages (Igboid, Ijoid). *(bibliography to be pulled.)*
- **Nigerian National Archives** — primary colonial-era records.
- **Elechi Amadi** — novelist from Aluu (Ikwerre); works such as *The Concubine* render village cultural texture. **Fiction — cultural colour, not historical citation.**
- **WikiAfrica / Wikipedia** — cross-check and citation trails only, never a primary source.
- Design/UX references for the eventual visualizations: **Histropedia** (histropedia.com) and **Chronas** (chronas.org) — timeline- and map-based history exploration (already listed in the Educate-Org README).

> **On citations:** no reference in this platform's *content* should be asserted from memory. Each becomes a row in `sources` with a real, checkable citation, or it stays `unverified`. Fabricated citations are worse than none — they poison the reliability the platform exists to provide.

---

## 9. Validated final entity list (tiered)

The authoritative table list after validating the model against the first Ikwerre sample. **15 tables; the 7 Tier-1 tables alone deliver a working experience** (collect a group's entries, connect them, source them). Build order = tiers.

### Tier 1 — core (build first) · 7 tables
| table | key columns | purpose |
| --- | --- | --- |
| `places` | id, parent_id, name, `designation`, level_rank, iso_code, general_info | recursive geography |
| `peoples` | id, parent_id, name, `designation`, general_info | recursive people groups |
| `people_places` | people_id, place_id, `relationship` | a people spans many places |
| `entries` | id, `entry_type`, title, summary, body, significance, period_start, period_end, `date_precision`, is_approximate, period_note, era, **place_id**, **people_id**, is_endangered, is_written, is_restricted, `verification_status`, `workflow_status`, timestamps | the unit of knowledge (**replaces `history`**) |
| `entry_relationships` | from_entry_id, to_entry_id, `relation_type`, note | the entry↔entry graph |
| `sources` | id, `source_type`, author_or_informant, title, year, citation_or_url, reliability_tier, + oral: informant_name, role_standing, community, interview_date, location, language, consent_given | provenance records |
| `entry_sources` | entry_id, source_id, `confidence`, `verification_status`, reviewer, note | many-to-many; per-claim reliability |

*Belonging* to a group = `entries.people_id`; *connecting* entries = `entry_relationships`.

### Tier 2 — rich culture (add next) · 8 tables
| table | key columns | purpose |
| --- | --- | --- |
| `figures` | id, name, role, people_id, birth_note, death_note, is_restricted, general_info | persons (Eze, founders) |
| `figure_relationships` | from_figure_id, to_figure_id, `relation_type` | genealogy & succession |
| `entry_figures` | entry_id, figure_id, `role` | link entries to the people in them |
| `languages` | id, name, iso_code, classification, endangerment_status, people_id | first-class language |
| `dialects` | id, language_id, name, region_note | dialect variation |
| `lexicon` | id, language_id, dialect_id, word, pronunciation, meaning, example_sentence, audio_media_id | words + audio |
| `media` | id, media_type, url, caption, source_id, is_restricted | images/audio/video/maps |
| `entry_media` | entry_id, media_id, caption | attach media to entries |

*`figures` and `lexicon` get their own lightweight `figure_sources` / `lexicon_sources` joins when sourced — same pattern as `entry_sources`.*
