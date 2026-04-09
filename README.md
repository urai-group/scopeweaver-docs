# Scopeweaver web based documentation

This repository serves as a live documentation of the Scopeweaver project.

The website is live at: 
[https://urai-group.github.io/scopeweaver-docs/](https://urai-group.github.io/scopeweaver-docs/)

## How to add pages?

Create an appropriately named markdown file with the following 
content inside [`docs`](./docs/) directory:

```yml
---
title: "Page Title"                # The page title shown in the sidebar and page header
description: "Page description"  # Short description of the page
layout: default                  # Layout to use: default, minimal, etc.

# sidebar: auto             # Options: auto, hide, or custom
# collapsible: false        # Make sidebar section collapsible
# featured: true            # Highlight this page in navigation

# author: "Author Name"
# last_modified_at: 2026-04-07
# categories: [docs, guide]
---

You content starts here. Write your page as a markdown

```

## Advanced

To change the name of your file in the sidebar, use the `nav` property
in the [`_config.md`](./_config.yml).


```yaml
nav:
  - Home: index.md
  # the custom name would appear on the sidebar
  - Your Custom name: your-file-in-docs-directory.md
```

To add subdocs under a section, you can use create a folder inside of
[`docs`](/docs), and under this folder, have your markdown files. This
would automatically appear as a nested entry in the sidebar.

```
docs
|──tutorials/
    ├─── intro.md
    ├─── advanced.md
```

