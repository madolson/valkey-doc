VERDICT: NEEDS-WORK
COUNTS: blocker=0 minor=9 nit=7

# Site structure and AI readiness (`site-structure`)

https://valkey.io/topics/

All 79 topics are crawlable, in the sitemap, and server-rendered, but the pages carry almost no metadata: no meta description, no canonical, no lastmod, no version or date. The most important fix is to pass the `description` that every valkey-doc topic already has through to `<meta name="description">`, so that snippets stop being drawn from whatever text comes first, which on 9 pages is the "likely incorrect" review banner.

## Findings

### F1 [minor] METADATA: No topic page has a meta description, though every source topic defines one (`valkey-io.github.io@5900f06:templates/includes/head.html:17`)
**Observed:** 0 of 79 live topic pages have `<meta name="description">`. All 79 valkey-doc topics have `description:` in front matter.
**Problem:** head.html only reads `page.description` from the Zola stub, and the generated stubs never set one. docs-page.html parses the valkey-doc front matter but only keeps `title`. Search and AI snippets fall back to the first text on the page.
**Evidence:**
```
$ grep -l '<meta name="description"' all/*.html | wc -l
0
head.html:17  {% if page and page.description %}<meta name="description" content="{{ page.description}}" />
head.html:18  {% elif section and section.description %}<meta name="description" content="{{ section.description}}" />{% endif %}
docs-page.html:13      {% if frontmatter_data.title %}
docs-page.html:14          {% set frontmatter_title = frontmatter_data.title %}
valkey-doc@a95bf25c:topics/latency.md:3  description: Finding the causes of slow responses
```
**Fix:** In `templates/docs-page.html` after line 15 add `{% if frontmatter_data.description %}{% set frontmatter_description = frontmatter_data.description | trim %}{% endif %}`. In `templates/includes/head.html` line 17, add a branch first: `{% if frontmatter_description is defined and frontmatter_description %}<meta name="description" content="{{ frontmatter_description }}" /><meta property="og:description" content="{{ frontmatter_description }}" />{% elif ... %}`. This uses the same mechanism as `frontmatter_title` on head.html:35.

### F2 [minor] OTHER: "Likely incorrect" banner opens 9 core pages, and config lists 5 topics that no longer exist (`valkey-io.github.io@5900f06:config.toml:37`)
**Observed:** /topics/faq/ (linked from every footer), /topics/latency/, /topics/hashes/, /topics/admin/, /topics/history/ (the target of /about/), distlock, eval-intro, functions-intro and lua-api all begin `<main>` with "This page is under review. The page is likely incorrect...". `review_list` and `publish_hold` still list internals, internals-vm, internals-eventlib and rdd, which are no longer in valkey-doc and return 404.
**Problem:** Because no page has a meta description (F1), this banner is the first text in the main content and the likeliest snippet. It tells readers and AI summarizers to distrust pages that may already be fine.
**Evidence:**
```
docs-page.html:45  {% if page.path in config.extra.review_list %}
docs-page.html:46  <p><strong>This page is under review.</strong> The page is likely incorrect, contains invalid links, ...
$ grep -c 'This page is under review' faq.html latency.html hashes.html history.html admin.html
1 1 1 1 1
$ curl -s -o /dev/null -w '%{http_code}' https://valkey.io/topics/internals/
404
```
**Fix:** Re-review the 9 pages and drop each one from `config.toml:37-52` once it passes. Delete the stale internals/rdd entries from `review_list` and `publish_hold` now. Until the reviews are finished, move the banner in `docs-page.html:45-49` after the content or into `<aside>` so it is not the leading text.
Tracked: valkey-io/valkey-doc#302

### F3 [minor] CRAWL: /topics/license/ is an indexable placeholder listed in the sitemap (`https://valkey.io/topics/license/`)
**Observed:** The page returns 200, is listed in sitemap.xml, and is linked from the /topics/ index. Its whole body is the review banner plus one line.
**Problem:** This is a soft 404: thin content that is indexed and that readers reach from the index.
**Evidence:**
```
<div class="main-inner">
<p><strong>This page is under review.</strong> The page is likely incorrect, ...</p>
<hr />
<p><code>/topics/license/</code> may be available after revisions.</p>
</div>
$ grep -c '/topics/license/<' sitemap.xml
1
```
**Fix:** Either publish it (remove it from `publish_hold`, `config.toml:55`), or skip `publish_hold` topics when generating stubs in `build/init-topics-and-clients.sh:42`. If the page has to stay, emit `<meta name="robots" content="noindex">` from head.html when `page.path in config.extra.publish_hold`.

