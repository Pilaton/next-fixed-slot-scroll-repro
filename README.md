# Repro: scroll is not reset on navigation when a parallel slot renders only a `position: fixed` element

Next.js **16.3.0** regression. Works correctly on **16.2.11**.

## Run

```bash
pnpm install
pnpm dev
```

## Steps

1. Open `/`.
2. Scroll to the bottom.
3. Click **go to /about**.

**Expected:** `/about` opens at the top (the behaviour on 16.2.11 and earlier).

**Actual:** `/about` opens at the scroll offset carried over from `/`.

Navigating back to `/` shows the same thing. Both directions are affected.

## Toggle the cause

```ts
// next.config.ts
export default {
  experimental: {
    appNewScrollHandler: false, // <- scroll reset works again
  },
};
```

The repo ships with the flag commented out, i.e. the 16.3.0 default (`true`).

## What matters in this repo

The `@header` parallel slot renders exactly one element, and that element is
`position: fixed`:

- `app/@header/page.jsx` matches `/`
- `app/@header/[...rest]/page.jsx` matches everything else
- `components/header.jsx` is the only thing either of them renders

Because the two slot segments differ, every navigation is a segment change for
the slot, so it participates in the navigation and receives a scroll intent.

## Why it happens

`experimental.appNewScrollHandler` flipped its default in 16.3.0:

| version | `dist/server/config-shared.js` |
| --- | --- |
| 16.2.11 | `appNewScrollHandler: false` |
| 16.3.0 | `appNewScrollHandler: true` |

`InnerScrollAndFocusHandlerOld` skipped `fixed`/`sticky` elements when choosing
the scroll target, and said why:

```js
function shouldSkipElement(element) {
  // we ignore fixed or sticky positioned elements since they'll likely pass the "in-viewport" check
  // and will result in a situation we bail on scroll because of something like a fixed nav,
  // even though the actual page content is offscreen
  if (['sticky', 'fixed'].includes(getComputedStyle(element).position)) {
    return true;
  }
  ...
}
```

`InnerScrollHandlerNew` — the Fragment-ref fork that is now the default — has no
equivalent check. It hands the Fragment ref to `getScrollTargetState`, which
reads the top edge of whatever host children the slot rendered:

```js
return elementTop >= getScrollPaddingTop() && elementTop <= viewportHeight ? 1 : 2;
```

For a slot holding only a fixed header, `elementTop` is a small positive number
on every navigation (here: `8`), so the result is always `1` — "already in
viewport". The handler then claims the scroll intent and returns without
scrolling:

```js
didHandleScroll = true;
// Mark as scrolled so no other segment scrolls for this navigation.
scrollRef.current = false;
...
if (initialTargetState === 1) {
  return;
}
```

`accumulateScrollRef` assigns the **same** `scrollRef` object to every changed
cache node, so once the header slot has claimed it, the `children` slot's layout
effect sees `scrollRef.current === false` and does nothing. The actual page
content is never scrolled into view.

## Isolating it

Add this and the scroll reset starts working again, with no layout change at all:

```css
html { scroll-padding-top: 200px; }
```

It only raises the threshold inside `getScrollTargetState` — `8 >= 200` is false,
so the target is no longer considered visible. That it alone flips the outcome
pins the behaviour to that one comparison.
