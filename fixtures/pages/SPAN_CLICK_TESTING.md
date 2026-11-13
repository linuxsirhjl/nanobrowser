# Testing Span Click Listener Detection

## Issue Description

A `<span>` element with a JavaScript click listener may not appear in Chrome's accessibility tree (it gets flattened to text), but Chrome's `DOMSnapshot.captureSnapshot` marks it as `isClickable: true`. NanoBrowser's tier 2 (layout enrichment) should detect these elements and make them actionable.

## Test Cases

### 1. Basic span with click listener
```html
<span class="mt10">转文字</span>
<script>
  document.querySelector('.mt10').addEventListener('click', () => {
    console.log('Clicked!');
  });
</script>
```

**Expected behavior:**
- Chrome marks the span as `isClickable: true` in the DOM snapshot
- NanoBrowser tier 2 enrichment detects it via `layoutSignal()` returning `'click listener'`
- The span gets a ref: `[e1] clickable "转文字"`

### 2. Inline span without semantic role
```html
<div>
  Some text <span id="clickable">click here</span> more text
</div>
<script>
  document.getElementById('clickable').addEventListener('click', handler);
</script>
```

**Expected behavior:**
- Span is ignored/flattened in the AX tree
- Span appears in layout snapshot with `clickable: true`
- Gets ref from tier 2: `[e1] clickable "click here"`

### 3. Nested clickable elements
```html
<div id="outer">
  Outer text
  <span id="inner">Inner clickable</span>
</div>
<script>
  document.getElementById('outer').addEventListener('click', handler1);
  document.getElementById('inner').addEventListener('click', handler2);
</script>
```

**Expected behavior:**
- Outer div gets ref: `[e1] clickable "Outer text Inner clickable"`
- Inner span might be skipped per line 161 of enrich.ts: `if (insideActionable(node) && !isNativeControl(node) && !existing?.name) continue;`
- This is correct: clicking the outer element is sufficient

### 4. Non-clickable span (no listener)
```html
<span>普通文本</span>
```

**Expected behavior:**
- Not marked as clickable
- Appears as text only: `text "普通文本"`
- No ref assigned

## How to Test

1. Open the test panel (bind `open-test-panel` command in chrome://extensions/shortcuts)
2. Navigate to http://localhost:8787/span-click-listener.html (requires `pnpm fixtures`)
3. Click "Observe" and check the outline
4. Verify that spans with click listeners have refs and role "clickable"

## Debugging

If a span with a click listener is not detected:

1. **Check if Chrome marks it clickable:**
   - Look at the layout snapshot in the capture JSON
   - Find the element by backendNodeId
   - Check if its index appears in `nodes.isClickable.index`

2. **Check if it's rendered:**
   - `isRendered()` requires bounds with width > 0 and height > 0
   - Check `display !== 'none'`, `visibility !== 'hidden'`, `opacity !== '0'`

3. **Check if it's inside another actionable element:**
   - Line 161 of enrich.ts skips non-native-controls inside actionable ancestors
   - This is usually correct (parent is enough to click)
   - Exception: if the inner element has `existing?.name`, it keeps its ref

## Current Implementation Status

The system **already supports** span elements with click listeners via:
- Chrome's `DOMSnapshot.captureSnapshot` API (`isClickable` flag)
- `layoutSignal()` returning `'click listener'` (layout.ts:313)
- `enrichFrame()` adding these to the outline (enrich.ts:159-197)

The d-clickables fixture already tests this with the chip span.
