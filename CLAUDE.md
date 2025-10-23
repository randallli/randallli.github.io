# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a personal portfolio and game development website built with Jekyll using the Beautiful Jekyll theme. The site showcases table-top and indie games including "Last cupful" (a water-fight playground game), "Insider Fruit Trading" (a board game about trading), and "Sirtet" (a tetris-variant game). It's hosted on GitHub Pages at tripleli.com.

## Development Commands

### Local Development
```bash
# Install dependencies
bundle install

# Run local development server (serves at http://localhost:4000)
bundle exec jekyll serve

# Build the site (outputs to _site/)
bundle exec jekyll build

# Serve with live reload
bundle exec jekyll serve --livereload
```

### Deployment
The site automatically deploys to GitHub Pages when changes are pushed to the `main` branch. No manual deployment is needed.

## Architecture

### Jekyll Theme Structure
This site uses the **Beautiful Jekyll** theme (v6.0.1) as a gem-based theme:
- Theme files are loaded from the `beautiful-jekyll-theme` gem
- Local customizations override theme defaults
- Primary configuration in `_config.yml`

### Key Directories
- `_layouts/` - Custom page layouts (base, default, home, minimal, page, post)
- `_includes/` - Reusable HTML components (header, footer, nav, comments, analytics, etc.)
- `_posts/` - Blog posts using Jekyll's naming convention: `YYYY-MM-DD-title.md`
- `assets/` - Static assets (CSS, JavaScript, images, Rive animations)
- `_site/` - Generated static site (excluded from git)

### Content Pages
Game project pages at the root:
- `cup.md` - Last cupful game
- `fruittrading.md` - Insider Fruit Trading game
- `sirtet.md` - Sirtet game
- `aboutme.md` - About page
- `newsletter.md` - Newsletter signup
- `playtest.md` - Playtest information

Each uses Jekyll front matter with `layout: page` and includes redirect mappings for alternative URLs.

### Configuration (_config.yml)
Site identity and branding:
- Title: "Triple Li"
- Author: Randall Li
- Custom navigation with nested menu items
- Social links: email, GitHub, Patreon, LinkedIn, Discord
- Google Analytics: gtag "G-6PB8REBYBS"

Theme settings:
- Timezone: America/Toronto
- Pagination: 5 posts per page
- Markdown: kramdown with GFM input
- Permalinks: `/:year-:month-:day-:title/`

### Page Front Matter
All markdown pages require YAML front matter with `---` delimiters:
```yaml
---
layout: page  # or 'post' for blog posts
title: Page Title
subtitle: Optional subtitle
redirect_from:
  - /alternative-url
  - /another-url
---
```

Blog posts automatically use `layout: post` with comments and social sharing enabled by default.

## Content Management

### Adding Blog Posts
Create files in `_posts/` with naming format: `YYYY-MM-DD-title.md`

Required front matter:
```yaml
---
layout: post
title: "Post Title"
subtitle: "Optional subtitle"
tags: [tag1, tag2]
---
```

### Adding New Pages
Create `.md` files at the root or in subdirectories. All pages need front matter even if minimal:
```yaml
---
---
```

### URL Redirects
Use `jekyll-redirect-from` plugin in front matter:
```yaml
redirect_from:
  - /old-url
  - /alternative-url
```

## Important Notes

- **Never commit _site/**: Generated files excluded via .gitignore
- **Testing before commit**: Always run `bundle exec jekyll build` to catch errors
- **Theme updates**: Beautiful Jekyll is gem-based, update with `bundle update beautiful-jekyll-theme`
- **Mobile-first**: Theme is designed for responsive display
- **YAML front matter required**: All content pages must have front matter delimiters even if empty
