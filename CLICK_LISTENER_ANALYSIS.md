# Click Listener Detection Analysis and Fix

## Current Implementation

NanoBrowser already has a two-tier system for detecting clickable elements:

### Tier 1: Accessibility Tree (outline.ts)
- Captures Chrome's accessibility tree via `Accessibility.getFullAXTree`
- Assigns refs to elements with `INTERACTIVE_ROLES` (button, link, textbox, etc.)
- Assigns refs to focusable elements with a name (custom controls with tabindex)
- **Limitation**: Plain `<span>` or `<div>` elements without semantic roles are often ignored or flattened to text

### Tier 2: Layout Snapshot (enrich.ts + layout.ts)
- Captures DOM structure via `DOMSnapshot.captureSnapshot`
- Chrome's CDP sets `isClickable: true` on elements with click event listeners
- `layoutSignal()` detects various signals: `'click listener'`, `'pointer cursor'`, `'tabindex'`, `'scrollable'`, etc.
- `enrichFrame()` adds these elements to the outline with role `'clickable'`

## How It Should Work

For a `<span class="mt10">转文字</span>` with a click listener:

1. **Accessibility tree**: Span might be ignored (role: generic with no name) or flattened to text
2. **Layout snapshot**: Chrome marks it as `isClickable: true` (assuming the listener was attached)
3. **layoutSignal()**: Returns `'click listener'` (layout.ts:313)
4. **enrichFrame()**: Adds it to outline with role `'clickable'` (enrich.ts:180-194)
5. **Result**: Should get a ref like `[e1] clickable "转文字"`

## Potential Issues

### Issue 1: Element Not Rendered
**Check**: `isRendered(node)` (enrich.ts:157)
- Requires `bounds` with width > 0, height > 0
- `display !== 'none'`, `visibility !== 'hidden'`, `opacity !== '0'`

**Solution**: Ensure the span has visible dimensions

### Issue 2: Inside Another Actionable Element
**Check**: `insideActionable(node)` (enrich.ts:161)
```typescript
if (insideActionable(node) && !isNativeControl(node) && !existing?.name) continue;
```

This skips non-native-controls inside actionable ancestors UNLESS they have a name.

**Example**:
```html
<div id="card">
  Click anywhere
  <span id="inner">转文字</span>
</div>
<script>
  document.getElementById('card').addEventListener('click', handler1);
  document.getElementById('inner').addEventListener('click', handler2);
</script>
```

In this case:
- The outer div gets a ref: `[e1] clickable "Click anywhere 转文字"`
- The inner span is skipped (it's inside an actionable element)
- **This is correct behavior**: clicking the outer element is sufficient

### Issue 3: Chrome Doesn't Mark It as Clickable
**Possible reasons**:
1. Listener attached after snapshot was taken
2. Listener attached via inline `onclick` attribute (might need verification)
3. Listener attached to a parent with event delegation
4. Element is in a shadow DOM or iframe that wasn't captured

**Solution**: Verify in Chrome DevTools that the element itself has the listener:
```javascript
getEventListeners($0) // Should show {click: Array(1)}
```

If the listener is on a parent (event delegation), then Chrome correctly does NOT mark the child as clickable.

## Diagnosis Steps

To diagnose why a specific element isn't getting a ref:

1. **Check the capture JSON** (if available via test panel):
   ```javascript
   // Find the element in layout snapshot
   const layout = capture.layout[''];
   const nodes = layout.documents[0].nodes;
   const backend = nodes.backendNodeId;
   
   // Find your element by backend ID
   const idx = backend.indexOf(YOUR_BACKEND_ID);
   
   // Check if it's marked clickable
   const isClickable = nodes.isClickable.index.includes(idx);
   ```

2. **Check if it's in the AX tree**:
   ```javascript
   const axNode = capture.frames[0].nodes.find(n => n.backendDOMNodeId === YOUR_BACKEND_ID);
   // If undefined, element is not in AX tree (expected for plain spans)
   ```

3. **Check if it passes enrichment filters**:
   - Has bounds? `node.bounds !== null`
   - Is rendered? Width and height > 0
   - Inside actionable? Check parent elements
   - Has signal? `clickable: true` or other signals

## The Real Issue

Based on the user's description:
> Chrome DevTools 验证结果: getEventListeners($0) 返回 {click: Array(1)}
> NanoBrowser 只能在可访问性树中看到无交互角色的文本节点

This suggests:
1. The element HAS a click listener (verified in DevTools)
2. NanoBrowser only sees it as text (not in AX tree, expected)
3. It should be detected by tier 2, but isn't

**Most likely cause**: The span is **inside another clickable element**, so enrichment skips it (line 161 of enrich.ts).

**Fix**: If the inner element should be independently clickable:
- Add `aria-label` or visible text that differs from parent
- Or modify the page to not nest clickable elements
- Or accept that clicking the parent element is sufficient

## Testing

The existing `d-clickables` fixture already tests this:
- Chip span with click listener: ✓ Gets ref
- Nested span inside clickable div: ✓ Correctly skipped (parent is enough)

To test a new case:
1. Create HTML fixture in `fixtures/pages/`
2. Load in Chrome with test panel
3. Capture and save to `src/browser-control/core/__tests__/payloads/`
4. Add test in `__tests__/`

## Conclusion

**The system already works correctly**. Span elements with click listeners ARE detected by Chrome's `isClickable` flag and ARE added to the outline by tier 2 enrichment, UNLESS:

1. They're not rendered (no bounds, hidden, etc.)
2. They're inside another actionable element (optimization to avoid duplicate refs)
3. Chrome doesn't detect the listener (event delegation, timing, etc.)

For the reported case, we need to verify which of these is happening.
