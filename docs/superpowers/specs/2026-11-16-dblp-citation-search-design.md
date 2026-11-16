# dblp citation search — design

## Intent

Given an arXiv URL or a paper title, find the paper on dblp and return the BibTeX
for its published (non-CoRR) version. If dblp only has the arXiv (CoRR) version,
tell the user to cite the arXiv version. Lookups must stay light on dblp's servers.

## Goals

- Accept an arXiv URL (`arxiv.org/abs/...`, `/pdf/...`, with or without version) or a title.
- Find exact title matches on dblp, excluding CoRR entries.
- One match: return its BibTeX. Several matches: list them, the user picks, return that BibTeX.
- No non-CoRR match: say the arXiv version is the one to cite.

## Non-goals

- claude.ai chat support (see Constraints).
- Batch lookups, bibliography files, or writing to `.bib` files.
- Fuzzy matching or suggesting near-miss titles.

## Anti-goals

- Bursts of requests to dblp, or any attempt to get past dblp's bot check outside a real browser.

## Constraints

- dblp serves a bot-check page (HTTP 202, "Making sure you're not a bot!") to non-browser
  clients on every endpoint, including the search API and `/rec/<key>.bib`
  (checked 2026-11-16). Only a real browser passes it, so dblp access goes through
  Claude in Chrome. This makes the plugin Claude Code only.
- The arXiv API (`https://export.arxiv.org/api/query?id_list=<id>`) works without a browser
  and returns the title.

## Approaches

1. **dblp search API via web fetch.** Lightest on dblp, would also work in claude.ai.
   Rejected: blocked by the bot check.
2. **Chrome only via Claude in Chrome.** Picked. Works because the browser passes the check;
   each lookup is the same page loads a person would make.
3. **API with Chrome fallback.** Rejected: the API path never succeeds, so it adds nothing.

## Design (decided with the user)

- Claude Code plugin containing one skill.
- Title source: user-given title, or the arXiv API for an arXiv URL.
- dblp search and BibTeX retrieval in a Chrome tab via Claude in Chrome.
- Exact title matches only; if more than one non-CoRR match, list all and ask the user to pick.

## Calls made by Claude (correct if wrong)

- **Exact match** means equal after: lowercasing, collapsing whitespace, and dropping the
  trailing period dblp adds to every title. Nothing looser.
- **Page loads per lookup:** one dblp search page, then one BibTeX page for the chosen entry
  only. With several matches the list comes from the search page, and BibTeX is fetched
  after the user picks, not for all of them.
- **CoRR-only case:** state that the arXiv version is the one to cite; offer to fetch the
  CoRR BibTeX from dblp if the user wants it (one extra page load, only on request).
- **Tab handling:** open one new tab per lookup, close it when done.
- **Repo layout:** `.claude-plugin/plugin.json`, `.claude-plugin/marketplace.json` (so the repo
  installs with `/plugin marketplace add`), `skills/dblp-cite/SKILL.md`.

## Left to the builder

- Exact wording of the skill instructions and how it reads the dblp search page
  (page text vs. links) to get each entry's title, venue, year, and record key.
