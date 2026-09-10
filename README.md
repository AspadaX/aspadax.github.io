# aspadax.github.io

Personal Jekyll blog at <https://aspadax.github.io>. A minimal dark design with a short biography, a compact article list,
local layouts, system fonts, and one plain CSS file. No theme or JavaScript
is needed.

## Structure

```text
_posts/         Published articles in English and Chinese
_drafts/        Unpublished article template
_layouts/       Shared page shell, homepage, and post layout
assets/         main.css and any article images
_config.yml     Site settings and post defaults
index.md        English homepage at /
index-zh.md     Chinese homepage at /cn/
_data/          English and Chinese interface copy
404.html        Not-found page
```

Pages live at the root; their `permalink` controls the public URL. Each homepage combines the biography and its language’s article list. The
EN/CN switch changes the full interface and links to the matching translation
when reading a post (or to the selected homepage if none exists). Language
selection lives in the URL, so links are shareable and work without JavaScript.
The former standalone About and Articles pages have been removed.

## Publish an article

Add `_posts/YYYY-MM-DD-your-title.md`:

```markdown
---
title: "Your post title"
---

Your opening paragraph becomes the article preview.

## A section heading

Write the rest in Markdown.
```

The layout, author, and English language default come from `_config.yml`.
For Chinese articles add `lang: zh-CN`. Give translations the same
`translation_key` to link them. Add images under `assets/images/` and reference
them with `![Description](/assets/images/example.png)`. Publish diagrams as static
SVG images so they remain readable without JavaScript. Exclude working article
sources at the repository root from the site in `_config.yml`.

Start a draft in `_drafts/`, then move it into `_posts/` with a dated filename to
publish. Draft source remains public in this repository. Future-dated posts
appear only after a build on or after their publication date.

## Customize

- Edit `_config.yml` for the title, description, author, and GitHub username.
- Edit `assets/main.css` for colors, typography, spacing, and responsive styles.
- Edit `_layouts/default.html` for the shared navigation and footer.
- Edit `_data/translations.yml` for the English and Chinese biography and interface copy.

SEO metadata and RSS are supplied by `jekyll-seo-tag` and `jekyll-feed`.
The English and Chinese articles imported from Techlab preserve their content,
publication dates, original `.html` URLs, and `source_url` metadata.

## Preview and deploy

Install Ruby and Bundler, then run:

```sh
bundle install
bundle exec jekyll serve
```

Open <http://localhost:4000>. Add `--drafts` to preview drafts. Restart the server
after editing `_config.yml`. Run `bundle exec jekyll build` for a production build.

In [Settings → Pages](https://github.com/AspadaX/aspadax.github.io/settings/pages),
select **Deploy from a branch**, **main**, and **/ (root)**. Commits to `main`
then rebuild the site automatically; no custom workflow is needed.
