# Personal Site Reset Design

## Goal

Reset this GitHub Pages repository from an old site into a very simple personal website for Alex R, published at `alexr17.github.io`.

## Direction

The site should follow the spirit of simple personal sites such as Mitchell Hashimoto, DHH, Simon Eskildsen, Andy Pavlo, and Armin Ronacher: text-first, fast, plain, link-heavy, and easy to edit without a build system.

## Scope

- Remove the old custom domain by deleting `CNAME`.
- Replace the default GitHub Pages/Jekyll starter content with a static homepage.
- Add a small stylesheet with readable typography and responsive spacing.
- Include starter sections for a short bio, side projects, short-form articles, and links.
- Update `README.md` with editing and publishing notes.

## Architecture

The first version is plain HTML and CSS:

- `index.html` contains the complete homepage content.
- `styles.css` contains all site styling.
- `README.md` documents how to maintain the site.
- `_config.yml` can remain minimal for GitHub Pages metadata, but the site does not depend on a Jekyll theme.

No JavaScript, package manager, build tool, framework, analytics, or generated assets are needed.

## Testing

Because this is a static site, verification is limited to file presence, broken local references, and serving the site locally to confirm the files load.
