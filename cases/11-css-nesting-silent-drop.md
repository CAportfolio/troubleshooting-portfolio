# 11. Desktop menu expansion invisible at mid-range viewport widths

## Context

A B2B directory app built on a low-code platform. The app uses the platform's standard bottom-bar navigation component, which is designed for mobile. A previous implementation decision — not mine — had repositioned it to the top of the screen on desktop using CSS transforms. The menu contains seven visible items plus overflow items (logout, admin) that are meant to appear when the user taps a "More" button. The platform's JS dynamically adds the More button when it calculates that not all items fit in the available width.

The CSS was written using SCSS-style nested rules inside media query blocks, which the platform's global CSS editor accepts without complaint.

---

## Symptom as reported

The client reported that on their laptop the menu appeared correctly at the top of the screen, but the "More" button was not visible, meaning they could not access the logout or admin links. The same menu worked correctly on mobile and on larger desktop screens.

---

## Investigation

The client's laptop was reporting a viewport width of approximately 710px. My first step was to establish whether the More button existed in the DOM at all, or whether the platform's JS was simply not rendering it.

I asked the client to run three console checks with dev tools open, first confirming they had switched from the "top" frame context to the correct app iframe:

```javascript
console.log($(window).width()); // 710
console.log($('.fl-bottom-bar-menu-holder ul li[data-show-more]').length); // 1
console.log($('.fl-bottom-bar-menu-holder ul li[data-show-more]').is(':visible')); // false
```

The More button existed in the DOM but was not visible. That ruled out the platform's JS failing to render it — something was hiding it after the fact.

My next check was whether the element itself had a CSS property making it invisible:

```javascript
console.log($('.fl-bottom-bar-menu-holder ul li[data-show-more]').css('display')); // flex
console.log($('.fl-bottom-bar-menu-holder ul li[data-show-more]').css('visibility')); // visible
```

Both values were correct. jQuery's `:visible` returning false while `display: flex` and `visibility: visible` are both set typically means a parent element is clipping the child. I checked the menu holder:

```javascript
console.log($('.fl-bottom-bar-menu-holder').css('height')); // 60px
console.log($('.fl-bottom-bar-menu-holder').css('overflow')); // hidden
```

The holder was 60px tall with overflow hidden. That was the clipping mechanism. But I had explicitly set `height: 60px` and `overflow: hidden` in the CSS as part of the desktop layout — that was intentional. The design was that the collapsed state clips overflow items, and `height: 200px; overflow: visible` on the `.expanded` class reveals them when the More button is clicked. The More button itself should have been visible within the 60px row.

This pointed to the More button being pushed outside the 60px row rather than being clipped by it — meaning the main menu items were overflowing onto a second row and pushing the More button below the visible area. I checked flex-wrap:

```javascript
console.log($('.fl-bottom-bar-menu-holder ul').css('flex-wrap')); // wrap
```

Wrap was active. The items were wrapping — but the More button was landing on a wrapped second row, below the 60px visible height. At 710px the seven main items were fitting the row and the More button was being pushed off the bottom.

I had already added a `@media (min-width: 640px) and (max-width: 1280px)` block to reduce item sizes at this viewport range specifically to force the More button into the visible row. That block was working on my test environment but not on the client's device. I escalated to engineering with the console output and a summary of what I had ruled out.

Their analysis identified the root cause: the CSS was using SCSS-style nested rules inside the media query blocks — for example:

```css
.fl-bottom-bar-menu-holder {
  transform: none !important;
  ul {
    transform: none !important; /* ← nested */
    flex-wrap: wrap !important;
  }
}
```

Native CSS nesting is only supported in Chrome and Edge 112 and later (March 2023). On older browsers the inner `ul { transform: none !important }` rule is silently dropped. The platform's live CSS delivery pipeline was also behaving differently from Studio preview, which explained why the fix had appeared to work in my test but not on the client's browser. With the nested rule dropped, the platform's default `ul { transform: translate3d(0, 100%, 0) }` remained active, keeping all overflow items — including the More button — translated off screen below the holder even when the items were otherwise sized correctly.

Engineering confirmed the fix: flatten all nested rules to fully qualified selectors. I replaced the nested blocks across all three media query breakpoints:

```css
/* Before — nested, silently broken on older browsers */
.fl-bottom-bar-menu-holder {
  transform: none !important;
  ul {
    transform: none !important;
    flex-wrap: wrap !important;
  }
}

/* After — flat, works everywhere */
.fl-bottom-bar-menu-holder { transform: none !important; }
.fl-bottom-bar-menu-holder ul { transform: none !important; flex-wrap: wrap !important; }
```

---

## Root cause

SCSS-style nested CSS rules were used in a plain CSS editor. Native CSS nesting requires Chrome/Edge 112+. On the client's browser the nested `ul` rule was silently dropped, leaving the platform's default vertical translate active on the overflow item list. The More button existed in the DOM but was translated off screen. The symptom only appeared at mid-range viewport widths because at those widths the More button was already near the edge of the visible row — at larger widths there was enough room for it regardless.

---

## Resolution and prevention

Replaced all nested CSS blocks with fully qualified flat selectors across the three breakpoints. The client confirmed the More button was visible and functional after the fix was published.

The platform's CSS editor does not warn when SCSS syntax is used in a plain CSS context. The safe rule going forward is to avoid nested selectors entirely in this editor and write fully qualified selectors from the start. I've noted this in the project documentation.

This is also a case where Studio preview diverged from live behaviour — preview was running a modern browser that supported native nesting; the client's browser was not. Any CSS change that appears to work in Studio but fails on the client's device is worth checking for browser compatibility issues first.

---

## Skills demonstrated

CSS specificity and cascade debugging; cross-browser compatibility; DevTools computed styles and frame context switching; console-based remote triage; reading platform support analysis; SCSS vs plain CSS distinction.
