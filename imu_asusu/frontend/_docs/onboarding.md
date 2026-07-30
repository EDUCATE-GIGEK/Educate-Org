# Onboarding preferences — where regions and intent live

## What they are

A user's saved **regions** (`{ kind: 'place' | 'people', id, name }[]`) and **intent**
(`teach` | `research` | `explore`), chosen during onboarding. They drive the Home hub's entry
points and suggestion tiles, the sidebar, and — most importantly — the relevance ranking on
Collaborate.

---

## Two stores, one shape

| Who | Store | Durable? |
|---|---|---|
| Anonymous visitor | `localStorage` under `imu.prefs` | Yes — it is their only copy |
| Signed-in user | `public.user_preferences.preferences` (jsonb) | Yes; `localStorage` is a mirror |

The mirror exists so reads stay **synchronous**. That is what lets `usePreferences` keep returning
a plain `prefs` object, and it is why none of the six consuming components
(`Home`, `Explore`, `Timeline`, `Collaborate`, `RegionSearch`, `Onboarding`) needed changing when
the database was introduced.

`services/preferences.js` is the only module that knows either store exists.

---

## Why a separate table

`public.user` is world-readable — its `public_read_user` policy is `using (true)` for both `anon`
and `authenticated`. A `preferences` column there would publish every user's regions of interest
to anyone with the anon key.

Nothing ever needs to read *another* user's preferences: Collaborate ranks against the reader's
own regions, client-side. So `user_preferences` carries owner-only RLS (`for all to authenticated`,
matched on `user_id`), which costs nothing and keeps interests private.

One jsonb blob rather than a `user_regions` join table, because preferences are always read whole
and this matches `manuscripts.contexts`, which already stores id arrays the same way. Normalize if
we ever need to query across users by region.

---

## Reconciliation on sign-in

Handled once per profile in `usePreferences`:

1. Read the account's preferences.
2. **If the account has regions, they win** — write them to the local mirror.
3. **Otherwise, if localStorage has regions, adopt them onto the account.** This is someone who
   onboarded before signing up; their choices must not be silently dropped.
4. Either way, mark the sync done — including on failure, so a network error can't leave callers
   waiting forever on something they can answer from the mirror.

Writes go to both stores, local first, so a slow network never delays a re-render.

---

## `isSyncing`, and why it matters

`usePreferences` returns `isSyncing`, true while a signed-in user's stored preferences are still
in flight (including while the profile itself is loading — hence `isProfileLoading` on
`AuthContext`).

**Anything that branches on `prefs` to make a routing decision must wait for it.** Two places do:

- `RootRedirect` — decides onboarding vs. the hub. Deciding early would send a user who onboarded
  on another device (or who simply logged out on this one) back through onboarding they have
  already completed.
- `Onboarding` — gates the flow so the picker's initial state is seeded once, from real values,
  rather than from an empty mirror.

There is deliberately **no** synchronous `hasCompletedOnboarding()` helper any more. For a
signed-in user that question cannot be answered from localStorage alone, and a helper that looks
like it can is how the bug comes back.

---

## Logging out

`signOut` clears the local mirror. The account keeps everything, so signing back in restores it —
but the next person on a shared machine does not inherit the last one's interests. Shared lab and
staff-room computers are the normal case for the institutions this platform is built for, which is
what makes this worth doing rather than a nicety.

`usePreferences().clear()` is a different operation: a deliberate reset of the account's
preferences, which clears both stores.
