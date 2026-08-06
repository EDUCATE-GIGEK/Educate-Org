# Researcher — user stories

**Persona** Researcher · **Intent** `research` · **ID prefix** `RES`

Someone following a specific question who needs to see the sources and the
disagreements. Format and field definitions live in
[`_docs/user-stories.md`](../_docs/user-stories.md).

**Numbering:** `RES-03` is the worked example in the format spec and is
reproduced below unchanged, so this file holds the whole persona. The ten new
stories are `RES-01`, `RES-02`, and `RES-04`–`RES-12`.

**State of play:** every story here is Tier 1 or Tier 2 data that already exists
in the SQL — and **nothing in the frontend reads `sources` or `entry_sources`
today.** The reader sees `entries.verification_status` as a badge and nothing
behind it. That is the gap this persona is about.

---

### RES-01 — See every source behind a claim

**As a** researcher
**I want** the full list of sources attached to an entry, with what each one says
about it
**So that** I can decide whether to believe the claim on the evidence, rather
than on a badge someone else assigned

|                     |                                                                                                                                                                     |
| ------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Priority**        | Must                                                                                                                                                                |
| **Tier**            | 1                                                                                                                                                                   |
| **Backing tables**  | [`entries`](../uml_diagrams/structural/02-Entries-Knowledge-Graph.md), [`entry_sources`, `sources`](../uml_diagrams/structural/03-Provenance.md)                    |
| **Restricted data** | Yes — sources with `is_restricted = true` are excluded server-side and are not counted in any total shown; the same applies to sources of a restricted entry         |

**Acceptance criteria**

1. **Given** an entry has three rows in `entry_sources` **when** I open its
   provenance **then** all three are listed, each with its `source_type`,
   `stance`, and `confidence`.
2. **Given** a source has `stance = 'contradicts'` and `confidence = 'high'`
   **when** it is listed **then** both are shown separately — a source that
   strongly contradicts the claim must not be rendered as weak support.
3. **Given** an entry has no rows in `entry_sources` at all **when** I open its
   provenance **then** it says so in words, rather than showing an empty panel
   that reads as "nothing to worry about".
4. **Given** one of the entry's sources is `is_restricted = true` **when** the
   page is served **then** that source is absent from the response and the
   visible source count matches what is actually listed.
5. **Given** a source is an oral testimony with `consent_given = false` **when**
   the entry is served **then** neither the source nor any quotation of it
   appears.

**Notes** — `entry_sources` is a join with a composite PK, so one source can back
many entries and one entry can cite many sources; the list is per-entry, not
per-source.

---

### RES-02 — Tell triangulation from repetition

**As a** researcher
**I want** to see the *mix of source types* behind a verified entry
**So that** I can tell genuine corroboration from two web pages copying each
other, which the word "verified" alone does not distinguish

|                     |                                                                                                                                                  |
| ------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Priority**        | Must                                                                                                                                             |
| **Tier**            | 1                                                                                                                                                |
| **Backing tables**  | [`entries`](../uml_diagrams/structural/02-Entries-Knowledge-Graph.md), [`entry_sources`, `sources`](../uml_diagrams/structural/03-Provenance.md) |
| **Restricted data** | Yes — excluded sources are excluded from the type mix too, so the summary never describes evidence the reader cannot see                          |

**Acceptance criteria**

1. **Given** an entry marked `verified` whose supporting sources are all
   `source_type = 'web'` **when** I view its provenance **then** the single type
   is visible, so I can see the agreement is not independent.
2. **Given** an entry supported by one `book` and one `oral_tradition` **when**
   I view its provenance **then** the two distinct types are shown as such —
   this is the condition the model calls triangulation.
3. **Given** an entry marked `verified` with exactly one supporting source
   **when** I view it **then** the mismatch between the verdict and the evidence
   is visible to me without opening every source.
4. **Given** `verification_status` is an editorial judgement written onto
   `entries` **when** it is displayed **then** it is presented as a verdict with
   its `updated_at`, not as a number the database computed.

**Notes** — the rollup is deliberately not trigger-computed, so a `verified`
label can lag its `entry_sources` rows. Whether the UI should actively warn on
that mismatch is the open question inherited from RES-03.

