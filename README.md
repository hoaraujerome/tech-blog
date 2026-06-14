# tech-blog

[![Hugo](https://img.shields.io/badge/Hugo-0.152.2-FF4088?logo=hugo)](https://gohugo.io/)
[![GitHub Pages](https://img.shields.io/badge/GitHub%20Pages-deployed-222?logo=github)](https://blog.hoaraujerome.com/)
[![License: MIT](https://img.shields.io/badge/code-MIT-blue.svg)](LICENSE)

Personal tech blog by [Jerome Hoarau](https://github.com/hoaraujerome) — hands-on notes on platform engineering, Kubernetes, cloud (AWS & Azure), Terraform, and infrastructure as code.

**Live site:** [blog.hoaraujerome.com](https://blog.hoaraujerome.com/)

## What you'll find here

- Kubernetes & homelab experiments (kubeadm, AWS, AKS)
- Cloud infrastructure design and trade-offs
- Terraform, CDKTF, and infrastructure-as-code workflows
- DevOps practices, cert-manager, Traefik, and real-world troubleshooting

Posts live in `content/posts/`. The [About](https://blog.hoaraujerome.com/about/) page has more background.

## Stack

| Layer | Choice |
|-------|--------|
| Generator | [Hugo](https://gohugo.io/) 0.152.2 (extended) |
| Theme | [PaperMod](https://github.com/adityatelange/hugo-PaperMod) (git submodule) |
| Hosting | GitHub Pages |
| CI/CD | `.github/workflows/hugo.yaml` |
| Local env | [devbox](https://www.jetify.com/devbox) |

## Quick start

### Clone

```bash
git clone https://github.com/hoaraujerome/tech-blog.git
cd tech-blog
git submodule update --init --recursive
```

The PaperMod theme is a submodule — run the last command after every fresh clone.

### Run locally

**With devbox:**

```bash
devbox shell
hugo server -D
```

Open [http://localhost:1313](http://localhost:1313). Use `-D` to preview draft posts.

**With Hugo installed:**

```bash
hugo server -D
```

### Create a post

```bash
hugo new posts/my-new-post.md
```

Edit the generated file in `content/posts/`, then remove `draft = true` (or set it to `false`) before publishing.

### Production build

```bash
hugo --gc --minify
```

Output goes to `public/`.

## Deployment

Pushes to `main` trigger the GitHub Actions workflow, which builds the site and deploys it to GitHub Pages.

Custom domain: configure **Settings → Pages → Custom domain** on the repo (see [GitHub Pages custom domains](https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site)).

## Project layout

```
content/          # Posts and pages (Markdown + TOML frontmatter)
layouts/          # Custom layouts (override theme)
themes/PaperMod/  # Hugo theme (submodule — don't edit directly)
hugo.toml         # Site configuration
devbox.json       # Local dev environment
```

## License

- **Source code & configuration:** [MIT License](LICENSE)
- **Written content in `content/`:** [CC BY-NC 4.0](CONTENT_LICENSE.md)

## Connect

- [GitHub](https://github.com/hoaraujerome)
- [LinkedIn](https://www.linkedin.com/in/hoaraujerome/)
