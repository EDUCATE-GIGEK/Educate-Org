# User Stories — format and conventions

The template every story in this project follows, and the checks a story is not
finished without. Stories live in `uml_diagrams/use_case/<persona>.md`, one file
per persona; this file defines their shape.

---

## The four personas

Three come straight from the onboarding `intent` field
(`teach` | `research` | `explore`) — see `frontend/_docs/onboarding.md`. The
fourth has no intent value because they don't use the app the same way.

| ID prefix | Persona     | Intent value | What they are doing                                                      |
| --------- | ----------- | ------------ | ------------------------------------------------------------------------ |
| `EXP`     | Explorer    | `explore`    | Encountering a culture for the first time; browsing without a goal       |
| `RES`     | Researcher  | `research`   | Following a specific question; needs to see sources and disagreements    |
| `TCH`     | Teacher     | `teach`      | Turning what exists into something teachable at a stated education level |
| `CON`     | Contributor | —            | Collecting and sourcing new knowledge: interviews, lexicon, media        |

**The Contributor is not optional.** `data/collection/` (entry, source and
lexicon templates plus an interview guide) describes a real workflow that no
current use-case file covers, and every table in Tier 1 assumes someone fills
it. A platform with three read personas and no write persona has no content.

---

## The format

```markdown
### <ID> — <short title>

**As a** <persona>
**I want** <capability>
**So that** <the reason — a user outcome, never a feature restatement>

|                     |                                                               |
| ------------------- | ------------------------------------------------------------- |
| **Priority**        | Must / Should / Could                                         |
| **Tier**            | 1 / 2 / App / **Blocked**                                     |
| **Backing tables**  | `table`, `table`                                              |
| **Restricted data** | Yes — how it is handled / No — cannot surface restricted rows |

**Acceptance criteria**

1. **Given** <starting state> **when** <action> **then** <observable result>.
2. **Given** … **when** … **then** …

**Notes** — open questions, dependencies, anything a reader would otherwise
assume is an oversight.
```

---

## The fields, and why each one exists

### Priority

Answers exactly one question: **does v1 ship without this?** The yardstick is
the v1 goal — _an interactive learning platform to preserve information about
the Nigerian Ikwerre-Igbo ethnic group_ — not how good an idea something is.

| Level      | The test                                                                                                          | If you're wrong                       |
| ---------- | ----------------------------------------------------------------------------------------------------------------- | ------------------------------------- |
| **Must**   | v1 is not shippable without it. Cutting it means the product doesn't function, or it breaks a promise we've made. | We ship something broken or dishonest |
| **Should** | v1 is materially worse without it, but still shippable and still honest.                                          | Users notice and are annoyed          |
| **Could**  | Genuinely nice. Cut it the moment the schedule tightens.                                                          | Nobody notices                        |

### Tier

The single most useful field in this project, because **the backend is in two
halves and one of them is not built yet.**

| Value         | Meaning                                                                 |
| ------------- | ----------------------------------------------------------------------- |
| `1`           | Needs only Tier 1 tables — buildable against `history_model_tier1.sql`  |
| `2`           | Needs Tier 2 (figures, languages, lexicon, media)                       |
| `App`         | Needs only the live Supabase tables (`user`, `manuscripts`, `notes`, …) |
| **`Blocked`** | Needs Tier 1/2 **and** app-layer data in the same query                 |

A `Blocked` story is not a bad story — it is a story that cannot ship until the
UUID/`bigint` split in `00-Overview.md#known-gaps` is resolved. Marking it early
is the point. Anything joining `manuscripts` to `entries` is `Blocked` today.

### Backing tables

Name the actual tables, linked to the structural diagrams. This is what stops a
story from quietly assuming a column that does not exist — the failure mode that
produced five obsolete diagrams. If you cannot name the tables, the story is not
ready.

### Restricted data

Every story that displays content must answer this. `is_restricted` exists on
`entries`, `sources`, `media`, and `figures`, and marks knowledge that was
**collected but must never be publicly displayed**. "No" is a fine answer; a
blank is not.

### So that

Must state a user outcome. "So that I can see a timeline" is a feature
restatement and tells you nothing. "So that I can tell whether two accounts of
the founding actually disagree" is a reason — and it implies a design.

---

## Worked example

### RES-03 — See where a disputed account splits

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

## Conventions

**One capability per story.** If the acceptance criteria split cleanly into two
groups that could ship in different weeks, it is two stories.

**Acceptance criteria are Given/When/Then, and observable.** "The timeline works
correctly" is not testable. Write what a person can see.

**Criteria state the data condition, not the UI mechanism.** "Given the entry has
`date_precision = 'relative'`" — not "given the user clicks the tooltip". The UI
will change; the data condition is what the story is actually about.

**Number stories within a persona and never renumber.** `RES-03` stays `RES-03`
after `RES-02` is deleted. Numbers appear in commits and branch names.

**Priority is a decision, not an estimate.** See
[Priority](#priority) — it is a claim about what v1 cannot ship without, and
nothing else.

---

## Cross-cutting checks

Before a story is marked ready, walk these. They exist because each one has
already caused a modelling correction in this project.

1. **Restricted knowledge** — can this surface a row with `is_restricted = true`?
   Filtering must happen server-side.
2. **Consent** — does this display an oral source? `consent_given` gates it.
3. **Fuzzy time** — does this assume a real date? Most entries have none.
   `period_start` may be null with `period_note = "≈4 generations before
present"`. A timeline that cannot render that is not finished.
4. **Contested accounts** — does this pick one version of something? If a
   `contradicts` link can exist between two entries, the story must say what the
   UI does with it. Silently choosing one is the failure mode the whole model
   exists to prevent.
5. **The two status columns** — is this about `verification_status` (is it
   trusted?) or `workflow_status` (is it published?)? Stories routinely conflate
   them. A `published` entry may be `disputed`.
6. **Tree depth** — does this assume a fixed number of levels? `places` and
   `peoples` nest arbitrarily; "state" and "LGA" are labels, not levels. Anything
   gathering a group's full set needs a recursive CTE over descendants.
7. **The database split** — does this join app-layer data to history-model data?
   If so it is `Blocked`, and the story should say so rather than describe a
   query that cannot be written.
