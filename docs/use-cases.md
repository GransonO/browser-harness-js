# Sample Use Cases

Concrete examples of what an agent can do with `browser-harness-js`. Each snippet runs via `browser-harness-js '<code>'` or `POST /eval`.

---

## 1. Research & Data Extraction

### 1.1 Scrape data from an authenticated dashboard

Attach to the user's already-logged-in Chrome — no credential handling needed.

```js
await session.connect()
const tabs = await listPageTargets()
await session.use(tabs[0].targetId)

// Navigate to the internal report page
await session.Page.enable()
await session.Page.navigate({ url: 'https://app.company.com/reports/monthly' })
await session.waitFor('Page.loadEventFired', undefined, 15_000)

// Extract table rows
const { result } = await session.Runtime.evaluate({
  expression: `
    Array.from(document.querySelectorAll('table tbody tr')).map(row =>
      Array.from(row.querySelectorAll('td')).map(td => td.innerText.trim())
    )
  `,
  returnByValue: true,
})
return JSON.stringify(result.value)
```

Because you attach to the live browser, session cookies, SSO tokens, and MFA are already satisfied — the agent never touches credentials.

---

### 1.2 Monitor a product for price drops

```js
await session.connect()
const tabs = await listPageTargets()
await session.use(tabs[0].targetId)
await session.Page.enable()
await session.Page.navigate({ url: 'https://example-shop.com/product/12345' })
await session.waitFor('Page.loadEventFired', undefined, 10_000)

const { result } = await session.Runtime.evaluate({
  expression: `document.querySelector('[data-price]')?.dataset.price`,
  returnByValue: true,
})
const price = parseFloat(result.value)
return JSON.stringify({ price, alert: price < 49.99 })
```

Run this on a schedule. The agent tells you when to buy.

---

### 1.3 Aggregate content from multiple tabs simultaneously

```js
await session.connect()
const urls = [
  'https://news.ycombinator.com',
  'https://lobste.rs',
  'https://thenewstack.io',
]

// Open each URL in its own tab
const targetIds = []
for (const url of urls) {
  const { targetId } = await session.Target.createTarget({ url: 'about:blank' })
  await session.use(targetId)
  await session.Page.enable()
  await session.Page.navigate({ url })
  targetIds.push(targetId)
}

// Wait for all to load, then extract headlines
const results = []
for (const targetId of targetIds) {
  await session.use(targetId)
  await session.waitFor('Page.loadEventFired', undefined, 15_000)
  const { result } = await session.Runtime.evaluate({
    expression: `
      Array.from(document.querySelectorAll('a[href]'))
        .slice(0, 10)
        .map(a => ({ title: a.innerText.trim(), href: a.href }))
        .filter(i => i.title.length > 10)
    `,
    returnByValue: true,
  })
  results.push({ targetId, headlines: result.value })
}

return JSON.stringify(results, null, 2)
```

---

## 2. Document & PDF Generation

### 2.1 Export any page as a PDF

```js
await session.connect()
const tabs = await listPageTargets()
await session.use(tabs[0].targetId)
await session.Page.enable()
await session.Page.navigate({ url: 'https://example.com/invoice/789' })
await session.waitFor('Page.loadEventFired', undefined, 10_000)

const { data } = await session.Page.printToPDF({
  printBackground: true,
  marginTop: 0.5,
  marginBottom: 0.5,
  marginLeft: 0.5,
  marginRight: 0.5,
  paperWidth: 8.5,
  paperHeight: 11,
})

const { tmpdir } = await import('node:os')
const out = `${tmpdir()}/invoice-789.pdf`
await Bun.write(out, Buffer.from(data, 'base64'))
return out
```

Works on any page the browser can render — dashboards, reports, web apps — not just static HTML.

---

### 2.2 Screenshot a specific element for a report

```js
await session.connect()
const tabs = await listPageTargets()
await session.use(tabs[0].targetId)

await session.DOM.enable()
const { root } = await session.DOM.getDocument({})
const { nodeId } = await session.DOM.querySelector({
  nodeId: root.nodeId,
  selector: '#quarterly-chart',
})
const { model } = await session.DOM.getBoxModel({ nodeId })
const [x, y] = model.border   // [x1,y1, x2,y1, x2,y2, x1,y2]
const { data } = await session.Page.captureScreenshot({
  format: 'png',
  clip: { x, y, width: model.width, height: model.height, scale: 1 },
})

const { tmpdir } = await import('node:os')
const out = `${tmpdir()}/chart.png`
await Bun.write(out, Buffer.from(data, 'base64'))
return out
```

