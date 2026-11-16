# Privacy policy

dblp-citation-search runs inside your Claude Code session. Its author runs no server and
collects, stores or receives nothing.

## What the plugin sends, and where

To look up a citation, the plugin sends:

- **To the arXiv API (export.arxiv.org):** the arXiv ID you give it, to get the paper's title and
  authors. This is a plain HTTPS request from your machine.
- **To dblp (dblp.org):** the paper title and, when known, the first author's surname, as a
  search query; then a request for the chosen paper's BibTeX page. These requests are made
  in your own Chrome browser through Claude in Chrome, so dblp sees them as ordinary visits
  from your browser.

Nothing else is sent. arXiv and dblp handle these requests under their own privacy policies.

## Data handled by Claude

Your requests and the results go through Claude like any other Claude Code session, under
Anthropic's terms and privacy policy.

## Contact

Questions: open an issue at https://github.com/shawonashraf/dblp-citation-search/issues
