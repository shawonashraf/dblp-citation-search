# dblp Citation Search Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** A Claude Code plugin whose skill turns an arXiv URL or paper title into the dblp BibTeX of the paper's published (non-CoRR) version, or says to cite arXiv.

**Architecture:** One skill (`SKILL.md`, instructions only, no scripts). Title comes from the user or from the arXiv API via `curl`. dblp search and BibTeX pages are loaded in a Chrome tab through the Claude in Chrome tools, because dblp serves a bot-check page to non-browser clients.

**Tech Stack:** Claude Code plugin manifest, Agent Skill markdown, Claude in Chrome MCP tools, `curl`.

**Spec:** `docs/superpowers/specs/2026-11-16-dblp-citation-search-design.md`

## Global Constraints

- Claude Code only; no claude.ai support.
- Never fetch dblp with `curl`, WebFetch, or any non-browser client. Never try to solve dblp's bot check outside the browser.
- Per lookup: at most one dblp search page load, plus one BibTeX page load per entry the user asks for.
- Exact match = equal after lowercasing, collapsing whitespace, dropping one trailing period. Nothing looser.
- CoRR entries (venue `CoRR`) never count as a published version.
- One new tab per lookup, closed at the end.
- Plugin name `dblp-citation-search`; skill name `dblp-cite`.
- No `Co-Authored-By` or "Generated with Claude Code" lines in commits.

## Review Focus

1. **arXiv URL variants** — `/abs/2401.01234`, `/pdf/2401.01234v2`, `/pdf/2401.01234v2.pdf`, `/html/...`, old-style `cs/0112017`: all resolve to the right ID with the version suffix removed. Covered by Task 2, Step 3 case E.
2. **Chrome not connected** — the Chrome tools are missing or error: the skill stops and tells the user to start Claude Code with `--chrome` (or connect the extension); it does not fall back to fetching dblp another way. Covered by Task 2, Step 3 case F.
3. **Bot-check page in the tab** — the first dblp load shows "Making sure you're not a bot!": the skill waits for it to finish and re-reads the page, without reloading in a loop. Covered by Task 2, Step 3 case A (first run in a fresh browser session).
4. **Title not on dblp at all** — distinct message from "only CoRR": "not found on dblp, cite the arXiv version". Covered by Task 2, Step 3 case D.
5. **Title differs only by case or trailing period** — arXiv "Attention is all you need" vs dblp "Attention Is All You Need." still matches. Covered by Task 2, Step 3 case A (enter the title in lowercase).

---

### Task 1: Plugin and marketplace manifests

**Files:**
- Create: `.claude-plugin/plugin.json`
- Create: `.claude-plugin/marketplace.json`

**Interfaces:**
- Produces: plugin `dblp-citation-search`, listed in marketplace `dblp-citation-search` with `"source": "./"`. Task 2 places the skill at `skills/dblp-cite/SKILL.md`, which the plugin picks up by default.

- [ ] **Step 1: Write `.claude-plugin/plugin.json`**

Fields: `name: "dblp-citation-search"`, `version: "0.1.0"`, `description` (one sentence from the Goal above), `author: {"name": "Shawon Ashraf"}`. No email.

- [ ] **Step 2: Write `.claude-plugin/marketplace.json`**

Fields: `name: "dblp-citation-search"`, `owner: {"name": "Shawon Ashraf"}`, `plugins: [{"name": "dblp-citation-search", "source": "./", "description": <same sentence>}]`.

- [ ] **Step 3: Validate**

Run: `claude plugin validate .`
Expected: no errors (a warning about having no skills yet is acceptable at this point).

- [ ] **Step 4: Commit**

```bash
git add .claude-plugin/plugin.json .claude-plugin/marketplace.json
git commit -m "add plugin and marketplace manifests"
```

### Task 2: The `dblp-cite` skill

**Files:**
- Create: `skills/dblp-cite/SKILL.md`
- Modify: `README.md` (install + usage)

**Interfaces:**
- Consumes: plugin from Task 1.
- Produces: skill invocable as `/dblp-citation-search:dblp-cite <arXiv URL or title>` and triggered automatically by requests like "get the dblp bibtex for <arXiv URL>".

