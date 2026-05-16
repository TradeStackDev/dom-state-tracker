# 🔍 dom-state-tracker

> **Real-time DOM state diffing and serialization for AI agent observation pipelines.**

[![Python](https://img.shields.io/badge/Python-3.11+-3776ab?style=flat&logo=python&logoColor=white)](https://python.org)
[![JavaScript](https://img.shields.io/badge/JavaScript-ES2022-F7DF1E?style=flat&logo=javascript&logoColor=black)](https://developer.mozilla.org/en-US/docs/Web/JavaScript)
[![Playwright](https://img.shields.io/badge/Playwright-1.44-2EAD33?style=flat)](https://playwright.dev)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

---

## Overview

`dom-state-tracker` is a lightweight Python + JavaScript library for capturing, diffing, and serializing the state of web pages in a format optimized for AI agent consumption.

It hooks into a live browser session (via Playwright or CDP) and continuously captures:

- Structured DOM snapshots (accessibility tree + raw DOM)
- Interactive element inventories with bounding boxes
- Real-time DOM mutation diffs
- User interaction events (clicks, keypresses, scrolls)
- Network activity relevant to page state changes

Designed as a companion library to [`webenv-sim`](https://github.com/adam/webenv-sim), but usable standalone in any browser automation pipeline.

---

## Why Not Just Use Screenshots?

Screenshots are expensive (high token count, slow inference) and lose structured information. `dom-state-tracker` gives your agent:

| Signal | Screenshot | dom-state-tracker |
|--------|-----------|-------------------|
| Interactive elements | ❌ (must detect visually) | ✅ (exact list + coords) |
| Text content | ❌ (OCR required) | ✅ (structured) |
| ARIA roles & labels | ❌ | ✅ |
| Real-time diffs | ❌ | ✅ |
| Token efficiency | ❌ High | ✅ Low |
| Deterministic | ❌ | ✅ |

---

## Quick Start

```bash
pip install dom-state-tracker
playwright install chromium
```

```python
from dom_state_tracker import DOMTracker
from playwright.async_api import async_playwright

async with async_playwright() as p:
    browser = await p.chromium.launch()
    page = await browser.new_page()

    tracker = DOMTracker(page)
    await tracker.attach()

    await page.goto("https://example.com")

    # Get full page state
    state = await tracker.snapshot()
    print(state.interactive_elements)
    print(state.aria_tree)

    # Perform an action
    await page.click("#submit-button")

    # Get only what changed
    diff = await tracker.diff()
    print(diff.added_elements)
    print(diff.removed_elements)
    print(diff.modified_elements)
```

---

## Observation Format

### Full Snapshot

```json
{
  "url": "https://shop.example.com/checkout",
  "title": "Checkout — Example Shop",
  "timestamp": 1716830400.123,
  "viewport": { "width": 1280, "height": 800 },
  "scroll": { "x": 0, "y": 340 },
  "interactive_elements": [
    {
      "id": "input-email",
      "tag": "input",
      "type": "email",
      "placeholder": "Email address",
      "value": "",
      "aria_label": "Email address",
      "bbox": { "x": 320, "y": 240, "width": 400, "height": 44 },
      "visible": true,
      "enabled": true
    },
    {
      "id": "btn-place-order",
      "tag": "button",
      "text": "Place Order",
      "bbox": { "x": 320, "y": 640, "width": 400, "height": 52 },
      "visible": true,
      "enabled": false
    }
  ],
  "aria_tree": { ... },
  "page_text": "Checkout\nOrder Summary\n...",
  "forms": [
    {
      "id": "checkout-form",
      "fields": ["input-email", "input-card", "input-expiry", "input-cvv"],
      "submit_button": "btn-place-order"
    }
  ]
}
```

### DOM Diff

```json
{
  "trigger": "click",
  "timestamp": 1716830415.456,
  "added_elements": [
    { "id": "toast-success", "tag": "div", "text": "Item added to cart!", "bbox": {...} }
  ],
  "removed_elements": [],
  "modified_elements": [
    {
      "id": "cart-count",
      "changes": { "text": { "before": "0", "after": "1" } }
    }
  ],
  "url_changed": false,
  "network_requests": [
    { "method": "POST", "url": "/api/cart/add", "status": 200 }
  ]
}
```

---

## Performance Optimizations

`dom-state-tracker` uses several techniques to minimize payload size and latency:

- **Incremental diffing** — only transmit changed nodes, not full tree
- **Element deduplication** — stable IDs across snapshots for reliable diffing
- **Compression** — ZSTD compression on serialized snapshots (~60% size reduction)
- **Selective capture** — configurable filters to exclude invisible or irrelevant elements
- **Batched mutations** — MutationObserver events are batched and debounced

### Benchmark

| Metric | Value |
|--------|-------|
| Snapshot latency (median) | 38ms |
| Snapshot latency (p99) | 95ms |
| Avg snapshot size (raw) | 42KB |
| Avg snapshot size (compressed) | 17KB |
| Diff latency (median) | 12ms |

---

## Integration with webenv-sim

```python
from webenv_sim import SimSession
from dom_state_tracker import DOMTracker

async with SimSession(task=task) as session:
    # DOMTracker is built into SimSession
    state = await session.observe()  # Returns DOMTracker snapshot
    diff = await session.last_diff() # Returns DOMTracker diff
```

---

## Configuration

```python
tracker = DOMTracker(
    page,
    include_invisible=False,      # Skip off-screen elements
    include_aria_tree=True,       # Include ARIA accessibility tree
    include_network=True,         # Track XHR/fetch during mutations
    compress_snapshots=True,      # ZSTD compression
    debounce_ms=50,               # Batch mutation events
    max_elements=500              # Cap interactive element list
)
```

---

## License

MIT © Adam
