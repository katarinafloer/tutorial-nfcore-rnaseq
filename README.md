# nf-core RNA-seq tutorial

This repository contains a Quarto website for an nf-core RNA-seq tutorial.

View the live tutorial here:

https://katarinafloer.github.io/tutorial-nfcore-rnaseq/

## Preview locally

Install Quarto, then run:

```bash
quarto preview
```

## Render locally

```bash
quarto render
```

The rendered site is written to `_site/`.

## Publish with GitHub Pages

1. Push this repository to GitHub.
2. In the repository settings, open **Pages**.
3. Set **Build and deployment** to **GitHub Actions**.
4. Push to `main`; the workflow in `.github/workflows/publish.yml` will render and publish the site.

Before publishing, update `site-url` and `repo-url` in `_quarto.yml`.
