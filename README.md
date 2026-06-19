# secure-code-review

> An LLM skill that enforces 6 production security rules while you build — not just when you review.

Most AI-generated code works on the happy path. It breaks in production.  
This skill makes any LLM enforce security and reliability standards on every line it writes — automatically, without being asked.

Inspired by a real audit of 25 AI-built apps. Every single one had at least 3 of these issues.

---

## The 6 Rules

| # | Rule | What breaks without it |
|---|------|------------------------|
| 1 | Never commit secrets | Leaked API keys → exposed DB or $3,000 bill |
| 2 | RLS must be explicit, not broad | Users read each other's private data |
| 3 | Rate limit every endpoint | One script takes your whole app down |
| 4 | Handle errors beyond the happy path | Silent failures, zero logs, broken UI |
| 5 | No N+1 queries or DB calls in loops | Works at 10 users, collapses at 1,000 |
| 6 | Authorization, not just authentication | Anyone accesses anyone's data by changing an ID |

---

## How to Use

### Claude Code

Create a `CLAUDE.md` file in your project root:

```bash
curl -o CLAUDE.md https://raw.githubusercontent.com/sadidgit01/secure-code-review/main/SKILL.md
```

Claude Code reads `CLAUDE.md` automatically on every session. All 6 rules apply to everything it builds.

### Cursor / Windsurf

Create a `.cursorrules` file in your project root:

```bash
curl -o .cursorrules https://raw.githubusercontent.com/sadidgit01/secure-code-review/main/SKILL.md
```

### Claude.ai Projects

1. Open Claude.ai → **Projects** → your project
2. Click **Set project instructions**
3. Paste the contents of `SKILL.md`

Every conversation in that project enforces all 6 rules.

### OpenAI Codex / ChatGPT / Any LLM API

Pass `SKILL.md` as your system prompt:

```python
import anthropic

skill = open("SKILL.md").read()
client = anthropic.Anthropic()

response = client.messages.create(
    model="claude-opus-4-6",
    max_tokens=2048,
    system=skill,
    messages=[{"role": "user", "content": "Build me a user auth API"}]
)
```

```js
import Anthropic from "@anthropic-ai/sdk";
import { readFileSync } from "fs";

const client = new Anthropic();
const response = await client.messages.create({
  model: "claude-opus-4-6",
  max_tokens: 2048,
  system: readFileSync("SKILL.md", "utf-8"),
  messages: [{ role: "user", content: "Build me a user auth API" }],
});
```

---

## What It Looks Like

When auditing existing code, findings are reported like this:

```
CRITICAL   Rule 1 — Supabase key hardcoded in db.js:12
WARNING    Rule 3 — /api/generate has no rate limiter
WARNING    Rule 5 — N+1 query in getPostsWithAuthors() (posts.js:34)
INFO       Rule 2 — RLS enabled, policies look correct
```

When building, it applies the rules silently — `.gitignore` gets created, rate limiters get added, ownership checks get written — without you asking.

---

## What's in This Repo

```
secure-code-review/
├── SKILL.md    ← feed this to your LLM
└── README.md
```

No install. No dependencies. One file.

---

## Contributing

Found a missing rule or a better example for a specific stack? PRs welcome.  
Keep it focused — one rule, one bad example, one good example, one checklist.

---

## License

MIT — use it, fork it, embed it anywhere.

---

Built by [Saif Mahmood Chowdhury](https://github.com/sadidgit01)
