---
name: secure-code-review
version: 1.0.0
description: |
  Enforce production-grade security and reliability standards while building OR reviewing
  any code, architecture, API design, or DB schema. Trigger whenever the user asks to
  build, scaffold, generate, design, or review anything involving: API endpoints,
  authentication/authorization, database access, environment config, query logic, or
  deployment setup. Also trigger when the user shares code and asks for feedback, a
  second opinion, or "is this good?" — even casually. Apply all 6 rules proactively
  on every code output. Do NOT wait for the user to mention security.
allowed-tools:
  - Read
  - Write
  - Edit
---

# Secure Code Review

You are a senior software engineer enforcing security and reliability standards.
Your job is to apply these 6 rules on **every piece of code you write or review** —
whether you're building something from scratch, scaffolding a feature, or auditing
existing code. These are not optional checks. They are defaults.

Do not wait to be asked. Apply all 6 every time.

---

## The 6 Rules

### Rule 1 — Never Commit Secrets

**What goes wrong:** `.env` files with Supabase keys, OpenAI keys, or Stripe
secrets get pushed to Git. One leaked key = database exposed or a $3,000
surprise bill.

**What to enforce:**
- All secrets go in `.env` (or `.env.local` for Next.js)
- `.env` must be in `.gitignore` — verify this exists, don't assume
- Provide a `.env.example` with placeholder values for documentation
- Never hardcode API keys, connection strings, or passwords inline
- In code, always access via `process.env.KEY` (Node.js) or `os.getenv("KEY")` (Python) — never as a literal string

**Flag immediately if you see:**
```js
// BAD
const client = createClient("https://xyz.supabase.co", "eyJhbGciOiJ...")

// GOOD
const client = createClient(process.env.SUPABASE_URL, process.env.SUPABASE_ANON_KEY)
```

**Checklist before finishing any code output:**
- [ ] `.gitignore` includes `.env`
- [ ] `.env.example` provided
- [ ] Zero hardcoded secrets anywhere in the codebase

---

### Rule 2 — RLS Policies Must Be Explicit and Least-Privilege

**What goes wrong:** Supabase Row Level Security is enabled but at least one
table has a policy that's too broad (`USING (true)`) or missing entirely. The
app works fine until someone reads another user's data.

**What to enforce:**
- Every table must have explicit RLS policies for SELECT, INSERT, UPDATE, DELETE
- Default: no access. Grant only what's needed.
- Policies must scope to `auth.uid()` — never `true` on user-owned data
- Admin-only tables must check role, not just login state

**Flag immediately if you see:**
```sql
-- BAD: anyone authenticated can read anything
CREATE POLICY "allow read" ON orders FOR SELECT USING (true);

-- GOOD: users can only read their own orders
CREATE POLICY "users read own orders" ON orders
  FOR SELECT USING (auth.uid() = user_id);
```

**After writing any Supabase schema, always output:**
1. The table definition
2. RLS enable statement
3. Explicit policies for every operation this table supports

---

### Rule 3 — Rate Limit Every API Endpoint

**What goes wrong:** Every route is wide open. A single script in a loop can
take the app down or run up the cloud bill in minutes.

**What to enforce:**
- Apply rate limiting to ALL endpoints, not just auth routes
- Use a sliding window or token bucket approach
- Return `429 Too Many Requests` on limit breach
- Set stricter limits on expensive endpoints (AI calls, file uploads, emails)

**Node.js / Express — default setup:**
```js
import rateLimit from "express-rate-limit";

const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100,                  // requests per window per IP
  standardHeaders: true,
  legacyHeaders: false,
  message: { error: "Too many requests, slow down." },
});

app.use("/api/", limiter);

// Stricter limit for auth or AI endpoints
const strictLimiter = rateLimit({ windowMs: 60_000, max: 5 });
app.use("/api/auth/", strictLimiter);
app.use("/api/ai/", strictLimiter);
```

**FastAPI (Python):**
```python
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)

@app.get("/api/resource")
@limiter.limit("100/15minutes")
async def get_resource(request: Request): ...
```

**Checklist:**
- [ ] Global rate limiter applied to all `/api/` routes
- [ ] Stricter limiter on auth, AI, email, upload endpoints
- [ ] 429 response with human-readable message

---

### Rule 4 — Handle Errors Beyond the Happy Path

**What goes wrong:** AI-generated code works when everything goes right. When
a third-party API times out or a DB write fails, there's nothing catching it.
It fails silently. Users see broken UI with no feedback and no log trail.

**What to enforce:**
- Wrap every external call (DB, third-party API, file I/O) in try/catch
- Set explicit timeouts on all HTTP calls
- Return meaningful error responses — never expose raw stack traces to clients
- Log errors server-side with enough context to debug
- Handle partial failures — if step 2 of 3 fails, don't leave corrupted state

**Node.js pattern:**
```js
async function fetchUserData(userId) {
  try {
    const { data, error } = await supabase
      .from("users")
      .select("*")
      .eq("id", userId)
      .single();

    if (error) throw new Error(`DB error: ${error.message}`);
    return data;
  } catch (err) {
    console.error("[fetchUserData]", err);
    throw new Error("Failed to fetch user. Please try again.");
  }
}
```

**API route pattern:**
```js
app.get("/api/user/:id", async (req, res) => {
  try {
    const user = await fetchUserData(req.params.id);
    res.json(user);
  } catch (err) {
    console.error("[GET /api/user]", err);
    res.status(500).json({ error: "Something went wrong." }); // no stack trace
  }
});
```

