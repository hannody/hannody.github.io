# Publishing a New Article

End-to-end steps to take a new post from draft to live on https://hannody.github.io/.

## 1. Create the markdown file

Place the post under `content/posts/` (use a subfolder like `K8s/` to group by topic):

```bash
hugo new content/posts/<topic>/<slug>.md
```

Or create it manually:

```
content/posts/<topic>/<my-post>.md
```

## 2. Add required front matter

Every post must start with YAML front matter. Minimum fields:

```yaml
---
title: "Human Readable Title"
date: 2026-05-23
draft: false
slug: "url-safe-slug"
tags:
  - tag-one
  - tag-two
---
```

Notes:
- `draft: false` — drafts are not built by default.
- `slug` controls the URL: `/posts/<topic-lowercased>/<slug>/`.
- `tags` must be a YAML list (one item per line with `- `).
- Optional `aliases: ["/old/path/"]` to redirect from a previous URL.

## 3. Write the content

Standard Markdown. Common conventions enforced by markdownlint:
- No more than one consecutive blank line (MD012).
- Headings start at `##` (the `title` field becomes the `<h1>`).
- Fenced code blocks must declare a language: ` ```bash `, ` ```yaml `, etc.

## 4. Preview locally

```bash
hugo server -D
```

Open http://localhost:1313/ and verify:
- The post appears on the home page and under `/posts/`.
- The slug, tags, and date render correctly.
- Code blocks, images, and links look right.

Stop with `Ctrl+C` when done.

## 5. Lint (optional but recommended)

If `markdownlint-cli2` is installed:

```bash
markdownlint-cli2 "content/**/*.md"
```

Fix any errors before committing — CI will run the same check.

## 6. Build check (optional)

```bash
hugo --minify
```

Should complete without warnings about missing layouts or broken references.

## 7. Commit and push

```bash
git add content/posts/<topic>/<my-post>.md
git commit -m "post: add <short title>"
git push
```

## 8. Watch the deploy

Two workflows run automatically:

1. **Deploy Hugo** — builds the site and pushes `public/` to the `gh-pages` branch.
2. **pages-build-deployment** — GitHub's internal job that publishes `gh-pages` to the CDN.

Check both:

```bash
gh run list --workflow "Deploy Hugo" --limit 3
gh run list --workflow "pages-build-deployment" --limit 3
```

Both should show ✓. If `pages-build-deployment` fails (occasional `401 Bad credentials`), re-run it:

```bash
gh run rerun $(gh run list --workflow "pages-build-deployment" --status failure --limit 1 --json databaseId --jq '.[0].databaseId') --failed
```

## 9. Verify the live site

GitHub Pages caches via Fastly with `max-age=600` (~10 min). Bypass the cache with a query string:

```bash
curl -s "https://hannody.github.io/posts/?v=$RANDOM" | rg -o "<your title or slug>"
```

Or open the post URL directly:

```
https://hannody.github.io/posts/<topic>/<slug>/
```

If the old version still shows in your browser, hard refresh (Cmd+Shift+R) or open a private window.

## Troubleshooting

| Symptom                          | Likely cause                              | Fix                                                                                                                                            |
| -------------------------------- | ----------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| Post missing from local server   | `draft: true` or future `date`            | Set `draft: false`, check date                                                                                                                 |
| `Deploy Hugo` fails on submodule | `themes/PaperMod` pointer is stale        | `cd themes/PaperMod && git fetch && git checkout master && cd ../.. && git add themes/PaperMod && git commit -m "Update PaperMod" && git push` |
| CI green but live site stale     | `pages-build-deployment` 401 or CDN cache | Re-run the failed job (step 8); wait up to 10 min for cache                                                                                    |
| Markdownlint MD012 in CI         | Multiple consecutive blank lines          | Collapse to a single blank line                                                                                                                |
| Wrong URL after publish          | `slug` missing or mismatched              | Set `slug:` in front matter; add `aliases:` if URL changed                                                                                     |
