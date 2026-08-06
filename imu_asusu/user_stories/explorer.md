# Explorer — user stories

**Persona** Explorer · **Intent** `explore` · **ID prefix** `EXP`

Someone encountering the culture for the first time, browsing without a goal.
Usually signed out. Format and field definitions live in
[`_docs/user-stories.md`](../_docs/user-stories.md).

**On the `Tier` column:** two stories below read Tier 1 tables *and* app-layer
tables, but never in the same query — the region ids travel through the client,
not through a join. They are marked `1 + App` rather than `Blocked`, because
`Blocked` means a query that cannot be written. See the note on each.

---

### EXP-01 — Choose the places and peoples I follow

**As an** explorer
**I want** to pick the places and people groups I care about and have that
choice survive the session
**So that** the app opens on something I actually want to read instead of a
generic front page I have to navigate out of every time

|                     |                                                                                                                                                                                     |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Priority**        | Must                                                                                                                                                                                |
| **Tier**            | 1 + App                                                                                                                                                                             |
| **Backing tables**  | [`places`, `peoples`, `designations`](../uml_diagrams/structural/01-Places-And-Peoples.md), [`user_preferences`, `user`](../uml_diagrams/structural/06-User-Account.md)             |
| **Restricted data** | No — the picker lists nodes in the two trees, and neither `places` nor `peoples` carries `is_restricted`                                                                            |

**Acceptance criteria**

1. **Given** I have no saved regions in either store **when** I open the app
   **then** I reach the region picker, and the decision waits for the stored
   preferences to resolve rather than being made from an empty local mirror.
2. **Given** the `peoples` tree holds a node at any depth **when** I search for
   it **then** it is offered with its `designation` label, so I can tell an
   Ethnic Group from a Clan of the same name before I pick one.
3. **Given** I chose regions while signed out and my account has none **when** I
   sign in **then** the local choices are adopted onto the account rather than
   discarded.
4. **Given** my account already holds regions **when** I sign in on a second
   device **then** the account's regions win and I am not sent back through
   onboarding I have already completed.
5. **Given** I sign out on a shared machine **when** the next person opens the
   app **then** none of my regions are visible to them.

**Notes** — preferences are one JSONB blob on `user_preferences`, mirrored into
`localStorage` so reads stay synchronous; a region is `{ kind, id, name }` where
`id` is a Tier-1 UUID held as text. Nothing joins the two halves in a single
query, so this is not `Blocked`.

---

### EXP-02 — Be pointed only at groups that have something to read

**As an** explorer
**I want** the suggestions and the timeline gallery to lead me to groups whose
records are actually readable
**So that** my first click doesn't land on an empty page and teach me the
platform has nothing in it

|                     |                                                                                                                                                                    |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Priority**        | Should                                                                                                                                                             |
| **Tier**            | 1 + App                                                                                                                                                            |
| **Backing tables**  | [`entries`](../uml_diagrams/structural/02-Entries-Knowledge-Graph.md), [`peoples`](../uml_diagrams/structural/01-Places-And-Peoples.md), `user_preferences`        |
| **Restricted data** | Yes — the counts are taken from the same filtered read the pages use, so a restricted or unpublished entry raises no count                                         |

**Acceptance criteria**

1. **Given** a people group whose entries are all `workflow_status = 'draft'` or
   `'in_review'` **when** the gallery is built **then** that group is absent,
   because its visible count is zero.
2. **Given** a group whose only entries are `is_restricted = true` **when** the
   gallery is built **then** it is likewise absent — the count comes from the
   filtered read, not from a raw row count.
3. **Given** I have saved regions **when** I open the gallery **then** my saved
   groups sort first, and every other readable group stays reachable below them.
4. **Given** I have saved nothing at all **when** I open the gallery **then** I
   see the list of readable groups, not an empty graph.
5. **Given** a group has entries but none of them are dated **when** its card is
   drawn **then** the card omits the year range instead of showing a fabricated
   one.

**Notes** — visible counts and period ranges are derived client-side from
filtered reads of `entries`. The filtering is what makes the suggestion honest;
if it ever moves to a raw `count(*)`, this story breaks silently.

---

