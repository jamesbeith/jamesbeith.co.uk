# James Beith

Technical writing about Python & Django.

[www.jamesbeith.co.uk](https://www.jamesbeith.co.uk/)

---

## Site

The site is built using [Hugo](https://gohugo.io/). The theme is [PaperMod](https://github.com/adityatelange/hugo-PaperMod) which is installed unmodified as a Git Submodule.

## Content

To create new content use.

```shell
hugo new content content/til/$(date '+%Y-%m-%d')/title.md
```

To run the local webserver use.

```shell
hugo server --buildDrafts --buildFuture
```

## Linting

[Prettier](https://prettier.io/) is used.

To run the linting tools, first ensure Node.js is installed using a version manager such as [fnm](https://github.com/Schniz/fnm?tab=readme-ov-file#installation). Then enable the Corepack (Yarn) package manager.

```shell
corepack enable
```

Install the dependencies using.

```shell
yarn install
```

You can then run the linting tools with.

```shell
yarn lint
```

You can also run the linting tools to automatically apply all fixes with.

```shell
yarn fix
```

## Deployment

The site is deployed using [Cloudflare Pages](https://developers.cloudflare.com/pages/framework-guides/deploy-a-hugo-site/).
