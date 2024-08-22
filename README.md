# James Beith

Technical writing about Python & Django.

[www.jamesbeith.co.uk](https://www.jamesbeith.co.uk/)

---

## Site

The site is built using [Hugo](https://gohugo.io/). The theme is [PaperMod](https://github.com/adityatelange/hugo-PaperMod) which is installed unmodified as a Git Submodule.

## Content

Content is written in Markdown. Hugo uses [Goldmark](https://github.com/yuin/goldmark/), a [CommonMark](https://spec.commonmark.org/current/) compliant parser, to render Markdown to HTML.

To create new content use.

```shell
hugo new content content/til/$(date '+%Y-%m-%d')/title.md
```

To run the local webserver use.

```shell
hugo server --buildDrafts --buildFuture
```

## Linting

[Vale](https://vale.sh/) and [Prettier](https://prettier.io/) are used.

To install Vale, first install the [Homebrew](https://brew.sh/) package manager. Then you can install Vale using.

```shell
brew install vale
```

You then need to download and install Vale’s external configuration sources using.

```shell
vale sync
```

To install Prettier, first install Node.js using a version manager such as [fnm](https://github.com/Schniz/fnm?tab=readme-ov-file#installation). Then enable the Corepack (Yarn) package manager.

```shell
corepack enable
```

Then you can install Prettier using.

```shell
yarn install
```

To run the linting tools use.

```shell
yarn lint
```

You can also run the linting tools to automatically apply all available fixes using.

```shell
yarn fix
```

## Deployment

The site is deployed using [Cloudflare Pages](https://developers.cloudflare.com/pages/framework-guides/deploy-a-hugo-site/).