### EXP-03 — Browse a group's history by kind of thing, not by date

**As an** explorer
**I want** to filter what a group has by the kind of history it is — festivals,
proverbs, crafts, origin traditions
**So that** I can follow what interests me without first knowing when any of it
happened, which is the one thing I don't know

|                     |                                                                          |
| ------------------- | ------------------------------------------------------------------------ |
| **Priority**        | Must                                                                     |
| **Tier**            | 1                                                                        |
| **Backing tables**  | [`entries`](../uml_diagrams/structural/02-Entries-Knowledge-Graph.md)   |
| **Restricted data** | Yes — restricted and unpublished entries are excluded before the facets are computed, so they cannot even be counted |

**Acceptance criteria**

1. **Given** a group's readable entries cover six of the thirty `entry_type`
   values **when** the facet chips are drawn **then** exactly those six appear —
   never the full vocabulary, and never a chip that filters to nothing.
2. **Given** I select the `festival` facet **when** the list re-renders **then**
   only `entry_type = 'festival'` entries remain, and the facets themselves are
   thematic — there is no era or date filter here.
3. **Given** the group has descendant groups in the `peoples` tree **when** I
   view its history **then** entries whose `people_id` is this node are shown
   first, and entries inherited from descendants are grouped separately, so a
   parent does not appear to repeat its children's records as its own.
4. **Given** I type a phrase into the search box **when** it matches an entry's
   title, summary, significance, or body **then** that entry stays visible under
   the current facet.
5. **Given** a group has no readable entries **when** I open its history
   **then** I am told there are no published entries yet for it, rather than
   shown a blank section that reads as a loading failure.

**Notes** — the facets are computed from the entries actually returned, which is
what keeps them in step with what the reader is allowed to see.

---

### EXP-04 — See where a group sits and where it lives

**As an** explorer
**I want** to see what a group is part of, what it contains, and which places it
is tied to
**So that** I can place an unfamiliar name in a structure instead of reading a
page about a word that means nothing to me

|                     |                                                                                                                                                          |
| ------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Priority**        | Must                                                                                                                                                     |
| **Tier**            | 1                                                                                                                                                        |
| **Backing tables**  | [`peoples`, `places`, `people_places`, `designations`](../uml_diagrams/structural/01-Places-And-Peoples.md)                                              |
| **Restricted data** | No — neither tree carries `is_restricted`                                                                                                                |

**Acceptance criteria**

1. **Given** a node's `designation_id` resolves to `Clan` **when** the page
   renders **then** the page calls it a Clan, taking the word from the
   `designations` lookup rather than from a hardcoded level in the UI.
2. **Given** a node has a `parent_id` **when** the page renders **then** a
   "part of" breadcrumb links to the parent, at whatever depth that parent sits.
3. **Given** a node has children **when** the page renders **then** they are
   listed with their own designation labels, and a node with no children shows
   no empty "contains" section.
4. **Given** a people group has four rows in `people_places` **when** I view it
   **then** all four places are shown, each labelled with its `relationship` —
   `homeland`, `diaspora`, or `historical` — rather than a single "location".
5. **Given** a place tree nests five levels deep in one branch and three in
   another **when** either is rendered **then** both work; nothing in the page
   assumes a fixed number of levels.

**Notes** — the two trees are deliberately not merged, which is why "where does
this group live" is a set of links with a relationship on each, not one field.

---

### EXP-05 — Not be told a date the record does not have

**As an** explorer
**I want** undated and approximately dated history shown as exactly that
**So that** I don't walk away believing a precise year for something the record
only places "about four generations ago"

|                     |                                                                        |
| ------------------- | ---------------------------------------------------------------------- |
| **Priority**        | Must                                                                   |
| **Tier**            | 1                                                                      |
| **Backing tables**  | [`entries`](../uml_diagrams/structural/02-Entries-Knowledge-Graph.md) |
| **Restricted data** | No                                                                     |

**Acceptance criteria**

1. **Given** an entry has `period_start = null` and `era = 'pre-colonial'`
   **when** it is displayed **then** the era label is shown and no year appears
   anywhere on it.
2. **Given** an entry has `is_approximate = true` **when** its period is shown
   **then** the year is prefixed to read as approximate, not as an exact date.