---

### RES-03 — See where a disputed account splits

> Reproduced from the worked example in
> [`_docs/user-stories.md`](../_docs/user-stories.md). Unchanged.

**As a** researcher
**I want** to see both sides of a contested claim with each side's sources
**So that** I can judge the disagreement myself instead of trusting a single
curated version

|                     |                                                                                                                                                                         |
| ------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Priority**        | Must                                                                                                                                                                    |
| **Tier**            | 1                                                                                                                                                                       |
| **Backing tables**  | [`entries`](../uml_diagrams/structural/02-Entries-Knowledge-Graph.md), `entry_relationships`, [`sources`, `entry_sources`](../uml_diagrams/structural/03-Provenance.md) |
| **Restricted data** | Yes — restricted entries and sources are excluded from the public view entirely, including from the disputed-pair count                                                 |

**Acceptance criteria**

1. **Given** an entry with `verification_status = 'disputed'` **when** I open it
   **then** I see it labelled as disputed rather than presented as settled fact.
2. **Given** that entry is linked to another by a `contradicts` relationship
   **when** I view the dispute **then** both entries are shown side by side,
   neither marked as the correct one.
3. **Given** each entry has rows in `entry_sources` **when** I view either side
   **then** each source is listed with its `stance` and `confidence`, and its
   `source_type` is visible so I can see whether the agreement is genuine
   triangulation or two sources of the same kind.
4. **Given** a source is an oral testimony **when** it is displayed **then** the
   informant's standing and the interview date are shown, and nothing is shown
   at all if `consent_given` is false.
5. **Given** either entry has `is_restricted = true` **when** the page is
   rendered for a public reader **then** that entry is absent from the response,
   not hidden in the client.

**Notes** — `verification_status` is an editorial judgement written onto
`entries`, not a trigger-computed rollup; a `disputed` label can therefore lag
its `entry_sources` rows. Whether the UI should warn on that mismatch is open.

---

### RES-04 — Follow a chain of causation across entries

**As a** researcher
**I want** to trace an entry backwards and forwards through its relationships
**So that** I can reconstruct how the record says one thing led to another,
instead of assembling that chain by hand from separate pages

|                     |                                                                                              |
| ------------------- | ---------------------------------------------------------------------------------------------- |
| **Priority**        | Should                                                                                       |
| **Tier**            | 1                                                                                            |
| **Backing tables**  | [`entries`, `entry_relationships`](../uml_diagrams/structural/02-Entries-Knowledge-Graph.md) |
| **Restricted data** | Yes — a chain never routes through a restricted entry; the edge is dropped with the row       |

**Acceptance criteria**

1. **Given** entries linked `A --caused--> B --commemorates--> C` **when** I open
   A **then** I can walk to B and on to C without losing the direction of each
   link.
2. **Given** an entry is the `to_entry_id` of a relation **when** I open it
   **then** the relation appears as incoming, because the read path queries both
   columns rather than only `from_entry_id`.
3. **Given** a relation carries a `note` **when** the link is shown **then** the
   note is shown with it — it is where the reason for the link is recorded.
4. **Given** a `contradicts` edge sits between two entries **when** a causal
   chain is traced **then** it is not walked as if it were a causal step; a
   dispute is not a stage in a sequence.
