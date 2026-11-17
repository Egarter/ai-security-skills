# Next.js stack guide

Read during step 2 when `next` is in package.json. It adds where to look and what AI-built Next.js apps get wrong; the core questions in SKILL.md still apply.

## Map these entry points

Every one of these runs on the server and can be called directly by anyone, without your UI. Each is a ledger row.

```sh
# Route handlers and API routes
find . -path ./node_modules -prune -o \( -path '*/app/*' -name 'route.[jt]s' -o -path '*/pages/api/*' \) -print
# Server actions: files or functions marked "use server"
grep -rln --include='*.ts' --include='*.tsx' -E "['\"]use server['\"]" --exclude-dir=node_modules .
# Middleware (proxy.ts since Next.js 16) and its matcher
ls middleware.* src/middleware.* proxy.* src/proxy.* 2>/dev/null
# Where the session is read
grep -rn --include='*.ts' --include='*.tsx' -E "getServerSession|auth\(\)|currentUser\(|getSession\(|cookies\(\)" --exclude-dir=node_modules . | head -50
```

Also list: pages under an `admin` segment, `cron`/`jobs`/`webhook` routes, upload handlers, and anything that calls an LLM, email or SMS provider.

## What to check

**1. Server actions are public endpoints.** A `"use server"` function becomes a POST endpoint the browser calls by id. Hiding the button, or only rendering the form for admins, doesn't stop anyone calling it with any arguments. Every action must read the session itself, check the role and the record's org/owner, and validate its arguments. An action used only from an admin page is the classic miss.

**2. Middleware is not your auth.** Treat middleware/proxy as a convenience redirect, not the access check:
- Check the `matcher`: paths it leaves out (often `/api/*`, `/_next`, file-extension patterns, a new route someone added later) get no check at all.
- Next.js before 15.2.3 / 14.2.25 / 13.5.9 / 12.3.5 lets a request skip middleware entirely with the `x-middleware-subrequest` header (CVE-2025-29927). Check the installed version in the lockfile.
- A check in `layout.tsx` doesn't protect route handlers or server actions, and isn't re-run on every navigation. Each route handler, action and page that reads data must check on its own, ideally through one data-access function that requires a session.

**3. Ownership and tenant scoping (IDOR).** `/api/projects/[id]` and actions taking an `id` must load the record scoped to the caller: `where: { id, orgId: session.orgId }` (Prisma `findFirst`), not `findUnique({ where: { id } })` followed by nothing. The org id comes from the session or a membership lookup, never from the URL, body, a header or a cookie the client sets. Check update and delete paths as carefully as reads; they are missed more often.

**4. Mass assignment.** `prisma.x.update({ data: body })`, `data: { ...body }`, or `create({ data: await req.json() })` lets a caller set any column: `role`, `orgId`, `ownerId`, `plan`, `credits`, `emailVerified`. Look for a schema (zod `.strict()` or an explicit field pick) between the request and the ORM. Same for Drizzle `.set(body)` and Mongoose `findByIdAndUpdate(id, body)`.

**5. Data sent to the client.** Props passed from a server component to a `"use client"` component are serialized into the page HTML. Passing a whole user or org row ships `passwordHash`, reset tokens, API keys, other members' emails. Look for `findMany()`/`findUnique()` without `select` whose result goes to a client component or `Response.json()`. Secrets must only be read in files that import `server-only`; anything prefixed `NEXT_PUBLIC_` is public.

**6. Caching per-user data.** A per-user or per-org response in a cached function (`unstable_cache`, `"use cache"`, `fetch` with `force-cache` or `revalidate`) or a statically rendered route can be served to the next visitor. Check that cache keys include the user/org, or that personal routes are dynamic.

**7. Raw SQL.** `$queryRawUnsafe`, `$executeRawUnsafe` and `Prisma.raw()` with string interpolation are injectable. The tagged-template `$queryRaw\`... ${x}\`` is parameterized and safe.

**8. Auth library setup (Auth.js / NextAuth, Clerk, Lucia, custom).**
- `AUTH_SECRET` / `NEXTAUTH_SECRET` set from the environment, not a literal or fallback.
- Role and org in the session: where do they come from? A `jwt`/`session` callback that copies a value the user can edit (their profile, `user_metadata`) makes the role user-controlled.
- Stale authority: any session that stores role, plan or org at login (a JWT, a signed or encrypted cookie) keeps them after the user is demoted or removed, until it expires (or forever, with no expiry). Find each admin or owner check that reads `session.role` / `session.orgId` and confirm it re-reads the membership row instead.
- Credentials `authorize()`: password hashing (bcrypt/argon2, not plain or fast hashes), rate limiting, same error for unknown email and wrong password.
- `allowDangerousEmailAccountLinking: true` lets anyone who controls a matching email on another provider take the account.
- Custom sessions: signed or encrypted cookie (`iron-session`, `jose`), `httpOnly`, `secure`, `sameSite`. A cookie holding `userId` or `role` in plain text is an account takeover.

**9. Redirects.** `redirect(searchParams.get("next"))`, `callbackUrl`, `returnTo`: allow only same-site paths (start with `/`, not `//`).

**10. Uploads.** Writing to `public/` or serving uploads from your own origin lets an uploaded `.html` or `.svg` run scripts as your site. `path.join(dir, file.name)` with the client's filename allows `../` traversal. Check server-side size and type limits and who may read each file.

**11. Webhooks and cron.** Route handlers must verify the signature over `await req.text()` (not `req.json()`, which loses the raw body), and dedupe event ids. Cron routes (`/api/cron/*`) must check `Authorization: Bearer ${process.env.CRON_SECRET}` or equivalent; otherwise anyone can trigger the job.

**12. Config.** `next.config`: `images.remotePatterns` with `hostname: "**"` turns the image optimizer into a fetch-anything proxy you pay for; `productionBrowserSourceMaps: true` ships source; check `headers()` for a CSP and frame policy. `dangerouslySetInnerHTML` with user or AI text needs a sanitizer (DOMPurify).

## Typical fixes

- One `requireUser()` / `requireMember(orgId, role)` helper in a `server-only` data-access file, called first in every route handler, action and data function.
- Prisma queries always carry the scope: `findFirst({ where: { id, orgId } })`, `updateMany({ where: { id, orgId }, data })` and check the count.
- A zod schema per input, `.strict()`, and an explicit `select` on every query whose result leaves the server.
