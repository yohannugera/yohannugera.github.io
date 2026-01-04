---
date:
    created: 2026-01-04
---

# First Blog for 2026

I want to make a habit of writing/publishing everything that I do technically here so that I can keep a track of what
I'm doing and to improve my writing skills. So, as my first job, I wanted to simplify my blogging experience with a
github actions process to automatically upload what I'm writing to my GitHub static-hosted site.

This is the script of workflow I'm gonna use:

```text
name: Documentation
on:
  push:
    branches:
      - master
      - main
permissions:
  contents: read
  pages: write
  id-token: write
jobs:
  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    steps:
      - uses: actions/configure-pages@v5
      - uses: actions/checkout@v5
      - uses: actions/setup-python@v5
        with:
          python-version: 3.x
      - run: pip install -r requirements.txt
      - run: mkdocs build --clean
      - uses: actions/upload-pages-artifact@v4
        with:
          path: site
      - uses: actions/deploy-pages@v4
        id: deployment
```

Let's see if it works!