- [ ] **Step 1: Write `skills/dblp-cite/SKILL.md`**

Frontmatter: `name: dblp-cite`; `description` saying when to use it (user gives an arXiv URL or paper title and wants a BibTeX citation / the published version from dblp).

Body, as numbered steps Claude follows:
1. **Get the title.** If the input is an arXiv URL, extract the ID (new-style `NNNN.NNNNN` or old-style `archive/NNNNNNN`), strip any `vN` and `.pdf`, then run `curl -s "https://export.arxiv.org/api/query?id_list=<id>"` and take the `<title>` inside `<entry>` (not the feed title), collapsing line breaks. Otherwise use the given text as the title. Tell the user the title being searched.
2. **Check Chrome.** Load the Claude in Chrome tools (one ToolSearch call: `tabs_context_mcp`, `tabs_create_mcp`, `navigate`, `get_page_text`, `find`, `tabs_close_mcp`). If unavailable or erroring, stop and tell the user to restart with `claude --chrome` or connect the extension. Never fall back to curl/WebFetch for dblp.
3. **Search dblp.** `tabs_context_mcp`, then `tabs_create_mcp`, then `navigate` to `https://dblp.org/search?q=<URL-encoded title>`. If the page is the bot check, wait a few seconds and read again (max 3 reads, no reloads). Read the results.
4. **Match.** Keep entries whose title equals the searched title under the exact-match rule. For each, record title, venue, year, and dblp record key (from the entry's link, `https://dblp.org/rec/<key>`). Split into CoRR and non-CoRR.
5. **Decide.**
   - No exact matches: say the paper was not found on dblp; cite the arXiv version.
   - Only CoRR: say dblp has only the arXiv version; cite that. Offer to fetch its BibTeX.
   - One non-CoRR: go to step 6 with it.
   - Several non-CoRR: list them (venue, year, key), ask the user to pick, then step 6 with the pick.
6. **Get BibTeX.** `navigate` the same tab to `https://dblp.org/rec/<key>.bib`, read the page text, return it in a ```bibtex block unchanged.
7. **Close the tab** with `tabs_close_mcp`, including on every early exit after step 3.

- [ ] **Step 2: Validate**

Run: `claude plugin validate .`
Expected: no errors; the skill `dblp-cite` is listed.

- [ ] **Step 3: End-to-end runs**

Start: `claude --chrome --plugin-dir .` and run each case as a fresh request.

- A. `/dblp-citation-search:dblp-cite attention is all you need` (lowercase) in a browser that has not visited dblp this session → title matches; BibTeX for key `conf/nips/VaswaniSPUJGKP17` (or a list containing it, if dblp has more than one non-CoRR entry); tab closed.
- B. `/dblp-citation-search:dblp-cite https://arxiv.org/abs/1706.03762` → title "Attention Is All You Need" printed, same result as A.
- C. A paper with the same title at a conference and in a journal (pick one by searching dblp manually in the browser) → both listed, user is asked to pick, only the picked BibTeX page is loaded.
- D. A made-up title, e.g. `A Study That Does Not Exist On Purpose 12345` → "not found on dblp, cite arXiv".
- E. `https://arxiv.org/pdf/1706.03762v7.pdf` and `https://arxiv.org/abs/cs/0112017` → correct IDs, titles fetched from arXiv.
- F. `claude --no-chrome --plugin-dir .`, then case A → tells the user to enable Chrome; no dblp request is made.
- G. An arXiv paper whose dblp search shows only a CoRR entry (pick one by searching dblp manually) → "only the arXiv version on dblp" plus the offer.

Expected: each case behaves as stated. Fix `SKILL.md` and re-run any case that fails.

- [ ] **Step 4: Update `README.md`**

Add: what it does (one paragraph), requirement (Claude Code with Claude in Chrome), install (`/plugin marketplace add <path or GitHub repo>` then `/plugin install dblp-citation-search@dblp-citation-search`), usage example, and why it uses the browser (dblp's bot check blocks non-browser clients).

- [ ] **Step 5: Commit**

```bash
git add skills/dblp-cite/SKILL.md README.md
git commit -m "add dblp-cite skill"
```
