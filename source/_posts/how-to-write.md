---
title: 写博客速查：Markdown 与 Front Matter
date: 2026-09-30 11:00:00
categories: 教程
tags: [Markdown, 速查]
cover: /img/cover-2.svg
description: 以后写文章时翻出来看的速查表。
---

## 新建一篇文章

在仓库的 `source/_posts/` 里新建 `文章英文短名.md`，开头写：

```yaml
---
title: 文章标题
date: 2026-10-05 20:00:00
categories: 学习
tags: [comp9900, react]
cover: /img/cover-3.svg   # 可换成任意图片地址
description: 首页卡片上显示的一句话简介
---
```

提交后 1–2 分钟网站自动更新。

## 常用 Markdown

- **加粗**、*斜体*、`行内代码`
- [链接](https://github.com/zhangxurui2024-lgtm)
- 图片：`![说明](/img/banner.svg)`

> 引用块会显示成带紫色边的卡片。

{% note info %}
Butterfly 还支持提示块：`{% raw %}{% note info %}...{% endnote %}{% endraw %}`，有 info / success / warning / danger 等样式。
{% endnote %}
