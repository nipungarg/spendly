# Spec: Registration

## Overview

Turn the existing `GET /register` page into a working sign-up flow. The form
in `templates/register.html` already collects name, email, and password and
posts to `/register`, but the route only renders the template. This step
adds a `POST /register` handler that validates the input, hashes the
password, inserts a new row in the `users` table, shows a success message, 
and redirects to the login page on success. It is the first feature that actually 
writes data created by an end user, and it unblocks the login flow 
that lands in the next step.

## Depends on

- Step 1 — Database setup (the `users` table, `get_db()`, and werkzeug must
  already be in place — they are, per `database/db.py`).

## Routes

- `POST /register` — validates the registration form, creates a user, and
  redirects to `/login` on success — public.

The existing `GET /register` route stays as-is. No other new routes.

## Database changes

No schema changes. The existing `users` table already has the columns we
need: `id`, `name`, `email`, `password_hash`, `created_at`. The
`UNIQUE(email)` constraint is what we lean on for duplicate detection.

## Templates

- **Create:** none.
- **Modify:** `templates/register.html` — only if needed to surface field-
  level errors or to repopulate the `name`/`email` inputs after a failed
  submit (the `{% if error %}` block already exists, so a single `error`
  string is enough; repopulation is a nice-to-have).

## Files to change

- `app.py` — accept `POST` on `/register`, call the new DB helper, render
  errors or redirect on success.
- `database/db.py` — add `create_user(name, email, password)` and
  `get_user_by_email(email)` helpers (parameterised queries, password
  hashed inside `create_user`).
- `templates/register.html` — optional tweak to repopulate `name` and
  `email` on validation failure.

## Files to create

- None.

## New dependencies

No new dependencies. `werkzeug.security.generate_password_hash` is already
imported in `database/db.py`.

## Rules for implementation

- No SQLAlchemy or other ORMs — raw `sqlite3` only.
- Parameterised queries only (`?` placeholders) — never f-strings in SQL.
- Passwords hashed with `werkzeug.security.generate_password_hash`; never
  store or log plaintext.
- All DB access goes through helpers in `database/db.py` — no inline
  queries inside the route function.
- All templates extend `base.html`; use existing CSS variables in
  `static/css/style.css` — never hardcode hex values.
- Use `url_for('login')` / `url_for('register')` for redirects and links —
  never hardcode URL paths.
- Validation rules:
  - `name` — required, trimmed, 1–100 characters.
  - `email` — required, trimmed, lowercased before insert, basic format
    check (must contain `@` and a `.` after it).
  - `password` — required, minimum 8 characters.
- On any validation failure or duplicate email, re-render
  `register.html` with a single `error` string and HTTP 400. Do not flash
  — sessions are not wired up yet (that lands in Step 3).
- On success, redirect (HTTP 302) to `url_for('login')`.
- Catch `sqlite3.IntegrityError` from the UNIQUE email constraint and
  surface it as `"An account with that email already exists."` — do not
  pre-check + insert in two steps (race-prone).
- The `GET /register` behaviour must not regress.

## Definition of done

- [ ] `POST /register` with valid name/email/password creates a row in
      `users` with a hashed password and redirects to `/login`.
- [ ] Submitting the form with a missing field re-renders `register.html`
      with a clear inline error and HTTP 400.
- [ ] Submitting an email that already exists re-renders `register.html`
      with `"An account with that email already exists."` and HTTP 400.
- [ ] Submitting a password shorter than 8 characters re-renders with an
      inline error.
- [ ] Inspecting the DB after a successful registration shows the
      password stored as a `pbkdf2:` / `scrypt:` hash, never plaintext.
- [ ] `GET /register` still renders the form unchanged.
- [ ] No new pip packages were added; `requirements.txt` is unchanged.
- [ ] All DB access lives in `database/db.py`; the route function only
      validates input, calls a helper, and renders or redirects.
