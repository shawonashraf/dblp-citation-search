---
name: dblp-cite
description: Find the BibTeX citation for a paper on dblp, given an arXiv URL or a paper title. Returns the published (non-arXiv/CoRR) version's BibTeX if dblp has one, otherwise says to cite the arXiv version. Use when the user asks for a citation, BibTeX, or the published version of an arXiv paper.
---

# dblp citation search

Turn an arXiv URL or a paper title into the dblp BibTeX of the paper's published version.

## Rules

- dblp blocks non-browser clients with a bot check. Access dblp **only** through the Claude in
  Chrome tools. Never fetch dblp with curl, WebFetch, or any other client, and never try to get
  around the bot check.
- Keep dblp traffic to what a person would do: one search page (a second only when step 3 asks
  the user for an author), then one BibTeX page per entry the user wants. No extra pages, no
  retries by reloading.
- Use one new tab for the lookup and close it at the end, including when you stop early.

## 1. Get the title (and first author)

**arXiv URL or arXiv ID** (`arxiv.org/abs/...`, `/pdf/...`, `/html/...`, or a bare ID such as
`1706.03762`, `arXiv:1706.03762`, `cs/0112017`): extract the ID. New-style IDs look like
`2401.01234`; old-style like `cs/0112017` or `hep-th/9901001`. Drop an `arXiv:` prefix, a
trailing `vN`, and `.pdf`. Then run:

```bash
curl -s "https://export.arxiv.org/api/query?id_list=<id>"
```

From the `<entry>` element (not the feed's own `<title>`), take the `<title>` (join its lines
with single spaces) and the first `<author><name>`. Decode XML entities in both (`&amp;` → `&`,
`&lt;` → `<`, `&gt;` → `>`, `&quot;` → `"`, `&apos;` → `'`). The first author's surname is the last word
of that name. If there is no `<entry>` or it has no title, tell the user the arXiv ID was not
found and stop.

**Title given directly:** use it as is. There is no author.

Tell the user which title you are searching for.

## 2. Open a tab

Load the Chrome tools in one ToolSearch call:
`select:mcp__claude-in-chrome__tabs_context_mcp,mcp__claude-in-chrome__tabs_create_mcp,mcp__claude-in-chrome__navigate,mcp__claude-in-chrome__javascript_tool,mcp__claude-in-chrome__tabs_close_mcp`

If they are not available, or `tabs_context_mcp` fails, stop and tell the user to start Claude
Code with `claude --chrome` (or connect the Claude in Chrome extension). Do not look up dblp any
other way.

Call `tabs_context_mcp`, then `tabs_create_mcp`, and use the new tab for everything below.

## 3. Search dblp

dblp lists results newest first, 30 per page, so a common title alone can push the paper off
the first page. Add the first author's surname to the query when you have it.

`navigate` to `https://dblp.org/search?q=<URL-encoded title + " " + surname>` (title only if
there is no author).

Read the results with `javascript_tool`:

```js
const TITLE = "...";  // replace with JSON.stringify(<searched title>), never paste it raw
const norm = s => s.toLowerCase().replace(/\s+/g, ' ').trim().replace(/\.$/, '');
const sleep = ms => new Promise(r => setTimeout(r, ms));
// Wait up to 15 s for dblp's bot check to finish and the results to render.
const isBot = () => /not a bot/i.test(document.title);
const info = () => document.querySelector('#completesearch-info-matches')?.innerText.trim() || '';
for (let i = 0; i < 30 && (isBot() || !info()); i++) await sleep(500);
const status = info();
const entries = [...document.querySelectorAll('#completesearch-publs li.entry')];
({
  ready: !isBot() && status !== '',
  botCheck: isBot(),
  found: /no match/i.test(status) ? 0 : Number((status.match(/([\d,]+)\s+match/) || [])[1]?.replace(/,/g, '')) || null,
  loaded: entries.length,
  matches: entries
    .filter(li => norm(li.querySelector('.title')?.textContent || '') === norm(TITLE))
    .map(li => {
      const bib = [...li.querySelectorAll('a')].map(a => a.href).find(h => h.includes('view=bibtex')) || '';
      return {
        title: li.querySelector('.title').textContent,
        cite: li.querySelector('cite').textContent,
        preprint: li.classList.contains('informal'),
        // Split on "/": the Chrome tool hides strings like "conf/nips/VaswaniSPUJGKP17" as
        // base64. Join keyParts with "/" to get the record key.
        keyParts: ((bib.match(/\/rec\/(.+)\.html/) || [])[1] || '').split('/'),
      };
    }),
})
```

- `ready` false, or the script errors (the page can reload itself once the bot check passes):
  run the script again, without reloading the page. At most 3 runs in total. Still not ready:
  tell the user dblp's bot check did not finish, close the tab, stop.
- Each match's record key is its `keyParts` joined with `/` (e.g. `conf/nips/VaswaniSPUJGKP17`).
- `preprint` true means dblp files the entry under "Informal and Other Publications": CoRR
  (key starts with `journals/corr/`) and other preprint servers. Every other match is a
  published version.

**No matches, and `found` is larger than `loaded`** (more results exist than the page shows):
- Title-only search: ask the user for the first author's surname, then search once more with it
  and re-run the script.
- Search already included the surname, or the user doesn't know it: go to step 4 as
  **Not confirmed**.

## 4. Decide

- **Not confirmed:** tell the user the title was not on the first page of the N dblp results,
  so you could not confirm whether a published version exists. Cite the arXiv version unless
  they know the paper appeared somewhere.
- **No matches:** the paper is not on dblp under this title. Tell the user to cite the arXiv
  version.
- **Only preprint matches:** dblp has only preprint versions. Tell the user to cite the arXiv
  version. If one of them is the CoRR entry, offer to fetch its dblp BibTeX, and fetch it
  (step 5) only if they say yes.
- **One published match:** go to step 5 with it.
- **Several published matches:** list them, numbered, with venue and year (from `cite`) and the
  key. Ask the user which one they want. Go to step 5 with their pick.

## 5. Get the BibTeX

`navigate` the same tab to `https://dblp.org/rec/<key>.html?view=bibtex` (this page shows the
BibTeX inside the page). Read it with `javascript_tool`:

```js
for (let i = 0; i < 30 && !document.querySelector('pre'); i++) await new Promise(r => setTimeout(r, 500));
document.querySelector('pre')?.textContent.split('\n')
  ?? (/not a bot/i.test(document.title) ? 'BOT_CHECK' : 'NO_BIBTEX')
```

`BOT_CHECK` or an error: run it again, without reloading, at most 3 runs in total. `NO_BIBTEX`,
or still blocked: tell the user, give them the record URL `https://dblp.org/rec/<key>`, close the
tab, stop.

Join the lines with newlines and give the user the BibTeX unchanged, in a ```bibtex block,
with one line saying which venue and year it is.

## 6. Close the tab

`tabs_close_mcp` on the tab you created. Do this on every exit after step 2.
