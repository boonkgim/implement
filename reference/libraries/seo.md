# seo

Assumes a web app's crawlable pages — anything a search engine or a shared-link preview
reads. Framework-illustrated with Next.js-style routing and `curl`/grep verification where
useful; not framework-required — the underlying check (does the byte stream a crawler
receives contain what this rule requires) applies to any server-rendered or SSG'd page.

## Rule 1: A rendered `<a>` tag is a link; a data reference is not

Internal-linking correctness is judged by the response HTML, not the data layer feeding it.
A URL sitting in a registry, a `pillarLinks`/`crossLinks` array, or a CMS field is a value,
not a link — it becomes a link only once something maps it to a rendered `<a href="...">` in
the DOM. Code review that stops at "the array includes the spoke URL" misses the case where
the array is imported but never mapped to JSX: the page compiles clean, the registry looks
complete, and the crawler sees nothing. Verify by reading the response, not the source file.

```
DON'T: grep -q '"/pricing/enterprise"' data/spokes.ts   # proves the string exists, not that it renders
DO:    curl -s https://site/hub-page | grep -c '<a href="/pricing/enterprise"'   # must be ≥1
```

## Rule 2: Every hub page renders its full spoke index, not the top N

A hub/pillar page ships a visible, rendered section — "Browse all X," "Explore Y" — that
loops over every spoke in its category and renders each as an `<a>` with descriptive anchor
text. Not the top 6 with the rest implied, and not a section that only appears as sibling nav
on the spoke pages themselves. A hub that looks finished because it links its 6 highest-value
spokes still leaves the other 34 reachable only by guessing the URL — which a crawler doesn't
do.

## Rule 3: Two inbound link sources per page, siblings count as one

Every indexed page needs at least two rendered inbound links from at least two distinct
source types: hub/pillar page, footer, header/mega menu, homepage, or a cross-link section in
a page template. Pages that link to each other only within the same registry or category
(sibling cross-links) count as **one** source no matter how many siblings do it. The footer
carries every hub/pillar page plus the highest-volume spokes; a site with multiple page
categories (products, solutions, industries, or similar) gets a second sitewide channel via a
header mega menu, rendered to the same `<a>`-tag standard as the footer — task-oriented
navigation, not a replacement for it. A page that only has sibling links looks connected in
the registry and is one edit away from orphaned, because it never picked up a link from any
page that actually drives traffic.

## Rule 4: Batch large rollouts, verify rendering before the next batch

Creating more than roughly 20 pages of the same kind in one slice gets split into batches by
category (all industry pages, then all channel pages, ...), each its own phase. A batch isn't
done when its pages exist — it's done when the hub→spoke links, footer entries, and inbound
link count for that batch are verified rendered. Don't start the next batch on an unverified
one. This is the concrete mechanism behind "113 pages created, 69% weakly linked": content
generation outpaces its own link wiring, and nobody re-checks batch 1 by the time batch 5
ships.

```
DO:    curl -s https://site/hub-page | grep '<a href="/spoke-'  # for every spoke in this batch, before batch N+1 starts
```

## Rule 5: Nothing important sits more than 2-3 clicks deep

Important pages are reachable from the homepage within 3 clicks; spokes are reachable within
2 given the footer → hub → spoke path Rules 2-3 require. Nothing meant to be indexed sits 4+
levels deep with no shortcut in. Crawl depth is a proxy for crawl priority — a page a crawler
has to work to reach gets crawled less often and ranks accordingly.

## Rule 6: robots.txt and the sitemap are both correct and stay in sync

`robots.txt` returns 200, is valid syntax, never blocks CSS/JS/image resources, declares at
least one `Sitemap:` directive, and carries no blanket `Disallow: /` for real user agents. The
sitemap returns 200, is valid XML, stays under 50,000 URLs / 50MB per file, and lists only
canonical 200 URLs — no redirecting URLs, no duplicates, no `noindex`ed pages. Every URL
carries a `lastmod` that reflects an actual content change, not the sitemap's build time. A
sitemap a crawler catches lying — dead links, phantom `lastmod` bumps — gets discounted as a
signal wholesale, not just on the bad rows.

## Rule 7: Every page self-declares one canonical, and it resolves

`<link rel="canonical">` on every page, an absolute URL, either self-referencing or pointing
at the one preferred URL for that content — and that URL returns 200, never a redirect, never
a 404. www/non-www, http/https, trailing-slash, and tracking-param variants of a URL all 301
to the single canonical version. Leave canonicalization undeclared and a search engine picks
one for you; it isn't always the one you'd have chosen.

## Rule 8: Redirects resolve in one hop, indexed URLs stay clean

Max one redirect hop — never A→B→C — no circular redirects, and 301 (not 302/307) for
anything permanent. Indexed URLs are lowercase, hyphenated, and free of session IDs or
tracking parameters, with consistent trailing-slash usage across the whole site. Chained or
parameterized URLs burn crawl budget on what's effectively the same page under N addresses
instead of on new content.

## Rule 9: Titles, descriptions, and headings are unique per page

Every page carries one unique `<title>` (50-60 characters, primary keyword present) and one
unique `<meta description>` (150-160 characters, a reason to click — not a keyword list).
Exactly one `<h1>` per page, and a heading order that doesn't skip levels (h2 straight to h4
is a broken outline). Duplicate or missing titles/descriptions collapse two distinct pages
into one entry in search results; a skipped heading level breaks the document structure both
assistive tech and crawlers rely on to understand the page.

