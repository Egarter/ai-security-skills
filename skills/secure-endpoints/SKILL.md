---
name: secure-endpoints
description: Make server code check who is calling and what they may do. Use whenever you write or change an API route, server action, Supabase edge function, Firebase/Cloud function, payment or other webhook, email-sending code, redirect, or code that fetches a URL, and when the user asks whether verify_jwt protects edge functions or whether users can change their own plan.
---

# Secure endpoints

Every endpoint the frontend calls can be called directly, without the UI, by anyone who reads the site's code. Hiding a button doesn't protect the action behind it. **The server decides; the browser only asks.**

## Every handler, in this order

1. **Who is calling?** Read the session from the request and refuse without one.

   Supabase edge function:

   ```ts
   const supabase = createClient(
     Deno.env.get("SUPABASE_URL")!,
     Deno.env.get("SUPABASE_ANON_KEY")!,
     { global: { headers: { Authorization: req.headers.get("Authorization") ?? "" } } },
   );
   const { data: { user } } = await supabase.auth.getUser();
   if (!user) return new Response("Unauthorized", { status: 401 });
   ```

   `verify_jwt = true` is **not** enough on its own: the public anon key is a valid JWT, so it lets through anyone who copies it from the site.

2. **May they touch this record?** Being signed in isn't ownership. `/api/orders/123` must refuse other users' orders: load the record and compare its owner to `user.id`, or query through the user's own client so RLS applies.

3. **Take identity and money from the server, never the body.** The user id comes from the session. Prices, discounts, plans and roles come from the database or a server-side price list; the request sends only an item id. Recipients of emails come from your data, not a `to` field.

4. **Then act**, and return a fixed error message on failure (log the details server-side).

## Endpoints that are public on purpose

Contact forms, waitlists, sign-up helpers: add a CAPTCHA or a rate limit, cap field sizes, and never let the caller choose the recipient or the amount.

## Webhooks

Anyone can POST a fake "payment succeeded". Verify the signature over the **raw** body before trusting anything in it:

```ts
const body = await req.text();
const signature = req.headers.get("stripe-signature") ?? "";
let event;
try {
  event = await stripe.webhooks.constructEventAsync(body, signature, Deno.env.get("STRIPE_WEBHOOK_SECRET")!);
} catch {
  return new Response("Bad signature", { status: 400 });
}
```

For Lemon Squeezy, Paddle and others, verify their HMAC with a constant-time compare. Keep the signing secret in an environment variable. Make handlers idempotent (store the event id) so a replayed event doesn't grant credits twice.

## Other things handlers get wrong

- **HTML emails:** escape every request value you put into the HTML (`& < > " '`), and build links on the server. Otherwise anyone can send fake links that look like they come from your app.
- **Redirects:** after login or checkout, only follow same-site paths (starts with `/` and not `//`) or an allowlist of origins; fall back to `/`. For Stripe `success_url` / `return_url`, use a fixed `SITE_URL` env var, never `req.headers.get("origin")`.
- **Fetching URLs the caller gives you:** allow https and a fixed list of hostnames. If users must enter any URL, resolve DNS and refuse private, loopback and link-local addresses (10., 127., 169.254., 172.16–31., 192.168.), and set `redirect: "error"`.
- **SQL and filters:** never concatenate request text into SQL or into PostgREST `.or()` / `.filter()` strings. Use parameters and the query builder's typed methods.
- **Limits:** "5 projects on the free plan" must be enforced by the server or the database (a check or trigger), not by a disabled button.
- **Races and one-time things:** codes, coupons, refunds and credit spends must be consumed atomically (`UPDATE ... WHERE used = false RETURNING`), or two requests at once both succeed.
- **Database (RPC) functions** are endpoints too: see the `supabase-rls` skill.

## Check your work

1. List what the frontend calls: `grep -rnE "/api/|functions\.invoke|\.rpc\(|fetch\(" src app 2>/dev/null`. For each, ask: signed out, or signed in as someone else, does the server refuse?
2. Shippable's code scan flags edge functions with no caller check, unsigned payment webhooks, email HTML injection, request-built redirects, SSRF, client-trusted prices and roles, SQL built from text, and raw errors returned to the caller, then hands your agent a checklist for the logic bugs patterns can't confirm:

   ```sh
   npx -y shippable-dev scan localhost:5173 --no-open
   ```

   Replace the port with the dev server's. For the deployed site, see https://shippable.dev. Only scan apps the user owns.

More: https://shippable.dev/leaks/unprotected-endpoints
