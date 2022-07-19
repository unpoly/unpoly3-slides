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



----

Modification time or content hash?
----------------------------------


Servers can use both `Last-Modified` and `ETag`, but `ETag` always takes precedence.

It's easier to mix in additional data into an `ETag`, e.g. the ID of the logged in user or the currently deployed commit revision.


---

Some other cases from Unpoly's own features:

- Polling stops when the reloading fragment is aborted
- Pending validations are aborted when the observed field is aborted


----

## Strict target derivation

Disable with:

```js
up.fragment.config.verifyDerivedTarget = false
up.fragment.config.targetDerivers.push('*')
```

---

## Cache Revalidation

Disable with:

```js
up.fragment.config.autoRevalidate = false
up.network.config.cacheEvictAge = up.network.config.cacheExpireAge = 15 * 60 * 1000
```
