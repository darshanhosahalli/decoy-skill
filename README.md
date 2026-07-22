# decoy-skill

A Claude Code slash command that analyzes your codebase and generates a complete set of API mock presets for the [Decoy](https://github.com/darshanbs/decoy) Chrome extension — covering every scenario your frontend might encounter, in one shot.

---

## What it does

When you run `/decoy-presets` inside Claude Code, it reads your source files, finds every API call, resolves all TypeScript types, traces your response-handling logic, and outputs a single JSON file you can import directly into Decoy.

No backend needed. No manual mock writing. Open the side panel, activate, and every request returns exactly the response you want.

### Scope detection

The skill automatically decides what to analyze based on your git state:

| Situation | What gets analyzed |
|---|---|
| No git repo | Entire `src/` directory |
| On `main` / `master` / `develop` / `trunk` / `staging` | Entire `src/` directory |
| Feature branch | Union of committed changes + staged changes + unstaged changes since branch point |
| Feature branch with no changes yet | Entire `src/` directory (fallback) |

This means it works whether you are mid-work with uncommitted changes, on a clean feature branch, or in a project with no version control at all.

### API call detection

The skill finds HTTP calls made through any of these patterns:

- `fetch(url, { method })` — native fetch
- `axios.get / .post / .put / .patch / .delete(url)` — axios methods
- `axios({ method, url })` — axios config object form
- `api.get / api.post / ...` — any named axios instance
- `useSWR(url, fetcher)` — SWR hooks
- `useQuery({ queryFn })` — TanStack Query / react-query
- `useMutation({ mutationFn })` — TanStack Query mutations
- Custom hooks that wrap any of the above internally

For each call it extracts the URL (normalizing dynamic segments like `/users/${id}` → `/users/:id`), the HTTP method, the request body type, and the response type.

### TypeScript type resolution

For every request and response type, the skill:

- Resolves `interface` and `type` definitions across the codebase
- Distinguishes mandatory fields (no `?`) from optional fields (`?`)
- Generates separate scenarios for `string | null` and `T | undefined` unions
- Generates one scenario per value for string literal unions and enums (`'admin' | 'member' | 'viewer'`)
- Handles array fields — both populated and empty
- Resolves nested types recursively up to 3 levels deep
- Uses realistic placeholder values — real-looking names, emails, dates, and IDs

### Branching logic analysis

After finding each API call, the skill traces the code that handles the response and generates a scenario for each branch:

| Code pattern | Scenario generated |
|---|---|
| `if (res.data.length === 0)` | Empty array response |
| `if (res.status === 404)` | 404 Not Found |
| `switch (res.data.role)` | One entry per switch case |
| `.catch()` / `try-catch` | 500 Server Error |
| `if (!res.ok)` | 400 Bad Request |
| `res.data?.field` (optional chain) | Variant where field is null or missing |
| `if (user.isPremium)` | Both true and false variants |

### Universal scenarios

Regardless of what the code does, every endpoint always gets these 10 presets:

| Scenario | Status | Latency |
|---|---|---|
| Success — full data | 200 | 200ms |
| Empty state | 200 | 200ms |
| Slow response — 30s | 200 | 30,000ms |
| Slow response — 40s | 200 | 40,000ms |
| Slow response — 60s | 200 | 60,000ms |
| 401 Unauthorized | 401 | 100ms |
| 405 Method Not Allowed | 405 | 100ms |
| 429 Too Many Requests | 429 | 100ms |
| 500 Server Error | 500 | 100ms |
| 502 Bad Gateway | 502 | 100ms |

The slow response presets are specifically for testing loading states, skeleton screens, and frontend timeout handling without needing to throttle a real server.

---

## Installation

### Option 1 — Project level (recommended)

Installs the command for one project only. Run from the root of your project:

```bash
mkdir -p .claude/commands
curl -o .claude/commands/decoy-presets.md \
  https://raw.githubusercontent.com/darshanbs/decoy-skill/main/skill/SKILL.md
```

The `/decoy-presets` command is available immediately in Claude Code. Commit the file to share it with your team — everyone on the project gets the command automatically.

### Option 2 — Global (available in every project)

```bash
mkdir -p ~/.claude/commands
curl -o ~/.claude/commands/decoy-presets.md \
  https://raw.githubusercontent.com/darshanbs/decoy-skill/main/skill/SKILL.md
```

### Option 3 — Manual

1. Open [`skill/SKILL.md`](./skill/SKILL.md) in this repo
2. Click **Raw**
3. Copy all the content
4. Create `.claude/commands/decoy-presets.md` in your project
5. Paste and save

---

## Usage

1. Open your project in Claude Code
2. Run the command:
   ```
   /decoy-presets
   ```
3. Claude analyzes the codebase and outputs a JSON block
4. Save the JSON as `decoy-presets.json`
5. Open the Decoy extension → **Presets** tab → **Upload** → select the file
6. All scenarios appear as presets ready to activate

---

## Updating

Re-run the same curl command to get the latest version:

```bash
curl -o .claude/commands/decoy-presets.md \
  https://raw.githubusercontent.com/darshanbs/decoy-skill/main/skill/SKILL.md
```

---

## Requirements

- [Claude Code](https://claude.ai/code) — the CLI or IDE extension
- [Decoy Chrome extension](https://chromewebstore.google.com/detail/decoy) — to import and activate the generated presets
- A TypeScript or JavaScript project (React, Next.js, Vue, plain TS — anything with API calls in `src/`)

---

## Output format

The generated JSON matches Decoy's preset import format exactly:

```json
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
      "savedAt": "2026-07-22T10:00:00.000Z",
      "expiresAt": 9999999999999
    }
  ]
}
```

---

## Repository structure

```
skill/
├── SKILL.md          ← the slash command — place this in .claude/commands/
└── installer/
    └── SKILL.md      ← installer skill for the claude.ai skills directory
```
