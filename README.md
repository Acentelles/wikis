# wikis

Personal research wikis, built with [mdBook](https://rust-lang.github.io/mdBook/)
and published to GitHub Pages.

Published books:

- `isogenies/` , Isogeny-Based Cryptography Notes

## Local build

```sh
cargo install mdbook mdbook-katex
mdbook serve isogenies/book
```

## Deploy

Pushes to `main` trigger `.github/workflows/deploy.yml`, which builds each
book into `site/<name>/` and publishes the whole `site/` as the Pages
artifact. The landing page at `landing/index.html` is copied to `site/index.html`.

The site is unlisted (`noindex, nofollow` on the landing page), but the
published HTML is readable by anyone with the URL. The repo itself is
private.
