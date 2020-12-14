---
title: "Make a hugo blog"
date: 2020-12-14T19:38:06+01:00
slug: "make-hugo-blog"
description: "About making a hugo blog site"
keywords: ["blog", "hugo"]
draft: false
tags: ["blog", "hugo"]
math: false
toc: false
---

## Motivation

It's planned to preserve the solutions to whatever (technical) problems I encountered and deem
worthy of being stored forever. Other people might find some pages useful and if so then even
better.

## Hugo

Hugo seems to be a quite handy tool. Write `md` and get a static webpage. Great!
As theme the `solar-theme-hugo` was chosen. Support for tags was added because this seems quite
necessary to navigate the posts.

## Build

There is no automatic workflow for now. Manually create the sources with:
```
hugo --minify
```

## Deploy