### F4 [minor] DUPLICATE: No `rel="canonical"` on any page, and alias redirects return 200 HTML (`valkey-io.github.io@5900f06:templates/includes/head.html:1`)
**Observed:** None of the sampled pages has a canonical link: topics, /docs/, /clients/, and the alias pages. The aliases /docs/topics/<page>/, /about/ and /topics/encryption/ return `200` with a JS redirect plus a meta refresh inside `<noscript>`.
**Problem:** GitHub Pages cannot send real 301s for aliases, so the canonical link is the only explicit duplicate signal, and it is missing on both the targets and the aliases.
**Evidence:**
```
$ curl -s -w '%{http_code}' https://valkey.io/docs/topics/latency/
<!doctype html>
<meta charset="utf-8">
<title>Redirect</title>
<script>
  const target = "https://valkey.io/topics/latency/";
  ...
<noscript>
  <meta http-equiv="refresh" content="0; url=https://valkey.io/topics/latency/">
</noscript>
200
$ grep -c 'rel="canonical"' page_topics_latency_.html page_docs_.html page_clients_.html
0 0 0
```
**Fix:** Add `<link rel="canonical" href="{{ current_url | safe }}">` to `templates/includes/head.html` after line 30. Zola sets `current_url` on every page and section.

### F5 [minor] NAVIGATION: Most in-content links point at slashless URLs and cost a 301 hop (`valkey-io.github.io@5900f06:templates/macros/docs.html:34`)
**Observed:** Of 509 in-content `/topics/` and `/commands/` links across the 79 topic pages, 399 have no trailing slash. 143 of the 162 unique internal targets answer 301. Footer links `/about`, `/topics/faq`, `/connect`, `/leadership` and `/code_of_conduct` do the same (`templates/default.html:101-113`), and `/about` then chains into a JS alias redirect.
**Problem:** Every internal link spends crawl budget and reader latency on a redirect. The canonical URLs in the sitemap all end in `/`.
**Evidence:**
```
docs.html:34  regex_replace(pattern="\\]\\(\\.\\./commands/(?P<fname>\\w.*?)(.md)?\\)", rep="](/commands/$fname)")
docs.html:37  regex_replace(pattern="\\]\\(\\.\\./topics/(?P<fname>\\w.*?)(.md)?\\)", rep="](/topics/$fname)")
$ curl -s -o /dev/null -w '%{http_code} %{redirect_url}' https://valkey.io/topics/latency
301 https://valkey.io/topics/latency/
slash=110 noslash=399
```
**Fix:** In `templates/macros/docs.html` lines 33-46, end each `rep` with a slash: `](/commands/$fname/)`, `](/commands/$fname/#$hash)`, `](/topics/$fname/)`, `](/topics/$fname/#$hash)`. In `default.html:101-113`, change the footer hrefs to `/topics/history/`, `/topics/faq/`, `/connect/`, `/leadership/` and `/code_of_conduct/`.

### F6 [minor] SEMANTIC-HTML: Multiple `<h1>` on 11 topic pages and on /topics/ (`valkey-doc@a95bf25c:README.md:54`)
**Observed:** The template emits `<h1 class="page-title">Documentation: X</h1>`, and the markdown adds more H1s through `# ` or setext `===` headings. Pages affected: cli 3, indexing 10, sentinel-clients 8, search-data-formats 5, search-query 5, search-expressions 4, modules-intro, modules-native-types, search-configurables, search-observables and valkey-bundle 2 each. /topics/ has 2 (`Documentation by Topic` and `Valkey documentation` from index.md).
**Problem:** The heading outline is flattened, so section headings rank equal to the page title. This breaks README's own rule.
**Evidence:**
```
valkey-doc README.md:54-55  The title field is used instead of an
                            H1 heading in each of the pages.
valkey-doc topics/indexing.md:21-22  Simple numerical indexes with sorted sets / ===
valkey-doc topics/search-query.md:12  # Pure Vector Queries
live cli.html: <h1 class="page-title">Documentation: CLI  <h1 id="interactive-mode">Interactive mode  <h1 id="special-modes-of-operation">...
```
**Fix:** In valkey-doc, demote those headings one level and add a markdownlint MD025 (single H1, `front_matter_title: "^\\s*title\\s*[:=]"`) check to CI. Delete the `# Valkey documentation` line from `topics/index.md`.

