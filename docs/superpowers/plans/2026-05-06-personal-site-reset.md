# Personal Site Reset Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the old GitHub Pages starter site with a plain static personal site for Alex R.

**Architecture:** Use one `index.html` page and one `styles.css` stylesheet. Keep the site framework-free and publishable by GitHub Pages at `alexr17.github.io`.

**Tech Stack:** HTML, CSS, GitHub Pages.

---

### Task 1: Static Site Reset

**Files:**
- Create: `index.html`
- Create: `styles.css`
- Modify: `README.md`
- Modify: `_config.yml`
- Delete: `CNAME`

- [ ] **Step 1: Remove the old custom domain**

Run: `rm CNAME`

Expected: `CNAME` no longer exists, so GitHub Pages uses `alexr17.github.io`.

- [ ] **Step 2: Replace the homepage**

Create `index.html` with a text-first personal homepage containing intro, side projects, notes, and links.

- [ ] **Step 3: Add simple styling**

Create `styles.css` with narrow readable layout, system fonts, plain links, responsive spacing, and no decorative framework styling.

- [ ] **Step 4: Update repository docs and metadata**

Replace `README.md` with concise maintenance notes and update `_config.yml` with site title and description.

- [ ] **Step 5: Verify**

Run:

```bash
test -f index.html
test -f styles.css
test ! -f CNAME
ruby -run -e httpd . -p 4000
```

Expected: required files exist, `CNAME` is absent, and the static site can be served locally.

- [ ] **Step 6: Commit and push**

Run:

```bash
git status --short
git add -A
git commit -m "Reset site for personal homepage"
git push origin master
```