---

## 3. Form Automation

### 3.1 Fill and submit a multi-step form

```js
await session.connect()
const tabs = await listPageTargets()
await session.use(tabs[0].targetId)
await session.DOM.enable()

// Step 1: fill personal details
await session.Runtime.evaluate({
  expression: `
    document.querySelector('#first-name').value = 'Jane';
    document.querySelector('#last-name').value = 'Smith';
    document.querySelector('#email').value = 'jane@example.com';
  `,
})

// Click Next
const { root } = await session.DOM.getDocument({})
const { nodeId } = await session.DOM.querySelector({ nodeId: root.nodeId, selector: '#next-btn' })
const { model } = await session.DOM.getBoxModel({ nodeId })
const cx = model.border[0] + model.width / 2
const cy = model.border[1] + model.height / 2
await session.Input.dispatchMouseEvent({ type: 'mousePressed', x: cx, y: cy, button: 'left', clickCount: 1 })
await session.Input.dispatchMouseEvent({ type: 'mouseReleased', x: cx, y: cy, button: 'left', clickCount: 1 })

// Wait for step 2 to load
await session.Network.enable({})
await session.waitFor(
  'Network.responseReceived',
  p => p.response.url.includes('/step/2'),
  10_000
)
```

---

### 3.2 Upload a file to a web form

Works regardless of whether the `<input type="file">` is hidden or styled over.

```js
await session.connect()
const tabs = await listPageTargets()
await session.use(tabs[0].targetId)
await session.DOM.enable()

const { root } = await session.DOM.getDocument({ depth: -1 })
const { nodeId } = await session.DOM.querySelector({
  nodeId: root.nodeId,
  selector: 'input[type="file"]',
})

await session.DOM.setFileInputFiles({
  nodeId,
  files: ['/home/user/reports/q3-2025.xlsx'],
})

// Verify the upload request fired
await session.Network.enable({})
const upload = await session.waitFor(
  'Network.requestWillBeSent',
  p => p.request.method === 'POST' && p.request.url.includes('/upload'),
  15_000
)
return upload.request.url
```

---

## 4. Network Inspection & API Debugging

### 4.1 Capture the API response behind a button click

Useful for understanding what a web app sends/receives without reading minified source.

```js
await session.connect()
const tabs = await listPageTargets()
await session.use(tabs[0].targetId)
await session.Network.enable({})

// Pre-arm the listener, then trigger the action
const responseReady = session.waitFor(
  'Network.responseReceived',
  p => p.response.url.includes('/api/') && p.response.status === 200,
  10_000
)

// Click the "Load More" button by evaluating a click directly
await session.Runtime.evaluate({
  expression: `document.querySelector('[data-action="load-more"]').click()`,
})

const ev = await responseReady
const { body } = await session.Network.getResponseBody({ requestId: ev.requestId })
return body
```

---

### 4.2 Mock a feature flag to test a UI branch

```js
await session.connect()
const tabs = await listPageTargets()
await session.use(tabs[0].targetId)

await session.Fetch.enable({
  patterns: [{ urlPattern: '*/api/flags*', requestStage: 'Response' }],
})

// Intercept flag responses and force the flag on
session.onEvent(async (method, params) => {
  if (method === 'Fetch.requestPaused') {
    await session.Fetch.fulfillRequest({
      requestId: params.requestId,
      responseCode: 200,
      responseHeaders: [{ name: 'content-type', value: 'application/json' }],
      body: Buffer.from(JSON.stringify({ new_checkout: true, dark_mode: true })).toString('base64'),
    })
  }
})

await session.Page.enable()
await session.Page.reload({})
await session.waitFor('Page.loadEventFired', undefined, 10_000)

// Take a screenshot of the UI with flags forced on
const { data } = await session.Page.captureScreenshot({ format: 'png' })
const { tmpdir } = await import('node:os')
await Bun.write(`${tmpdir()}/with-flags.png`, Buffer.from(data, 'base64'))
```

---

### 4.3 Log all outbound requests during a user flow

