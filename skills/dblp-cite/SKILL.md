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

**arXiv URL** (`arxiv.org/abs/...`, `/pdf/...`, `/html/...`): extract the ID. New-style IDs look
like `2401.01234`; old-style like `cs/0112017` or `hep-th/9901001`. Drop a trailing `vN` and
`.pdf`. Then run:

```bash
curl -s "https://export.arxiv.org/api/query?id_list=<id>"
```

From the `<entry>` element (not the feed's own `<title>`), take the `<title>` (join its lines
with single spaces) and the first `<author><name>`. The first author's surname is the last word
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
const norm = s => s.toLowerCase().replace(/\s+/g, ' ').trim().replace(/\.$/, '');
const want = norm(TITLE);  // replace TITLE with the searched title as a JS string literal
({
  botCheck: /not a bot/i.test(document.title),
  found: (document.body.innerText.match(/found (\S+) match/) || [])[1] || null,
  loaded: document.querySelectorAll('li.entry').length,
  matches: [...document.querySelectorAll('li.entry')]
    .filter(li => norm(li.querySelector('.title')?.textContent || '') === want)
    .map(li => {
      const bib = [...li.querySelectorAll('a')].map(a => a.href).find(h => h.includes('view=bibtex')) || '';
      return {
        title: li.querySelector('.title').textContent,
        cite: li.querySelector('cite').textContent,
        // Split on "/": the Chrome tool hides strings like "conf/nips/VaswaniSPUJGKP17" as
        // base64. Join keyParts with "/" to get the record key.
        keyParts: ((bib.match(/\/rec\/(.+)\.html/) || [])[1] || '').split('/'),
      };
    }),
})
```

- `botCheck` true: the browser is solving dblp's check. Wait about 5 seconds and run the script
  again. Do this at most 3 times, without reloading. Still blocked: tell the user, close the
  tab, stop.
- Each match's record key is its `keyParts` joined with `/` (e.g. `conf/nips/VaswaniSPUJGKP17`).
- A match whose key starts with `journals/corr/` is the arXiv (CoRR) version. Every other match
  is a published version.

**Title-only search with no matches**, when `found` is larger than `loaded` (more results exist
than the page shows): ask the user for the first author's surname, then search once more with
it and re-run the script. If they don't know it, go to step 4 with no matches.

## 4. Decide

- **No matches:** the paper is not on dblp under this title. Tell the user to cite the arXiv
  version.
- **Only CoRR matches:** dblp has only the arXiv version. Tell the user to cite the arXiv version,
  and offer to fetch its dblp BibTeX. Fetch it (step 5) only if they say yes.
- **One published match:** go to step 5 with it.
- **Several published matches:** list them, numbered, with venue and year (from `cite`) and the
  key. Ask the user which one they want. Go to step 5 with their pick.

## 5. Get the BibTeX

`navigate` the same tab to `https://dblp.org/rec/<key>.html?view=bibtex` (this page shows the
BibTeX inside the page). Read it with `javascript_tool`:

```js
document.querySelector('pre').textContent.split('\n')
```

Join the lines with newlines and give the user the BibTeX unchanged, in a ```bibtex block,
with one line saying which venue and year it is.

## 6. Close the tab

`tabs_close_mcp` on the tab you created. Do this on every exit after step 2.
