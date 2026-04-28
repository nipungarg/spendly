# Spec: Login and Logout

## Overview

Step 2 lets users register, but the only thing they can do afterwards is
re-render forms — there's no way to actually sign in. This step introduces
session-based authentication: `POST /login` verifies an email + password
against `users.password_hash`, stores `user_id` in the Flask session on
success, and `GET /logout` clears the session and redirects to the
landing page. The navbar adapts to authentication state, and a small
helper layer (`current_user()`, `login_required` decorator) is added so
later steps (profile, expenses) can gate access cheaply. This is the
backbone for every logged-in feature that follows.

## Depends on

- Step 1 — Database setup (the `users` table with hashed passwords).
- Step 2 — Registration (so there are accounts to sign in to, and so
  `?registered=1` already redirects here).

## Routes

- `POST /login` — verifies credentials, sets `session["user_id"]`,
  redirects to `/profile` (or to the `next` query param if present and
  same-origin) — public.
- `GET /logout` — clears the session and redirects to `/` — logged-in
  (a logged-out hit just no-ops and redirects).

The existing `GET /login`, `GET /register`, and `POST /register` routes
stay as-is. The `/logout` placeholder string returned today gets
replaced.

## Database changes

No schema changes. All needed columns already exist on `users` from
Step 1.

## Templates

- **Create:** none.
- **Modify:**
  - `templates/base.html` — navbar shows `Sign in` / `Get started` when
    logged-out, and `Profile` / `Logout` when logged-in.
  - `templates/login.html` — preserve the entered `email` after a failed
    submit (mirrors the Step 2 register-form repopulation).

## Files to change

- `app.py` — set `app.secret_key`, add `POST` handling on `/login`,
  replace the `/logout` stub, add `current_user()` helper and
  `login_required` decorator, expose `current_user` to all templates via
  `@app.context_processor`.
- `database/db.py` — add `get_user_by_id(user_id)` helper (mirrors the
  Step 2 `get_user_by_email`).
- `templates/base.html` — auth-aware navbar.
- `templates/login.html` — repopulate `email` field after error.

## Files to create

- None.

## New dependencies

No new pip packages. `werkzeug.security.check_password_hash` ships with
the existing `werkzeug==3.0.3`.

## Rules for implementation

- No SQLAlchemy or other ORMs — raw `sqlite3` only.
- Parameterised queries only (`?` placeholders) — never f-strings in SQL.
- Passwords verified with `werkzeug.security.check_password_hash`; never
  log or echo the submitted password.
- All DB access lives in `database/db.py` — no inline queries in routes.
- All templates extend `base.html`; reuse existing CSS variables in
  `static/css/style.css` — never hardcode hex values.
- Use `url_for(...)` for every internal link and redirect.
- `app.secret_key` must come from `os.environ.get("SECRET_KEY", <dev-fallback>)`
  with a clearly-marked dev fallback so local runs still work without
  extra setup. Document the env var inline near where it is read.
- Use the same generic error message for "no such email" and "wrong
  password" — `"Invalid email or password."` — to avoid account
  enumeration.
- Validate inputs before hitting the DB:
  - `email` — required, trimmed, lowercased before lookup.
  - `password` — required (any length; the registration step is what
    enforces length).
- On any failure: re-render `login.html` with `error=...` and HTTP 400,
  preserving the entered `email` value.
- On success: `session.clear()` first (defence-in-depth against fixation),
  set `session["user_id"]`, then redirect (302) to `/profile`. If a
  `next` form/query param is present, only honour it when it begins with
  `/` and does not begin with `//` (open-redirect guard).
- `GET /logout` must use `session.clear()` (not `session.pop`) and
  redirect to `url_for("landing")`.
- `current_user()` must return `None` when no `user_id` in session, and a
  `sqlite3.Row` (from `get_user_by_id`) otherwise — never raise on a
  missing or stale id; just clear the session and return `None` if the
  id no longer maps to a row.
- `login_required` is added but **not yet applied** to existing stub
  routes — those stubs remain untouched until their own steps. The
  decorator exists so Step 4 (profile) can adopt it.
- Do not auto-login the user on registration in this step either —
  Step 2's `?registered=1` flow stays intact.

## Definition of done

- [ ] Submitting `/login` with a known email + correct password redirects
      302 to `/profile` (which will still render its current Step-4 stub
      string — that's fine for this step).
- [ ] After a successful login, hitting `/login` again still works (the
      route does not crash on an already-authenticated session).
- [ ] Submitting `/login` with an unknown email returns HTTP 400 and
      renders `"Invalid email or password."`; the `email` field is
      repopulated.
- [ ] Submitting `/login` with a known email but wrong password returns
      the **same** generic error (no enumeration leak).
- [ ] Submitting `/login` with an empty field returns HTTP 400 with a
      clear inline error.
- [ ] `GET /logout` clears the session and redirects (302) to `/`.
      Visiting any page afterwards shows the logged-out navbar.
- [ ] The navbar in `base.html` shows `Sign in` / `Get started` when
      logged-out and `Profile` / `Logout` when logged-in, on every page
      that extends `base.html`.
- [ ] `?registered=1` from Step 2 still surfaces the success banner on
      `GET /login` and the form still works.
- [ ] `app.secret_key` reads from `SECRET_KEY` env var with a documented
      dev fallback; the app starts without setting any env var.
- [ ] No new pip packages; `requirements.txt` is unchanged.
- [ ] All DB access lives in `database/db.py`; the route functions only
      validate input, call helpers, set/clear session, and render or
      redirect.
