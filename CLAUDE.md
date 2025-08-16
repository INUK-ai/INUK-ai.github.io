# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Korean IT blog built with Jekyll using the Chirpy theme. The site features technical writing about programming topics (JavaScript, React, CSS, Spring Security) and is deployed on GitHub Pages at `https://INUK-ai.github.io`.

## Development Commands

### Local Development
```bash
# Start development server (preferred)
./tools/run
# Alternative: bundle exec jekyll serve -H 0.0.0.0 -l

# Build site for production
JEKYLL_ENV=production bundle exec jekyll build

# Test built site
./tools/test
# With custom config: ./tools/test -c "_config.yml,_config.custom.yml"
```

### JavaScript/CSS Build System
```bash
# Build JavaScript bundles for production
npm run build

# Watch for changes during development  
npm run watch

# Lint SCSS files
npm test

# Fix linting issues automatically
npm run fixlint
```

### Ruby Dependencies
```bash
# Install Ruby dependencies
bundle install

# Update dependencies
bundle update
```

## Site Architecture

### Content Structure
- **Posts**: `_posts/` - Blog posts in Korean with front matter (categories: AIBE/FRONT-END, tags)
- **Pages**: `_tabs/` - Static pages (about, archives, categories, tags)
- **Data**: `_data/` - Site configuration, locales (24+ languages), contact info
- **Layouts**: `_layouts/` - Jekyll templates for different page types

### Theme System
- **Base Theme**: Jekyll-theme-chirpy v6.5.3
- **Language**: Korean (ko-KR) with Seoul timezone
- **Comments**: Giscus integration for GitHub-based comments
- **PWA**: Enabled with offline caching
- **Features**: Dark/light mode, search, analytics support, SEO optimization

### JavaScript Architecture
- **Source**: `_javascript/` - Modular ES6+ JavaScript  
- **Build**: Rollup.js with Babel transpilation and Terser minification
- **Output**: `assets/js/dist/` - Minified bundles (commons, home, categories, page, post, misc)
- **Components**: Back-to-top, clipboard, image handling, search, sidebar, TOC

### Styling System
- **Source**: `_sass/` - Organized SCSS with theme support
- **Structure**: addon/, colors/, layout/ directories for modular styles
- **Features**: Syntax highlighting, responsive design, theme switching
- **Linting**: Stylelint with SCSS standard configuration

### Post Guidelines
- File naming: `YYYY-MM-DD-Title.md` in `_posts/`
- Front matter: title, date (+019:00 timezone), categories, tags
- Categories: Use existing structure like `[AIBE, FRONT-END]`
- Language: Content written in Korean
- Images: Store in `assets/img/` directory

### Configuration Notes
- Main config: `_config.yml` (Jekyll settings, theme config, SEO)
- Gemspec: `jekyll-theme-chirpy.gemspec` (theme dependencies)  
- Package.json: JavaScript build tools and linting configuration
- Excluded from build: docs/, tools/, README.md, LICENSE, config files

### Deployment
- Platform: GitHub Pages
- Branch: Automatic deployment from main branch
- Build: GitHub Actions handles Jekyll build and deployment
- Domain: `INUK-ai.github.io` with custom domain support available