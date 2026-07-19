# History Data Model — Conceptual Spec

**Status:** Draft / proposal. Vocabulary and relationships only — **no SQL yet**. The goal is to lock the conceptual model and its controlled vocabularies first, gather a representative sample of real Ikwerre data against it, let that sample refine the model, and *then* write migrations.

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

### `entries` — typed units of knowledge
| column | notes |
| --- | --- |
| `id` | |
| `entry_type` | controlled enum, doubles as the "best-aspect" lenses — §5 |
| `title` | |
| `summary` | short, teachable |
| `body` | rich text / jsonb |
| `significance` | why it matters |
| `is_endangered` | boolean (dying craft, fading oral lit) |
| `is_written` | boolean (written vs oral transmission) |
| `status` | `draft` \| `in_review` \| `published` |
| **time** | `period_start`, `period_end`, `date_precision` (`year`\|`decade`\|`century`\|`era`\|`relative`), `is_approximate`, `era` (§5) |
| **links** | `place_id?`, `people_id?` — an entry attaches to a place, a people, or both |

Fuzzy time flags let "founded ~4 generations ago" sit honestly on a timeline.

### `entry_relationships` — the connective tissue (the key table)
| column | notes |
| --- | --- |
| `from_entry` | |
| `to_entry` | |
| `relation_type` | §5 — caused, followed_by, part_of, founded_by … |

This is what powers the timeline's connection threads and the knowledge-graph view. Example chain:
> `migration` **caused** `settlement_founding` **founded_by** `figure(Eze …)` **who instituted** `festival` **that commemorates** the `migration`.

### `figures` — people as first-class
Persons (Eze, founder, warrior, priest, hero): `name`, `role`, lifespan, `people_id`, plus self-referencing `figure_relationships` (`parent_of`, `succeeded_by`, `married`) for genealogies and succession lines.

### Language (first-class — the Ikwerre language is under pressure)
`languages`, `dialects`, `lexicon` (word, meaning, **audio**, example sentence), orthography notes, endangerment status.

### `media`
`type` (`image`\|`audio`\|`video`\|`map`\|`document`), `url`, `caption`, `source_id`, linked to entries / figures / places.

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

**`entry_type`** (starter — the cultural lenses): origin_tradition, migration, settlement_founding, institution, title, deity_spirit, shrine_site, cosmology, festival, rite_of_passage, masquerade, proverb, folktale, praise_name, naming_custom, craft, architecture, attire, cuisine, music_dance, economy, agriculture, trade_route, marriage_custom, lineage, figure, event, conflict, alliance, colonial_encounter, modern_identity, diaspora.

**`relation_type`**: caused, followed_by, part_of, founded_by, practiced_by, located_at, derived_from, commemorates, succeeded_by, in_conflict_with, related_to.

**`era`**: pre-colonial, colonial, post-independence, contemporary.

---

## 6. Migration path from the current schema

This is an **evolution, not a teardown**:

1. Create `places` + `peoples` + `people_places`; backfill from `continents`/`countries`/`states`/`local_governments` (→ `places`) and `ethnic_groups`/`tribes` (→ `peoples`), assigning each a `designation`.
2. Add `entry_type`, time columns, `status` to `history`; rename/reshape toward `entries`. Backfill the ~6 existing rows.
3. Add `entry_relationships`, `sources`, `entry_sources`, `figures`, `media`, `languages`/`lexicon` as new tables.
4. Repoint `history`'s four geo/ethnic FKs to the two generic `place_id` / `people_id` links.
5. Preserve RLS on every new table.

Migrations land in the **backend repo**; the live Supabase project keeps running throughout (schema changes are additive first, destructive last).

---

## 7. Open decisions

1. **`designation` strictness** — enum vs **lookup table (recommended)** vs free text.
2. **Table naming** — `places` / `peoples` acceptable? 
3. **Contested-history presentation** — **multi-perspective + sourced (recommended)** vs a single curated canonical view. Note: the Educate-Org README frames the group as the "Ikwere-Igbo ethnic group," which is *itself* a position on a contested question (see §8). The provenance model lets us hold that framing while still showing the evidence.
4. **First build focus** — (a) schema spec + migration, (b) Ikwerre data-gathering plan, (c) timeline/graph prototype.

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
