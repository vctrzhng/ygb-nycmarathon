---
name: a11y-responsive-review
description: Audit this site (index.html) for responsive layout bugs and accessibility issues — run before shipping any visual/structural change. Checks viewport overflow, color contrast, keyboard navigation/focus, and a11y semantics using a headless-Chrome+CDP harness (no Playwright/Puppeteer needed).
---

# Accessibility + responsive review for ygb-nycmarathon

This is a single static `index.html` (dark-themed fundraiser page: sticky header,
hero with live stats/progress bar, canvas skyline, donor list, training chart,
footer). It has no build step and no test suite — the only way to verify a
change is to actually render it and probe it. Use this skill any time you touch
layout, CSS breakpoints, color tokens, focus/keyboard behavior, or markup
structure (headings, landmarks, lists).

## Why this exists

A real audit (see git history around the "make sure regular, low-vision,
keyboard-only, and different-device users can all reliably complete the core
actions" task) found multiple bugs that looked fine in isolation but broke
under specific conditions:
- A `position: sticky` full-bleed trick that caused a permanent ~24px
  horizontal overflow on every screen size — invisible until you actually
  measured `scrollWidth` vs `clientWidth`.
- A 4-column stats row that had silently overflowed-and-clipped (via
  `overflow: hidden` on an ancestor) at narrow widths for who knows how long.
- An SVG chart whose text was sized in `viewBox` units, so it rendered at
  ~4px actual pixels on a phone — invisible unless you measured the real
  scale factor, not just eyeballed a screenshot.
- A contrast failure caused by stacking `opacity` on top of an
  already-translucent color token (two translucencies compounding).
- Anchor links (`#supporters`, `#training`) that scrolled the page correctly
  but never moved keyboard/AT focus, because the targets weren't focusable.

None of these were visible from reading the CSS/JS. They only show up when
you actually drive a browser and measure.

## Setup: headless Chrome + CDP (no Playwright/Puppeteer in this env)

```bash
# Terminal 1: serve the file
python3 -m http.server 8765 &

# Launch headless Chrome with remote debugging open to localhost
pkill -f "remote-debugging-port=9333" 2>/dev/null
rm -rf /tmp/cdp-profile && mkdir -p /tmp/cdp-profile
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" \
  --headless=new --disable-gpu --remote-debugging-port=9333 \
  "--remote-allow-origins=*" --user-data-dir=/tmp/cdp-profile \
  --window-size=1440,900 about:blank &

# One-time: pip install websocket-client (no other deps needed)
pip3 install --quiet websocket-client
```

Drive it from Python via the CDP websocket (`Page.navigate`,
`Runtime.evaluate`, `Page.captureScreenshot`, `Emulation.setDeviceMetricsOverride`,
`Input.dispatchKeyEvent`). Build a small `CDP` helper class once (navigate,
eval, set_size, screenshot, close) and reuse it across checks — see the
pattern below; recreate it fresh each session rather than relying on a
checked-in script.

```python
import json, base64, time, urllib.request, websocket

class CDP:
    def __init__(self, port=9333):
        targets = json.loads(urllib.request.urlopen(f"http://localhost:{port}/json").read())
        ws_url = [t["webSocketDebuggerUrl"] for t in targets if t["type"] == "page"][0]
        self.ws = websocket.create_connection(ws_url, timeout=30)
        self._id = 0
    def send(self, method, params=None):
        self._id += 1
        self.ws.send(json.dumps({"id": self._id, "method": method, "params": params or {}}))
        while True:
            d = json.loads(self.ws.recv())
            if d.get("id") == self._id:
                return d
    def eval(self, expr):
        r = self.send("Runtime.evaluate", {"expression": expr, "returnByValue": True})["result"]
        return r.get("result", {}).get("value")
    def navigate(self, url):
        self.send("Page.enable"); self.send("Page.navigate", {"url": url}); time.sleep(1.0)
    def set_size(self, w, h):
        self.send("Emulation.setDeviceMetricsOverride", {"width": w, "height": h, "deviceScaleFactor": 1, "mobile": False})
        time.sleep(0.15)
    def screenshot(self, path, clip=None):
        params = {"format": "png"}
        if clip: params["clip"] = clip
        res = self.send("Page.captureScreenshot", params)
        open(path, "wb").write(base64.b64decode(res["result"]["data"]))
```

**Gotcha:** `html { scroll-behavior: smooth }` makes `window.scrollTo()` via
`Runtime.evaluate` async/animated — reads taken immediately after will be
mid-animation. Set `document.documentElement.style.scrollBehavior = 'auto'`
in the page before scripting scrolls, or wait ~300-500ms after each scroll.

## Checks to run

### 1. Responsive overflow sweep (do this first — cheapest, highest signal)

Sweep viewport width in ~10-20px steps from 320 to 1440+ and compare
`document.documentElement.scrollWidth` vs `clientWidth` at each. Any width
where `scrollWidth > clientWidth` is real horizontal overflow — find the
exact offending element by scanning `document.querySelectorAll('body *')`
for any `getBoundingClientRect()` with `right > clientWidth` or `left < 0`.

```js
[...document.querySelectorAll('body *')].filter(el => {
  const r = el.getBoundingClientRect();
  return r.right > document.documentElement.clientWidth + 1 || r.left < -1;
}).map(el => ({tag: el.tagName, cls: el.className, left: r.left, right: r.right}))
```

Don't stop at the first viewport that looks fine — overflow in this codebase
has appeared in narrow, specific width *bands* (e.g. only 480–620px, or only
601–619px) that a coarse check (just 375/768/1440) would miss. Sweep in fine
steps across the full range.

Also check: 200% zoom equivalent (test at ~half the target viewport width,
since full-page zoom ≈ shrinking the CSS px viewport), and 320px exactly
(WCAG 1.4.10 reflow minimum).

### 2. Color contrast

Compute actual WCAG contrast ratios in-page (don't eyeball it) — resolve each
text element's effective foreground/background by walking up the DOM
compositing any translucent layers, then apply the standard relative-luminance
formula. Check every distinct text/background pairing used (muted text, gold
text, button text, chips, focus rings as non-text 3:1). Small text (<18px,
or <14px bold) needs 4.5:1; large text needs 3:1; non-text UI indicators
(focus rings) need 3:1.

Watch specifically for **stacked translucency** — a `color` with an alpha
channel *and* a CSS `opacity` on the same element compounds multiplicatively
and is an easy way to silently drop below threshold even when the base color
token is fine elsewhere.

### 3. Keyboard navigation and focus

Drive real `Input.dispatchKeyEvent` Tab/Enter presses (don't just read the
HTML and assume tab order) and inspect `document.activeElement` after each.
Check:
- Tab order matches visual/logical order, and is a closed loop over exactly
  the actually-interactive elements.
- Every focused state is visibly distinguishable — screenshot a focused
  element and confirm there's a visible ring, don't just trust that no
  `outline: none` exists in the CSS.
- Enter activates links/buttons as expected.
- **Anchor links to in-page sections** (`href="#foo"`): confirm both that the
  scroll happens *and* that `document.activeElement` lands on the target
  after activation. If the target isn't natively focusable, this needs
  `tabindex="-1"` on the target plus a `hashchange` listener calling
  `.focus({preventScroll: true})` — scrolling alone is not enough for
  screen-reader users to get oriented.
- Any custom interactive widgets (not plain `<a>`/`<button>`) need Space and
  Esc handled explicitly; plain links/buttons get this for free from the
  browser and don't need extra JS.

### 4. Semantics

- Exactly one `<main>` landmark, plus a skip link as the first focusable
  element pointing into it.
- Heading hierarchy has no skipped levels (h1 → h2 → h3, never h1 → h3) and
  every major content section has a real heading, not just a styled `<p>`.
- The `<canvas>` skyline and the SVG mileage chart are decorative/duplicate
  visualizations of data that's *also* present as real text elsewhere (stats
  bar, donor list, training-stat numbers) — they should be `aria-hidden`
  rather than given a fake accessible name. If a visualization's *only* copy
  of some data point lives inside it, don't hide it — add a `.sr-only` text
  summary instead.
- Lists of repeated items (donor chips, etc.) should be real `<ul>/<li>`,
  not `<div>/<span>` soup, with `list-style: none` to preserve the visual.
  If a list item concatenates two pieces of text with only CSS spacing
  between them (e.g. name + amount), add a `.sr-only` separator
  (`<span class="sr-only">, </span>`) so it doesn't read as one run-on word.
- No forms currently exist on this page — if one gets added, every input
  needs a programmatically associated `<label>`.

### 5. SVG/Canvas-specific scaling trap

The mileage chart's `viewBox` is fixed but its rendered width isn't — text
sized in viewBox units shrinks with the container. If you change this chart,
re-verify the counter-scaling math (`renderTraining()`'s `fz()` helper) still
keeps on-screen text at a legible fixed px size across 320–1440px, and that
counter-scaled text doesn't get *physically too long* for the chart's fixed
height at small scale (clipped by the SVG's own viewport) — there's a
`svgScale >= 0.55` cutoff that hides the rotated y-axis title below that
scale for exactly this reason.

## Reporting

For each issue: what it is, how you confirmed it (the actual measurement —
not "looks fine"), the fix applied, and whether it's fully resolved or a
documented remaining risk (e.g. true OS-level "text-only zoom" that doesn't
scale the viewport is a known, accepted gap here since evergreen browsers
default to full-page zoom instead, which *is* verified clean down to 320px).
