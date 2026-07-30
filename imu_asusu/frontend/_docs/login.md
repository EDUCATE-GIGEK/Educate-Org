# Login — Feature Spec

## What it is

The Login page lets users authenticate to access personal features (Manuscripts). Two methods are supported: email + password and Google OAuth. The page matches the app's theme (Playfair Display headings, Lato body, orange-background-100 palette).

---

## Auth methods

### Email + Password
- Uses `signInWithPassword({ email, password })` from `src/services/auth/signInWithPassword.js`
- Calls `supabase.auth.signInWithPassword`
- On success: navigates to `/app/country`
- On failure: displays error message inline

### Google OAuth
- Uses `signInWithOAuth({ provider: "google" })` from `src/services/auth/signInWithOAuth.js`
- Calls `supabase.auth.signInWithOAuth` — redirects browser to Google
- After Google approval: Supabase redirects to `${window.location.origin}/app/country`
- On first login: DB trigger `on_auth_user_created` auto-creates a `public.user` profile row

---

## Form fields

| Field | Input type | Validation | Notes |
|---|---|---|---|
| Country | Select (`<select>`) | Required | Dropdown of ~13 countries |
| Name | Text input | Required | Stored in `sessionStorage` on email login |
| Email | Email input | Required | |
| Password | Password input | Required | `autoComplete="current-password"` |

> Country and Name are captured on the form but are not yet persisted to `public.user` on email sign-in — only stored in `sessionStorage` under `"login_profile"`. This is a TODO.

---

## Countries list

Nigeria, Ghana, Kenya, South Africa, United Kingdom, United States, Canada, Australia, Germany, France, Italy, Netherlands, Other

---

## State

Managed via `react-hook-form`:
- `register` for all inputs
- `handleSubmit` for form submission
- `formState.errors` for field-level validation messages
- `formState.isSubmitting` to disable the submit button during sign-in

A separate `serverError` string state (`useState`) holds API-level errors shown below the form fields.

---

## Flow

```
User lands on /login
  │
  ├─ Fills email + password → clicks "Sign in"
  │     └─ signInWithPassword() → navigate("/app/country")
  │
  └─ Clicks "Sign in with Google"
        └─ signInWithOAuth() → browser redirects to Google
              └─ Google approves → Supabase callback → navigate("/app/country")
                    └─ (first time) trigger creates public.user row
```

---

## Styled components

All built with `tailwind-styled-components` (`tw`):

| Component | Role |
|---|---|
| `PageWrapper` | Full-page `min-h-screen` with `bg-orange-background-100` |
| `BackBtn` | Absolute top-left back button |
| `Card` | Centered `max-w-sm` container |
| `BrandName` | `font-heading` h1 — "EDUCATÉ" |
| `Tagline` | Subtitle under brand name |
| `StyledSelect` | Country dropdown |
| `StyledInput` | Name / Email / Password fields |
| `SignInBtn` | Primary submit button |
| `Divider` | "or" separator between email and Google |
| `GoogleBtn` | White outlined button with Google SVG icon |
| `ErrorText` | Per-field validation error |
| `ServerError` | Centred API error below the form |

---

## Components

| Component | Path |
|---|---|
| Page | `src/pages/Login.jsx` |
| signInWithPassword | `src/services/auth/signInWithPassword.js` |
| signInWithOAuth | `src/services/auth/signInWithOAuth.js` |

---

## Route

```
/login  →  Login page (outside the /app layout, no auth guard)
```

Linked from:
- "Log in" button in `AppLayout` (top-right, shown when no session)
- Inline login prompt on the Manuscripts page

---

## Pending / TODO

- [ ] Persist Country and Name to `public.user` after email sign-in (currently only saved to `sessionStorage`)
- [ ] Add a Sign Up flow (currently only sign-in is implemented)
- [ ] Add forgot password / reset password link
- [ ] Redirect back to the page the user was on before being asked to log in (instead of always going to `/app/country`)
