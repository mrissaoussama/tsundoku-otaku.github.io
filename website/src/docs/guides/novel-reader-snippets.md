---
title: Novel reader (Advanced)
titleTemplate: Guides
description: Custom CSS/JS snippets, the Tsundoku JS objects, theme variables, and the HTML structure of the WebView novel renderer.
---

# Novel reader snippets

Custom CSS/JS snippets, theme variables, and the `window.Tsundoku` objects run only in the
**WebView** renderer. The Native (TextView) renderer ignores all of them, a limit of Android's
`TextView`, not a setting.

## Enable WebView

1. Open a novel chapter.
2. In reader settings set **Rendering mode** to **WebView**.
3. Open the **Advanced** tab for the controls below.

## Advanced tab controls

- **Enable embedded CSS** (on): keep `<style>` tags and inline `style` attributes shipped inside
  the chapter HTML/EPUB. Off strips them.
- **Enable embedded JS** (off): keep `<script>` tags shipped inside the chapter. Off strips them.
- **Source CSS priority** (off): off makes the reader's base CSS use `!important` so it wins over
  source styles. On drops `!important` so source styles win.
- **CSS Snippets** / **JavaScript Snippets**: named blocks you add, each independently
  enabled/disabled. Enabled ones run on every chapter. Each JavaScript snippet also has a
  **Re-run on infinite-scroll append** toggle, see
  [When your JS runs again](#when-your-js-runs-again).
- **Regex Find & Replace**: see [Regex rules](#regex-rules).

## Content pipeline

Chapter text passes through this order before rendering:

1. Strip chapter title (if enabled)
2. Normalize: plain-text for `.txt`/`.text` URLs, otherwise auto-detect HTML / Markdown and
   convert Markdown to HTML
3. Regex find/replace rules
4. Force lowercase (if enabled)
5. Translate (if enabled)
6. Sanitize for the renderer

Sanitize always removes `<noscript>` and HTML comments. It strips `<script>`/`<style>`/inline
styles unless the matching **Enable embedded** toggle is on, and strips media when **Block images
and videos** is on. Your snippets are injected after sanitize, so they always run.

## Injection order

The page is built with the theme variables, base `<style>`, the `window.Tsundoku` and
`window.TsundokuTheme` objects, and your enabled CSS already in `<head>`. After `onPageFinished`
the app re-injects the `Tsundoku` object, then your one-off **JavaScript** box, then your enabled
JS snippets in list order.

### When your JS runs again

Some transitions build a brand-new document, some only mutate the current one. That difference is
the whole reason snippets have to be idempotent.

| Transition | Document | What runs |
| --- | --- | --- |
| First chapter load | new | one-off box, then all enabled snippets |
| Chapter change **without** infinite scroll, or a jump from the chapter list | new | one-off box, then all enabled snippets |
| Infinite-scroll append (next chapter) or prepend (previous chapter) | same, nodes inserted | only snippets with **Re-run on infinite-scroll append** |
| Reader setting change | same | one-off box, then all enabled snippets |
| Edit mode on/off | same | one-off box, then all enabled snippets |

A new document wipes everything you added, so a fresh run starts clean. In the "same" rows nothing
is cleaned up for you: earlier chapters are still in the DOM, and so are the nodes, listeners and
observers your previous run created.

On an append, your snippet runs *after* the new `<tsundoku-chapter>` and its
`.tsundoku-chapter-divider` are inserted, so it can query them immediately. Appended chapters are
never trimmed, so `document.querySelectorAll('tsundoku-chapter')` returns every chapter loaded so
far, including the ones you already processed. Skip the ones you have handled, see
[Mark what you processed](#mark-what-you-processed).

Turn the toggle on for code that must decorate every chapter (buttons, per-chapter styling, word
counts). Leave it off for one-shot setup that would stack on each appended chapter.

For CSS, **later snippets win**. Use `!important` to beat a base rule when **Source CSS priority**
is off.

## `window.Tsundoku`

```js
window.Tsundoku.novelUrl          // string, normalized novel URL
window.Tsundoku.currentChapter    // { id, title, number, path, url }
window.Tsundoku.chapters          // [ { id, title, number, path, url }, ... ] in reading order

window.Tsundoku.runtime.isEditMode             // boolean
window.Tsundoku.runtime.isInfScroll            // boolean
window.Tsundoku.runtime.textSelectionBlocked   // boolean
window.Tsundoku.runtime.forcedLowercase        // boolean

window.Tsundoku.runtime.menuVisible            // boolean, reader menu (top/bottom bars) shown
window.Tsundoku.runtime.immersive              // boolean, always the inverse of menuVisible
window.Tsundoku.runtime.ttsState               // "stopped" | "playing" | "paused"
window.Tsundoku.runtime.loadingChapter         // boolean, a chapter body is still loading
```

Each chapter object has `id` (number, `-1` if unknown), `title` (string), `number` (number, `-1`
if unknown), `path` (relative URL) and `url` (absolute URL).

`runtime.*` is re-pushed on chapter change and on settings change. `menuVisible` / `immersive`,
`ttsState` and `loadingChapter` are also updated live between those pushes and each fire a
[window event](#events) you can subscribe to instead of polling.

The reader also adds these internal fields. They are read-only, do not set them yourself:

```js
// Added by the scroll tracker (present once a chapter has loaded)
window.Tsundoku.runtime.progress                 // number 0..1, whole-document progress
window.Tsundoku.runtime.chapterProgress          // number 0..1, current-chapter progress
window.Tsundoku.runtime.currentChapterId         // string id of the chapter in view, null outside infinite scroll
window.Tsundoku.runtime.infiniteScrollInstalled  // boolean, scroll listener installed
window.Tsundoku.runtime.loadingNext              // boolean, a next chapter is loading
window.Tsundoku.runtime.setLoadingNext(v)        // function, used by the app
window.Tsundoku.runtime.noMoreChapters           // boolean, reached the last chapter
window.Tsundoku.runtime.setNoMoreChapters(v)     // function, used by the app
window.Tsundoku.runtime.knownDividerCount        // number, cached divider count
window.Tsundoku.runtime.lastChapterIdxSeen       // number, index of the last reported chapter
window.Tsundoku.runtime.resetChapterTracking()   // function, forces the next frame to re-report

// Added in edit mode
window.Tsundoku.runtime.editInputBound           // boolean, edit listener installed
window.Tsundoku.runtime.inputListener            // function ref, the edit listener

// Infinite-scroll boundary helpers (on window, not runtime)
window.chapterBoundaries            // [ { chapterId, startOffset, height }, ... ]
window.addChapterBoundary(id, startOffset, height)
window.updateChapterBoundaries()    // recompute from the divider DOM
```

Need another field exposed? Open an issue or PR with the use case.

## Events

The reader dispatches these `CustomEvent`s on `window` so snippets can react instead of polling.
Subscribe with `addEventListener` and read `event.detail` for the payload.

```js
// Reader menu (top app bar + bottom bar) shown/hidden.
// detail: { menuVisible, immersive }  — or { visible } when fired by the reader-chrome path.
window.addEventListener('tsundoku:menuvisibilitychange', function (e) {
  var d = e.detail || {};
  var visible = d.visible !== undefined ? d.visible : d.menuVisible;
  // ...
});

// User tapped next / previous chapter.
// detail: { direction: 'next' | 'prev' }
window.addEventListener('tsundoku:chapternavigate', function (e) { /* e.detail.direction */ });

// A chapter body started/finished loading. Mirrors runtime.loadingChapter.
// detail: { loading: boolean }
window.addEventListener('tsundoku:chapterloading', function (e) { /* e.detail.loading */ });

// TTS state changed. Mirrors runtime.ttsState.
// detail: { state: 'stopped' | 'playing' | 'paused' }
window.addEventListener('tsundoku:ttsstatechange', function (e) { /* e.detail.state */ });

// Reading progress advanced, throttled. Mirrors runtime.progress / runtime.chapterProgress.
// detail: { progress, chapterProgress, chapterId, isLast }
window.addEventListener('tsundoku:progresschange', function (e) {
  var pct = Math.round(e.detail.chapterProgress * 100); // this chapter, 0..100
});
```

A menu toggle can fire `tsundoku:menuvisibilitychange` twice, once per internal path, with different
detail keys. Read `detail.visible`, fall back to `detail.menuVisible`, and keep the handler
idempotent.

### progress vs chapterProgress

Both are `0..1` and both are measured from the scroll position, not from time spent.

- **`chapterProgress`** counts within the chapter you are looking at, from its divider to the next
  one. It drives the reader's slider and the read/unread mark, so it is what you want for a
  per-chapter indicator.
- **`progress`** counts across the entire loaded document, which in infinite scroll is every chapter
  appended so far. Use it for a page-level scrollbar.

Say three chapters are loaded and you are halfway down the second: `chapterProgress` is about `0.5`
while `progress` is about `0.5` of all three chapters, so roughly `0.33`. Reaching the end of chapter
two sends `chapterProgress` to `1`, then it drops back near `0` as chapter three scrolls into view,
while `progress` keeps climbing. Without infinite scroll there is one chapter per document and the
two values are identical.

`chapterId` in the detail is the chapter in view, and it is `null` unless infinite scroll has more
than one chapter loaded. `isLast` says whether that chapter is the last one currently loaded. Prefer
these values over deriving your own percentage from `scrollTop`, which is wrong under infinite scroll
and in wide-viewport modes.

## `window.TsundokuTheme`

The active reader colors as a JS object. Same values as the CSS variables below, for when you need
them in JS. Every key:

```js
window.TsundokuTheme.mdSysColorPrimary
window.TsundokuTheme.mdSysColorOnPrimary
window.TsundokuTheme.mdSysColorPrimaryContainer
window.TsundokuTheme.mdSysColorOnPrimaryContainer
window.TsundokuTheme.mdSysColorSecondary
window.TsundokuTheme.mdSysColorOnSecondary
window.TsundokuTheme.mdSysColorSecondaryContainer
window.TsundokuTheme.mdSysColorOnSecondaryContainer
window.TsundokuTheme.mdSysColorTertiary
window.TsundokuTheme.mdSysColorOnTertiary
window.TsundokuTheme.mdSysColorTertiaryContainer
window.TsundokuTheme.mdSysColorOnTertiaryContainer
window.TsundokuTheme.mdSysColorError
window.TsundokuTheme.mdSysColorOnError
window.TsundokuTheme.mdSysColorErrorContainer
window.TsundokuTheme.mdSysColorOnErrorContainer
window.TsundokuTheme.mdSysColorBackground
window.TsundokuTheme.mdSysColorOnBackground
window.TsundokuTheme.mdSysColorSurface
window.TsundokuTheme.mdSysColorOnSurface
window.TsundokuTheme.mdSysColorSurfaceVariant
window.TsundokuTheme.mdSysColorOnSurfaceVariant
window.TsundokuTheme.mdSysColorOutline
window.TsundokuTheme.tsundokuReaderBackground
window.TsundokuTheme.tsundokuReaderText
```

Each value is a hex string like `"#006A6A"`.

## CSS theme variables

Defined on `:root`, usable in any CSS/JS snippet:

```css
/* Reader colors (track your selected reader theme) */
--tsundoku-reader-background
--tsundoku-reader-text

/* Material 3 palette */
--md-sys-color-primary        --md-sys-color-on-primary
--md-sys-color-secondary      --md-sys-color-on-secondary
--md-sys-color-tertiary       --md-sys-color-on-tertiary
--md-sys-color-error          --md-sys-color-on-error
--md-sys-color-background     --md-sys-color-on-background
--md-sys-color-surface        --md-sys-color-on-surface
--md-sys-color-surface-variant --md-sys-color-on-surface-variant
--md-sys-color-outline
/* plus the *-container / on-*-container variants */
```

### Safe-area insets

Two `:root` variables carry the height (in px) of the reader menu bars so fixed-position elements
can stay clear of them. They are `0` while the menu is hidden and updated live as it toggles:

```css
--tsundoku-safe-top      /* top app bar height (+ system bar) */
--tsundoku-safe-bottom   /* bottom bar height (+ system bar) */
```

Use them with a fallback so a snippet still works before the first update:

```css
/* pin above the bottom bar */
bottom: calc(8px + var(--tsundoku-safe-bottom, 0px));
```

The vars cover only the transient menu bars. The **novel status bar** takes real layout space, so
nothing ever renders under it and no snippet needs to account for it. Pair the vars with
[`tsundoku:menuvisibilitychange`](#events) to slide your own UI in and out with the menu.

Example, styling blockquotes inside the chapter to follow the reader theme:

```css
tsundoku-chapter blockquote {
  color: var(--md-sys-color-on-surface-variant);
  border-left: 3px solid var(--md-sys-color-primary);
}
```

## HTML structure and selectors

Chapter content is wrapped in a custom element:

```html
<tsundoku-chapter
  data-tsundoku-chapter="1"
  data-chapter-id="123"
  data-chapter-title="Chapter 1"
  data-chapter-number="1"
  data-chapter-path="/novel/ch-1"
  data-chapter-url="https://.../novel/ch-1">
  ...chapter content...
</tsundoku-chapter>
```

Other markers a snippet can target:

| Selector | Meaning |
| --- | --- |
| `tsundoku-chapter` | The chapter content wrapper (target this, not `body`). |
| `#tsundoku-chapters-container` | Wraps all loaded chapters in infinite-scroll mode. |
| `.tsundoku-chapter-divider` | Boundary between chapters in infinite scroll (carries the same `data-chapter-*` attributes). |
| `.tsundoku-plain-text` / `[data-tsundoku-plain-text="1"]` | Chapter detected as plain text. |
| `[data-tsundoku-markdown="1"]` | Content rendered from Markdown. |
| `[data-tsundoku-editable="1"]`, `[contenteditable="true"]` | Edit mode is active. |
| `.td-tts-highlight-bg`, `.td-tts-highlight-underline`, `.td-tts-highlight-outline` | Applied by TTS to the spoken segment. Style these to restyle the read-aloud highlight. |

Reserved ids the app owns. Do not reuse them: `tsundoku-custom-style` (your injected CSS),
`edit-mode-style`, `next-chapter-btn-container`.

## Regex rules

Each rule has a title, a find pattern, a replacement, and toggles: **Use Regex** (literal text when
off), case sensitive, and match whole word. Rules run at pipeline step 3, on the chapter HTML
before rendering, so patterns may need to account for tags.

## Safe snippet practices

- Scope to `tsundoku-chapter` or your own ids/classes, avoid bare `*`.
- Do not replace `document.body` or a whole chapter's `innerHTML`. Wrap or insert nodes instead.
- Guard against missing nodes, a snippet can run before a chapter is in the DOM:

```js
const target = document.querySelector('tsundoku-chapter');
if (target) {
  // your logic
}
```

### Mark what you processed

Your snippet sees every chapter still in the document, not only the new one, on every re-run listed
in [When your JS runs again](#when-your-js-runs-again). Flag each chapter you touch and skip flagged
ones. This is the whole pattern:

```js
document.querySelectorAll('tsundoku-chapter').forEach(function (chapter) {
  if (chapter.dataset.myWordCount) return;   // already done, skip
  chapter.dataset.myWordCount = '1';         // claim it first

  const div = document.createElement('div');
  div.textContent = chapter.innerText.trim().split(/\s+/).length + ' words';
  chapter.prepend(div);
});
```

Enable **Re-run on infinite-scroll append** and that snippet decorates each appended chapter exactly
once, no observer involved.

### Do you still need a MutationObserver?

Not for chapters. **Re-run on infinite-scroll append** covers appends and prepends, and it runs after
the nodes are inserted, which an observer only learns about later.

An observer is still useful for DOM changes the app makes with no snippet re-run behind them, like
images finishing load or the TTS highlight classes moving between paragraphs. If you install one:

- **Install it once.** The reader re-runs snippets inside the *same* document on setting changes, edit
  mode toggles and appends, so `new MutationObserver(...)` at the top level stacks up one observer
  per run, each doing the same work. Park it on `window` and reuse or disconnect it:

```js
if (window.__myObserver) window.__myObserver.disconnect();
window.__myObserver = new MutationObserver(onMutation);
window.__myObserver.observe(document.body, { childList: true, subtree: true });
```

- **Do not rescan the document per mutation batch.** A chapter insert fires many mutations. Either
  look only at `mutation.addedNodes`, or coalesce into one pass with `requestAnimationFrame`.
- **Expect your own writes back.** If the callback mutates the subtree it observes, it will fire
  again. The processed flag above is what stops the loop.

### Cautions

- **Edit mode saves your markup.** Saving an edited chapter stores the live `innerHTML`, so anything
  a snippet injected is written into the chapter permanently. Bail out early on
  `Tsundoku.runtime.isEditMode`, and listen for `tsundoku:chapternavigate` or re-check the flag if you
  cache state.
- **TTS reads `p, li, blockquote, h1`-`h6`, `pre`.** Injecting one of those tags into the chapter adds
  a spoken paragraph and shifts TTS paragraph indices. Use `div` or `span` for your own UI.
- **Word-level rewrites are per-chapter work.** Splitting every text node (bionic reading, ruby text,
  furigana) on a long chapter costs a visible pause, so do it once per chapter behind the processed
  flag and never on every mutation.
- **Never write the `runtime.*` fields or the reserved ids.** They are pushed by the app and your
  values are overwritten on the next update.

## Troubleshooting

- **Snippet ignored:** confirm renderer is **WebView** and the snippet is enabled. Native mode has
  a **Show raw HTML** option to inspect the source markup. This option only shows the source's contents, not HTML/CSS/JS injected by the app or your snippets.
- **Style won't apply:** move the rule to a later snippet, or add `!important` (base styles carry
  `!important` unless **Source CSS priority** is on).
