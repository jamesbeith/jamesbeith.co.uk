---
title: "How to optimise the order of HTML head elements using Capo"
date: "2025-06-23T11:00:00+10:00"
---

I recently came across [Capo](https://rviscomi.github.io/capo.js/), a tool that helps identify HTML head elements that are out of order.

Using the Chrome extension, it offered a few suggestions for my project.

- Move the [es-module-shims](https://github.com/guybedford/es-module-shims) asynchronous script and `importmap` synchronous script to be a lot higher
- All the synchronous styles `<link rel=stylesheet>` to come before deferred scripts `<script defer src>`
- Remove some `preload` elements which were having little to no effect (as the files were already discoverable by other `link` elements)
- Put everything else such as `meta`, non-stylesheet `link`, and JSON `script` elements last

You can see how Capo categorises all the elements into 11 groups on the docs [rules page](https://rviscomi.github.io/capo.js/user/rules/).
