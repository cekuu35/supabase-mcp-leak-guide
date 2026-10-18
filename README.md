# The service_role Key Hiding in Your .mcp.json — the AI-Vibe-Coder Secret Leak

A year ago, the classic Supabase leak was a `.env` file committed to a public repo. Everyone learned to gitignore `.env`. Scanners learned to flag it. We mostly moved on.

But the way people build changed. Now you wire Supabase into Cursor, Claude Code, Windsurf, or VS Code through an MCP server, paste a key into a JSON config so your AI assistant can talk to your database, and keep shipping. That config file *feels* like an editor setting, not a secret. So it gets committed. And it's carrying the one credential that makes Row Level Security irrelevant.

This post is about that new leak surface — where it hides, why it's worse than a `.env` slip, and the exact order you fix it in. (Order matters more than you'd think.)

## First, get the two keys straight

Supabase gives you two classes of API key, and the whole security story hinges on telling them apart.

**Publishable / anon keys** (`sb_publishable_...`, or the legacy `anon` JWT) are **public by design**. They are meant to ship in your browser bundle. Anyone can read them out of your JavaScript, and that is fine — because every request they make is still filtered through Row Level Security. The key identifies your project; RLS decides what that request is allowed to see. A publishable key with good RLS behind it is not a leak. It's the front door, and the front door is supposed to be visible.

**Secret / service_role keys** (`sb_secret_...`, or the legacy `service_role` JWT) are the opposite. Straight from the Supabase docs, they provide *full access* to your project's data, **bypassing Row Level Security entirely**. This key is a backend-only credential — for servers, Edge Functions, admin panels. It does not care what policies you wrote. It reads every row in every table, writes anything, deletes anything.

So here's the painful irony: developers panic when they realize their anon key is visible in the browser (it's fine), while the key that actually ends up in public places is the service_role key. And the newest place it ends up is your AI tooling config.

## Where it actually hides now

When you connect Supabase to an AI coding assistant, the credential goes into a JSON file that lives *inside your repo*:

- `.mcp.json` (Claude Code, project root)
- `.cursor/mcp.json` (Cursor)
- `.vscode/mcp.json` (VS Code)
- `~/.codeium/windsurf/mcp_config.json` (Windsurf)
- any `mcp.json` / `mcp-config.json` a tutorial told you to create

A typical entry looks like this:

```json
{
  "mcpServers": {
    "supabase": {
      "command": "npx",
      "args": ["-y", "@supabase/mcp-server-supabase"],
      "env": {
        "SUPABASE_ACCESS_TOKEN": "sbp_0a1b2c3d...",
        "SUPABASE_SERVICE_ROLE_KEY": "sb_secret_9f8e7d..."
      }
    }
  }
}
```

There are actually **three** different disasters that show up in these files:

1. **`sb_secret_...` / service_role** — bypasses RLS, full data access.
2. **A Supabase personal access token (`sbp_...`)** — controls your whole account through the Management API: it can list projects, run SQL, even create or pause projects. This is arguably *worse* than a single project's service_role key.
3. **A Postgres connection string** — `postgresql://postgres:PASSWORD@db.ref.supabase.co:5432/postgres`. That's your database password in plaintext, i.e. direct superuser access to the box.

All three routinely land in MCP config because that's exactly what the quickstarts tell you to paste in.

## Why this is worse than the old `.env` mistake

Three reasons.

**It doesn't look like a secret.** `.env` screams "credentials." `.cursor/mcp.json` reads like "my editor preferences." People commit editor config without a second thought — and often *want* to commit it so teammates get the same setup.

**The tooling gap.** Plenty of repos have `.env` in `.gitignore` and nothing else. `.mcp.json`, `.cursor/`, and `.vscode/` are frequently tracked on purpose. Your existing ignore rules do not cover them.

**It's already in your git history.** This is the part people miss. Even if you delete the key from the current file and commit the fix, the secret still sits in a past commit. `git log`, any fork, any clone, GitHub's cached views, and every secret-scanning bot that already crawled your repo still have it. Deleting a secret from the latest commit does **not** un-leak it.

## Check yourself in 30 seconds

From your repo root:

```bash
# Is a secret sitting in your history right now?
git log -p -S 'sb_secret_' -- . | head
git log -p -S 'service_role' -- . | head
git log -p -S 'sbp_' -- . | head

# Are the AI config files even tracked?
git ls-files | grep -E '(^|/)(\.mcp\.json|\.cursor/|\.vscode/mcp\.json)'
```

For a real audit, run a proper scanner over the whole history — `gitleaks detect` or `trufflehog git file://.` will find these across every commit, not just HEAD.

## Fix it in the right order

If you found something, do these steps **in this order**. The order is the whole point.

**1. Rotate first — before anything else.** In the Supabase dashboard: for API keys, revoke the exposed secret key and create a new one; for a personal access token, revoke it under Account → Access Tokens; for a leaked DB password, reset it under Project Settings → Database. Rotation is what actually protects you, because it invalidates the leaked credential everywhere at once. Cleaning git history without rotating is theater — the secret was already public.

**2. Then purge it from history.** Now that the key is dead, scrub the value so bots stop flagging your repo: `git filter-repo --replace-text` or the BFG Repo-Cleaner, then force-push. (Coordinate with collaborators — this rewrites history.)

**3. Stop it happening again.** Add the config paths to `.gitignore`:

```gitignore
.mcp.json
.cursor/
.vscode/mcp.json
.env*
```

Commit a `.mcp.json.example` with `${SUPABASE_ACCESS_TOKEN}` placeholders instead of real values, and keep the real ones in your shell environment or a secrets manager.

**4. Reduce the blast radius.** The Supabase MCP server supports `--read-only` and `--project-ref` flags. Scope the token to a single project and read-only access so that even a future leak can't write or delete. Give your AI assistant the least privilege that still lets it help.

## The takeaway

The anon key in your browser was never the problem. The service_role key in your `.cursor/mcp.json` is. RLS is a genuinely strong wall — but service_role walks straight through it, and AI tooling has quietly created a brand-new way to commit that key to a public repo without realizing it.

So: audit your history, rotate anything you find, *then* clean up. In that order.

---

If you want to actually *see* this happen end to end, I put together a small reproducible demo — a repo with a service_role key wired into an MCP config, and a walkthrough of exactly what an attacker can pull once RLS is out of the picture and how the rotate-then-purge fix plays out: **[github.com/cekuu35/supabase-rls-leak-demo](https://github.com/cekuu35/supabase-rls-leak-demo)**. It's free, no signup.

And if you'd rather run a full pass over your own project — RLS policies, key placement, and config-file leaks like the one above — I keep a paid **[Supabase RLS Audit Kit](https://cengokurtoglu.gumroad.com/l/supabase-rls-audit-kit)** ($29) with the checklist and test scripts I use. Totally optional; the demo repo above already covers the leak in this post.

## Free pre-launch checklist

10 checks that catch the common Next.js + Supabase launch mistakes (free, instant download): https://cengokurtoglu.gumroad.com/l/nextjs-supabase-10-checks-free

More security tooling and review kits: https://cengokurtoglu.gumroad.com
