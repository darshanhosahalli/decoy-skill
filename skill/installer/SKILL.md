---
name: decoy-install
description: Installs the /decoy-presets command into your project. Run once inside your repo — Claude Code will write the command file and it becomes permanently available as /decoy-presets in this project.
---

# Decoy Preset Installer

Install the Decoy preset generator as a permanent Claude Code command in the current repository.

## What this does

Writes `.claude/commands/decoy-presets.md` into the current project. After installation, `/decoy-presets` becomes a Claude Code slash command that analyzes your codebase and generates a Decoy mock preset JSON ready to import into the Decoy Chrome extension.

## Steps

1. Check whether `.claude/commands/` exists in the current working directory. If not, create it.

2. Write the following content exactly as-is to `.claude/commands/decoy-presets.md`:

```markdown
---
description: Analyzes your codebase or branch diff and generates a Decoy preset JSON covering every API scenario — success, errors, edge cases, enum variants, optional fields. Import the output directly into the Decoy Chrome extension via Presets → Import.
---

# Decoy Preset Generator

Generate a Decoy-compatible preset file by analyzing API calls and TypeScript types in the current repository.

## Step 1 — Determine analysis scope

First check whether a git repository exists:
\`\`\`bash
git rev-parse --git-dir 2>/dev/null
\`\`\`

**If the command fails (no git repo):**
- Analyze the **entire codebase** — all `.ts`, `.tsx`, `.js`, `.jsx` files under `src/`
- Skip all remaining git steps and go to Step 2

**If git repo exists**, get the current branch:
\`\`\`bash
git rev-parse --abbrev-ref HEAD
\`\`\`

If the branch name is `main`, `master`, `develop`, `trunk`, or `staging`:
- Analyze the **entire codebase** — all `.ts`, `.tsx`, `.js`, `.jsx` files under `src/`
- Skip remaining git steps and go to Step 2

Otherwise (feature branch), collect files from three sources and analyze their union:

1. **Committed changes** since branch point:
   \`\`\`bash
   git merge-base HEAD $(git remote show origin | grep 'HEAD branch' | awk '{print $NF}')
   git diff <merge-base>...HEAD --name-only
   \`\`\`

2. **Staged (uncommitted) changes:**
   \`\`\`bash
   git diff --cached --name-only
   \`\`\`

3. **Unstaged changes:**
   \`\`\`bash
   git diff --name-only
   \`\`\`

Deduplicate the three lists. Analyze only files with `.ts`, `.tsx`, `.js`, or `.jsx` extensions.

If the combined list is empty (brand new branch with no changes yet) — fall back to analyzing the entire `src/` directory.

## Step 2 — Extract every API call

In each file, find every HTTP call made via:
- `fetch(url, { method })`
- `axios.get/post/put/patch/delete(url)`
- `axios({ method, url })`
- `api.get/post/...` (any axios instance)
- `useSWR`, `useQuery`, `react-query` hooks wrapping a fetch
- Any custom hook whose name starts with `use` and calls one of the above internally

For each call extract:
- **URL**: the full string or template literal. Normalize dynamic segments (`/users/${id}` → `/users/:id`)
- **Method**: GET/POST/PUT/PATCH/DELETE — default to GET if not specified
- **Request body type**: the TypeScript generic or variable passed as body/data
- **Response type**: the TypeScript generic — e.g. `axios.get<User[]>(url)` → `User[]`

## Step 3 — Resolve TypeScript types

For every request and response type found:

1. Find the `interface` or `type` definition in the codebase
2. For each field:
   - **Mandatory**: no `?` suffix → always include in generated JSON
   - **Optional**: `?` suffix → generate two variants: one with it, one without
3. For `string | null` or `T | undefined` union types → generate both filled and null variants
4. For fields typed as an `enum` or string literal union (`'admin' | 'viewer' | 'editor'`):
   - Generate **one scenario per possible value**
5. For array fields → generate both a populated array and an empty array scenario
6. For nested types → resolve recursively, max 3 levels deep
7. Use realistic placeholder values — not `"string"` or `0`. Use names, emails, dates, IDs that look real

## Step 4 — Trace branching logic

After the API call, trace the code that handles the response. Look for:

| Pattern | Scenario to generate |
|---|---|
| `if (res.data.length === 0)` | empty array scenario |
| `if (res.status === 404)` | 404 scenario |
| `switch (res.data.role)` | one scenario per case value |
| `.catch()` / `try-catch` | 500 error scenario |
| `if (!res.ok)` | 400 error scenario |
| `res.data?.field` (optional chain) | scenario where field is null/missing |
| `if (user.isPremium)` | both true and false variants |

Each distinct branch = one additional scenario.

## Step 5 — Build the scenario list

For every endpoint, always include at minimum:

1. **Success** — full realistic data, all mandatory fields, optional fields included
2. **Empty state** — empty arrays, null optional fields, zero counts
3. **Not found** — status 404, `{ "error": "Not found" }`
4. **Server error** — status 500, `{ "error": "Internal server error" }`
5. **One scenario per enum value** found in the response type
6. **One scenario per branching condition** found in the response handler

## Step 6 — Output the preset file

Output a JSON object in this exact format that Decoy can import directly:

\`\`\`json
{
  "__decoy_type__": "preset",
  "entries": [
    {
      "domain": "",
      "title": "GET /api/users — success (admin)",
      "description": "All users returned, role=admin variant",
      "url": "/api/users",
      "method": "GET",
      "status": 200,
      "latency": 200,
      "requestHeaders": "{}",
      "requestPayload": "",
      "responseHeaders": "{\"Content-Type\":\"application/json\"}",
      "responsePayload": "[{\"id\":1,\"name\":\"Alice\",\"email\":\"alice@example.com\",\"role\":\"admin\"}]",
      "raw": "",
      "savedAt": "<current ISO timestamp>",
      "expiresAt": 9999999999999
    }
  ]
}
\`\`\`

Rules:
- `domain`: always empty string — Decoy fills this on import from the active tab
- `title`: `"METHOD /path — scenario name"` — be specific, mention the variant
- `description`: one sentence explaining what this scenario tests
- `responsePayload`: must be a **JSON string** (the entire value is a string, not a nested object)
- `responseHeaders`: always `"{\"Content-Type\":\"application/json\"}"` unless the endpoint returns something else
- `requestPayload`: empty string for GET/DELETE, JSON string of the request body for POST/PUT/PATCH
- `latency`: 200ms for success responses, 100ms for error responses
- `savedAt`: use the current date/time in ISO 8601 format
- `expiresAt`: always `9999999999999` (far future — presets don't expire)

## Step 7 — Final instruction to user

After outputting the JSON, tell the user:

> Save this as `decoy-presets.json`, then in Decoy open the **Presets** tab → click **Import** → select the file. All scenarios will appear as presets ready to activate.
```

3. After writing the file, tell the user:

> ✓ Installed. `/decoy-presets` is now available as a Claude Code command in this project.
> Run `/decoy-presets` in Claude Code to generate mock presets from your codebase.
> 
> To make it available globally across all your projects, move the file to:
> `~/.claude/commands/decoy-presets.md`
