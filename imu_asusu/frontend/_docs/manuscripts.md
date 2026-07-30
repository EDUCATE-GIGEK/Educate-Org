# Manuscripts — Feature Spec

## What it is

A manuscript is a personal record created by a logged-in user to preserve Ikwerre cultural knowledge. It can be a story, oral account, observation, or research note. Manuscripts are private to the user who creates them and are tagged with geographic/cultural context so they can be connected to the broader archive.

---

## Auth requirement

- Only logged-in users can view or create manuscripts.
- If a user is not logged in and clicks **+ Add Manuscript**, an inline prompt appears asking them to log in. The form does not open.
- The manuscripts list query is disabled (`enabled: !!profile?.id`) until the user's profile is loaded.

---

## Data model — `public.manuscripts`

| Column | Type | Required | Notes |
|---|---|---|---|
| `id` | BIGINT | auto | Auto-increment via `manuscripts_id_seq` |
| `user_id` | BIGINT | yes | FK → `public.user.id` |
| `title` | TEXT | yes | Short title for the manuscript |
| `manuscript_description` | TEXT | no | The full content / notes |
| `contexts` | JSONB | no | `{ states[], localGovernments[], ethnicGroups[], tribes[] }` — arrays of string IDs |
| `education_level` | `education_level` enum | no | One of `preschool`, `kindergarten`, `high_school`, `undergrad`, `grad` — the intended student audience |
| `created_at` | TIMESTAMPTZ | auto | Default `now()` |

---

## Form fields

| Field | Input type | Validation |
|---|---|---|
| Title | Text input | Required |
| States | Multi-select (ManuscriptContextSelect) | Optional |
| Local Governments | Multi-select (ManuscriptContextSelect) | Optional |
| Ethnic Groups | Multi-select (ManuscriptContextSelect) | Optional |
| Tribes | Multi-select (ManuscriptContextSelect) | Optional |
| Description | Textarea | Optional |

---

## Context selectors (ManuscriptContextSelect)

UI style matches Handshake's multi-select pattern:
- Pill button showing the label + count badge when selections exist
- Floating dropdown with search input and checkboxes
- Selected items appear as removable chips below the pill
- Click outside the dropdown to close it

Data is fetched from Supabase:
- States → `getAllStates()`
- Local Governments → `getAllLocalGovernments()`
- Ethnic Groups → `getAllEthnicGroups()`
- Tribes → `getAllTribes()`

---

## Operations

### Create
- `POST` equivalent via `supabase.from("manuscripts").insert(...)`
- Requires: `user_id` (from `profile.id`), `title`
- On success: invalidates `["manuscripts", profile.id]` query, resets form, closes form panel

### Read
- `getManuscriptsByUser(userId)` fetches manuscripts for the current user filtered by `user_id`, ordered by `created_at DESC`
- Displayed as a list of `ManuscriptCard` components below the Add button
- Shows a spinner while loading, empty state message if no manuscripts exist

### Update
- Triggered by clicking **Edit** on a `ManuscriptCard`
- Scrolls to top of page and opens the form pre-filled with existing values
- Saves via `supabase.from("manuscripts").update(...).eq("id", id)`
- On success: invalidates query, resets form, closes form panel

### Delete
- Triggered by clicking **Delete** on a `ManuscriptCard`
- Inline two-step confirmation: Delete → "Yes, delete" / Cancel
- The card being deleted shows a disabled state while the mutation is pending
- On success: invalidates query, card is removed from the list

---

## ManuscriptCard

Each card in the list displays:
- **Title** (heading font)
- **Date** created (e.g. "12 Jun 2026")
- **Context tags** — count-based pills (e.g. "2 States", "1 Ethnic Group") derived from the `contexts` JSONB field
- **Description** — truncated to 3 lines (`line-clamp-3`)
- **Edit** button — opens the form pre-filled
- **Delete** button — triggers inline confirmation before deleting

---

## Page layout

```
Manuscripts (title)
Description paragraph

[Form panel — visible when creating or editing]

[+ Add Manuscript button]
[Login prompt — shown when unauthenticated user clicks Add]

[Manuscript cards list — visible when logged in and form is closed]
  → Spinner while loading
  → Empty state if no manuscripts
  → ManuscriptCard per result
```

---

## Components

| Component | Path |
|---|---|
| Page | `src/pages/Manuscripts.jsx` |
| Manuscript card | `src/features/Manuscripts/ManuscriptCard.jsx` |
| Context selector | `src/features/Manuscripts/ManuscriptContextSelect.jsx` |
| API service | `src/services/apiManuscripts.js` |

---

## Pending / TODO

- [x] Add RLS policy so users can only read/write their own manuscripts
- [ ] Show resolved names in context tags (e.g. "Rivers State") instead of counts
- [ ] Consider pagination for the manuscript list
- [ ] Add a confirmation modal (instead of inline confirm) for delete
