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

The build is automated with github actions. Release artifacts are created automatically too. Only
tags run the github actions.
```yaml
name: CI

on:
  push:
    tags:
    - 'v*'

  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v2
      
      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v2.4.13
        with:
          extended: true

      - name: Build
        run: hugo --minify

      - name: Package
        run: zip -r release public

      - name: Create Release
        id: create_release
        uses: actions/create-release@v1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          tag_name: ${{ github.ref }}
          release_name: Release ${{ github.ref }}
          draft: false
          prerelease: false
          
      - name: Upload Release Asset
        id: upload-release-asset 
        uses: actions/upload-release-asset@v1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          upload_url: ${{ steps.create_release.outputs.upload_url }}
          asset_path: ./release.zip
          asset_name: release.zip
          asset_content_type: application/zip
```

## Deploy

The deployment is as easy as downloading the `release.zip` file and pointing a web server to the 
unzipped folder.
```bash
#!/usr/bin/env bash

curl -OL https://github.com/ftann/ctor-blog/releases/latest/download/release.zip
unzip release.zip
mv public blog
chown 2000.2000 -R blog
docker restart nginx
```
