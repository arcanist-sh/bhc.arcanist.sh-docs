# bhc.arcanist.sh-docs

Documentation site for [bhc](https://github.com/arcanist-sh/bhc) — the Basel Haskell Compiler.

**Live site:** https://bhc.arcanist.sh/docs

## Overview

This repository contains the documentation for bhc. Built with [Zola](https://www.getzola.org/) using the `brutalist-blueprint` theme.

## Development

### Prerequisites

- [Zola](https://www.getzola.org/documentation/getting-started/installation/) (0.18+)
- Or use [mise](https://mise.jdx.dev/) to manage the environment:
  ```bash
  mise install
  ```

### Local Development

```bash
# Start development server with live reload
zola serve

# Build for production
zola build

# Check for errors
zola check
```

The development server runs at `http://127.0.0.1:1111` by default.

## Project Structure

```
.
├── config.toml          # Zola configuration
├── content/
│   └── docs/            # Documentation pages
│       ├── profiles/    # Runtime profiles
│       └── targets/     # Compilation targets
├── static/              # Static assets
└── themes/
    └── brutalist-blueprint/  # Brutalist documentation theme
        ├── sass/             # SCSS stylesheets
        └── templates/        # HTML templates
```

## Content Structure

- **Installation** - Getting bhc installed
- **Quick Start** - First steps with bhc
- **Compatibility** - GHC compatibility information
- **Profiles** - Runtime profiles (default, server, numeric, edge)
- **Targets** - Compilation targets (native, WASM)

## Adding Documentation

1. Create a new `.md` file in the appropriate directory under `content/docs/`
2. Add frontmatter with `title`, `description`, and `weight` (for ordering)
3. Write content in Markdown

Example:

```markdown
+++
title = "My New Page"
description = "Description of the page"
weight = 5
+++

Content goes here...
```

## Deployment

The site is automatically deployed on push to the main branch.

## License

MIT
