# dblp-citation-search

A Claude Code plugin that finds the right BibTeX for a paper. Give it an arXiv URL or a paper
title. It searches [dblp](https://dblp.org) for an exact title match and returns the BibTeX of
the published version (conference or journal). If dblp only has the arXiv (CoRR) version, or
does not have the paper at all, it tells you to cite the arXiv version. If several published
versions match, it lists them and asks which one you want.

## Requirements

- Claude Code with the Claude in Chrome extension connected
  (start Claude Code with `claude --chrome`).

dblp puts a bot check in front of its whole site, including its search API, so non-browser
clients get a challenge page instead of results. The plugin therefore does the lookup in a
Chrome tab, loading the same pages a person would: one search page and one BibTeX page.
Titles from arXiv URLs come from the arXiv API.

## Install

```
/plugin marketplace add shawonashraf/dblp-citation-search
/plugin install dblp-citation-search@dblp-citation-search
```

## Usage

```
/dblp-citation-search:dblp-cite https://arxiv.org/abs/1706.03762
/dblp-citation-search:dblp-cite Attention Is All You Need
```

Or just ask, e.g. "get me the dblp BibTeX for https://arxiv.org/abs/2408.09667".

For a title without an arXiv URL, if the title is common and the paper is not on the first page
of dblp results, the plugin asks for the first author's surname to narrow the search.
