# Search (Job Hunt)

Repo: https://github.com/MelvTheGoat/Search (private)

## In short

A daily Python tool that finds ML and AI jobs, labels whether each is reachable from Nigeria (remote-open, Nigeria, Africa, visa sponsor or restricted, with evidence), scores fit 0–100 against a CV using a local embedding model, and keeps a tracker that never loses your notes. It checks drafted cover letters so they can't contain numbers that aren't in the CV. It never applies for you.

## Key facts

| | |
|---|---|
| Language | Python 3.10+ |
| Core tech | requests, sentence-transformers (MiniLM), SQLite, openpyxl, python-docx |
| Sources | 405 company boards on 9 ATSs, plus job boards |
| Tests | 74 passing |
| Not measured yet | How well the fit score predicts interviews |

Note: this report describes the tool only. It leaves out personal details from the repo (CV content, letters and applications).

## Files

| File | What's in it |
|---|---|
| [00-start-here.md](00-start-here.md) | **Start here.** The whole project in simple English (good for NotebookLM) |
| [01-system-design.md](01-system-design.md) | Parts, diagram, stack, data flow, trade-offs, 10x |
| [02-how-to-write-the-system-design.md](02-how-to-write-the-system-design.md) | Whiteboard steps |
| [03-linkedin-post.md](03-linkedin-post.md) | LinkedIn post |
| [04-blog-post.md](04-blog-post.md) | Blog post |
| [05-explain-to-non-technical.md](05-explain-to-non-technical.md) | Plain-language explanation |
| [06-explain-to-technical.md](06-explain-to-technical.md) | Engineer-level explanation |
| [07-defend-in-interview.md](07-defend-in-interview.md) | Pitch, questions, weak spots |
| [08-ten-points-to-know.md](08-ten-points-to-know.md) | 10 key facts |
| [09-what-this-proves-i-know.md](09-what-this-proves-i-know.md) | Skills and related topics |
| [10-system-design-for-beginners.md](10-system-design-for-beginners.md) | The whole system explained step by step, for beginners |
| [11-technical-terms.md](11-technical-terms.md) | Every technical term: what it means and why it's used here |
| [12-tools-and-why.md](12-tools-and-why.md) | Every tool: what it is and why it was picked |
| [13-talking-it-through.md](13-talking-it-through.md) | The whole build talked through like a conversation, naming every file as it is created |
