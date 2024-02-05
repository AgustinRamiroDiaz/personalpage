+++
authors = ["Duck Quack"]
title = "Dinamically get the URL of a GitHub Pages site in a GitHub Action"
description = ""
date = 2024-02-05
[taxonomies]
tags = ["GitHub", "GitHub Actions", "CI/CD"]
[extra]
toc = false
[extra.comments]
id = ""

+++

# How to get the URL of a GitHub Pages site in a GitHub Action

```yml
# this script is executed on every push to main
on:
  push:
    branches:
      - main

name: Build and deploy GH Pages
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: checkout # we first checkout the code
        uses: actions/checkout@v4
      - name: get url # then we get the url of the github pages site
        env:
          GH_TOKEN: ${{ github.token }}
        run:
          | # we use the gh cli to get the url and save it in an environment variable
          url=$(gh api "repos/$GITHUB_REPOSITORY/pages" --jq '.html_url')
          echo "url=$url" >> $GITHUB_ENV
      - name: build_and_deploy
        uses: shalzz/zola-deploy-action@v0.18.0
        env:
          # Target branch
          PAGES_BRANCH: gh-pages
          BUILD_FLAGS: --base-url ${{ env.url }} # we pass the url to zola
          # Provide personal access token
          # TOKEN: ${{ secrets.TOKEN }}
          # Or if publishing to the same repo, use the automatic token
          TOKEN: ${{ secrets.GITHUB_TOKEN }}
```