```js
await session.connect()
const tabs = await listPageTargets()
await session.use(tabs[0].targetId)
await session.Network.enable({})

const log = []
const off = session.onEvent((method, params) => {
  if (method === 'Network.requestWillBeSent') {
    log.push({
      method: params.request.method,
      url: params.request.url,
      postData: params.request.postData ?? null,
    })
  }
})

// Do the flow you want to observe
await session.Page.enable()
await session.Page.navigate({ url: 'https://app.example.com/checkout' })
await session.waitFor('Page.loadEventFired', undefined, 10_000)

off()
return JSON.stringify(log, null, 2)
```

---

## 5. Session & Cookie Management

### 5.1 Export a browser session for backup or reuse

```js
await session.connect()
const tabs = await listPageTargets()
await session.use(tabs[0].targetId)
await session.Network.enable({})

// Dump all cookies visible to the current page
const { cookies } = await session.Network.getCookies({})

// Also grab localStorage
const { result } = await session.Runtime.evaluate({
  expression: `JSON.stringify(Object.entries(localStorage))`,
  returnByValue: true,
})

const snapshot = {
  cookies,
  localStorage: JSON.parse(result.value),
  timestamp: new Date().toISOString(),
}

const { tmpdir } = await import('node:os')
await Bun.write(`${tmpdir()}/session-snapshot.json`, JSON.stringify(snapshot, null, 2))
return `Exported ${cookies.length} cookies`
```

---

### 5.2 Restore a saved session into Chrome

```js
const { readFileSync } = await import('node:fs')
const snapshot = JSON.parse(readFileSync('/tmp/session-snapshot.json', 'utf8'))

await session.connect()
const tabs = await listPageTargets()
await session.use(tabs[0].targetId)
await session.Network.enable({})

// Restore cookies
await session.Network.setCookies({ cookies: snapshot.cookies })

// Restore localStorage (must be on the right origin first)
await session.Page.enable()
await session.Page.navigate({ url: 'https://app.example.com' })
await session.waitFor('Page.loadEventFired', undefined, 10_000)

for (const [key, value] of snapshot.localStorage) {
  await session.Runtime.evaluate({
    expression: `localStorage.setItem(${JSON.stringify(key)}, ${JSON.stringify(value)})`,
  })
}

await session.Page.reload({})
return 'Session restored'
```

---

## 6. Testing & Quality Assurance

### 6.1 Agent-written E2E test (no test framework required)

The agent describes what to verify; the harness provides the primitives.

```js
await session.connect()
const tabs = await listPageTargets()
await session.use(tabs[0].targetId)
await session.Page.enable()
await session.Network.enable({})

const failures = []

// Test 1: login page loads
await session.Page.navigate({ url: 'https://app.example.com/login' })
await session.waitFor('Page.loadEventFired', undefined, 10_000)
const { result: titleResult } = await session.Runtime.evaluate({
  expression: `document.title`,
  returnByValue: true,
})
if (!titleResult.value.includes('Login')) {
  failures.push(`Expected title to contain 'Login', got: ${titleResult.value}`)
}

// Test 2: submit button exists and is enabled
const { result: btnResult } = await session.Runtime.evaluate({
  expression: `
    const btn = document.querySelector('button[type="submit"]')
    btn ? { found: true, disabled: btn.disabled } : { found: false }
  `,
  returnByValue: true,
})
if (!btnResult.value.found) failures.push('Submit button not found')
if (btnResult.value.disabled) failures.push('Submit button is disabled on load')

// Test 3: API returns 200 on form submit
const responseReady = session.waitFor(
  'Network.responseReceived',
  p => p.response.url.includes('/api/login'),
  10_000
)
await session.Runtime.evaluate({
  expression: `
    document.querySelector('#email').value = 'test@example.com';
    document.querySelector('#password').value = 'hunter2';
    document.querySelector('button[type="submit"]').click();
  `,
})
try {
  const ev = await responseReady
  if (ev.response.status !== 200) {
    failures.push(`Login API returned ${ev.response.status}`)
  }
} catch {
  failures.push('Login API request not detected within 10s')
}

return JSON.stringify({ passed: failures.length === 0, failures })
```

---

### 6.2 Visual regression check via screenshot diff

Take a screenshot, compare pixel dimensions to a baseline, and flag regressions.

