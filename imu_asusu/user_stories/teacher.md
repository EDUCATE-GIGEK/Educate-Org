# Teacher — user stories

**Persona** Teacher · **Intent** `teach` · **ID prefix** `TCH`

An instructor turning what the platform holds into something teachable at a
stated education level. Format and field definitions live in
[`_docs/user-stories.md`](../_docs/user-stories.md).

**On the `Tier` column — read this before marking anything `Blocked`.** The
format spec says "anything joining `manuscripts` to `entries` is `Blocked`
today". That rule is now **out of date**: `manuscripts.contexts` stores
`{ places: uuid[], peoples: uuid[] }`, the Tier 1 tables are live in the same
Supabase project, and the `manuscript-fact-check` and `manuscript-generate` edge
functions already resolve contexts into entries through the
`entries_for_contexts` RPC under the caller's own RLS. Stories that do this are
marked **`App + 1`** rather than `Blocked`, because they ship today. What remains
genuinely broken is narrower: `manuscripts.cultural_history_id` still points at
the dead `cultural_history` table (gap 3 in
[`00-Overview.md`](../uml_diagrams/structural/00-Overview.md#known-gaps)), and
the id-type split still prevents a real FK.

**`TCH-01` supersedes the earlier `TER1` draft** — same story, renumbered to the
prefix the format spec defines.

---

### TCH-01 — Start a manuscript for a topic I have to teach

**As a** teacher
**I want** to open a blank manuscript and start writing about a topic
**So that** I can build the teaching material my subject has no textbook for,
instead of waiting for one to exist

|                     |                                                                                                                                    |
| ------------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| **Priority**        | Must                                                                                                                               |
| **Tier**            | App                                                                                                                                |
| **Backing tables**  | [`manuscripts`](../uml_diagrams/structural/07-Manuscripts-Collaborate.md), [`user`](../uml_diagrams/structural/06-User-Account.md) |
| **Restricted data** | No — a manuscript holds the author's own writing                                                                                   |

**Acceptance criteria**

1. **Given** I am signed out **when** I click Add Manuscript **then** I am allowed to create and edit a singular manuscript.
2. **Given** I signed out **when** I try to create a second manuscript **then** I am unable to and prompted to login.
   it is saved against my `user_id` and appears in my list.
3. **Given** a newly created manuscript **when** it is saved **then**
   `is_public` is false — nothing becomes visible to others because I did not
   read a setting.
4. **Given** another signed-in user **when** they query manuscripts **then**
   mine are not returned, because the select policy is owner-only until I share.
5. **Given** I am signed in **when** I create a manuscript with a title **then**

**Notes** — supersedes the `TER1` draft. Manuscript body, attachments, and the
editor's own behaviour are out of scope here; this story is only "a manuscript
now exists and is mine".

---

### TCH-02 — Tie a manuscript to the places and peoples it covers

**As a** teacher
**I want** to tag my manuscript with the groups and places it is about
**So that** the platform's own records can be brought to bear on what I write,
rather than me copying facts across by hand

|                     |                                                                                                                                                                       |
| ------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Priority**        | Must                                                                                                                                                                  |
| **Tier**            | App + 1                                                                                                                                                               |
| **Backing tables**  | [`manuscripts`](../uml_diagrams/structural/07-Manuscripts-Collaborate.md), [`places`, `peoples`, `designations`](../uml_diagrams/structural/01-Places-And-Peoples.md) |
| **Restricted data** | No — the picker lists tree nodes, which carry no `is_restricted` flag                                                                                                 |

**Acceptance criteria**

1. **Given** the context picker **when** it lists options **then** each carries
   its `designation` label, so I can tell an Ethnic Group from a Clan with the
   same name.
2. **Given** I select contexts **when** the manuscript is saved **then**
   `contexts` holds `{ places: [...], peoples: [...] }` as UUID arrays matching
   the Tier 1 ids.
3. **Given** I select an ethnic group **when** the platform later resolves my
   contexts **then** entries anywhere in that group's subtree are in scope, not
   only entries whose `people_id` is the node itself.
4. **Given** I save a manuscript with no contexts at all **when** it is stored
   **then** it saves — but every grounding feature must tell me it has nothing
   to work from ([TCH-04](#tch-04--draft-a-passage-grounded-in-the-records)).

**Notes** — `manuscripts.md` in the frontend docs still describes `contexts` as
`{ states, localGovernments, ethnicGroups, tribes }`. The live shape is
`{ places, peoples }` with UUIDs; that doc needs updating.

---

### TCH-03 — Say who the material is for

**As a** teacher
**I want** to state the level of the students I am writing for
**So that** the help I get is pitched at my class rather than at an average
reader who does not exist

|                     |                                                                           |
| ------------------- | ------------------------------------------------------------------------- |
| **Priority**        | Must                                                                      |
| **Tier**            | App                                                                       |
| **Backing tables**  | [`manuscripts`](../uml_diagrams/structural/07-Manuscripts-Collaborate.md) |
| **Restricted data** | No                                                                        |

**Acceptance criteria**

1. **Given** the details panel **when** I set a level **then** it is one of the
   five `education_level` values (`preschool`, `kindergarten`, `high_school`,
   `undergrad`, `grad`) and is stored on the manuscript.
2. **Given** a level is set **when** I ask for generated or assisted text
   **then** that level is passed as the audience and the register of the output
   changes with it.
3. **Given** no level is set **when** I ask for generated text **then** the
   assistant assumes `high_school` and says which level it wrote for, rather
   than leaving me to guess.
4. **Given** I share the manuscript **when** another instructor sees its card
   **then** the intended student level is visible, so they can judge fit before
   opening it.

**Notes** — criterion 4 contradicts
[`07-Manuscripts-Collaborate.md`](../uml_diagrams/structural/07-Manuscripts-Collaborate.md),
which calls `education_level` "an AI input, not a display field", while
`collaborate.md` lists it among what a reader sees. It is currently both; the
diagram's note is the one that is wrong.

---

### TCH-04 — Draft a passage grounded in the records

**As a** teacher
**I want** a first draft of a passage built from the platform's own historical
entries
**So that** I start from what this archive actually holds instead of from a
model's general impression of West African history

|                     |                                                                                                                                                  |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Priority**        | Must                                                                                                                                             |
| **Tier**            | App + 1                                                                                                                                          |
| **Backing tables**  | [`manuscripts`](../uml_diagrams/structural/07-Manuscripts-Collaborate.md), [`entries`](../uml_diagrams/structural/02-Entries-Knowledge-Graph.md) |
| **Restricted data** | Yes — the entry read runs under my own JWT and RLS, so restricted and unpublished entries never enter the prompt                                 |

**Acceptance criteria**

1. **Given** my manuscript has contexts **when** I ask for a passage **then**
   the entries used are drawn from the subtree of those contexts and the
   concrete claims in the output do not contradict them.
2. **Given** my manuscript has no contexts **when** I ask for a passage **then**
   I am told there is nothing to ground against, rather than being handed
   confident prose written from the model's own knowledge.
3. **Given** the contexts resolve to entries **when** the instruction asks for
   something those entries do not cover **then** the passage supplies structure,
   framing, and questions — and states the gap — instead of inventing dates or
   names.
4. **Given** an entry is `is_restricted = true` or not `published` **when** the
   grounding set is assembled **then** it is absent, because the function reads
   with my permissions and not with a service-role key.
5. **Given** my existing draft is very long **when** the request is built
   **then** it is truncated so it cannot crowd the source entries out of the
   context window.

**Notes** — a fabricated fact in teaching material is the failure this whole
model exists to prevent; criteria 2 and 3 are the ones that make that true in
practice rather than in the prompt only.

---

### TCH-05 — Have my draft checked against the records

**As a** teacher
**I want** my draft checked claim by claim against the platform's entries
**So that** I find out I have a date wrong before thirty students learn it that
way

|                     |                                                                                                                                                  |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Priority**        | Must                                                                                                                                             |
| **Tier**            | App + 1                                                                                                                                          |
| **Backing tables**  | [`manuscripts`](../uml_diagrams/structural/07-Manuscripts-Collaborate.md), [`entries`](../uml_diagrams/structural/02-Entries-Knowledge-Graph.md) |
| **Restricted data** | Yes — the same RLS-scoped read as TCH-04; a restricted entry is never quoted back at me as evidence                                              |

**Acceptance criteria**

1. **Given** a draft claim that an entry directly affirms **when** the check
   runs **then** it is returned as supported and cites the entry it relied on.
2. **Given** a claim the entries neither affirm nor conflict with **when** the
   check runs **then** the verdict is "no evidence", cites nothing, and does not
   speculate about whether the claim is true.
3. **Given** a claim an entry conflicts with **when** the check runs **then** a
   correction is supplied that can replace the text in place, faithful to the
   entry.
4. **Given** a returned claim **when** I look for it in my draft **then** the
   excerpt is a verbatim substring, so it can be located and highlighted rather
   than approximately matched.
5. **Given** a manuscript id I do not own **when** the check is called with it
   **then** it fails as not found, because ownership is enforced by RLS rather
   than by a separate check the caller could skip.
6. **Given** my manuscript has no contexts **when** the check runs **then** it
   returns nothing checked and says so, rather than falling back on the model's
   own history knowledge.

**Notes** — this is the function that makes provenance pay off. It is also the
one place where "helpful" and "honest" pull apart, which is why criterion 2 is a
Must and not a nicety.

---

### TCH-06 — Not have a disputed record used as settled proof

**As a** teacher
**I want** the checker to tell me when its evidence is itself contested or
unverified
**So that** I don't teach one side of a live disagreement as established fact
because a green tick told me to

|                     |                                                                                                                                                                             |
| ------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Priority**        | Should                                                                                                                                                                      |
| **Tier**            | App + 1                                                                                                                                                                     |
| **Backing tables**  | [`entries`, `entry_relationships`](../uml_diagrams/structural/02-Entries-Knowledge-Graph.md), [`entry_sources`](../uml_diagrams/structural/03-Provenance.md), `manuscripts` |
| **Restricted data** | Yes — restricted entries and sources stay out of the evidence set entirely                                                                                                  |

**Acceptance criteria**

1. **Given** the entry supporting a verdict has
   `verification_status = 'disputed'` **when** the result is shown **then** the
   verdict is qualified as contested rather than presented as plain support.
2. **Given** the supporting entry is `unverified` **when** the result is shown
   **then** that is stated alongside the verdict.
3. **Given** two entries linked by `contradicts` both bear on my claim **when**
   the result is shown **then** both are surfaced, and the checker does not pick
   the one that happens to agree with my draft.
4. **Given** a verdict citing a disputed entry **when** I accept its correction
   **then** I am warned that I am adopting one side of a modelled disagreement.

**Notes** — **not currently possible.** The fact-check function reshapes each
entry into `{ id, label, category, text }` and drops `verification_status`
before the model ever sees it, so an unverified claim can come back as
"supported". Fixing this means widening that projection and teaching the prompt
a fourth kind of answer. This is the most important unbuilt story in this file.

---

### TCH-07 — Share a manuscript, and take it back

**As a** teacher
**I want** to publish a manuscript to other instructors and be able to unpublish
it
**So that** I can share work when it is ready without losing control of it
afterwards

|                     |                                                                           |
| ------------------- | ------------------------------------------------------------------------- |
| **Priority**        | Must                                                                      |
| **Tier**            | App                                                                       |
| **Backing tables**  | [`manuscripts`](../uml_diagrams/structural/07-Manuscripts-Collaborate.md) |
| **Restricted data** | No                                                                        |

**Acceptance criteria**

1. **Given** a private manuscript **when** I set `is_public = true` **then** it
   appears on other signed-in users' Collaborate page; nothing else about my
   account is shared by that act.
2. **Given** a public manuscript **when** a reader opens it **then** they can
   read, upvote, and fork it, and cannot change a word of mine.
3. **Given** I set `is_public` back to false **when** the page reloads **then**
   it disappears from other people's Collaborate, while my own list still shows
   it.
4. **Given** I have attached a source file **when** the manuscript is shared
   **then** the file is not shared with it — the text is the shared artefact.
5. **Given** I share a manuscript **when** the select policies are applied
   **then** I still see my own private manuscripts too, because the policies are
   permissive and OR together.

**Notes** — "private by default, sharing is not surrendering" is the trust model
the whole feature rests on; a reader path that could ever write to my row breaks
it.

---

### TCH-08 — Build on another instructor's work

**As a** teacher
**I want** to take a copy of a shared manuscript and adapt it for my own class
**So that** I start from what a colleague already researched instead of paying
the whole research cost again for a subject with thin literature

|                     |                                                                           |
| ------------------- | ------------------------------------------------------------------------- |
| **Priority**        | Must                                                                      |
| **Tier**            | App                                                                       |
| **Backing tables**  | [`manuscripts`](../uml_diagrams/structural/07-Manuscripts-Collaborate.md) |
| **Restricted data** | No                                                                        |

**Acceptance criteria**

1. **Given** a public manuscript **when** I fork it **then** a copy of its title,
   summary, body, contexts, and student level is created, owned by me and
   private.
2. **Given** the fork is created **when** it is saved **then** `forked_from`
   records the original, so where it came from is recoverable later.
3. **Given** the fork is created **when** the transaction completes **then** the
   original's `fork_count` has gone up by one and nothing else about the
   original has changed.
4. **Given** I do not own the original **when** the counter is written **then**
   the write goes through the server-side function, because direct writes to a
   row I do not own are refused.
5. **Given** the original author deletes their manuscript **when** I open my
   fork **then** it still exists, with its `forked_from` cleared rather than
   cascaded away.
6. **Given** the original has attached files **when** I fork it **then** they
   are not copied and I am not shown a broken attachment.

**Notes** — there is deliberately no merge, no review queue, and no shared
editing. Those need a different trust model and are a separate feature.

---

### TCH-09 — See work about the regions I actually teach first

**As a** teacher
**I want** shared manuscripts ranked by how well they match the regions I saved
**So that** the first thing I see is usable in my classroom, not the most
popular thing on the platform

|                     |                                                                                                                                                                                          |
| ------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Priority**        | Should                                                                                                                                                                                   |
| **Tier**            | App                                                                                                                                                                                      |
| **Backing tables**  | [`manuscripts`, `manuscript_upvotes`, `manuscript_views`](../uml_diagrams/structural/07-Manuscripts-Collaborate.md), [`user_preferences`](../uml_diagrams/structural/06-User-Account.md) |
| **Restricted data** | No                                                                                                                                                                                       |

**Acceptance criteria**

1. **Given** a manuscript covering a region I saved and a more-upvoted one
   covering a region I did not **when** the shelf is built **then** the
   on-region manuscript ranks higher.
2. **Given** two manuscripts equally on-topic **when** they are ranked **then**
   forks outweigh upvotes and upvotes outweigh views, in that order.
3. **Given** two manuscripts with identical scores **when** they are ranked
   **then** the newer one comes first, so a first contribution is not
   permanently buried.
4. **Given** my own public manuscripts **when** the shelf is built **then** they
   are excluded from it but still appear in the browsable list below.
5. **Given** a reader opens the same manuscript ten times **when** the view
   count is computed **then** it counts once; an author reading their own work
   counts for nothing.
6. **Given** I have saved no regions **when** the shelf is built **then** it
   falls back to community signals rather than rendering empty.

**Notes** — scoring runs client-side because the strongest input (saved regions)
lives in browser preferences. If preferences move server-side, the ranking moves
with them.

---

### TCH-10 — Carry the evidence into the material

**As a** teacher
**I want** the entries a passage was built from listed with the passage
**So that** my students can follow the material back to its sources, and I can
show my department the work is grounded

|                     |                                                                                                                                                                                                                             |
| ------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Priority**        | Should                                                                                                                                                                                                                      |
| **Tier**            | App + 1                                                                                                                                                                                                                     |
| **Backing tables**  | [`manuscripts`](../uml_diagrams/structural/07-Manuscripts-Collaborate.md), [`entries`](../uml_diagrams/structural/02-Entries-Knowledge-Graph.md), [`entry_sources`, `sources`](../uml_diagrams/structural/03-Provenance.md) |
| **Restricted data** | Yes — restricted entries, restricted sources, and non-consented testimony are absent from any trail written into a manuscript                                                                                               |

**Acceptance criteria**

1. **Given** a passage generated or checked against entries **when** I ask for
   its evidence **then** each entry is listed with its title, type, and
   `verification_status`.
2. **Given** an entry in that list has sources **when** the trail is written
   **then** each source's `citation_or_url` and type is included as held —
   nothing is completed from elsewhere.
3. **Given** an entry has no sources **when** it appears in the trail **then**
   it is marked as unsourced rather than cited as though it were.
4. **Given** an entry is only relatively dated **when** it appears in the trail
   **then** its `period_note` is carried across, not rounded into a year.
5. **Given** I share the manuscript **when** a reader opens it **then** the
   trail travels with the text.

**Notes** — the generate function currently returns HTML with no entry ids at
all, while the fact-check function does return the sources it used. The trail
therefore has to be captured at generation time or it is unrecoverable
afterwards. Depends on the provenance reads in
[RES-01](researcher.md#res-01--see-every-source-behind-a-claim).

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
