---
title: Novel reader snippets
titleTemplate: Guides
description: How custom CSS/JS snippets work in the novel reader.
---

# Novel reader snippets

This guide explains how custom CSS and JavaScript snippets behave in the **Novel WebView reader**:

- where snippets are applied,
- which rules win when multiple styles/scripts exist,

## Before you start

Snippets are intended for the **WebView novel renderer**.

1. Open a novel chapter.
2. Open reader settings.
3. Set rendering mode to **WebView**.
4. Go to **Advanced** and use the CSS/JS snippet sections.

If you use the TextView renderer, WebView snippets do not run.

## Execution order

For a normal chapter load, the order is:

1. **Chapter preprocessing (native Kotlin):**
   - optional hide chapter title
   - optional force lowercase
   - optional translation
2. **Content sanitization:**
   - strips chapter-inline `<script>`, `<style>`, and `<noscript>` blocks
   - optional media stripping
   - applies enabled **Regex Find/Replace** rules
3. **EPUB extraction (if enabled):**
   - if `Enable EPUB CSS` is on, `<style data-epub-css>` is moved into the document `<head>`
   - if `Enable EPUB JS` is on, `<script data-epub-js>` is moved into the document `<head>`
4. **Initial HTML is loaded into WebView** (with base style + extracted EPUB head content + `window.__TSUNDOKU_*` variables).
5. **After page finished (`onPageFinished`) the app injects in this order:**
   - Reader settings CSS/JS
   - EPUB JS/CSS
   - Custom CSS
   - Custom JS

User CSS/JS runs **after** base content load, and after EPUB head content is attached.

## CSS priority and merge order

The injected style text is assembled in this order:

1. generated base style (font size, line height, margins, colors, text selection, paragraph spacing, etc.)
3. enabled CSS snippets (in list order)

Because CSS is appended in that order, **later rules override earlier ones** when specificity is equal.

## JavaScript behavior and order

The runtime JS layers are:

1. During initial HTML build, the app writes `window.__TSUNDOKU_*` variables:
   - `__TSUNDOKU_CHAPTER_TITLE`
   - `__TSUNDOKU_CHAPTER_NUMBER`
   - `__TSUNDOKU_CHAPTER_URL`
   - `__TSUNDOKU_NOVEL_URL`
   - `__TSUNDOKU_IS_EDIT_MODE`
   - `__TSUNDOKU_IS_INF_SCROLL`
   - `__TSUNDOKU_TEXT_SELECTION_BLOCKED`
   - `__TSUNDOKU_FORCED_LOWERCASE`
2. After page finishes loading, object `window.Tsundoku` is created:
   - `chapterTitle`, `chapterNumber`, `chapterUrl`, `novelUrl`
   - `isEditMode`, `isInfScroll`, `textSelectionBlocked`, `forcedLowercase`
 

If you need more variables exposed to JS, open an issue with your use case or submit a PR.

## EPUB CSS/JS options

The reader has EPUB toggles in Advanced:

- **Enable EPUB CSS**
- **Enable EPUB JS**

These toggle whether EPUB-embedded blocks marked as `data-epub-css` and `data-epub-js` are kept and injected into `<head>`.

## Js run order:
For JS and CSS, Reader settings load first, then EPUB, then user code

## Safe snippet practices

- Keep snippets small and focused.
- Prefer id/class selectors scoped to chapter content.
- Avoid heavy loops/mutation observers without guards.
- Guard against missing nodes:

```js
const target = document.querySelector('.chapter-content');
if (target) {
  // your logic
}
```
- If your code has loops, guard against infinite loops.
- For CSS, prefer scoped selectors over `*` when possible.

## Troubleshooting

### Snippet seems ignored

- Confirm renderer is **WebView**.
- Ensure snippet is **Enabled**.
- Check for selector mismatch on current source HTML (Textview mode has a raw html option).
- Temporarily disable other snippets to detect override conflicts.

### Style conflict

- Move conflicting rule to a later snippet or combine into one snippet.
- Some elements like body might have !important applies on some of the styles. Apply 
!important on your style if it doesn't work.