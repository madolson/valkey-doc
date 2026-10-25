VERDICT: MINOR-ISSUES
COUNTS: blocker=0 minor=3 nit=5

# Valkey Documentation (`index`)

https://valkey.io/topics/

All 60 relative links resolve and the live page matches a95bf25c. The main gap is coverage: the curated list leaves out the module docs (JSON, Search, Bloom, LDAP: 11 pages), and readers can only reach them through "Valkey Bundle" or the alphabetical sidebar.

## Findings

### F1 [minor] MISSING: Module docs (JSON, Search, Bloom, LDAP) absent from the curated list (`topics/index.md:15`)
**Doc says:**
> * [Valkey Bundle](valkey-bundle.md): Get started with Valkey Bundle.
**Problem:** 22 of the 87 topic .md files are not linked from index.md. The data-type pages (strings, lists, hashes, sets, sorted-sets, streams-intro, bitmaps, bitfields, geospatial, hyperloglogs) are reachable through data-types.md, so leaving them out is fine. The module pages are only linked from valkey-bundle.md, and nothing in "Get started with Valkey Bundle" tells a reader that the JSON, Search, or Bloom reference lives there. A reader who scans the curated list for "JSON" or "search" finds nothing.
**Evidence:** Files in topics/ with no link from index.md: `bloomfilters.md ldap.md search.md search-configurables.md search-data-formats.md search-expressions.md search-monitoring.md search-observables.md search-query.md valkey-json.md history.md` (the data-type pages are omitted from this list on purpose). valkey-bundle.md is the only page that links bloomfilters.md, ldap.md, search.md, and valkey-json.md.
**Fix:** Add a subsection under "3. Extending Valkey":
```
### Valkey Bundle modules
* [Valkey JSON](valkey-json.md): Store, query, and modify JSON documents.
* [Valkey Search](search.md): Vector, full-text, and secondary-index search. See also [Queries](search-query.md), [Expressions](search-expressions.md), [Data formats](search-data-formats.md), [Configuration](search-configurables.md), [Monitoring](search-monitoring.md), [Observables](search-observables.md).
* [Bloom filters](bloomfilters.md): Probabilistic membership tests with the valkey-bloom module.
* [LDAP authentication](ldap.md): Authenticate users against LDAP or Active Directory.
```
Tracked: valkey-io/valkey-doc#303 (partially), valkey-io/valkey-doc#513 (nav.yml would supersede)

### F2 [minor] CONFUSING: "Valkey as an LRU cache" mislabels the eviction page (`topics/index.md:31`)
**Doc says:**
> * [Valkey as an LRU cache](lru-cache.md): How to configure Valkey as a cache with a fixed amount of memory and automatic eviction of keys.
**Problem:** The target page is titled "Key eviction" and covers all eviction policies, LFU included. A reader looking for LFU or `maxmemory-policy` will skip a link that says LRU, and the sidebar shows the same page as "Key eviction", so the page appears under two names.
**Evidence:** topics/lru-cache.md:2-3: `title: Key eviction` / `description: Overview of Valkey key eviction policies (LRU, LFU, etc.)`
**Fix:** `* [Key eviction](lru-cache.md): How to use Valkey as a cache with a fixed memory limit, and how the LRU, LFU, and other eviction policies choose keys to remove.`

### F3 [minor] AI-NATIVE: Opening paragraph is about repo hosting and license, not what the page is for (`topics/index.md:5`)
**Doc says:**
> The Valkey documentation is managed in markdown files in the [valkey-doc repository](...). It's released under the [Creative Commons Attribution-ShareAlike 4.0 International license](...).
**Problem:** This is the first text under the "Documentation by Topic" banner. It tells readers how the docs are maintained, not what the page offers or where to start. Contributor and license information belongs at the bottom.
**Evidence:** The live page renders this paragraph straight after `<h1>Documentation by Topic`.
**Fix:** Move the paragraph to the end of the file. Open with: "Guides and reference for using, operating, and extending Valkey, grouped by task. New to Valkey? Start with the [Quick start](quickstart.md). For individual commands, see the [command reference](../commands/)."

### F4 [nit] STYLE: Two H1s on the rendered page (`topics/index.md:10`)
**Doc says:**
> # Valkey Documentation
**Problem:** templates/docs.html already outputs `<h1>Documentation by Topic</h1>`, so the live page has two H1s back to back, and the second one also repeats the front matter title.
**Evidence:** Live page headings: `<h1>Documentation by Topic` then `<h1 id="valkey-documentation">Valkey Documentation`.
**Fix:** Delete line 10.

### F5 [nit] STYLE: Typo "scan" (`topics/index.md:49`)
**Doc says:**
> Modules scan implement new data types
**Fix:** "Modules can implement new data types".

### F6 [nit] BROKEN: "Expires" links to the EXPIRE command, not the expiration topic (`topics/index.md:24`)
**Doc says:**
> * [Expires](../commands/expire.md): How to set a Time To Live (TTL) on key ...
**Problem:** Every other entry in "Core Concepts" links to a topic page. The concept is explained in keyspace.md, and the command page covers only one of the TTL commands. "on key" is also missing an article.
**Evidence:** topics/keyspace.md:65: `## Key expiration`.
**Fix:** `* [Key expiration](keyspace.md#key-expiration): How to set a time to live (TTL) on a key so the server removes it automatically when it expires.`

### F7 [nit] CONFUSING: RDMA entry omits that the feature is experimental (`topics/index.md:84`)
**Doc says:**
> * [RDMA](RDMA.md): An overview of RDMA support.
**Evidence:** topics/RDMA.md:2: `title: "RDMA experimental support"`. topics/RDMA.md:255: `Valkey Over RDMA is experimental, it may be changed or be removed in any minor or major version.`
**Fix:** `* [RDMA](RDMA.md): Experimental support for Valkey over RDMA.`

### F8 [nit] MISSING: History page not linked (`topics/index.md:13`)
**Problem:** history.md ("How the Valkey project started") is not linked from the curated list. Only faq.md links to it.
**Fix:** Add `* [History](history.md): How the Valkey project started.` under "1. Introduction".
Tracked: valkey-io/valkey-doc#303
