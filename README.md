# lab.schoonens.nl

Personal blog by Camiel Schoonens, built with [Jekyll](https://jekyllrb.com) and the [Chirpy](https://github.com/cotes2020/jekyll-theme-chirpy) theme. Hosted on [GitHub Pages](https://pages.github.com) and available at [lab.schoonens.nl](https://lab.schoonens.nl).

## About

A personal blog covering tech, tools, and tinkering. Posts are written in Markdown and support categories, tags, and RSS.

## Writing a Post

Create a new file in `_posts/` following this naming convention: _posts/YYYY-MM-DD-your-post-title.md

With this front matter:

---
layout: post
title: "Your Post Title"
date: YYYY-MM-DD HH:MM:SS +0200
categories: [Category]
tags: [tag1, tag2]
description: A short summary of the post.
pin: false
toc: false
published: true
---
```

## Deployment

Pushing to `main` triggers a GitHub Actions workflow that builds and deploys the site automatically. Build status can be monitored in the [Actions tab](https://github.com/camielschoonens/camielschoonens.github.io/actions).

## RSS Feed

Available at [lab.schoonens.nl/feed.xml](https://lab.schoonens.nl/feed.xml).
