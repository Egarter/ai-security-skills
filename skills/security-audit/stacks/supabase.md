# Supabase stack guide

Read during step 2 when the project has `supabase/` or `@supabase/supabase-js`. The browser talks to the database directly with the public anon key, so the database rules are the access control. The core questions in SKILL.md still apply.

## Map these

- Every table, view, function and bucket in `supabase/migrations/*.sql`, **and** every table, `.rpc()` and bucket the frontend calls (`grep -rnE "\.from\(|\.rpc\(|storage\.from\(" src app`). A name the app uses but no migration creates was made in the dashboard: its rules are unknown, so mark it needs validation.
- Every folder in `supabase/functions/` and its `verify_jwt` setting in `supabase/config.toml`.
- Auth settings in `supabase/config.toml`: `enable_confirmations`, `enable_anonymous_sign_ins`, `minimum_password_length`.

## What to check

- **Tables:** RLS enabled on each; policies per operation scoped to `auth.uid()` or a membership check; no `USING (true)` on private data; no `TO authenticated USING (true)` as if it meant "owner"; INSERT/UPDATE `WITH CHECK` present.
- **Columns:** an owner UPDATE policy on a row that holds `role`, `plan`, `credits`, `points` or similar lets users edit them. Look for column grants or a separate table.
- **Roles:** policies reading `user_metadata` / `raw_user_meta_data` are user-controlled.
- **Functions:** `SECURITY DEFINER` without a caller check, without `SET search_path`, or without `REVOKE EXECUTE ... FROM PUBLIC, anon`. Functions that take a team or user id must check the caller belongs to it. Functions that move points, credits or stock must check balances and run atomically.
- **Views:** without `security_invoker = true` they skip RLS; any view granted to `anon` must return only public data.
- **Storage:** public buckets serve every file by URL; read policies need an owner or folder check. Judge a bucket by what the app uploads into it (find the `storage.from("<bucket>").upload(` calls and the page that uses them), not by its name or config: user photos, documents, receipts and IDs are personal data even when the bucket is marked public, while logos and catalog images really are public.
- **Edge functions:** `verify_jwt` alone lets anyone with the anon key in; each function must call `auth.getUser()` and check the record or team it acts on. Service-role clients must never act on ids from the body without that check.
- **Anonymous sign-ins** on: every `TO authenticated` policy also applies to throwaway anonymous users.

The `supabase-rls` and `secure-endpoints` skills in this repo carry the fixes.