3. **Given** an entry has `date_precision = 'relative'` and a `period_note`
   **when** it is displayed **then** the note is shown as written — a relative
   date with its reference point stripped is worse than no date.
4. **Given** an entry has no `period_start`, no `period_end`, and no `era`
   **when** it is displayed **then** it is labelled undated and still appears in
   the list; it is not sorted as year 0 or dropped.
5. **Given** a `period_start` is negative **when** it is displayed **then** it
   reads as BCE rather than as a negative number.

**Notes** — this is cross-cutting check 3 in the format spec. Most of the first
Ikwerre sample is date-free oral tradition, so this is the normal case, not the
edge case.

---

### EXP-06 — Follow how one thing led to another

**As an** explorer
**I want** to see a group's history laid out with the links between its parts
drawn
**So that** I can follow a thread — a migration into a founding into the
festival that commemorates it — instead of reading disconnected facts

|                     |                                                                                            |
| ------------------- | ------------------------------------------------------------------------------------------ |
| **Priority**        | Must                                                                                       |
| **Tier**            | 1                                                                                          |
| **Backing tables**  | [`entries`, `entry_relationships`](../uml_diagrams/structural/02-Entries-Knowledge-Graph.md) |
| **Restricted data** | Yes — restricted entries are absent from the entry set, and any edge that touched one is therefore not drawn |

**Acceptance criteria**

1. **Given** an edge in `entry_relationships` whose other endpoint is not in the
   visible entry set **when** the graph is drawn **then** that edge is omitted
   rather than left dangling — including when the endpoint is hidden by
   filtering rather than absent.
2. **Given** two connected entries **when** I open either one **then** the
   relation is named (`caused`, `followed_by`, `commemorates`, …) and its
   direction is clear, since `from_entry_id → to_entry_id` reads left to right.
3. **Given** an entry is the *target* of a relation rather than its source
   **when** I open it **then** the link still appears, marked as incoming.
4. **Given** two entries are linked by `contradicts` **when** the graph is drawn
   **then** that link is visually distinct from a causal one and neither entry
   is presented as the correct account.
5. **Given** a group's entries are all undated **when** its timeline is opened
   **then** it still renders using relational or era ordering, rather than
   collapsing every entry onto one point.

**Notes** — a timeline is derived, not stored: it is the entries belonging to a
group plus the relationships running between them. There is no `timelines`
table, and this story must not imply one.

---

### EXP-07 — Know how much of what I am reading is checked

**As an** explorer
**I want** each piece of history to carry how well established it is
**So that** I can tell a corroborated account from something a single person
said once, without having to be a researcher to notice the difference

|                     |                                                                        |
| ------------------- | ---------------------------------------------------------------------- |
| **Priority**        | Must                                                                   |
| **Tier**            | 1                                                                      |
| **Backing tables**  | [`entries`](../uml_diagrams/structural/02-Entries-Knowledge-Graph.md) |
| **Restricted data** | No — the badge is drawn from rows already cleared for display          |

**Acceptance criteria**

1. **Given** an entry has `verification_status = 'unverified'` **when** it is
   displayed **then** it is labelled unverified — the default state is stated,
   not left blank for the reader to read as fine.
2. **Given** an entry has `verification_status = 'disputed'` **when** it is
   displayed **then** it is labelled disputed and still shown; a contested
   account is a state to display, not a reason to hide it.
3. **Given** an entry has `workflow_status = 'published'` and
   `verification_status = 'disputed'` **when** it is displayed **then** nothing
   in the interface implies that being published means being trusted.
4. **Given** an entry is marked `is_endangered` **when** it is displayed
   **then** that is shown as a separate signal from verification, because "at
   risk of being lost" and "not yet corroborated" are different claims.

**Notes** — the badge is the explorer's whole view of provenance. The sources
behind it are [RES-01](researcher.md); this story deliberately stops at the
label so the two can ship separately.

---

### EXP-08 — Hear how a word is actually said

**As an** explorer
**I want** to hear a recorded pronunciation alongside a word and its meaning
**So that** I learn the language as it sounds rather than as a spelling I will
guess at wrongly

