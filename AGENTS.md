# AGENTS.md

This is a Hugo-based static blog site using the PaperMod theme. It hosts
personal writing — experiences, opinions, and project announcements — not
technical tutorials or how-to content.

## Project Structure

```
tech-blog/
├── content/          # Blog posts and pages (Markdown + YAML frontmatter)
├── layouts/          # Custom layouts and partials (overrides theme)
├── themes/PaperMod/  # Git submodule - Hugo theme (do not edit directly)
├── hugo.toml         # Site configuration
├── devbox.json       # Development environment (Hugo + Git)
├── public/           # Generated static site (gitignored)
└── resources/        # Hugo cache (gitignored)
```

## Build Commands

### Local Development
```bash
hugo server                   # Start dev server at http://localhost:1313
hugo server -D               # Include draft posts
hugo server --disableFastRender  # Full rebuild on changes
```

### Production Build
```bash
hugo              # Build to /public/ for deployment
hugo --gc         # Build and clean up unused cache files
hugo --minify     # Minify HTML/CSS/JS output
```

### Theme Management
```bash
git submodule update --init --recursive  # Reinitialize submodules after clone
```

### Linting
```bash
# No JavaScript/TypeScript linting - this is a static Hugo site
# For content validation, Hugo implicitly validates frontmatter on build
```

### Testing
```bash
# No automated tests - static content site
# Manual verification: hugo server, navigate to posts, check rendering
```

## Blogging Goals

* Personal experiences
* Opinions
* Judgment about specific topics
* Announcing projects
* NO technical content since LLM

## Code Style Guidelines

### Page Focus

* Homepage = “Why should you read this?”
* About = “Why should I be trusted?”

### Hugo Configuration (hugo.toml)
- Use TOML syntax with lowercase keys and underscores
- Use 2-space indentation for nested tables
- Quote strings that contain special characters or URLs

### Content Frontmatter
All posts must use YAML frontmatter (Hugo shortcode syntax `+++`):

```yaml
+++
date = 'YYYY-MM-DDTHH:MM:SS-04:00'
title = 'Post Title'
draft = true  # Remove or set to false before publishing
+++
```

Frontmatter fields:
- `date`: Required. ISO 8601 format with timezone offset
- `title`: Required. Descriptive title
- `draft`: Optional. Set to `false` or omit when publishing

### Markdown Content
- Use ATX-style headings (`##` not underline style)
- Keep lines under 100 characters where practical
- No emoji in content (plain text or HTML entities only)
- Avoid code blocks, command snippets, and technical walkthroughs — LLMs cover that better

### File Naming
- Post files: lowercase with hyphens (e.g., `why-i-shipped-project-x.md`)
- New posts: `hugo new posts/your-post-title.md`

### Go Templates (layouts/)
- Indent with 2 spaces
- Use Hugo template functions (`{{ .Title }}`, `{{ range }}`, etc.)
- Prefer `{{-` and `-}}` to trim whitespace around template actions
- Use `absURL` for absolute URLs in sitemap and feeds

### Directory Structure
- Content posts go in `content/posts/`
- Static pages go in `content/` (about.md, etc.)
- Custom layouts override theme files in `layouts/`

## Content Guidelines

### Writing Style
- Personal, direct, and opinionated
- Share experiences, judgments, and what you learned — not step-by-step instructions
- Announce projects with context: why you built it, what you believe, what surprised you
- Use clear, descriptive headings
- Do not write tutorials, troubleshooting guides, or reference docs

### Licensing
- Written content in `content/`: CC BY-NC 4.0
- Source code and configuration: MIT License
- Do not include proprietary or confidential information

### SEO
- Use descriptive titles (50-60 characters)
- Include relevant keywords in frontmatter (optional)

## Common Tasks

### Creating a New Post
```bash
hugo new posts/my-new-post.md
# Edit the generated file with your content
```

### Publishing a Post
1. Remove `draft = true` from frontmatter or set to `false`
2. Verify with `hugo server -D` (draft flag shows all drafts)
3. Build with `hugo` when ready to deploy

### Adding Images
Place images in `static/images/` and reference with:
```markdown
![Alt text](/images/filename.png)
```

## Deployment

This site uses GitHub Pages. The workflow is defined in `.github/workflows/hugo.yaml`.
Push to main branch triggers automatic build and deployment.

## Environment Configuration

The `devbox.json` specifies:
- Hugo 0.152.2
- Git 2.51.2

Use `devbox run hugo` or install Hugo locally for builds.
