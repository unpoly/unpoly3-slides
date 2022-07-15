---
marp: true
title: Unpoly 3
theme: unpoly3-slides
---



Disabling outside a form submission
-----------------------------------

Fields marked with `[up-validate]` and `[up-watch]` (former `[up-disable]`) can also disable other fields while loading:

```html
<select up-validate=".employees" up-watch-disable=".employees">
```

Or just use `{ disable }` and `[up-disable]` options for any render operation:

```js
up.reload('.city-options', { disable: '.country-select' })
```
