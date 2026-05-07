# alexr17.github.io

Personal website for Alex R.

This is intentionally simple: plain HTML and CSS, published by GitHub Pages at
`https://alexr17.github.io`.

## Editing

- Update the homepage in `index.html`.
- Update styling in `styles.css`.
- Add project links and short notes directly to the lists on the homepage.
- Add longer notes as Markdown files under `notes/` with front matter and
  `layout: note`.

## Local Preview

From this repository:

```bash
ruby -run -e httpd . -p 4000
```

Then open `http://localhost:4000`.

GitHub Pages renders note Markdown files to `.html` URLs. To preview generated
note pages locally, use a local Jekyll/GitHub Pages build instead of the static
Ruby file server.
