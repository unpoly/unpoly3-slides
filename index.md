---
marp: true
title: Unpoly 3
theme: unpoly3-slides
---

<!-- _class: no-watermark -->

<div class="title">
  <img src="./images/unpoly3.svg" alt="Unpoly 2" class="title--logo">
  <div class="title--author">
    Henning Koch&nbsp; <a href="https://twitter.com/triskweline">@triskweline</a>
  </div>  
</div>


---

# About Unpoly

[Unpoly](https://unpoly.com) is an unobtrusive JavaScript framework for server-side web applications.\
It enables fast and flexible frontends while keeping rendering logic on the server.

**This presentation is for experienced Unpoly users\
who want to learn about the major changes in Unpoly 3.**



----

Unpoly 3 objectives
===================

<div class="row">
<div class="col" style="flex-grow: 1.5">

### Concurrency

- User clicking faster than the server can respond
- Multiple requests targeting the same fragment
- Forms where everything depends on everything else
- Responses arrive in different order than requests
- Multiple users working on the same backend data (stale caches)
- User losing connection while requests are in flight
- Lie-Fi (spotty Wi-Fi, EDGE, tunnel)
</div>
<div class="col">

### Quality of live improvements

- Optional targets
- More control over `[up-hungry]`
- Shorter data attributes
- More render callbacks
- Strict target derivation
- Better feedback
- Log separated by interactions
- Foreign overlays
</div>


---

<h1 class="color-secondary">This will be a smooth update</h1>

- Upgrade from v2 to v3 will be *much* smoother than going from v1 to v2.
- Almost all are breaking changes are polyfilled by [`unpoly-migrate.js`](https://unpoly.com/changes/upgrading).
- Unpoly 3 keeps aliases for deprecated APIs going back to 2016.\
  You can upgrade from v1 to v3 (without going through v2).



---

<div class="row">
<div class="col">

### Renamed events are aliased

`up.on('up:proxy:load')` will bind to\
`up:request:load`.

### Renamed functions are aliased

`up.modal.close()` will call\
`up.layer.dismiss()`

### Renamed options are aliased

`{ reveal: false }` will be renamed to\
`{ scroll: false }`

</div>
<div class="col">

### Renamed packages are aliased

`up.proxy.config` will return\
`up.network.config`.

### Renamed HTML attributes are aliased

`<a up-close>` will translate to\
`<a up-dismiss>`.
</div>


---

<p>Calls to old APIs will be forwarded to the new version and log a deprecation notice with a trace.
</p>

<img src="./images/log-deprecation.png" alt="New log" class="picture" style="display: block; margin: 0.6em 0 1.1em 0">

<p>This way you upgrade Unpoly, revive your application with few changes,<br>
then replace deprecated API calls under green tests.</p>


---

## Removing aliases from your build

All aliases are shipped as a separate file `unpoly-migrate.js`.

If you prefer errors to warnings, you can set <code>up.migrate.config.logLevel = 'error'</code>.

Once you have fixed deprecated usages you can remove this from your build.

---
<!-- _class: topic -->

# Concurrency


---

# Cache revalidation

Unpoly always cached `GET` responses for a few minutes:

- Problem with stale content
- Many projects configured cache expiry down to a few seconds
- Every non-GET request cleared the entire cache


The cache in Unpoly 3 now follows a *stale-while-revalidate* pattern:

- After rendering stale content from the cache, Unpoly automatically reloads the fragment.
- This is called *cache revalidation*.
- Longer cache times
- The user never sees stale content


---
<!-- _class: no-watermark -->

<img src="images/cache.svg" height="100%">

---

`up.network.config.cacheExpireAge`
----------------------------------

- Defaults to 15 seconds (down from 5 minutes in Unpoly 2)
- When cached content exceeds this age we perform cache revalidation\
 (reload a rendered fragment)


`up.network.config.cacheEvictAge`
---------------------------------

- Defaults to 90 minutes (lower for low-memory devices)
- When cached content exceeds this age we delete it from the cache


---

## Revalidation happens after `up.render()` settles


```js
await up.render(..)
// Fragment is updated, but animation and revalidation may still be underway
```

```js
await up.render(..).finished
// Fragment is fully updated, transitioned and revalidated
```

---

Does this cause more requests?
------------------------------

**No**.

- As many requests as a plain Rails app.
- As many requests as an Unpoly 2 app with short cache expiry.
- **Optionally** we can optimize reloading so it is effectively free for the server.



-----

GET conditional on modification time 
------------------------------------

Browser sends:

```http
GET /foo HTTP/1.1
```

Server responds with content and modification time (e.g. `#updated_at`):


```http
HTTP/1.1 200 OK
Last-Modified: Wed, 15 Nov 2000 13:11:22 GMT
...
```

Browser reloads:

```http
GET /foo HTTP/1.1
If-Modified-Since: Wed, 15 Nov 2000 13:11:22 GMT
```

Server checks modification time and responds **without content**:

```http
HTTP/1.1 304 Not Modified
```


-----

GET conditional on content hash
-------------------------------

Browser sends:

```http
GET /foo HTTP/1.1
```

Server responds with content and a hash over the underlying data:


```http
HTTP/1.1 200 OK
ETag: "x234dff"
...
```

Browser reloads:

```http
GET /foo HTTP/1.1
If-None-Match: "x234dff"
```

Server checks data hash and responds **without content**:

```http
HTTP/1.1 304 Not Modified
```


----


Servers can use both `Last-Modified` and `ETag`, but `ETag` take precedence.

It's easier to mix in additional data into an `ETag`, e.g. the ID of the logged in user or the currently deployed commit revision.


----

Unpoly 3 supports conditional GET when reloading
-------------------------------------------------

Unpoly remembers the `Last-Modified` and `ETag` headers a fragment was delivered with:

```html
<div class='messages' up-time='Wed, 21 Oct 2015 07:28:00 GMT' etag='"x234dff"'>
...
</div>
```

When the fragment is reloaded:

```http
GET /messages HTTP/1.1
X-Up-Target: .messages
If-Modified-Since: Wed, 21 Oct 2015 07:28:00 GMT
If-None-Match: "x234dff"
```

If no fresher data exists, the server may skip rendering and respond with:

```http
HTTP/1.1 304 Not Modified
```

In practice you would use either `Last-Modified` or `ETag`, but never both.




----

All web apps should support conditional GET
-------------------------------------------

This improves **all** cases where we access a previously visited page:

- Cache revalidation.
- Page loads without Unpoly
- [`[up-poll]`](https://unpoly.com/up-poll)


----


Implementing conditional GET in Ruby on Rails
=============================================

If you're not a Rails user, skip to a slide without Ruby code.


----
<!-- _class: no-watermark -->

```ruby
class PostsController < AppplicationController

  # Produce different ETags for different users
  etagger { current_user&.id }

  def show
    @post = Post.find(params[:id])

    # Produces ETag from (1) class name (2) @post.id
    # (3) @post.updated_at (4) view template code (5) flashes.
    # Renders 304 Not Modified if ETag matches If-None-Match header.
    fresh_when @post
  end
  
  def index
    @posts = Post.order(created_at: :desc)

    # Produces ETag from (1) class name (2) scope conditions
    # (3) @posts.maximum(:updated_at) (4) view template code (5) flashes.
    # Renders 304 Not Modified if ETag matches If-None-Match header.
    fresh_when @posts
  end

end
```




----

Default ETags
----------------------------------------

Even without `fresh_when` Rails produces a default ETag by hashing the response body.\
You still pay the rendering time, but won't transmit unchanged HTML.

This default ETag has issues with random tokens (CSRF token, CSP nonce).\
Use [`Rack::SteadyETag`](https://github.com/makandra/rack-steady_etag) to address this.



----


All of this is optional!
------------------------

Cache revalidation will work with or without conditional GET.

Without cache revalidation will cause as many requests as an Unpoly 2 app with short cache expiry.


----


Concurrent updates to the same fragment
=======================================

When two requests target `main`, what should happen?

```
+------+-------------+
| side |    main     |
+------+-------------+
```

### Unpoly 1: Do nothing

Responses would be rendered in whatever order they arrive.

### Unpoly 2: Abort all other requests when navigating

The last request would be the one rendered.\
This also aborted requests in unrelated regions like `side`.

### Unpoly 3: Abort requests within the targeted fragment only

New default render option `{ abort: 'target' }` (navigation or not)


---


Aborting requests targeting a fragment
--------------------------------------


Clicking this link will cancel requests targeting `.region` or its descendants:
    
```html
<a href="/path" up-target=".region">
```

Same when rendering programmatically:

```js
up.render({ url: '/path', target; '.region' })
```

Aborting without rendering:

```js
up.fragment.abort('.region')
```


---

Reacting to a fragment being aborted
-----------------------------------

Use `up.fragment.onAborted(fragment, callback)` to react when fragment or its ancestor is aborted on its layer.

A fragment is also aborted before it is removed from the DOM (through `up.destroy()` or a fragment update).


---

The following compiler will follow a link after 5 seconds:

```js
up.compiler('a[auto-follow]', function(link) {
  let timer = setTimeout(() => up.follow(link), 5000)
  return () => clearTimeout(timer)
})
```

We now have a race condition:

- Timer starts
- User follows different link
- While the user link is loading, the timer elapses and follows the original link
- The user request is aborted

Using `up.fragment.onAborted()` we can stop the timer if the link is targeted while waiting:

```js
up.compiler('a[auto-follow]', function(link) {
  let timer = setTimeout(() => up.follow(link), 5000)
  up.fragment.onAborted(link, () => clearTimeout(timer))
})
```


----

Examples for `up.fragment.onAborted()` from Unpoly's own code:

- Polling stops when the fragment is aborted
- Pending validations are aborted when the observed field is aborted


---

Exemptions
----------

To not abort the targeted fragments, use `{ abort: false }` or `[up-abort=false]`.

To make a request that will *not* be aborted by another fragment update, use `{ abortable: false }` or `[up-abortable=false]`.



Preloading
----------

Preloading never aborts targeted fragments.

Programmatic preloading with `up.link.preload()` is not abortable by default.\
You can use this to populate the cache while the user is navigating.



---
<!-- _class: no-watermark -->


Forms where everything depends on everything
======================================



```html
<form method="post" action="/purchases">
  <select name="continent" up-validate="[name=country]">...</select>
  <select name="country" up-validate="[name=price]">...</select>
  <input name="weight" up-validate="[name=price]"> kg
  <output name="price">23 €</output>
  <button>Buy stamps</button>
</form>
```

- User changes continent
- Request targeting `[name=country]` starts
- User changes weight
- Request targeting `[name=price]` starts
- User changes continent again
- Request targeting `[name=country]` starts
- Responses arrive and render in random order


---

Disabling form elements while loading
--------------------------------------

Forms with `[up-disable]` attribute disable all fields and buttons while submitting or validating.\
This prevents user input while the form is loading:


```html
<form up-submit up-disable>
  <input type="text" name="email"> <!-- will be disabled during submission -->
  <button>Submit</button>          <!-- will be disabled during submission -->
</form>
```

You can also only disable the submit button:

```html
<form up-submit up-disable="button">
```

Or any given CSS selector:

```html
<form up-submit up-disable="input[name=email]">
```


----

Concurrent `[up-validate]`: Consistency without disabling
--------------------------------------------------------

- Sometimes we don't want to disable because of user speed or optics (gray fields)
- In cases where we don't want to disable, there is a second solution for forms with many `[up-validate]` dependencies
- Multiple `[up-validate]` targets are batched into a single render pass with multiple targets
- Duplicate or nested targets are merged
- Only one concurrent request
- Form will eventually show a consistent state, regardless how fast the user clicks or how slow the network is


----


```html
<form method="post" action="/purchases">
  <select name="continent" up-validate="[name=country]">...</select>
  <select name="country" up-validate="[name=price]">...</select>
  <input name="weight" up-validate="[name=price]"> kg
  <output name="price">23 €</output>
  <button>Buy stamps</button>
</form>
```

<div class="row" style="font-size: 0.8em">
<div class="col">

**Unpoly 2**:

- User changes continent
- Request targeting `[name=country]` starts
- User changes weight
- Request targeting `[name=price]` starts
- User changes continent again
- Request targeting `[name=country]` starts
- Responses arrive and render in random order
</div>
<div class="col" style="flex-grow: 1.2">

**Unpoly 3**:

- User changes continent
- Request targeting `[name=country]` starts
- User changes weight
- User changes continent again
- Response for `[name=country]` received and rendered
- Request targeting `[name=price], [name=country]` starts
- Response for `[name=price], [name=country]` received and rendered
</div>
</div>


----


Watch rework
============

- `up.observe()` is now `up.watch()`
- `[up-observe]` is now `[up-watch]`
- Per-field `{ event, feedback, delay, disable }` options for watching and validation


---



```html
<form method="post" action="/purchases">
  <select name="continent" up-validate="[name=country]" up-watch-disable="[name=country]">...</select>
  <select name="country" up-validate="[name=price]">...</select>
  <input name="weight" up-validate="[name=price]" up-watch-event="input"> kg
  <output name="price">23 €</output>
  <button>Buy stamps</button>
</form>
```

This disables the country select while new countries are loading after a continent changes.

This updates the price while the user is typing in the weight field (on `input` instead of on `change`)



---

Handling disconnects
====================

You can now handle connection loss with `{ onOffline }` or `[up-on-offline]` callbacks:

```html
<a href="..." up-on-offline="if (confirm('Retry'?) event.retry()">Post bid</a>
```

Or globally:

```js
up.on('up:fragment:offline', (event) => if (confirm('Retry'?)) event.retry())
```


----

Handling "Lie-Fi"
-----------------

Often our device reports a connection, but we're effectively offline:

- Smartphone in EDGE cell
- Car drives into tunnel
- Overcrowded Wi-fi with massive packet loss

Unpoly 3 handles Wi-Fi with timeouts:

- All requests have a default timeout of 90 seconds (`up.network.config.timeout`)
- Customize timeouts per-request with `{ timeout }`, `[up-timeout]` options
- Timeouts will now trigger `onOffline()` (used to by `onAborted()`)


----

Expired pages remain accessible while offline
---------------------------------------------

- Cached content will remain navigatable for 90 minutes
- Revalidation will fail
- Clicking uncached content will not change the page and trigger `onOffline()`


---


Limitations: This isn't full offline support (yet)
--------------------------------------------------

- The cache is still in-memory and dies with the browser tab
- To fill up the cache the device must be online for the first part of the session
- For a full offline experience (content with empty cache) we recommend a [service worker](https://web.dev/offline-fallback-page/) or a canned solution like [UpUp](https://www.talater.com/upup/) (similarities in name are coincidental)




---
<!-- _class: topic -->

# Quality of life


----


Optional targets
================

The following would require fragments matching `.content` and `.navigation`.

The fragment `.flashes` will only be updated if there is a match in both the current page and server response (like `[up-hungry]`).

```html
<a href="/post" up-target=".content, .flashes:maybe, .navigation">
```

----


More control over `[up-hungry]`
===============================

Elements with an `[up-hungry]` attribute are updated whenever the server sends a matching element, even if the element isn't targeted.

Unpoly 3 lets you control which layer and which target to piggy-back on.


---

Controlling which layer to piggy-back on
----------------------------------------

By default Unpoly only considers `[up-hungry]` fragments in the updating layer.

With `[up-if-layer=any]` a hungry fragment will be considered for updates in *any* layer.

A use case is notification elements in the application layout:

```html
<div class="flashes" up-if-layer="any">...</div>
<div class="unread-messages" up-if-layer="any">...</div>
```


---

Controlling which targets to piggy-back on
------------------------------------------

By default Unpoly considers `[up-hungry]` fragments for any update in its layer.

With `[up-if-target]` you can restrict which updates a hungry fragment will piggy-back on.

A use case is a canonical `<link>` element that should only update when we update a [main element](https://unpoly.com/main),
but not when we update smaller fragments:

```html
<link rel="canonical" href="..." up-if-target=":main" />
```

----


Shorter data attributes
=======================


Unpoly always had [`[up-data]`](https://unpoly.com/up-data) to attach structured data to an element.

This is verbose when we're attaching simple key/value pairs:

```html
<div class="user" up-data="<%= { name: @user.name }.to_json %>">
```

It would feel more natural to use HTML5 data attributes instead:

```html
<div class="user" data-name="<%= @user.name %>">
```


---

In Unpoly 3 Simple data key/values can be attached with both `[up-data]` and HTML5 data attributes.

These three elements produce the same compiler data:

```html
<div up-data='{ "foo": "one", "bar": "two" }'></div>
<div data-foo='one' data-bar='two'></div>
<div up-data='{ "foo": "one" }' data-bar='bar'></div>
```

```js
up.compiler('div', function(element, data) {
  console.log(data.foo) // is always "one"
  console.log(data.bar) // is always "two"
})
```

Note that HTML5 data attributes are always strings.



---

Highlighting the targeted fragment
==================================

Targeted fragments get an `.up-loading` class.\
This lets you highlight the part of the screen that's loading.

Targeted fragments also get an [`[aria-busy]`](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Attributes/aria-busy) attribute.

---

A link targeting a fragment:

```html
<a href="/path" up-target=".target">
<div class="target">old text</div>
```

While the request is loading:

```html
<a href="/path" up-target=".target" up-active>
<div class="target" class="up-loading" aria-busy>old text</div>
```

Once the fragment is updated all classes and attributes are removed:

```html
<a href="/path" up-target=".target">
<div class="target">new text</div>
```

---

Log separated by interactions
=============================

The log has a lot of debug information. It's often hard to find where the relevant output begins and ends.

The new log shows which user interaction triggered an event chain.

---

<!-- _class: no-watermark no-padding -->

<img src="./images/log-interaction-event.png" alt="New log" class="picture" style="width: 80%; margin: auto;">

---

Playing nice with foreign overlays
=================================

Unpoly 2 sometimes clashes with overlays from other libraries ("foreign overlay") like Bootstrap or TinyMCE:

- Clicking a foreign overlay closes an Unpoly overlay
- Unpoly steals focus from a foreign overlay

This happens when foreign overlays look "on top" visually (`z-index: 99999999999`), but their elements attach to the `<body>`. For Unpoly this looks like content on the root layer. 

This could often be fixed by attaching the foreign overlay to the correct Unpoly layer (pseudo-code):

```js
OtherOverlay.open({ content: 'foo', onOpen(overlay) { up.layer.element.attach(overlay) }})
```

However, the solution is custom to every library.

---

Making Unpoly aware of foreign overlays
----------------------------------------

You can push a selector into `up.layer.config.foreignOverlaySelectors` and Unpoly will no longer have layer-related opinions over that region. You no longer need to re-attach the foreign overlay element.

Example from `unpoly-bootstrap5.js`:

```js
up.layer.config.foreignOverlaySelectors.push(
'.modal',
'.popover',
'.dropdown-menu'
)
```


---

More control over the progress bar
==================================

Unpoly 2 shows a progress bar when any request takes too long to load.

This may be unwanted for requests are loading in the background, or have longer load
times in the best of cases (e.g. a large report).

Unpoly 3 gives you more control over if and when the progress bar shows.


---


Background requests
--------------------

Pass { background: true } or [up-background] when rendering or making a request

Background requests never trigger the progress bar.\
Background requests are also deprioritized.

### Uses cases from Unpoly

- Polling requests are background requests automatically
- Preload requests are background requests automatically


---

Custom response times
---------------------

You can also set `{ badResponseTime }`, `[up-bad-response-time]`.

```html
<a href="/huge-report" up-bad-response-time="10_000">Open report</a>
```


----

Extensive render callbacks
==========================

You may now pass callback functions to intervene at many points
in the rendering lifecycle.

```js
up.render({
  url: '/path',
  onLoaded(event)        { /* Content was loaded from cache or server */ },
  focus(fragment, opts)  { /* Set focus */ },
  scroll(fragment, opts) { /* Set scroll positions */ },
  onRendered(result)     { /* Fragment was updated */ },
  onFailRendered(result) { /* Fragment was updated from failed response */ },
  onRevalidated(result)  { /* Stale content was re-rendered */ },
  onFinished(result)     { /* All finished, including animation and revalidation */ }
  onOffline(event)       { /* Disconnection or timeout */ },
  onError(error)         { /* Any error */ }
})
```

---

Strict target derivation
========================


Unpoly often needs to *derive* a target selector from an element.\
Some features that do this are `[up-poll]`, `up.reload()`, `[up-hungry]`.

```js
up.reload(element) // Produces a target selector from the given element
```

To build the selector, Unpoly 2 uses the following element properties in decreasing
order of priority:

- The element's `[up-id]` attribute
- The element's `[id]` attribute
- The element's `[name]` attribute
- The element's `[class]` names, ignoring `up.fragment.config.badTargetClasses`.
- The element's tag name

This sometimes produces a weak selector that won't uniquely identify the element.

```html
<link rel="stylesheet" href="...">
<link rel="canonical" href="..." up-hungry> <!-- targets "link", matching the stylesheet -->
```

---

Unpoly is smarter and stricter
------------------------------

- Tag names are only used for unique elements (`<body>`, `<main>`, etc.)
- Smarter default derivers (see next slide)
- Derived targets are verified to match the derivee
- Throw an error if no target can be derived or wouldn't match the derivee
- This may throw some errors in apps with bad selectors.\
  You should probably fix this, but also `up.fragment.config.verifyDerivedTarget = false`


---

Configurable target derivation patterns
---------------------------------------

Unpoly 3 lets you configure patterns to use for target derivation.

The following patterns are configured by default:

```js
up.fragment.config.targetDerivers = [
  '[up-id]',      // [up-id="foo"]
  '[id]',         // #foo
  'html',         // html
  'head',         // head
  'body',         // body
  'main',         // main
  '[up-main]',    // [up-main="root"]
  'link[rel]',    // link[rel="canonical"]
  '*[name]',      // input[name="email"]
  'form[action]', // form[action="/users"]
  'a[href]',      // a[href="/users/"]
  '[class]',      // .foo (filtered by up.fragment.config.badTargetClasses)
]
```

You can also push a function if your deriver can't be expressed in a pattern string.


---

Detect failure when server sends incorrect HTTP status
======================================================

Unpoly can render different content for [failed responses](https://unpoly.com/server-errors).
E.g. a form submission updates the main element when successful, but re-renders the form for validation errors.

For this to work Unpoly requires servers to signal failure with an HTTP error code. E.g. an invalid form should render with HTTP 400 (Bad Request).

Misconfigured server endpoints will send HTTP 200 (OK) for everything. This is not always easy to fix, e.g. when screens are rendered by libraries outside your control.

---

Listeners to `up:fragment:loaded` can now force a failure, even for responses that are `200 OK`.

```js
up.on('up:fragment:loaded', (event) => {
  if (event.response.headers['X-Authentication-Error']) {
    // Force Unpoly to use render options for failure
    event.renderOptions.fail = true
  }
})
```

You may also globally customize what Unpoly considers an error:

```js
up.network.config.fail = (response) => 
  (response.status < 200 || response.status > 299) && response.status !== 304
```


---

Focus restoration
=================

Focus is saved automatically and restored when using the back and forward button.

Saved state includes:

- Which element is focused.
- The cursor position within a focused input element.
- The selection range within a focused input element.
- The scroll position within a focused input element.

---

Explicit focus restoration
------------------------

- `<a href="/path" up-follow up-focus="restore">`
- up.render({ focus: 'restore' })
- up.viewport.restoreFocus()

Note that Unpoly already had `{ focus: 'keep' }` to preserve focus within
an updating fragment.

---

IE11 removal
============

Unpoly 3 will no longer boot on IE11 or [legacy Edge (EdgeHTML)](https://en.wikipedia.org/wiki/EdgeHTML).\
If you need to support Internet Explorer 11, use Unpoly 2.

This allowed us to delete a lot of internal code.


---

Functions with a native replacement have been moved to `unpoly-migrate.js`:

| Deprecated function | Native replacement |
|---------------------|--------------------|
| `up.util.assign()` | `Object.assign()` |
| `up.util.values()` | `Object.values()` |
| `up.element.remove()` | `Element#remove()` |
| `up.element.matches()` | `Element#matches()` |
| `up.element.closest()` | `Element#closest()` |
| `up.element.replace()` | `Element#replaceWith()` |
| `up.element.all()` | `document.querySelectorAll()` |
| `up.element.toggleClass()` | `Element#classList.toggle()` |


---


Many other features and fixes
=============================

See [full CHANGELOG](https://github.com/unpoly/unpoly/blob/master/CHANGELOG.md).



---

Breaking changes
================

Probably none with [`unpoly-migrate.js`](https://unpoly.com/changes/upgrading).

You should test if you're seeing any errors with:

- Strict target derivation
- Cache revalidation

----

### Strict target derivation

Disable with:

```js
up.fragment.config.verifyDerivedTarget = false
up.fragment.config.targetDerivers.push('*')
```

---

### Cache Revalidation

Disable with:

```js
up.fragment.config.autoRevalidate = false
up.network.config.cacheEvictAge = up.network.config.cacheExpireAge = 15 * 60 * 1000
```


----

Unpoly 3 objectives
===================

Repeat card from intro


---

Availability
============

Unpoly 3 is scheduled to release in late summer 2022.


---

<!-- _class: no-watermark -->

<div class="title">
  <img src="./images/unpoly3.svg" alt="Unpoly 3" class="title--logo">
  <div class="title--author">
    Henning Koch&nbsp; <a href="https://twitter.com/triskweline">@triskweline</a>
  </div>  
</div>
