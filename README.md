# Spas Bojanov — Developer blog

A personal blog built with Jekyll and published by GitHub Pages from the root of
`master`.

## Write a post

Create a Markdown file in `_posts` named `YYYY-MM-DD-title-of-post.md`, for
example `_posts/2026-11-04-my-first-post.md`:

```markdown
---
layout: post
title: "My first post"
---

Write your post here. **Markdown** and [links](https://example.com) work.
```

The date in the filename is the publication date. Posts dated in the future are
not published until their date, when the site is next built. The home page lists
published posts automatically, newest first. Commit and push to `master` to
publish; GitHub Pages builds the site for you.

## Preview locally

Install Ruby and Jekyll, then run `jekyll serve` from this directory and open
<http://localhost:4000>. Jekyll processes the templates, so opening
`index.html` directly will not show the post list correctly.

Edit `index.html` to update the introduction and `styles.css` to adjust the design.