|                     |                                                                                                                  |
| ------------------- | ------------------------------------------------------------------------------------------------------------------ |
| **Priority**        | Should                                                                                                           |
| **Tier**            | 2                                                                                                                |
| **Backing tables**  | [`languages`, `dialects`, `lexicon`, `media`](../uml_diagrams/structural/05-Language-Lexicon-Media.md)          |
| **Restricted data** | Yes — `media.is_restricted` is filtered server-side, independently of the word it is attached to                 |

**Acceptance criteria**

1. **Given** a lexicon row has an `audio_media_id` **when** I open the word
   **then** the recording is playable next to its meaning and example sentence.
2. **Given** a lexicon row has no audio but has a `pronunciation` **when** I
   open the word **then** the written pronunciation is shown and no empty player
   appears.
3. **Given** a word belongs to a specific `dialect_id` **when** it is displayed
   **then** the dialect is named, so a form is not read as the whole language's.
4. **Given** the attached media row has `is_restricted = true` **when** the page
   is served **then** the audio is absent from the response, even though the
   word itself is public.
5. **Given** the language has an `endangerment_status` **when** the language is
   displayed **then** it is shown on the UNESCO scale wording, not a bespoke
   phrase.

**Notes** — Tier 2 exists in `history_model_tier2.sql`, and the frontend already
reads `languages` and `figures` for a group, but there is **no lexicon surface
built yet**. This story is data-ready and UI-unbuilt.

---

### EXP-09 — Keep a thought where I had it

**As an** explorer
**I want** to leave a short note attached to what I was reading and find it
still there next time
**So that** the question I had while reading is still with me when I come back,
instead of lost with the tab

|                     |                                                                            |
| ------------------- | -------------------------------------------------------------------------- |
| **Priority**        | Should                                                                     |
| **Tier**            | App                                                                        |
| **Backing tables**  | [`notes`, `user`](../uml_diagrams/structural/06-User-Account.md)          |
| **Restricted data** | No — a note holds the reader's own words                                   |

**Acceptance criteria**

1. **Given** I am signed out **when** I try to save a note **then** I am asked
   to sign in and my typed text is not discarded by the prompt.
2. **Given** I create a note from an open entry **when** it is saved **then** it
   records which entry and which group it came from, so the notes library can
   say where it belongs without a join it cannot make.
3. **Given** I move a note on screen **when** I return in a later session
   **then** it is where I left it.
4. **Given** I have reached the per-context note limit **when** I try to add
   another **then** I am told, and no note is silently dropped.
5. **Given** another signed-in user **when** they load the same group **then**
   they see none of my notes.

**Notes** — `notes.context_id` is `TEXT`, not `UUID`, because it holds a
`peoples.id` from the other database. There is no FK and there cannot be one;
`context_label` and `source_entry_title` are denormalized for the same reason.

---

### EXP-10 — Not lose what I did before I signed up

**As an** explorer
**I want** the choices I made while browsing anonymously to come with me when I
create an account
**So that** signing up feels like keeping my place rather than starting over,
which is the moment I would otherwise leave

|                     |                                                                                       |
| ------------------- | --------------------------------------------------------------------------------------- |
| **Priority**        | Should                                                                                |
| **Tier**            | App                                                                                   |
| **Backing tables**  | [`user`, `user_preferences`](../uml_diagrams/structural/06-User-Account.md), [`manuscripts`](../uml_diagrams/structural/07-Manuscripts-Collaborate.md) |
| **Restricted data** | No                                                                                    |

**Acceptance criteria**

1. **Given** I picked regions anonymously and my new account has none **when** I
   first sign in **then** those regions are written to the account.
2. **Given** I started a draft anonymously **when** I first sign in **then** it
   is migrated to my account rather than left in the browser.
3. **Given** the preference sync fails on a slow network **when** the app loads
   **then** it stops waiting and reads from the local mirror, rather than
   leaving me on a spinner.
4. **Given** the migration has already run once for this account **when** I sign
   in again **then** it does not run a second time and nothing is duplicated.

**Notes** — the anonymous store is the visitor's only copy, which is what makes
dropping it a real loss rather than an inconvenience.

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