### F7 [minor] JS: The topic-page index is loaded by JS from an HTML fragment URL that is in the sitemap (`valkey-io.github.io@5900f06:templates/docs-page.html:59`)
**Observed:** On every topic page, `<aside>` contains an empty `<ul id="topic-list"></ul>` that is filled by `fetch("/topics-list/")` on DOMContentLoaded. /topics-list/ is a bare list of `<li>` items with no `<head>`, and it is listed in sitemap.xml (as are /command-list/ and /commands/command-list/).
**Problem:** With no JS, crawlers and readers on a topic page get no cross-topic links. Every view costs an extra request. A headless fragment is offered for indexing.
**Evidence:**
```
docs-page.html:59  <h2 id="topic-list-title">Alphabetical Index</h2>
docs-page.html:60  <ul id="topic-list"></ul>
docs-page.html:66      f = fetch("/topics-list/");
$ grep topics-list sm.txt
https://valkey.io/topics-list/
```
**Fix:** Replace `docs-page.html:60-71` with `<ul id="topic-list">{% include "topics-list.html" %}</ul>`. topics-list.html already uses `get_section`, so it works as an include. Then set `render = false` in `content/topics-list/_index.md`, which also drops it from the sitemap. Tradeoff: the list is rebuilt 79 times per build (79×79 `load_data` calls), which costs build time. #707 restructures this sidebar and should be checked for whether it is still client-side.
Tracked: valkey-io/valkey-io.github.io#707 (related)

### F8 [minor] FRESHNESS: No version, last-updated date, or sitemap lastmod on any topic page, and the docs are unversioned (`valkey-io.github.io@5900f06:build/init-topics-and-clients.sh:50`)
**Observed:** Topic pages show no Valkey version and no date. No topic URL in sitemap.xml has `<lastmod>`, while blog URLs do. zola-deploy.yml checks out valkey-doc's default branch, so there is a single unversioned doc set. Only 15 of 88 topic files mention a version (for example hashes.md: "Valkey 9.0 we added ...").
**Problem:** A reader on 8.1 cannot tell whether a page describes their release. Crawlers get no change signal for the topic pages.
**Evidence:**
```
$ grep -A2 '/topics/latency/' sitemap.xml
        <loc>https://valkey.io/topics/latency/</loc>
    </url>
$ grep -A2 '/blog/1-billion-rps/' sitemap.xml
        <lastmod>2025-10-20T00:00:00Z</lastmod>
.github/workflows/zola-deploy.yml:30-31  repository: valkey-io/valkey-doc / path: valkey-doc
```
**Fix:** In the stub heredoc (`init-topics-and-clients.sh:50-54`), add `updated = "$(git -C "$1" log -1 --format=%cs -- "$base")"` and set `fetch-depth: 0` on the valkey-doc checkout in `zola-deploy.yml:27`. Zola's sitemap emits `<lastmod>` from `page.updated`. Render `Last updated {{ page.updated }}` in `docs-page.html` next to the edit box. Add one line to `topics/index.md` saying the docs track the latest release and that version-specific behavior is marked "Since Valkey X.Y".

### F9 [minor] CRAWL: Custom front matter drops the legacy alias, so /docs/topics/tls/ is 404 (`valkey-io.github.io@5900f06:build/init-topics-and-clients.sh:43`)
**Observed:** Topics with a custom-frontmatter file get only the aliases in that file. `tls.toml` lists `/topics/encryption/` but not `/docs/topics/tls/`. Every other topic keeps its `/docs/topics/<x>/` alias.
**Problem:** The legacy /docs/topics/tls/ URL, and any external links to it, now 404. The branch also appends with `>>`, so re-running the script locally duplicates the front matter.
**Evidence:**
```
init-topics-and-clients.sh:43  if [ -f "./build/custom-frontmatter/topics/$topic.toml" ]; then
init-topics-and-clients.sh:45      echo "+++" >> "./content/topics/$topic.md"
build/custom-frontmatter/topics/tls.toml:  aliases = [ "/topics/encryption/" ]
/docs/topics/tls/ 404
/docs/topics/encryption/ 404
/docs/topics/history/ 200
```
**Fix:** Add `"/docs/topics/tls/"` (and `"/docs/topics/encryption/"`, if that URL existed) to `build/custom-frontmatter/topics/tls.toml`. Change line 45 to `>` so the stub is truncated.

### F10 [nit] PAGE-EXPERIENCE: Render-blocking scripts in `<head>` on every doc page (`valkey-io.github.io@5900f06:templates/includes/head.html:61`)
**Observed:** `nav.js`, `carousel.js` (unused on doc pages) and Osano `osano.js` (250 KB, 1.18 s from this host) load synchronously before the body. /clients/ also loads `js.hsforms.net` embed v2.
**Problem:** These delay first render of the doc content. The HTML itself is small: latency 14.9 KB gzipped, modules-api-ref 92 KB gzipped.
**Evidence:**
```
head.html:61  <script src="/assets/js/nav.js"></script>
head.html:64  <script src="/assets/js/carousel.js"></script>
head.html:67  <script src="https://cmp.osano.com/.../osano.js?variant=one"></script>
osano 250532 1.181701
```
**Fix:** Add `defer` to nav.js and carousel.js, and include carousel.js only on templates that use a carousel. Osano is the LF standard and is normally placed first so it can block cookies. Leave it alone unless LF allows async.

