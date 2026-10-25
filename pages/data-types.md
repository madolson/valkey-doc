VERDICT: MINOR-ISSUES
COUNTS: blocker=0 minor=2 nit=5

# Data types (`data-types`)

https://valkey.io/topics/data-types/

Every link works and nothing on the page is wrong, but the page lists only Bloom and leaves out the JSON and Search modules that have doc pages and command groups. Fix that first.

## Findings

### F1 [minor] MISSING: JSON and Search are not listed (`topics/data-types.md:95`)
**Doc says:**
> ## Bloom Filter
> ...
> The Bloom filter data type / command support is provided by the `valkey-bloom` module.

**Problem:** Bloom is the only module data type on the page. JSON has its own concept page and command group. Search has its own pages and command group too. A reader who uses this page as the type index will not find either one, and will not learn that module types need the module loaded (or valkey-bundle).
**Evidence:** `topics/valkey-json.md:6`: "the JSON data type and commands are implemented in the [valkey-json](https://github.com/valkey-io/valkey-json) module". The topics/ directory has `search.md` and `search-data-formats.md`. The live https://valkey.io/commands/ page has `id="json"` and `id="search"` anchors next to `id="bloom"`.
**Fix:** Add a section after Bloom Filter:
"## JSON
[JSON](valkey-json.md) stores JSON documents natively and lets you query and modify them with JSONPath. The `valkey-json` module provides it.
* [Overview of JSON](valkey-json.md)
* [JSON command reference](../commands/#json)"
After that, add one line pointing to [Search](search.md) for secondary and vector indexes over hashes and JSON (`../commands/#search`). Also add one sentence in the intro saying that Bloom, JSON, and Search come from modules, which you must load or get through valkey-bundle.
Tracked: valkey-io/valkey-doc#299 (JSON only)

### F2 [minor] AI-NATIVE: Page has no Valkey-specific facts, only one-line generic descriptions (`topics/data-types.md:12`)
**Doc says:**
> [Strings](strings.md) are the most basic Valkey data type, representing a sequence of bytes.

**Problem:** Every section is one generic sentence plus two links. A reader comparing types gets none of the facts they would quote from this page: size limits, ordering guarantees, typical complexity, or features such as hash field expiration (added in 9.0.0). Most of these facts already exist on the linked pages.
**Evidence:** `topics/strings.md:96`: "By default, a single String can be a maximum of 512 MB." `topics/hashes.md:85` has a "Field expiration" section. `valkey@9.1.2:src/commands/hexpire.json`: `"since": "9.0.0"`.
**Fix:** Add a summary table under the intro with columns Type, What it stores, Key property or limit, Overview, and Commands. Examples: "String | bytes | max 512 MB (`proto-max-bulk-len`)", "Hash | field-value pairs | per-field TTL since 9.0 (`HEXPIRE`)". Keep the per-type sections.

### F3 [nit] CONFUSING: "caching" links to client-side caching (`topics/data-types.md:7`)
**Doc says:**
> from [caching](client-side-caching.md) to [queuing](lists.md)

**Problem:** The sentence is about using Valkey data types for caching. The link goes to the client-side caching (tracking/invalidation) protocol page, which is a different topic.
**Fix:** Unlink "caching", or link it to `lru-cache.md` (using Valkey as a cache with eviction).

### F4 [nit] AI-NATIVE: Redundant intro sentence and duplicated links in each section (`topics/data-types.md:10`)
**Doc says:**
> If you'd like to try a comprehensive tutorial for each data structure, see their overview pages below.

**Problem:** Line 10 repeats line 8 ("links to broader overviews"). Every section also links its overview page twice: inline in the first sentence and again as the first bullet (lines 14/17, 22/25, 30/34, 39/43, 48/51, 56/60, 65/68, 73/76, 81/85, 90/92, 97/102).
**Fix:** Delete line 10. Remove the inline link from each section's first sentence, or remove the "Overview of X" bullet. Pick one and apply it everywhere.

### F5 [nit] STYLE: Bitfields section has no command link, unlike every other section (`topics/data-types.md:86`)
**Doc says:**
> * The `BITFIELD` command.

**Problem:** Every other section links to its command reference. This one has plain code text with no link.
**Fix:** `* [`BITFIELD`](../commands/bitfield.md) and [`BITFIELD_RO`](../commands/bitfield_ro.md) commands`

### F6 [nit] STYLE: Wording and heading problems (`topics/data-types.md:97`)
**Doc says:**
> allows adding elements and checking if item/s are definitely not present
> The Bloom filter data type / command support is provided by the `valkey-bloom` module.
> ## Extensions

**Problem:** "item/s" and "data type / command" read like notes, not prose. The capitalization mixes "Bloom Filter" (heading) and "Bloom filters" (body), and "Sorted sets" (heading) and "Sorted Sets" (body). The "Extensions" heading does not say what the section answers.
**Fix:** "...lets you add items and check whether an item is definitely absent or possibly present, at a configured false positive rate." Then: "The `valkey-bloom` module provides the Bloom filter data type and commands." Use sentence case in headings and body text throughout ("Bloom filters", "Sorted sets"). Rename "Extensions" to "Extending Valkey with scripts and modules".

### F7 [nit] OUTDATED: Stale versions in external reference docs (`topics/data-types.md:30`)
**Doc says:**
> [Java HashSets](https://docs.oracle.com/javase/7/docs/api/java/util/HashSet.html), [Python sets](https://docs.python.org/3.10/library/stdtypes.html#set-types-set-frozenset)

**Problem:** The links are pinned to Java 7, Python 3.10, and Ruby 3.1.2 (line 40). They resolve (HTTP 200), but they point at end-of-life versions, and the page mixes Java 7 with Java 8 (line 40).
**Fix:** Use version-agnostic URLs: `https://docs.python.org/3/library/stdtypes.html#set-types-set-frozenset`, `https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/HashSet.html` (and the matching HashMap URL), and `https://ruby-doc.org/3.3/Hash.html`.
