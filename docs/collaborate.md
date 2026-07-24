# Collaborate — Feature Spec

**Status:** Built (2026-07-24) in Ịmụ-Asụsụ. Schema lives in `Backend-Imu-Asusu/supabase/migrations/20260724000000_collaborate_public_manuscripts.sql`; the page lives in `Frontend-Imu-Asusu/src/pages/Collaborate.jsx`, with a fuller implementation spec at `src/specs/collaborate/Collaborate.md`.

**Scope:** how educators share the teaching materials they write on the platform, and how the platform decides which of them to put in front of each person. Applies to manuscripts today; the same pattern is intended to carry over to any curriculum artefact a future Educate project produces.

---

## 1. Why the platform needs this

The platform's premise is that instructors in low-resource institutions should be able to build teaching materials on their own histories without depending on Western scholarship. But an instructor writing alone is still working alone — they carry the whole cost of research, sourcing, and drafting for a subject that, by definition, has thin published literature.

Collaborate is the answer to that: a place where the material one instructor has already built becomes the starting point for the next. The unit of sharing is the **manuscript**, and the unit of reuse is the **fork**.

Three principles shape it:

- **Private by default.** Nothing becomes visible because a user did not read a setting. An author opts in, per manuscript, and can opt back out.
- **Sharing is not surrendering.** A reader can never edit an author's manuscript. Building on someone's work means taking a copy — the original stays under its author's control, and the copy is honestly recorded as derived.
- **Relevance beats popularity.** A manuscript about a region an instructor actually teaches is more useful to them than a widely-upvoted one about somewhere else. The ranking is built to say so.

---

## 2. Public manuscripts

A manuscript carries an `is_public` flag, set from its details panel alongside its contexts and student level. When true, it appears on every signed-in user's Collaborate page: readable, upvotable, forkable.

What a reader sees on a shared manuscript: title, summary, author name, intended student level, the written content, and its standing (upvotes, forks, views). What they cannot do: change it, or see manuscripts that were never shared.

Attached source files are **not** shared with the manuscript today. The text is the artefact; the underlying research file stays with its author.

---

## 3. Forking

Forking copies a public manuscript's text — title, summary, body, contexts, student level — into a new manuscript owned by the forker, private, and recorded as `forked_from` the original. The source's `fork_count` goes up by one, in the same transaction.

Deliberate consequences:

- The original author's work is never mutated by anyone else. There is no merge, no review queue, no shared editing surface — those are a different feature with a different trust model (see §6).
- Provenance survives. A forked manuscript knows what it came from, and an author can see how many times their work has been taken up.
- Attached files are not duplicated. The forker attaches their own if they need one.

---

## 4. Relevance — what each person is shown first

The page opens with up to **ten** manuscripts chosen for the reader, above a browsable list of everything.

The score:

```
score = 10 × (the reader's saved regions the manuscript covers)
      +  3 × forks
      +  2 × upvotes
      +  0.1 × views
```

The weights encode a judgment. **Region overlap dominates by design** — the whole point of the platform is that an instructor's own region is where the material matters, so a manuscript on a region they saved should outrank a popular manuscript on a region they did not. The community signals then rank among manuscripts that are equally on-topic, weighted by the effort each represents: forking a manuscript is a stronger endorsement than upvoting it, which is stronger than reading it.

All three signals count **people, not events**: one upvote and one view per educator, and an author's own reading of their own manuscript counts for nothing. A manuscript ten instructors each read once outranks one that a single instructor opened ten times, which is the comparison the ranking needs to be able to make.

Ties break on recency, so an equally relevant newer contribution is never permanently buried under an older one — a new contributor's first manuscript can surface immediately.

The reader's own public manuscripts are kept out of this shelf (it is for discovering other people's work) but remain in the browsable list.

Scoring runs client-side, because the strongest input — the reader's saved regions — is held in browser preferences rather than in the database. If those preferences ever move server-side, the scoring moves with them.

---

## 5. Browsing everything

Below the shelf, every public manuscript, with:

- **Search** across title, summary, and author name.
- **Filter** by intended student level.
- **Sort** by most recent (default), most upvoted, A–Z, or oldest.

---

## 6. Explicitly not in scope

These are separate features with their own designs, not omissions:

- **Peer review** — commenting on, endorsing, or disputing another educator's manuscript.
- **Co-authoring** — two people editing one manuscript. Collaborate's trust model is deliberately copy-based; shared editing needs its own.
- **Following an author or subscribing to new public manuscripts.**

---

## 7. Data model summary

Added to `manuscripts`: `is_public`, `fork_count`, `upvote_count`, `view_count`, `forked_from`.

New tables `manuscript_upvotes (manuscript_id, user_id)` and `manuscript_views (manuscript_id, user_id)` — in both, the primary key is what enforces one per person, rather than the interface. `manuscript_views` has no insert policy at all: a view can only be recorded by the server-side function, which is what makes the number trustworthy.

The counts on `manuscripts` are denormalized because the page ranks and sorts on them for every card it draws; a trigger keeps the upvote count truthful, and the view function only moves the counter on a reader's first open.

Every write that touches a row the caller does not own — the upvote count, the view count, the source manuscript's fork count — goes through a server-side function. Row-level security on manuscripts stays owner-only for direct writes, which is what keeps an author's work theirs.