### F11 [nit] METADATA: Open Graph and Twitter tags are incomplete, and the docs index title is generic (`valkey-io.github.io@5900f06:templates/includes/head.html:9`)
**Observed:** Only `og:image` and `og:title` are present. There is no `og:description`, `og:url`, `og:type`, or `twitter:card`. The /topics/ title is `Valkey · Topics`, while its H1 is "Documentation by Topic".
**Evidence:**
```
<meta property="og:image" content="https://valkey.io/img/valkey-logo-og.png" /> <meta name="title" property="og:title" content="Valkey: Topics">
twitter: 0
```
**Fix:** In head.html after line 55, add `og:url` (`current_url`), `og:type` (`article` for pages), `og:description` (see F1) and `<meta name="twitter:card" content="summary">`. Set `title = "Documentation by Topic"` in `content/topics/_index.md`.

### F12 [nit] SEMANTIC-HTML: 88% of code blocks carry no language (`valkey-doc@a95bf25c:topics/`)
**Observed:** 1111 of 1268 `<pre>` blocks render as `data-lang="plain"`. The source has 309 bare fences plus many indented blocks (149 in latency.md alone).
**Evidence:**
```
   1111 <pre class="giallo" ...><code data-lang="plain">
     77 ... data-lang="c"
     47 ... data-lang="shellscript"
```
**Fix:** Add a valkey-doc style rule and a markdownlint MD040 (fenced-code-language) check. Tag valkey-cli sessions with something like ` ```console ` and config with ` ```ini `.

### F13 [nit] PAGE-EXPERIENCE: Topic images have no width or height, and one is 435 KB (`https://valkey.io/topics/lru-cache/`)
**Observed:** Images are emitted as bare markdown `<img src alt>`. Alt text is present but terse.
**Evidence:**
```
lru-cache.html:<img src="/topics/lru_comparison.png" alt="LRU comparison" />
lru_comparison.png 435399
```
**Fix:** Recompress lru_comparison.png in valkey-doc (or convert it to WebP). To prevent layout shift, emit width and height, either with a Zola image shortcode or by post-processing in `fix_links_docs`.

### F14 [nit] AI-ACCESS: The source link needs a login, and there is no read-only source link (`valkey-io.github.io@5900f06:templates/docs-page.html:75`)
**Observed:** The only link to the source is `github.com/valkey-io/valkey-doc/edit/main/topics/<slug>.md`, which sends anonymous clients to a login page. The raw file is public.
**Evidence:**
```
$ curl -s -o /dev/null -w '%{http_code} %{redirect_url}' https://github.com/valkey-io/valkey-doc/edit/main/topics/latency.md
302 https://github.com/login?return_to=...
$ curl -s -o /dev/null -w '%{http_code} %{content_type}' https://raw.githubusercontent.com/valkey-io/valkey-doc/main/topics/latency.md
200 text/plain; charset=utf-8
```
**Fix:** In `docs-page.html:75`, add a "View source" link to `https://github.com/valkey-io/valkey-doc/blob/main/topics/{{ page.slug }}.md`. An `llms.txt` or markdown alternate links would help non-Google agents only. Google says they neither help nor harm Search, so treat them as optional.

### F15 [nit] NAVIGATION: Breadcrumbs skip the Documentation level and have no structured data, and there are no previous/next links (`valkey-io.github.io@5900f06:templates/docs-page.html:31`)
**Observed:** The breadcrumb is `Topics > <title>`, with no link to /docs/ and no `BreadcrumbList` JSON-LD. Topic pages have no previous/next links.
**Fix:** Add a `/docs/` crumb before line 33 and a `BreadcrumbList` JSON-LD block. Add previous/next and section crumbs once the nav.yml menu lands.
Tracked: valkey-io/valkey-doc#513

### F16 [nit] OTHER: Uppercase slugs are case-sensitive (`https://valkey.io/topics/ARM/`)
**Observed:** `/topics/ARM/` returns 200 and `/topics/arm/` returns 404. The same applies to RDMA.
**Fix:** Add lowercase aliases through `build/custom-frontmatter/topics/ARM.toml` and `RDMA.toml`. Include `/docs/topics/ARM/` (see F9).

## Unverified
- Duplicate content between topic and command pages: I spot-checked titles only and did not compare bodies. No duplicate `<title>` exists across the 79 topic pages.
- Whether Zola 0.23 lets the built-in alias template be overridden to add `rel="canonical"` to alias pages: no `templates/internal/alias.html` exists in the repo, and I did not check Zola's source.
- Actual Google snippets for the review-banner pages (F2): no Search Console access.
- Lighthouse/LCP numbers: no browser run. F10 is based on script size and placement only.
