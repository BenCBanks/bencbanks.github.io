# BenCBanks.com

Ben Banks's personal website. GitHub Pages builds it from the `master` branch with Jekyll.

## Write or update a post

Use [Pages CMS](https://app.pagescms.org/) and choose `BenCBanks/bencbanks.github.io`. The [plain-language publishing guide](.github/PUBLISHING.md) covers drafting in Google Docs, adding an image, saving an unpublished draft, and publishing it. You do not need to edit Markdown, YAML, HTML, or Git commands to publish a post.

The editor reads [.pages.yml](.pages.yml). Its GitHub App must be installed for this repository before the editor can be used.

## Site source

Articles are in `_posts/`; layouts are in `_layouts/`; styles are in `assets/css/main.scss`. The existing GitHub Actions workflow builds and deploys changes to `master`. The custom domain is `bencbanks.com`.
