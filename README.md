# RSS → EPUB

A single-page, client-side tool that turns any RSS or Atom feed into EPUB files for reading on an e-ink device. Paste a feed URL, pick your options, and download either **one combined EPUB** of all articles or **one EPUB per article** (zipped). Everything runs in the browser — the only moving part is a tiny Cloudflare Worker that proxies the network requests.

## Features

- **Full-text articles**, not just feed excerpts — fetches each article page and extracts the readable content with Mozilla's Readability.
- **Combined or individual** output: one book with every article as a chapter, or a zip of separate EPUBs.
- **Images included and cleaned up** — embeds article images directly in the EPUB, filters out ads/trackers and tracking pixels, handles lazy-loaded images, and converts WebP/AVIF to JPEG for broad e-reader compatibility. Can be turned off for fast, text-only books.
- **Cleaner articles** — strips common end-of-article clutter: "Add as a preferred source on Google" / Google News promos, FTC affiliate disclosures, and author social-media follow lines (in-article product links are left intact).
- **Cover + table of contents** in combined mode: a cover image (feed logo, or the first article's image, with a text title page as fallback), a title/metadata page, and a clickable Contents page, plus the reader's built-in navigation.
- **Byline + source link** on each article — author and date when the feed provides them, plus a "Read original" link back to the source page.
- **Fetch controls**: cap the number of articles, limit to the last _N_ days, and skip articles you've already downloaded (remembered per feed in your browser).
- **Saved feeds**: keep a dropdown of your regular feeds, remembered in the browser, so you never retype a URL.

## How it works

```
feed URL ─▶ fetch (via Worker) ─▶ parse RSS/Atom items
                                       │
                 for each item ─▶ fetch full page ─▶ Readability extract
                                       │                    │
                                       ▼                    ▼
                                 embed + clean images   valid XHTML
                                       │
                 ┌─────────────────────┴─────────────────────┐
            one combined EPUB                         one EPUB per article
            (cover + TOC + chapters)                  (zipped)
                 └─────────────────────┬─────────────────────┘
                                       ▼
                                   download
```

Libraries ([JSZip](https://stuk.github.io/jszip/) and [@mozilla/readability](https://github.com/mozilla/readability)) load from a CDN at runtime, so there's no build step — `index.html` is the entire app.

## Setup

### 1. Host the page (GitHub Pages)

1. Add `index.html` to a repository.
2. **Settings → Pages → Source: Deploy from a branch → `main` / root.**
3. Open the Pages URL GitHub gives you.

### 2. Deploy the proxy (Cloudflare Worker)

A proxy is required because browsers block cross-origin requests, so the page can't fetch feeds or article pages directly. A personal Cloudflare Worker handles this (free tier is 100,000 requests/day — far more than you'll need).

1. **dash.cloudflare.com → Workers & Pages → Create → Create Worker.**
2. Name it (e.g. `rss-proxy`), **Deploy**, then **Edit code** and replace the default with:

   ```js
   export default {
     async fetch(request) {
       const CORS = {
         "Access-Control-Allow-Origin": "*", // or your Pages origin, e.g. https://you.github.io
         "Access-Control-Allow-Methods": "GET, OPTIONS",
         "Access-Control-Allow-Headers": "*",
       };
       if (request.method === "OPTIONS") return new Response(null, { headers: CORS });

       const target = new URL(request.url).searchParams.get("url");
       if (!target) return new Response("Missing ?url=", { status: 400, headers: CORS });

       let dest;
       try { dest = new URL(target); }
       catch { return new Response("Bad URL", { status: 400, headers: CORS }); }
       if (dest.protocol !== "http:" && dest.protocol !== "https:")
         return new Response("Only http/https allowed", { status: 400, headers: CORS });

       try {
         const upstream = await fetch(dest.toString(), {
           headers: {
             "User-Agent": "Mozilla/5.0 (compatible; RSS2EPUB/1.0; personal reader)",
             "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8",
           },
           redirect: "follow",
         });
         const body = await upstream.arrayBuffer();
         const headers = new Headers(CORS);
         headers.set("Content-Type", upstream.headers.get("Content-Type") || "text/plain; charset=utf-8");
         return new Response(body, { status: upstream.status, headers });
       } catch (e) {
         return new Response("Upstream fetch failed: " + e.message, { status: 502, headers: CORS });
       }
     }
   };
   ```

3. **Deploy** and copy the Worker URL (e.g. `https://rss-proxy.your-subdomain.workers.dev`).

### 3. Point the page at your Worker

In `index.html`, set the proxy line near the top of the `<script>` to your Worker URL, keeping `/?url=` on the end:

```js
const PROXY = "https://rss-proxy.your-subdomain.workers.dev/?url=";
```

Commit, and you're done.

## Usage

1. Paste a feed URL (e.g. `https://example.com/feed.xml`).
2. Set **Max articles** and, optionally, a **last _N_ days** window.
3. Choose **combined** or **one per article**, and toggle **Include images**.
4. Click **Generate**. The EPUB (or zip) downloads to your machine; copy it to your e-reader over USB.

**Skip already-downloaded** remembers article IDs per feed in `localStorage`, so repeat runs only grab new pieces. Use **Reset history** to clear that memory for the feed currently in the box.

**Saved feeds** — click **Save** to add the current URL to the **Saved feeds…** dropdown (labeled by site name); pick one anytime to fill the box, or **Remove** to delete the selected feed. The list is stored in your browser.

## Transferring to your device

EPUB is the standard e-ink format, so just copy the downloaded file to your reader like any other book (USB, or whatever method your device uses).

## Limitations

- **Extraction quality varies by site.** Most feeds come out clean; some will have stray navigation text or an imperfect image or two.
- **Paywalled articles** only yield what a logged-out visitor would see.
- **Image-heavy runs are slower**, since each image is a separate request through the Worker.
- **The ad filter is good, not perfect** — a site serving ad images from its own domain with innocuous names could occasionally slip one through.
- **Dedupe and saved feeds are per-browser** — running on a phone and a laptop keeps separate histories and feed lists.

## Hardening (optional)

By default the Worker allows any origin (`Access-Control-Allow-Origin: "*"`). To lock it to only your own site, set that header to your Pages origin instead, e.g. `https://your-username.github.io` (no trailing slash).

## Security & privacy

Runs entirely in your browser; no server stores anything. Feed and article requests pass through your own Cloudflare Worker, and downloaded EPUBs never leave your machine.

## License

Released under the [MIT License](LICENSE).
