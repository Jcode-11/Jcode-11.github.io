---
title: "如何发布第一篇文章"
description: "不需要后台系统：新建一篇 Markdown，推送到 GitHub，博客就会自动更新。"
slug: "publish-your-first-post"
date: 2026-09-18T10:00:00+08:00
categories:
  - 使用指南
tags:
  - 博客
  - 使用指南
---

这个博客的文章都放在 `content/post/` 目录中；一个文件夹就是一篇文章，正文写在其中的 `index.md`。

## 新建文章

新建一个文章目录，例如：

```text
content/post/我的第一篇文章/index.md
```

然后填写文章信息和正文：

```md
---
title: "我的第一篇文章"
description: "用一句话说明这篇文章。"
date: 2026-09-18T09:00:00+08:00
categories: [生活]
tags: [随笔]
---

从这里开始写正文。
```

## 发布

将修改提交并推送到 GitHub 的 `main` 分支。部署流程会自动构建网站并发布到 GitHub Pages。
