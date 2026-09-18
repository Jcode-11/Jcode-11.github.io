---
title: "How to Publish Your First Post"
description: "No admin panel needed: add a Markdown file, push to GitHub, and the blog updates automatically."
slug: "publish-your-first-post"
date: 2026-09-18T10:00:00+08:00
categories:
  - Guide
tags:
  - Blog
  - Guide
---

Every post in this blog lives under `content/post/`. Each post gets its own folder, with the body written in `index.md`.

## Create a post

Create a folder for the post, for example:

```text
content/post/my-first-post/index.md
```

Then add its front matter and content:

```md
---
title: "My First Post"
description: "A one-line summary of the post."
date: 2026-09-18T09:00:00+08:00
categories: [Life]
tags: [Notes]
---

Start writing here.
```

## Publish

Commit and push your changes to GitHub’s `main` branch. The deployment workflow will build and publish the site to GitHub Pages automatically.