```js
await session.connect()
const tabs = await listPageTargets()
await session.use(tabs[0].targetId)
await session.Page.enable()
await session.Page.navigate({ url: 'https://app.example.com/dashboard' })
await session.waitFor('Page.loadEventFired', undefined, 10_000)

// Full-page screenshot
const { data } = await session.Page.captureScreenshot({
  format: 'png',
  captureBeyondViewport: true,
})

const { tmpdir } = await import('node:os')
const ts = Date.now()
const path = `${tmpdir()}/dashboard-${ts}.png`
await Bun.write(path, Buffer.from(data, 'base64'))

// Basic size sanity check (real diff uses image comparison libraries)
const buf = Buffer.from(data, 'base64')
return JSON.stringify({ path, sizeBytes: buf.length, timestamp: ts })
```

---

## 7. Automation Helpers for Power Users

### 7.1 Bulk-close tabs matching a pattern

```js
await session.connect()
const tabs = await listPageTargets()

const toClose = tabs.filter(t =>
  t.url.includes('twitter.com') || t.url.includes('reddit.com')
)

for (const tab of toClose) {
  await session.Target.closeTarget({ targetId: tab.targetId })
}

return `Closed ${toClose.length} tab(s)`
```

---

### 7.2 Group open tabs by domain and report

```js
await session.connect()
const tabs = await listPageTargets()

const byDomain = {}
for (const tab of tabs) {
  try {
    const domain = new URL(tab.url).hostname
    if (!byDomain[domain]) byDomain[domain] = []
    byDomain[domain].push({ title: tab.title, url: tab.url })
  } catch {}
}

return JSON.stringify(byDomain, null, 2)
```

---

### 7.3 Download a file that requires login

Chrome's cookies carry the auth; use `Browser.setDownloadBehavior` to capture the file.

```js
await session.connect()
const tabs = await listPageTargets()
await session.use(tabs[0].targetId)

const { tmpdir } = await import('node:os')
const downloadDir = `${tmpdir()}/cdp-downloads`
await Bun.write(`${downloadDir}/.keep`, '')

await session.Browser.setDownloadBehavior({
  behavior: 'allow',
  downloadPath: downloadDir,
  eventsEnabled: true,
})

// Navigate to the protected download URL
await session.Page.enable()
await session.Page.navigate({ url: 'https://app.example.com/export/report.csv' })

const ev = await session.waitFor(
  'Browser.downloadWillBegin',
  p => p.suggestedFilename.endsWith('.csv'),
  15_000
)

await session.waitFor(
  'Browser.downloadProgress',
  p => p.guid === ev.guid && p.state === 'completed',
  60_000
)

return `${downloadDir}/${ev.suggestedFilename}`
```

---

### 7.4 Take a full-page screenshot of every open tab

```js
await session.connect()
const tabs = await listPageTargets()
const { tmpdir } = await import('node:os')
const saved = []

for (const tab of tabs) {
  await session.use(tab.targetId)
  await session.Page.enable()
  try {
    const { data } = await session.Page.captureScreenshot({ format: 'jpeg', quality: 70 })
    const safe = tab.title.replace(/[^a-z0-9]/gi, '_').slice(0, 40)
    const path = `${tmpdir()}/${safe}.jpg`
    await Bun.write(path, Buffer.from(data, 'base64'))
    saved.push({ title: tab.title, path })
  } catch (e) {
    saved.push({ title: tab.title, error: String(e) })
  }
}

return JSON.stringify(saved, null, 2)
```

---

## 8. AI Agent Copilot Patterns

### 8.1 "Summarise what's on my screen"

```js
await session.connect()
const tabs = await listPageTargets()
await session.use(tabs[0].targetId)

// Grab visible text and the page title
const { result } = await session.Runtime.evaluate({
  expression: `({
    title: document.title,
    url: location.href,
    bodyText: document.body?.innerText?.slice(0, 8000) ?? '',
  })`,
  returnByValue: true,
})

// Take a screenshot for the model to look at
const { data } = await session.Page.captureScreenshot({ format: 'jpeg', quality: 60 })
const { tmpdir } = await import('node:os')
const shot = `${tmpdir()}/screen.jpg`
await Bun.write(shot, Buffer.from(data, 'base64'))

return JSON.stringify({ ...result.value, screenshot: shot })
```

Pass both the text and the screenshot path to your LLM call for a richer summary.

---

### 8.2 "Fill this form using information from my notes"

