# Release checklist

Use this for every `develop` → `main` release. There is no staging environment
(decided 2026-11-15), so these steps are what stands between a change and
Olam's live data. Paste the filled-in checklist into the release PR.

Merging to `main` deploys immediately:
- **Backend:** Render auto-deploys `faithflow-backend` from `main`, running
  `migrate_schemas --shared` and `migrate_schemas` during the build.
- **Frontend:** `.github/workflows/deploy.yml` builds and syncs to S3, then
  invalidates CloudFront.

## 1. Before opening the release PR

- [ ] CI is green on `develop` for both repos.
- [ ] Every PR merged into `develop` since the last release is listed in the
      release PR description.
- [ ] Any new environment variable or secret is already set on Render / in
      GitHub secrets (names listed in the PR; never values).
- [ ] If the release touches login, permissions or tenant selection: run the
      read-only checks the PR asks for in a Render shell (for example, users
      with no church).

## 2. Migration rehearsal (only if the release adds migrations)

Neon can copy production instantly. Rehearse against the copy, never against
production itself.

1. Neon console → **Branches** → create a branch from the production branch,
   named `rehearsal-<date>`.
2. Copy that branch's connection string. Check the host is the rehearsal
   branch, not production.
3. From `faithflow-backend/`, run the migrations against it:
   ```bash
   DATABASE_URL='<rehearsal connection string>' python manage.py migrate_schemas --shared
   DATABASE_URL='<rehearsal connection string>' python manage.py migrate_schemas
   ```
4. Both must finish without errors. Note how long they took; if more than a
   minute, deploy during a quieter window.
5. Delete the rehearsal branch. It holds a full copy of real member data.

## 3. Deploy

- [ ] Pick a quiet time: **not Saturday evening or Sunday**, and not just
      before a church event.
- [ ] Tell the Olam admin contact that a short update is happening.
- [ ] Merge the `develop` → `main` PR in both repos (backend first if the
      frontend depends on a new endpoint).
- [ ] Watch the Render deploy log until it reports live. Watch the frontend
      deploy workflow until it finishes.

## 4. Smoke test (about 5 minutes, on the live site)

Use a test admin account and a test member account in Olam, not a real
member's.

- [ ] Admin logs in at the church subdomain.
- [ ] Members list loads; open one member.
- [ ] Giving / payments page loads (admin).
- [ ] Member account logs in and sees only their own data.
- [ ] Log out, then confirm the back button does not show logged-in pages.
- [ ] Forgot password sends an email (check the inbox, not the screen).
- [ ] Render logs show no new errors and no secrets or personal data.

## 5. If something is wrong

Decide fast. Rolling back is cheap; debugging on production is not.

- **Backend broken:** Render dashboard → `faithflow-backend` → **Events** →
  previous successful deploy → **Rollback**. Then revert the merge on `main`
  through a PR so the next deploy does not bring the bug back.
- **Frontend broken:** revert the merge commit on `main` through a PR; the
  deploy workflow republishes the previous build.
- **Data damaged** (bad migration or script): stop and do not run more
  writes. In Neon, restore the production branch to a point in time just
  before the deploy (Neon console → Branches → production → Restore).
  Anything written after that point is lost, so note the time first.
- **Migration already applied and rollback needed:** a code rollback alone
  can leave the schema ahead of the code. Check the migration is reversible
  (`python manage.py migrate_schemas <app> <previous_migration>`) on a Neon
  rehearsal branch before running it on production.

## 6. After

- [ ] Note the release date and what shipped in the release PR.
- [ ] If anything went wrong, write down what and why, and update this
      checklist.
