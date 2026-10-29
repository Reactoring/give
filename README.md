# Give

**A Claude Code skill that checks and acts on any system, through the cheapest path that works.**

Give is the method I built so Claude Code can verify scenarios and take actions on GitHub, Google Cloud, ServiceNow, internal portals or a local app on any port, without loading a single MCP server. It is a skill: one `SKILL.md`, a few Node scripts, and a catalogue that grows with use.

---

## Why not MCP

| MCP browser / service servers | Give |
|---|---|
| Every tool schema sits in the context, every session, used or not | One line of description until the skill is triggered |
| One tool call per click, type, screenshot | One script runs a whole scenario in one call |
| Returns full pages, accessibility trees, screenshots | Returns one JSON line: only what failed and what was asked |
| On failure, the agent takes a screenshot to understand | Errors list what *is* available ("button not found; available: Save, Cancel…") |
| Browser first, even when an API exists | CLI first, then API, then browser |
| A server to install, run, update and trust with credentials | Plain scripts, versioned, readable, nothing to run |
| Every session rediscovers the same flow | Working flows are saved as scripts and reused |

MCP gives the agent many generic tools. Give gives it a decision rule and a few sharp ones.

---

## Strategy: the cheapest path first

```
Request
  │
  ├─ 1. CLI exists and is logged in?   gh, gcloud, bq …          → use it
  ├─ 2. API exists?                    http.mjs / curl           → use it
  └─ 3. Only visible in the UI?        private Chrome + script   → use it
```

- **CLI** is the fastest and most reliable: already authenticated, structured output, no parsing.
- **API** when there is no CLI. `http.mjs` keeps the output compact: `--pick result[].number,result[].state` returns two fields instead of a 20 KB payload.
- **Browser** only when the thing to check *is* the screen: a form, a user journey, a render, or because the user asked.

Paths can be combined: create test data through the API, then check the screen in the browser.

---

## Architecture

```
SKILL.md                    decision rule, safety rules, how to report
bin/
  chrome.mjs                open / reuse / list the private test Chrome
  run.mjs                   run a browser script, return one JSON line
  http.mjs                  compact HTTP calls, auth from env vars
  list.mjs                  catalogue built from script headers
lib/
  browser.mjs               connection, tabs, helpers, noise filter
scripts/
  browser/*.mjs             check-page, snapshot, read, shot, service-specific flows
  api/*                     reusable API calls
  cli/*                     reusable CLI sequences
reference/scripts.md        how to write and save a new script (read only when needed)
```

Progressive loading: Claude sees one description line. It reads `SKILL.md` only when the skill triggers, and `reference/scripts.md` only when no existing script fits.

---

## One call, one line

```bash
node bin/run.mjs check-page --url https://staging.example.com/orders --expect "Orders"
```

```json
{"ok":false,"failedStep":"expect text","error":"\"Orders\" not found","network":["GET /api/orders → 500"],"console":["TypeError: items is undefined (main.js:212)"],"ms":1840}
```

The output contains only what matters: `ok`, the returned `data`, the failed step, console errors, HTTP requests ≥ 400, dialogs, and a screenshot path **only on failure**. Known noise (optional auth refreshes, cross-origin blocked images) is filtered and counted, not printed.

Exit code 0 or 1, so it composes with anything.

---

## Helpers that explain their own failures

Scripts use accessible names, not CSS selectors:

```js
// @description Create a ticket and check it appears in the list
// @params --title <text> [--priority High]
export default async function ({ params, goto, step, click, fill, select, waitText }) {
  await step('open form', () => goto('/tickets/new'));
  await step('fill', async () => {
    await fill('Title', params.title);
    await select('Priority', params.priority || 'Low');
    await click('Create');
  });
  await step('check', () => waitText(params.title));
}
```

- `click`, `fill`, `select` find elements by label, `aria-label`, placeholder or visible text, search inside an open dialog first, wait for disabled buttons to enable, and work with React controlled inputs and ARIA comboboxes (Radix, shadcn, MUI).
- When they fail, the error **lists what is available**. The agent fixes the script from the message, without a screenshot.
- `snapshot` returns a short numbered list of headings and interactive elements to explore an unknown page, instead of the whole DOM.

---

## The private test browser

- A visible Chrome **private window** on a debug port, opened on first use and **reused** after that.
- **The user signs in themselves** (SSO, MFA). The session serves every tab while Chrome stays open. Claude never types a password.
- Scripts open their own tab and never take over the tab the user is working in.
- Scripts never close the last tab, because that would close Chrome and lose the session.
- If Chrome was closed, the output says `relaunched` with a warning, so the agent tells the user to sign in again instead of failing mysteriously.

---

## Catalogue: the skill grows with use

```bash
node bin/list.mjs ticket
browser/servicenow-create.mjs   Create a ServiceNow record through the Table API…   | --table <t> --data '{…}'
```

- Before writing anything, the agent checks the catalogue.
- A one-off check is written in the session's scratch folder and forgotten.
- A flow that will come back is made **parameterised** (no URL, ID or secret hard-coded), gets a `@description` / `@params` header, and is saved under `scripts/`. It appears in the catalogue automatically.

The first time costs an exploration. Every next time is one command. Same idea as [Echo](https://github.com/Reactoring/echo), applied to actions.

---

## Safety

- **No secret in clear.** Never in a command, a saved script, a message or an output. GitHub through `gh`, Google Cloud through `gcloud auth print-access-token`, other APIs through env vars read by `http.mjs` (`--bearer-env`, `--basic-env`). If a variable is missing, the user sets it; it is never pasted in the chat.
- **Production is read-only by default.** No create, update, delete, form submit or POST/PUT/PATCH/DELETE without explicit approval for that exact action.
- **Visible actions are shown before they leave.** Issue, PR, comment, incident, message, deployment: target, title and content are shown, and one approval covers one action.
- **Least privilege.** A script does only what it is written to do.

---

## Reporting

Short: ✅ or ❌, what was checked, and the proof (failed step, error message, the request in 500, the console error), plus a fix suggestion when the project code is available. Never the raw JSON.

---

## Getting started

1. Put the skill in `~/.claude/skills/give` and run `npm install` (only dependency: `puppeteer-core`, it drives the Chrome already installed).
2. Log in your CLIs (`gh auth login`, `gcloud auth login`) and export API credentials as env vars.
3. Ask Claude: "check that the login page of staging loads", "create an issue for this bug", "test the signup form on localhost:5173".

## Limits

- The browser path drives a real Chrome: it is slower than CLI or API, which is why it comes last.
- Saved scripts follow the UI; a redesign can break them (the error message says what changed).
- The catalogue must stay curated: improve an existing script rather than add a near-duplicate.

---

*Author: Amine Dkhili.*
