---
name: capture-site
description: Fully capture a live website into Figma as editable design layers. Use this skill when the user asks to "capture site to figma", "import site to figma", "clone site to figma", "recreate website in figma", "copy production site to figma", or provides a URL and wants it in Figma. Requires Figma Remote MCP and Playwright MCP to be connected.
argument-hint: [url]
allowed-tools: mcp__figma-remote__generate_figma_design, mcp__plugin_playwright_playwright__browser_run_code, mcp__plugin_playwright_playwright__browser_navigate, mcp__plugin_playwright_playwright__browser_snapshot, mcp__plugin_playwright_playwright__browser_take_screenshot, WebFetch
---

You are executing a full-site Figma capture. Your goal is to import every page, every breakpoint, and key UI states of a live website into a structured Figma file — ready to be the foundation of a design system.

## Step 0: Gather inputs

Ask the user (in a single message):
1. **"What is the URL of the site you want to capture?"** (if not already provided in $ARGUMENTS)
2. **"Add to an existing Figma file, or create a new one?"**
   - If existing: ask for the Figma URL → extract `fileKey` from it
   - If new: auto-generate a file name from the domain (e.g. `costas.design — Full Capture`)

Then proceed autonomously without further interruptions unless blocked.

---

## Step 1: Preflight checks

Verify both MCPs are available before doing anything:
- Call `mcp__figma-remote__generate_figma_design` without parameters to confirm it's reachable
- Confirm Playwright tools are listed

If either is missing, stop and tell the user which MCP needs to be connected, with setup instructions.

---

## Step 2: Site discovery

Use this Playwright script to crawl the site and build a complete page list:

```javascript
async (page) => {
  await page.goto(TARGET_URL, { waitUntil: 'networkidle' });

  // Try sitemap first
  let sitemapUrls = [];
  try {
    const sitemapRes = await page.context().request.get(new URL('/sitemap.xml', TARGET_URL).href);
    if (sitemapRes.ok()) {
      const xml = await sitemapRes.text();
      sitemapUrls = [...xml.matchAll(/<loc>(.*?)<\/loc>/g)].map(m => m[1]);
    }
  } catch (e) {}

  // Extract nav links from the page
  const navLinks = await page.evaluate((base) => {
    const domain = new URL(base).hostname;
    return [...document.querySelectorAll('a[href]')]
      .map(a => { try { return new URL(a.href).href; } catch { return null; } })
      .filter(href =>
        href &&
        new URL(href).hostname === domain &&
        !href.match(/\.(pdf|zip|png|jpg|svg|webp|mp4|mp3)(\?|$)/i) &&
        !href.includes('#') &&
        !href.match(/\?(utm_|ref=)/i)
      );
  }, TARGET_URL);

  // Merge and deduplicate
  const all = [...new Set([...sitemapUrls, ...navLinks])];

  return all;
}
```

Deduplicate, filter to same-domain URLs only, strip tracking params.

Present the page list to the user:
> "Found N pages: [list]. Starting capture now."

If more than 20 pages, ask the user if they want to scope it (e.g. top-level only, or specific sections).

---

## Step 3: Define the capture matrix

For each page, capture:

**Breakpoints:**
| Name    | Width  |
|---------|--------|
| Mobile  | 375px  |
| Tablet  | 768px  |
| Desktop | 1440px |

**UI states** (only where interactive elements exist — detect via Playwright):
| State          | Trigger                                      |
|----------------|----------------------------------------------|
| Default        | Page load                                    |
| Nav open       | Click hamburger or hover primary nav         |
| Modal open     | Click first CTA that opens an overlay        |
| Form focused   | Click first form input                       |
| Accordion open | Click first accordion item (if present)      |
| Tab 2 active   | Click second tab (if tabs present)           |

**Frame naming convention:**
- `Home / Desktop`
- `Home / Mobile`
- `About / Tablet`
- `Contact / Desktop / Form Focused`
- `Work — Project Name / Desktop`

---

