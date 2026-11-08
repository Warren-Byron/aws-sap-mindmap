# AWS SAP Mindmap

Interactive mindmap for tracking study progress through the AWS Certified Solutions Architect – Professional (SAP-C02) course.

**Live site:** https://warren-byron.github.io/aws-sap-mindmap/

## How it works

- `site/mindmap.md` holds all the content: 16 course sections, 106 lectures, and condensed study notes under each lecture.
- `site/index.html` loads that file and renders it with [Markmap](https://markmap.js.org/).
- Every push to `main` deploys the `site/` folder to GitHub Pages via GitHub Actions.

## Tracking progress

Each lecture is a Markdown checkbox. Tick it by changing `[ ]` to `[x]`, then commit and push:

```markdown
- [x] **IAM**
- [ ] **IAM Access Analyzer**
```

The header bar shows overall progress, and each section shows a `done/total` badge that turns green when finished.

## Using the map

| Action | How |
|---|---|
| Expand / collapse a node | Click its circle |
| Expand a whole branch | Ctrl/⌘-click its circle |
| Search | Type in the search box (or press `/`); matching branches open and matches are highlighted |
| Section overview | **Sections** button |
| What's left | **Show to-do** opens only sections with unticked lectures |
| Zoom, fit, dark mode | Toolbar, bottom right |

Note icons: 💡 exam tip · ⚠️ gotcha · 🧭 decision guide · ✅ AWS-recommended

## Editing the map

- `##` = section, `- [ ] **Lecture**` = lecture, indented `-` bullets = notes.
- Display options (colours, initial depth, spacing) live in the YAML front matter at the top of `mindmap.md`.
- Inline `code`, **bold**, links and fenced code blocks all render inside nodes.

## Run locally

The page fetches `mindmap.md`, so it must be served over HTTP (opening `index.html` directly from disk will not work):

```bash
python3 -m http.server 8000 --directory site
```

Then open http://localhost:8000.

## Content note

These are personal study notes written in my own words. They do not include course slides, screenshots or other course material.
