Add a sources section to `content/posts/$ARGUMENTS/index.md`. Follow these rules exactly:

**Choosing sources:**
- Read the post and pick out the claims a reader might want to follow up: historical events, papers, statistics, CVEs, announcements, specifications, and the tools or projects the post discusses.
- Prefer primary sources: the original paper (as a `https://doi.org/...` link where one exists), the official project page or spec, the original announcement or mailing-list post, live status pages for statistics, and NVD for CVEs. Use Wikipedia only when there is no good primary source.
- Every URL must be real. Find candidates with WebSearch or from your own knowledge, then check that each one resolves (e.g. `curl -s -o /dev/null -L -m 15 -A 'Mozilla/5.0' -w '%{http_code}' <url>`). If a site blocks automated requests (403) but the URL is certainly correct (such as a DOI), keep it. Otherwise drop the link and never guess one.
- If you can't find a good source for something, leave it out and say so in the report. Don't put a weak source in its place.
- Don't link to other posts on this site. Don't link to sources that are already linked inline in the post, unless they belong to a claim that needs a source anyway.

**Format** (append as the very last section of the post, after one blank line):

```markdown
## Sources

- Author Name, [Title of the work](https://...), Publication, Year
- [Project or page title](https://...)
- [Page title](https://...) (Site name, Year)
- [CVE-YYYY-NNNN](https://nvd.nist.gov/vuln/detail/CVE-YYYY-NNNN) at NVD
```

- One bullet per source. The link text is the page's real title, or a short descriptive title when the real one is unwieldy. Put the author before it for papers and personal write-ups, and add the site/publication and year after it when they help.
- Put related links on one bullet when they belong together (e.g. a blog post and the project it describes).
- Order the bullets by where their topic first comes up in the post.
- Follow the CLAUDE.md writing style (British English, no em dashes).

**Other rules:**
- Don't change anything else in the post.
- If the post already has a `## Sources` section, update it in place (add the missing sources and keep the order) instead of adding a second one.

Afterwards, output:
1. **Added**: the list of sources, each with the claim or topic it supports.
2. **Not sourced**: claims or topics you couldn't find a good source for, if any.
