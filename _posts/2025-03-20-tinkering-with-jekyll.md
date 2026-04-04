---
layout: post
title: Tinkering with Jekyll and GitHub Pages
description: >
  Notes from setting up this site — what worked, what didn't, and a few things worth knowing.
---

Setting up a Jekyll site on GitHub Pages is mostly straightforward, with a few sharp edges. Here's what I ran into.

**Remote themes** — GitHub Pages supports a limited set of gems natively. If you want to use a theme not on that list, you use `remote_theme` instead of `theme`. The tradeoff is slower build times and slightly less control, but for a personal site it's fine.

**The `_data` folder** — A lot of the site's content lives here. Author info, resume data, navigation — all configurable without touching layout files. Worth understanding early.

**YAML is unforgiving** — A stray bracket or missing quote will silently break things. If a page renders blank, check your front matter and data files first.

**Free vs. PRO themes** — Some Hydejack layouts (like the resume layout) are PRO-only. The documentation isn't always clear about this upfront.

Overall, Jekyll + GitHub Pages is a solid setup for a site like this. Fast, free, and fully in your control.
