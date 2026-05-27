# tech-blog

## Local environment setup

```bash
hugo new site . --force
git submodule add --depth=1 https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
git submodule update --init --recursive # needed when you reclone your repo (submodules may not get cloned automatically)
```

## GitHub Pages Verified Domains

* Settings > Pages > Verified domains > Add domain
* Follow the instructions to verify your domain (aka TXT record creation)

## License

- Source code and configuration: MIT License
- Written content in `content/`: CC BY-NC 4.0