## Step 4: Pre-capture Playwright script

Run this before every capture to ensure full rendering:

```javascript
async (page, targetUrl, viewportWidth) => {
  // Strip CSP so capture script can inject
  await page.route('**/*', async (route) => {
    const response = await route.fetch();
    const headers = { ...response.headers() };
    delete headers['content-security-policy'];
    delete headers['content-security-policy-report-only'];
    await route.fulfill({ response, headers });
  });

  // Set viewport
  await page.setViewportSize({ width: viewportWidth, height: 900 });

  // Load page
  await page.goto(targetUrl, { waitUntil: 'networkidle' });

  // Kill animations so nothing is mid-transition
  await page.addStyleTag({
    content: `
      *, *::before, *::after {
        animation-duration: 0.001ms !important;
        animation-delay: 0.001ms !important;
        transition-duration: 0.001ms !important;
        transition-delay: 0.001ms !important;
        scroll-behavior: auto !important;
      }
    `
  });

  // Scroll full page to trigger lazy-load and reveal animations
  await page.evaluate(async () => {
    await new Promise((resolve) => {
      let total = 0;
      const step = 300;
      const cap = Math.min(document.body.scrollHeight, 10000);
      const timer = setInterval(() => {
        window.scrollBy(0, step);
        total += step;
        if (total >= cap) {
          clearInterval(timer);
          window.scrollTo(0, 0);
          resolve();
        }
      }, 80);
    });
  });

  // Wait for images and fonts
  await page.waitForTimeout(1000);
}
```

---

## Step 5: Capture execution

### For each page × breakpoint:

1. Run the pre-capture script (Step 4)
2. Inject the Figma capture script and submit:

```javascript
async (page) => {
  const r = await page.context().request.get('https://mcp.figma.com/mcp/html-to-design/capture.js');
  await page.evaluate((s) => {
    const el = document.createElement('script');
    el.textContent = s;
    document.head.appendChild(el);
  }, await r.text());
  await page.waitForTimeout(500);
  return await page.evaluate(() => window.figma.captureForDesign({
    captureId: 'CAPTURE_ID_HERE',
    endpoint: 'https://mcp.figma.com/mcp/capture/CAPTURE_ID_HERE/submit',
    selector: 'body'
  }));
}
```

3. Poll `generate_figma_design` with the `captureId` every 5 seconds until `status: completed` (max 10 attempts)
4. On completion, move to next capture

### Batching
- Generate all capture IDs upfront
- Run in parallel batches of 5 max
- Retry failed captures once before marking skipped

### For UI states:
After the default capture for a page, use Playwright to trigger each relevant state, then run a fresh capture with the state name appended to the frame name.

---

## Step 6: Figma file organization

Suggest this page structure in Figma (one Figma page per section):
```
🏠 Home
📄 About
💼 Work
📬 Contact
🔧 [Components]  ← reserved for design system work later
```

---

## Step 7: Post-capture report

After all captures complete, output:

```
✅ Site Capture Complete

📄 Pages captured: N
📐 Breakpoints: Mobile / Tablet / Desktop
🖱️  UI states: N
⚠️  Skipped: [any failed pages and why]

🔗 Figma file: [link]

Suggested next steps:
→ Review all frames in Figma
→ Identify repeated components (nav, cards, footer, buttons)
→ Extract a component library from the most reused patterns
→ Define color, typography, and spacing tokens
→ Apply a design system across all frames
```

---

## Error handling

| Situation | Action |
|---|---|
| Playwright MCP not available | Stop. Tell user to install it. |
| Figma Remote MCP not available | Stop. Tell user to connect it via /mcp. |
| Page returns 404 or redirect loop | Skip, log in final report |
| Capture still `pending` after 10 polls | Retry once with a new capture ID, then skip |
| Site requires login | Ask user if credentials or cookies can be provided |
| More than 20 pages | Ask user to confirm scope before proceeding |
| Figma rate limit | Wait 10s, reduce batch size to 3 |
