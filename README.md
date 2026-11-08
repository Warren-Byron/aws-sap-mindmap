# AWS SAP Mindmap

Interactive mindmap for tracking study progress through the AWS Certified Solutions Architect – Professional (SAP-C02) course.

**Live site:** https://<your-username>.github.io/aws-sap-mindmap/

## How it works

- `site/mindmap.md` holds all the mindmap content as plain Markdown.
- `site/index.html` loads that file and renders it with [Markmap](https://markmap.js.org/).
- Every push to `main` deploys the `site/` folder to GitHub Pages via GitHub Actions.

## Tracking progress

Each topic is a Markdown checkbox:

```markdown
- [ ] Not started
- [x] Done
```

Tick a box by changing `[ ]` to `[x]`, commit, and push. The progress bar at the top of the page counts ticked boxes automatically.

## Run locally

The page fetches `mindmap.md`, so it must be served over HTTP (opening `index.html` directly from disk will not work):

```bash
python3 -m http.server 8000 --directory site
```

Then open http://localhost:8000.
