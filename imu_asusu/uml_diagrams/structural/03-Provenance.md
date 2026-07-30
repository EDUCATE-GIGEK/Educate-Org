# Provenance

**Source:** `history_model_tier1.sql` §7–8, `history_model_tier2.sql` §9–10

```mermaid
classDiagram
    class sources {
        +UUID id PK
        +TEXT source_type
        +TEXT author_or_informant
        +TEXT title
        +INTEGER year
        +TEXT citation_or_url
        +TEXT reliability_tier
        +TEXT informant_name
        +TEXT role_standing
        +TEXT community
        +DATE interview_date
        +TEXT location
        +TEXT language
        +BOOLEAN consent_given
        +BOOLEAN is_restricted
        +TIMESTAMPTZ created_at
    }

    class entry_sources {
        +UUID entry_id PK_FK
        +UUID source_id PK_FK
        +TEXT stance
        +TEXT confidence
        +TEXT reviewer
        +TEXT note
    }

    class figure_sources {
        +UUID figure_id PK_FK
        +UUID source_id PK_FK
        +TEXT stance
        +TEXT confidence
        +TEXT reviewer
        +TEXT note
    }

    class lexicon_sources {
        +UUID lexicon_id PK_FK
        +UUID source_id PK_FK
        +TEXT stance
        +TEXT confidence
        +TEXT reviewer
        +TEXT note
    }

    class entries {
        +UUID id PK
        +TEXT verification_status
    }
    class figures {
        +UUID id PK
    }
    class lexicon {
        +UUID id PK
    }
    class media {
        +UUID id PK
        +UUID source_id FK
    }

    entries  "1" --> "*" entry_sources   : entry_id
    sources  "1" --> "*" entry_sources   : source_id
    figures  "1" --> "*" figure_sources  : figure_id
    sources  "1" --> "*" figure_sources  : source_id
    lexicon  "1" --> "*" lexicon_sources : lexicon_id
    sources  "1" --> "*" lexicon_sources : source_id
    sources  "0..1" --> "*" media        : source_id

    note for sources "source_type: oral_tradition | book | journal | archival | interview | museum | web<br/>Oral rows carry informant_name .. consent_given; NULL for written sources"
    note for entry_sources "stance: supports | contradicts | mentions<br/>confidence: low | medium | high"
```

## Description

The reliability backbone, and the part of the model that is genuinely unusual.
The old schema had **no provenance at all** — this is the largest single
addition the history model makes.

Because this history is under-documented in written records, *method matters
more than any single fact*. Nothing enters without a source.

### One shape, four attachment points

`sources` is a single table; four things attach to it via near-identical joins:

| Join | Attaches to |
| --- | --- |
| `entry_sources` | a claim in `entries` |
| `figure_sources` | a person in `figures` |
| `lexicon_sources` | a word in `lexicon` |
| `media.source_id` | direct FK — provenance of the asset itself (photographer, archive) |

The three `*_sources` joins carry an identical column set — `stance`,
`confidence`, `reviewer`, `note` — on purpose. One pattern, one set of API
shapes, one review UI.

### Stance and confidence answer different questions

- `stance` — *what does this source say about the claim?*
  `supports` / `contradicts` / `mentions`
- `confidence` — *how strongly does it back it?* `low` / `medium` / `high`

A source can `contradict` a claim with `high` confidence. Collapsing these into
one score would lose exactly the case the platform exists to handle.

### The rollup

`entries.verification_status` is the aggregate shown in the UI; the join records
what **each** source says. So:

- **`verified`** — independent source *types* agree (a scholarly text *and* an
  elder interview). Agreement between two web pages is not triangulation.
- **`disputed`** — the entry has both `supports` and `contradicts` rows. It is
  a state to display, not a problem to resolve.
- **`unverified`** — the default. Everything starts here.

Note the rollup is **not** computed by a trigger; it is an editorial judgement
written onto `entries`. The join table is the evidence, the column is the
verdict.

### Oral sources carry more

Rows with `source_type` in (`oral_tradition`, `interview`) use seven extra
columns that are `NULL` for written sources: `informant_name`, `role_standing`,
`community`, `interview_date`, `location`, `language`, `consent_given`.

**`consent_given` is an ethical gate, not metadata.** Consent is required before
recording an informant — see `data/collection/interview-guide.md`. Combined with
`is_restricted`, this is how the platform holds knowledge an elder shared but
asked not to be published.

### Why this matters beyond the database

This model is what a trustworthy AI fact-checker grounds against — the
`manuscript-fact-check` edge function has something real to check *against*
only because provenance is first-class. From the spec: *fabricated citations
are worse than none — they poison the reliability the platform exists to
provide.*
