# aspadax.github.io

A minimal Jekyll blog using the GitHub Pages-supported Minima theme.

Site address: <https://aspadax.github.io>

## Enable GitHub Pages once

Open [Settings → Pages](https://github.com/AspadaX/aspadax.github.io/settings/pages).
Under **Build and deployment**, set:

- **Source:** Deploy from a branch
- **Branch:** main
- **Folder:** / (root)

Click **Save**. GitHub will build and publish the site. Check the repository's
**Actions** tab for the build result. No custom workflow is needed.

## Publish a new article from GitHub

1. Open the [_posts folder](https://github.com/AspadaX/aspadax.github.io/tree/main/_posts).
2. Choose **Add file → Create new file**.
3. Name it `YYYY-MM-DD-your-post-title.md`, using the publication date.
4. Add the following, replacing the title and article text:

   ```markdown
   ---
   title: "Your post title"
   ---

   Your opening paragraph.

   ## A section heading

   The rest of your article goes here.
   ```

5. Commit the file to `main`. GitHub Pages rebuilds the blog automatically
   once Pages is enabled.

The post layout and author are supplied by `_config.yml`, so only `title` is
required in the front matter. Keep the opening and closing `---` lines.
The homepage lists posts newest first and shows each opening paragraph.
Future-dated posts are hidden until a build runs on or after their date;
there is no scheduled rebuild configured.

## Drafts and images

- Use `_drafts/post-template.md` as a starting point. Drafts do not appear on
  the published site. Move a finished draft into `_posts` and give it the
  dated filename above to publish it.
- This repository is public: draft source files are still visible on GitHub.
- Upload images to `assets/images/`, then embed them in an article with
  `![Description of the image](/assets/images/example.png)`.
- The initial `hello-world` post can be edited or deleted at any time.

## Customize

- `_config.yml`: site title, description, author, GitHub link, and settings.
- `about.md`: the About page.
- `_posts/`: published articles.

## Optional local preview

Install Ruby and Bundler, then run from this directory:

```sh
bundle install
bundle exec jekyll serve
```

Open <http://localhost:4000>. To preview drafts, use
`bundle exec jekyll serve --drafts`. Restart the server after changing
`_config.yml`.

For a production build, run `bundle exec jekyll build`.

See [GitHub's Jekyll guide](https://docs.github.com/en/pages/setting-up-a-github-pages-site-with-jekyll/creating-a-github-pages-site-with-jekyll)
for the supported setup.