```js
// Agent receives: { name: "Jane Smith", email: "jane@example.com", company: "Acme" }
// Agent finds matching inputs on the page and fills them

await session.connect()
const tabs = await listPageTargets()
await session.use(tabs[0].targetId)

// Discover form fields
const { result } = await session.Runtime.evaluate({
  expression: `
    Array.from(document.querySelectorAll('input, textarea, select')).map(el => ({
      tag: el.tagName.toLowerCase(),
      type: el.type,
      name: el.name,
      id: el.id,
      placeholder: el.placeholder,
      label: document.querySelector('label[for="' + el.id + '"]')?.innerText ?? '',
    }))
  `,
  returnByValue: true,
})

// The agent sends this field list to the LLM, which returns a fill map:
// { "#name": "Jane Smith", "#email": "jane@example.com", "#company": "Acme" }
// Then:
const fillMap = { '#name': 'Jane Smith', '#email': 'jane@example.com', '#company': 'Acme' }
for (const [selector, value] of Object.entries(fillMap)) {
  await session.Runtime.evaluate({
    expression: `
      const el = document.querySelector(${JSON.stringify(selector)})
      if (el) { el.value = ${JSON.stringify(value)}; el.dispatchEvent(new Event('input', {bubbles:true})) }
    `,
  })
}
```

---

### 8.3 Watch a page and alert when it changes

Run this in a long-lived agent session.

```js
await session.connect()
const tabs = await listPageTargets()
await session.use(tabs[0].targetId)
await session.DOM.enable()

// Snapshot the current text content hash
const snapshot = async () => {
  const { result } = await session.Runtime.evaluate({
    expression: `document.body?.innerText ?? ''`,
    returnByValue: true,
  })
  return result.value.trim()
}

globalThis.lastSnapshot = await snapshot()
globalThis.checkInterval = setInterval(async () => {
  const current = await snapshot()
  if (current !== globalThis.lastSnapshot) {
    globalThis.lastSnapshot = current
    // Signal the agent: write to a file or call a webhook
    const { tmpdir } = await import('node:os')
    await Bun.write(`${tmpdir()}/page-changed.txt`, new Date().toISOString())
  }
}, 5000)

return 'Watching for changes every 5s'
```

---

## 9. Developer Workflows

### 9.1 Inspect runtime state in a live app (no source access needed)

```js
await session.connect()
const tabs = await listPageTargets()
await session.use(tabs[0].targetId)

// Read the Redux / Zustand store from the window
const { result } = await session.Runtime.evaluate({
  expression: `
    const store = window.__REDUX_STORE__ || window.store
    store ? JSON.stringify(store.getState()) : 'no store found'
  `,
  returnByValue: true,
})
return result.value
```

---

### 9.2 Inject a debug helper into a page

```js
await session.connect()
const tabs = await listPageTargets()
await session.use(tabs[0].targetId)

// Add a persistent overlay that shows the hovered element's selector
await session.Runtime.evaluate({
  expression: `
    const tip = document.createElement('div')
    tip.style.cssText = 'position:fixed;bottom:10px;right:10px;background:#000;color:#0f0;padding:8px 12px;font:13px monospace;z-index:99999;border-radius:4px;pointer-events:none'
    document.body.appendChild(tip)
    document.addEventListener('mouseover', e => {
      const el = e.target
      const sel = el.id ? '#' + el.id : el.className ? '.' + el.className.trim().split(/\s+/)[0] : el.tagName.toLowerCase()
      tip.textContent = sel
    })
  `,
})
return 'Debug overlay injected'
```

---

### 9.3 Measure page load performance

```js
await session.connect()
const tabs = await listPageTargets()
await session.use(tabs[0].targetId)
await session.Page.enable()

const start = Date.now()
await session.Page.navigate({ url: 'https://example.com' })
await session.waitFor('Page.loadEventFired', undefined, 30_000)
const wallClock = Date.now() - start

// Use the Navigation Timing API for browser-side breakdown
const { result } = await session.Runtime.evaluate({
  expression: `
    const t = performance.timing
    JSON.stringify({
      dns:        t.domainLookupEnd - t.domainLookupStart,
      tcp:        t.connectEnd - t.connectStart,
      ttfb:       t.responseStart - t.requestStart,
      download:   t.responseEnd - t.responseStart,
      domParse:   t.domInteractive - t.responseEnd,
      domReady:   t.domContentLoadedEventEnd - t.navigationStart,
      loadEvent:  t.loadEventEnd - t.navigationStart,
    })
  `,
  returnByValue: true,
})

return JSON.stringify({ wallClockMs: wallClock, breakdown: JSON.parse(result.value) }, null, 2)
```