## Rule 10: Alt text and anchor text name the destination

Image `alt` text describes what's in the image (empty `alt=""` only for genuinely decorative
images) — never the filename, never the word "image." Internal link anchor text names what's
on the other end of the link — never "click here," "read more," or an empty anchor. Every
image also carries explicit `width`/`height`. "Click here" repeated as anchor text across a
site gives a crawler nothing to tell the linked pages apart by relevance.

## Rule 11: Structured data ships in the initial HTML and matches the page

JSON-LD only (not Microdata/RDFa), inside a `<script type="application/ld+json">` tag present
in the server-rendered response — never injected by client-side JS after load. Use the type
the page actually is: Organization + WebSite on the homepage, `BreadcrumbList` wherever
breadcrumbs render, `Article`/`BlogPosting` on posts, `Product` on product pages, `FAQPage` on
FAQ sections — with that type's required properties filled in. Structured data never describes
content that isn't visibly on the page. Schema that oversells the page (ratings nobody
collected, fields that don't correspond to visible content) gets the whole block ignored or
the page flagged; schema injected after hydration is invisible to a crawler that only reads
the first response.

## Rule 12: Open Graph and Twitter Card tags are complete and match canonical

`og:title`, `og:description`, `og:image` (≥1200×630), `og:url`, `og:type` on every shareable
page, with `og:url` matching the canonical — not a tracking or session variant of it.
`twitter:card` set appropriately (`summary_large_image` for visual pages) with matching
title/description/image. A shared link with a missing image or a stale title is a
click-through loss nobody notices until someone actually posts it.

## Rule 13: Meta essentials live in the server response, not a client-side effect

`<title>`, `<meta description>`, `<link canonical>`, `<meta robots>`, charset (first element
in `<head>`), `<html lang>`, and a non-scale-locked viewport tag are all present in the raw
server response — check with "view source" or `curl`, not the browser's rendered DOM.
Favicon is referenced at multiple sizes and returns 200.

```
DON'T: useEffect(() => { document.title = product.name }, [product])   # crawler's first read sees the fallback title
DO:    export const metadata = { title: product.name }                 // or equivalent server-rendered head
```

## Rule 14: No page meant to be indexed carries an unintended `noindex`

Check both `<meta name="robots" content="noindex">` in the response HTML and the
`X-Robots-Tag` response header — either one alone is enough to pull a page out of search
results, and neither shows up in the rendered page a reviewer eyeballs in a browser. A staging
config or middleware default that sets `X-Robots-Tag: noindex` sitewide and never gets
stripped for the production build is a common accidental source: the page looks and renders
fine, and it's still invisible to a crawler. A page carrying either directive must never appear
in the sitemap (Rule 6) — if one says indexed and the other says noindex, one of them is wrong.

## Rule 15: Ship the patterns that make Core Web Vitals pass

Hero image and critical fonts preloaded; the LCP element is present in the initial HTML with
no JS dependency. Below-fold images use `loading="lazy"`; above-fold images never do.
Third-party and non-critical scripts load `async`/`defer`; nothing render-blocking sits in
`<head>`. Images ship in WebP/AVIF with `srcset`, and every image has explicit
`width`/`height` (this also bounds CLS, along with `font-display: swap` on web fonts). Static
assets are gzip/brotli compressed, with `preconnect`/`dns-prefetch` on critical third-party
origins. Server TTFB (time to first byte) stays under 800ms, with 200ms as the ideal — LCP
can't start until TTFB finishes, so a slow origin response puts every other rule in this list
behind before the browser has painted a single pixel. A code review can't measure
milliseconds, but it can catch the missing `loading="lazy"`, the unpreloaded hero image, and
the render-blocking script that would have caused the LCP/INP/CLS regression at runtime.

## Rule 16: Security response headers ship alongside the SEO surface

HTTPS everywhere, HTTP→HTTPS via 301, no mixed content. HSTS with `max-age` of at least one
year. `X-Content-Type-Options: nosniff`, `X-Frame-Options`, `Referrer-Policy`, and a CSP (even
a basic one) all present. Browsers and search engines both treat a header-bare or
partially-insecure origin as lower-trust — this is table stakes, not a follow-up task.

## Rule 17: A navigation change ships in the same phase as the page it points to

Before a footer, header/nav, mega menu, breadcrumb, or cross-link array references an internal
path, confirm the route file exists — glob for it, don't assume it "will exist by the time
this ships." A navigation change and the page(s) it targets land in the same phase/slice; if
the pages come later, the link waits for them. This is what manufactures a 404 at authoring
time instead of crawl time: a footer link added in phase 2 for a page that lands in phase 4 is
a broken link for every phase in between, and it's easy to miss because the code that's wrong
today is the (reviewed, working) footer — the missing page is scheduled reading for later.

```
DON'T: add href="/solutions/enterprise" to footer.tsx in this slice, page ships two slices later
DO:    Glob app/**/solutions/enterprise/page.{tsx,jsx}; if no match, either create the page in this
       slice or leave the link out until the slice that does
```

## Not this

- A registry/array entry doing duty for a rendered `<a>` tag — grep the source, not the response.
- A hub page's spoke section that only shows the top N, or exists only as sibling nav on spokes.
- A footer or mega menu link added before its target page's route file exists.
- Structured data injected client-side, or describing content that isn't on the page.
- Title or meta description set in a `useEffect` instead of the server-rendered response.
- Batch N+1 of a large page rollout starting before batch N's links are verified rendered.