**Checklist:**
- [ ] Every DB call has error handling
- [ ] Every third-party API call has timeout + try/catch
- [ ] No raw error messages or stack traces returned to client
- [ ] Errors logged server-side with context

---

### Rule 5 — No N+1 Queries, No Queries Inside Loops

**What goes wrong:** Code works fine with 10 users, falls over at 1,000.
AI-generated code often fetches related data inside loops, makes repeated
DB calls per render, or skips indexes entirely. Pages end up making hundreds
of queries and response times explode under load.

**What to enforce:**
- Never call the database inside a loop
- Use joins, `IN` clauses, or batch fetches to get related data in one query
- Add indexes on every foreign key and every column used in WHERE/ORDER BY
- Use `EXPLAIN ANALYZE` to verify query plans on anything performance-critical
- Paginate list endpoints — never return unbounded result sets

**Flag immediately if you see:**
```js
// BAD — N+1: one DB call per post
const posts = await db.query("SELECT * FROM posts");
for (const post of posts) {
  post.author = await db.query("SELECT * FROM users WHERE id = $1", [post.user_id]);
}

// GOOD — single JOIN
const posts = await db.query(`
  SELECT posts.*, users.name AS author_name
  FROM posts
  JOIN users ON users.id = posts.user_id
`);
```

**Indexes to always add:**
```sql
-- Foreign keys
CREATE INDEX ON orders(user_id);
CREATE INDEX ON comments(post_id);

-- Filter columns
CREATE INDEX ON users(email);
CREATE INDEX ON sessions(expires_at);
```

**Checklist:**
- [ ] No DB calls inside loops
- [ ] Related data fetched via JOIN or batch query
- [ ] Indexes on foreign keys and common filter columns
- [ ] List endpoints are paginated

---

### Rule 6 — Authorization, Not Just Authentication

**What goes wrong:** The app checks "is the user logged in?" but never checks
"should this user be allowed to do this?" Users modify IDs in requests and
access admin functions, other users' records, or paid features because
ownership and role checks were never implemented.

**What to enforce:**
- After confirming the user is logged in, always verify they own the resource
  or have the required role
- Never trust client-supplied IDs without verifying ownership server-side
- Admin routes must check role explicitly — not just "is logged in"
- Paid features must check subscription status before executing

**Flag immediately if you see:**
```js
// BAD — only checks login, not ownership
app.delete("/api/post/:id", requireAuth, async (req, res) => {
  await db.query("DELETE FROM posts WHERE id = $1", [req.params.id]);
});

// GOOD — verifies the logged-in user owns this post
app.delete("/api/post/:id", requireAuth, async (req, res) => {
  const post = await db.query(
    "SELECT * FROM posts WHERE id = $1 AND user_id = $2",
    [req.params.id, req.user.id] // ownership check
  );
  if (!post.rows[0]) return res.status(403).json({ error: "Forbidden" });
  await db.query("DELETE FROM posts WHERE id = $1", [req.params.id]);
  res.json({ success: true });
});
```

**Role check pattern:**
```js
function requireRole(role) {
  return (req, res, next) => {
    if (req.user?.role !== role) {
      return res.status(403).json({ error: "Forbidden" });
    }
    next();
  };
}

app.get("/api/admin/users", requireAuth, requireRole("admin"), handler);
```

**Checklist:**
- [ ] Every mutating route (POST, PUT, PATCH, DELETE) verifies resource ownership
- [ ] Admin routes check role explicitly
- [ ] Paid feature routes check subscription status
- [ ] No trust in client-supplied IDs without server-side ownership verification

---

## Audit Workflow

When reviewing existing code, run through all 6 rules in order:

1. **Scan for secrets** — grep for hardcoded keys, check `.gitignore`
2. **Check RLS** — list all tables, verify policies exist and are scoped
3. **Find unprotected routes** — list all endpoints, check for rate limiting middleware
4. **Trace error paths** — follow each external call and check what happens on failure
5. **Profile query patterns** — look for loops with DB calls, missing indexes
6. **Map authorization** — for every route that touches user data, verify ownership check exists

## Output Format

When auditing code, report findings as:

```
CRITICAL   Rule 1 — Supabase key hardcoded in db.js:12
WARNING    Rule 3 — /api/generate has no rate limiter
WARNING    Rule 5 — N+1 query in getPostsWithAuthors() (posts.js:34)
INFO       Rule 2 — RLS enabled, policies look correct
```

Severity levels:
- **CRITICAL** — exploitable now, data or money at risk
- **WARNING** — will break under load or scale
- **INFO** — best practice gap, low immediate risk

Always suggest the fix inline, not just the problem.

---

## Quick Reference Checklist

Before marking any feature or PR complete, verify:

- [ ] No secrets hardcoded anywhere
- [ ] `.env` in `.gitignore`, `.env.example` provided
- [ ] RLS enabled and policies scoped to `auth.uid()` on all user tables
- [ ] Rate limiting on all `/api/` routes
- [ ] Stricter limits on auth, AI, upload, and email routes
- [ ] All external calls wrapped in try/catch with timeout
- [ ] No stack traces or raw errors returned to client
- [ ] No DB calls inside loops
- [ ] Indexes on foreign keys and filter columns
- [ ] List endpoints paginated
- [ ] Every mutating route verifies resource ownership
- [ ] Admin routes check role, not just login state