5. **Given** a relationship involves a person rather than another entry **when**
   I look for it **then** it is not in this chain — those live in `entry_figures`
   and `figure_relationships` ([RES-07](#res-07--follow-a-person-through-the-record)).

**Notes** — `entry_relationships` is directional and unique on
`(from, to, relation_type)`, so the same pair can carry more than one kind of
link.

---

### RES-05 — Find where the evidence is thin for a whole group

**As a** researcher
**I want** to filter a group's record by how well established each entry is
**So that** I can see at a glance which parts of this history still need work,
which is usually the question I arrived with

|                     |                                                                        |
| ------------------- | ---------------------------------------------------------------------- |
| **Priority**        | Should                                                                 |
| **Tier**            | 1                                                                      |
| **Backing tables**  | [`entries`](../uml_diagrams/structural/02-Entries-Knowledge-Graph.md) |
| **Restricted data** | Yes — the counts come from the filtered read, so restricted entries never appear in a total |

**Acceptance criteria**

1. **Given** a group's readable entries **when** I filter by
   `verification_status` **then** I get separate views of `verified`,
   `unverified`, and `disputed`, with a count for each.
2. **Given** the filter is offered **when** I read its labels **then** nothing
   in it implies `unverified` means unpublished — `verification_status` and
   `workflow_status` are shown as answers to different questions.
3. **Given** an entry is `verified` but still `workflow_status = 'draft'`
   **when** I browse as a public reader **then** it does not appear at all,
   under either filter.
4. **Given** the group has descendant groups **when** the counts are computed
   **then** they cover the whole subtree, and the view says whether a count
   includes descendants.

**Notes** — conflating the two status columns is the single most common
misreading of this model; the filter labels are where that misreading would
enter the product.

---

### RES-06 — Gather a group's whole record, clans included

**As a** researcher
**I want** everything held about a group *and* its sub-groups in one place
**So that** I am not silently missing the material filed one level down, which
is where a lot of clan-specific history lives

|                     |                                                                                                                                       |
| ------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| **Priority**        | Must                                                                                                                                  |
| **Tier**            | 1                                                                                                                                     |
| **Backing tables**  | [`entries`](../uml_diagrams/structural/02-Entries-Knowledge-Graph.md), [`peoples`, `places`](../uml_diagrams/structural/01-Places-And-Peoples.md) |
| **Restricted data** | Yes — the recursive read applies the same restriction filter at every level of the tree                                                |

**Acceptance criteria**

1. **Given** an entry whose `people_id` is a clan three levels below the ethnic
   group **when** I open the ethnic group's record **then** that entry is
   included, gathered by descending the tree rather than by a fixed join.
2. **Given** entries inherited from descendants **when** they are shown **then**
   they are distinguishable from entries whose `people_id` is the node itself.
3. **Given** a group with no descendants **when** I open it **then** the same
   read returns only its own entries and shows no empty "sub-groups" section.
4. **Given** an entry carries a `place_id` but no `people_id` **when** I gather
   by place **then** it is included there, because the two belonging links are
   independent and either may be null.

**Notes** — this is a `WITH RECURSIVE` traversal, not `WHERE people_id = ?`;
"state" and "LGA" are labels on nodes, not levels the query can count on.

---

### RES-07 — Follow a person through the record

**As a** researcher
**I want** to see one figure's appearances, kin, and succession together
**So that** I can test a genealogy against what each entry actually claims,
instead of inferring a lineage from scattered mentions

|                     |                                                                                                                                                     |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Priority**        | Should                                                                                                                                              |
| **Tier**            | 2                                                                                                                                                   |
| **Backing tables**  | [`figures`, `figure_relationships`, `entry_figures`](../uml_diagrams/structural/04-Figures.md), [`figure_sources`](../uml_diagrams/structural/03-Provenance.md) |
| **Restricted data** | Yes — `figures.is_restricted` is filtered server-side, and a restricted figure's kin edges disappear with it                                         |

**Acceptance criteria**

1. **Given** a figure appears in several entries **when** I open them **then**
   each appearance names its `role` — `founded_by`, `led_by`, `about`,
   `mentions`, `attributed_to` — rather than listing them all as "related".
2. **Given** a `parent_of` edge between two figures **when** it is displayed
   **then** the direction is unambiguous, and reciprocal relations (`married`,
   `sibling_of`) appear on both figures even though they are stored one way.
3. **Given** two competing genealogies for the same figure **when** they are
   displayed **then** both are shown as separately sourced records; they are not
   merged into one lineage.
4. **Given** a figure's `birth_note` reads "three generations before Akalaka"
   **when** it is displayed **then** it is shown as written, since these are
   free-text notes and not dates.
5. **Given** a figure has `is_restricted = true` **when** the page is served
   **then** the figure is absent from the response entirely.

**Notes** — figures are their own entity precisely because `entries` has nowhere
to put a lifespan and `entry_relationships` has the wrong vocabulary for kinship.

---

### RES-08 — Judge an oral testimony as testimony

**As a** researcher
**I want** to see who gave an oral account, in what standing, where and when
**So that** I can weigh it as evidence the way I would weigh a citation, rather
than treating "oral tradition" as one undifferentiated category

|                     |                                                                            |
| ------------------- | ---------------------------------------------------------------------------- |
| **Priority**        | Must                                                                       |
| **Tier**            | 1                                                                          |
| **Backing tables**  | [`sources`, `entry_sources`](../uml_diagrams/structural/03-Provenance.md) |
| **Restricted data** | Yes — `consent_given = false` suppresses the source outright, and `is_restricted` sources are never served |

**Acceptance criteria**

1. **Given** a source with `source_type = 'oral_tradition'` or `'interview'`
   **when** it is displayed **then** `role_standing`, `community`,
   `interview_date`, `location`, and `language` are shown where present.
2. **Given** such a source has `consent_given = false` **when** the entry is
   served **then** the source is omitted server-side — this is an ethical gate,
   not a display preference.
3. **Given** a written source **when** it is displayed **then** the seven oral
   fields are absent rather than rendered as empty rows.
4. **Given** the interview was conducted in a language other than the display
   language **when** the source is shown **then** the `language` field is
   visible, because a translated testimony is a different piece of evidence.

**Notes** — consent is required before recording an informant; the flag records
a real agreement made in the field, and the read path is where that agreement is
either kept or broken.

---

### RES-09 — Take a citation away with me

**As a** researcher
**I want** to export the sources behind what I have read, in citable form
**So that** I can use this material in my own work and someone else can check it
without going back through the interface

|                     |                                                                                                                                                  |
| ------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Priority**        | Should                                                                                                                                           |
| **Tier**            | 1                                                                                                                                                |
| **Backing tables**  | [`entries`](../uml_diagrams/structural/02-Entries-Knowledge-Graph.md), [`sources`, `entry_sources`](../uml_diagrams/structural/03-Provenance.md) |
| **Restricted data** | Yes — restricted sources and non-consented testimony are absent from the export, exactly as they are from the page                                |

**Acceptance criteria**

1. **Given** an entry with sources **when** I export its citations **then** each
   carries `author_or_informant`, `title`, `year`, and `citation_or_url` as
   held — no field is filled in from anywhere else.
2. **Given** a source row has an empty `citation_or_url` **when** it is exported
   **then** the gap is stated as a gap; nothing plausible is generated to fill
   it.
3. **Given** an oral source **when** it is exported **then** it is cited as oral
   testimony with informant standing, community, and interview date, subject to
   the same consent gate as [RES-08](#res-08--judge-an-oral-testimony-as-testimony).
4. **Given** a `disputed` entry **when** I export it **then** the export carries
   both sides' sources, not just the ones supporting the version I happened to
   read first.
5. **Given** an entry with no sources at all **when** I export it **then** the
   export says so.

**Notes** — "a fabricated citation is worse than none" is the project's stated
position; this story is where an interface would otherwise break it by being
helpful.

---

### RES-10 — See a language's classification as a claim, not a fact

**As a** researcher
**I want** a language's family classification presented as a sourced,
contestable position
**So that** I can see the evidence behind a classification that is itself part
of the dispute I am researching

|                     |                                                                                                                                                     |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Priority**        | Should                                                                                                                                              |
| **Tier**            | 2                                                                                                                                                   |
| **Backing tables**  | [`languages`, `dialects`, `lexicon`](../uml_diagrams/structural/05-Language-Lexicon-Media.md), [`lexicon_sources`, `sources`](../uml_diagrams/structural/03-Provenance.md) |
| **Restricted data** | Yes — restricted sources behind any cited claim are excluded server-side                                                                            |

**Acceptance criteria**

1. **Given** a language row with `classification = 'Igboid'` **when** it is
   displayed **then** the classification is attributed, not asserted as a
   property of the language.
2. **Given** the classification is contested **when** it is displayed **then**
   the competing position is shown with its own sources, on the same footing.
3. **Given** `people_id` is null **when** the language is displayed **then** it
   is not filed under a group anyway — the nullability is there because the
   mapping is exactly what is in question.
4. **Given** an `endangerment_status` **when** it is displayed **then** it uses
   the UNESCO wording, so it can be compared against the Atlas rather than read
   as a local judgement.

**Notes** — **modelling gap:** Tier 2 has `figure_sources` and `lexicon_sources`
but **no `language_sources` join**, so `languages.classification` has nowhere to
record its evidence. Two options: add the join, or model the contested
classification as an `entries` row (`modern_identity`) with a `contradicts` link
and let `entry_sources` carry it. This story cannot ship until that is decided.

---

### RES-11 — Find the claims resting on a single source

**As a** researcher
**I want** to list the entries backed by only one source, or none
**So that** I can target my own fieldwork at the parts of the record that most
need a second, independent account

|                     |                                                                                                                                                  |
| ------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Priority**        | Should                                                                                                                                           |
| **Tier**            | 1                                                                                                                                                |
| **Backing tables**  | [`entries`](../uml_diagrams/structural/02-Entries-Knowledge-Graph.md), [`entry_sources`, `sources`](../uml_diagrams/structural/03-Provenance.md) |
| **Restricted data** | Yes — restricted sources are excluded before counting, so a thin entry is never made to look well-sourced by evidence the reader cannot see       |

**Acceptance criteria**

1. **Given** a group's entries **when** I open this view **then** entries with
   exactly one visible source are listed, most-load-bearing first.
2. **Given** entries with no sources at all **when** the view is drawn **then**
   they are listed as their own case, not folded in with the single-source ones.
3. **Given** an entry cited by three sources that are all `source_type = 'book'`
   **when** independence is assessed **then** it counts as one source *type*,
   because that is the standard the model sets for `verified`.
4. **Given** an entry has a restricted source **when** the counts are shown
   **then** the number matches the sources actually listed for that entry.

**Notes** — the counterpart to [RES-02](#res-02--tell-triangulation-from-repetition):
that one asks whether a verdict is earned, this one asks where no verdict can be
reached yet.

---

### RES-12 — Compare what two groups hold about the same thing

**As a** researcher
**I want** to put two groups' entries on the same subject side by side
**So that** I can see whether neighbouring communities tell the same story,
which a single group's page can never show me

|                     |                                                                                                                                     |
| ------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| **Priority**        | Could                                                                                                                               |
| **Tier**            | 1                                                                                                                                   |
| **Backing tables**  | [`entries`, `entry_relationships`](../uml_diagrams/structural/02-Entries-Knowledge-Graph.md), [`peoples`](../uml_diagrams/structural/01-Places-And-Peoples.md) |
| **Restricted data** | Yes — restricted entries are absent from both columns and from any count of what the groups share                                    |

**Acceptance criteria**

1. **Given** two groups each with `origin_tradition` entries **when** I compare
   them **then** both sets are shown, filtered to the same `entry_type`, with
   neither presented as the reference version.
2. **Given** entries in the two sets are linked by `contradicts` **when** the
   comparison is drawn **then** the link is shown, because a modelled
   disagreement is the most informative thing on the page.
3. **Given** entries in the two sets are linked by `derived_from` **when** the
   comparison is drawn **then** that is distinguished from independent
   agreement — a borrowed account is not a second witness.
4. **Given** one group's entries are undated **when** the comparison is drawn
   **then** it still renders; alignment falls back to type and relation rather
   than to a time axis.

**Notes** — nothing in the schema links "the same subject" across groups except
`entry_relationships`, so this view is only as good as the curation of those
edges. Worth stating before anyone reads an empty comparison as an absence of
disagreement.

---

## Blank template

```markdown
### <ID> — <title>

**As a** <persona>
**I want** <capability>
**So that** <user outcome>

|                     |     |
| ------------------- | --- |
| **Priority**        |     |
| **Tier**            |     |
| **Backing tables**  |     |
| **Restricted data** |     |

**Acceptance criteria**

1. **Given** … **when** … **then** …
2. **Given** … **when** … **then** …

**Notes** —
```
