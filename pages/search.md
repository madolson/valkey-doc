VERDICT: NEEDS-WORK
COUNTS: blocker=1 minor=5 nit=3

# Valkey Search - Overview (`search`)

https://valkey.io/topics/search/

The whole page is published twice, front matter included. The Query Consistency section also gets the timeout default backwards: by default a timed-out query returns partial results, not an error. Verified against valkey-search main at c672fd7 (1.2.0-rc1-297, after the 1.3.0-rc1 tag) and the 1.2.1 tag.

## Findings

### F1 [blocker] INACCURATE: Query timeout default is partial results, not an error (`topics/search.md:104`)
**Doc says:**
> If a timeout occurs then by default an error is returned. The `SOMESHARDS` option of the `FT.SEARCH` command can be used to override this behavior to allow a result to be generated if only a subset of the cross-shard query operations succeed. The `INCONSISTENT` option of `FT.SEARCH` can be used to allow results from nodes with different views of the cluster.
**Problem:** The defaults are `SOMESHARDS` and `INCONSISTENT`, in both 1.2.1 and main. A reader who trusts this sentence assumes an incomplete query fails. It does not: it silently returns a partial, best-effort result, and they will never add `ALLSHARDS`/`CONSISTENT`. The same text appears again at line 216.
**Evidence:** valkey-search@c672fd7:src/valkey_search_options.cc:268-277 (identical at 1.2.1:217-226):
```
constexpr absl::string_view kEnablePartialResults{"enable-partial-results"};
static config::Boolean prefer_partial_results(kEnablePartialResults, true);
...
static config::Boolean prefer_consistent_results(kEnableConsistentResults,
                                                 false);
```
commands/ft.search.md:26 agrees: "This is the default when `search.enable-partial-results` is `no`; otherwise `SOMESHARDS` is the default." topics/search-configurables.md:22-23 lists the defaults as `yes` and `no`.
**Fix:** Replace with: "If a timeout occurs, the default (`search.enable-partial-results yes`) returns a best-effort result built from the shards that responded. Specify `ALLSHARDS` on `FT.SEARCH`, or set `search.enable-partial-results no`, to return an error instead. Likewise, `INCONSISTENT` is the default (`search.enable-consistent-results no`). Specify `CONSISTENT` to reject results from nodes with a different view of the cluster."

### F2 [minor] BROKEN: Entire page body duplicated, rendered twice on the live site (`topics/search.md:113`)
**Doc says:**
> ---
> title: "Valkey Search - Overview"
> description: Valkey Search Module Overview
> ---
**Problem:** Lines 113-224 are a verbatim copy of lines 1-112, including the front matter. On the live page the copied front matter renders as an h2 (`id="title-valkey-search-overview-description-valkey-search-module-overview"`), and every heading appears twice with `-1` anchors.
**Evidence:** `curl -sL https://valkey.io/topics/search/ | grep -c 'Use-Cases Where'` returns `2`. The live h2 list includes `see-search-info-fields-for-details`, then `title-valkey-search-overview-...`, followed by `use-cases-where-valkey-search-shines-1` ... `info-fields-1`.
**Fix:** Delete lines 113-224.
Tracked: valkey-io/valkey-io.github.io#702, valkey-io/valkey-doc#509

### F3 [minor] OUTDATED: Supported Commands list omits FT.HYBRID and FT.CURSOR (`topics/search.md:28`)
**Doc says:**
> FT.CREATE FT.DROPINDEX FT.INFO FT._LIST FT.SEARCH FT.AGGREGATE
**Problem:** Search 1.3 adds `FT.HYBRID` and `FT.CURSOR`, and valkey-doc already documents both (commands/ft.hybrid.md, commands/ft.cursor.md, from the "Documentation changes for Valkey Search 1.3" commit 058a8fd1). The overview still lists only the 1.2 set.
**Evidence:** valkey-search@1.3.0-rc1:src/commands/commands.h:51,53:
```
constexpr absl::string_view kHybridCommand{"FT.HYBRID"};
constexpr absl::string_view kCursorCommand{"FT.CURSOR"};
```
These are absent at 1.2.1:src/commands/commands.h:36-43.
**Fix:** Add `FT.HYBRID` and `FT.CURSOR` with "(1.3+)". Better: turn the list into a table of linked commands with the module version each one was introduced in.

### F4 [minor] INACCURATE: "Even if the prefix list is confined to a single slot" describes a configuration FT.CREATE rejects (`topics/search.md:90`)
**Doc says:**
> Index names without a hash tag are considered to be cluster-wide indexes (even if the prefix list is confined to a single slot).
**Problem:** If the index name has no hash tag, FT.CREATE rejects any prefix that has one. So a single-slot prefix list on a cluster-wide index can't be expressed, and the page never states this rule.
**Evidence:** valkey-search@c672fd7:src/commands/ft_create_parser.cc:284-291 (same at 1.2.1:249-261):
```
    if (index_hash_tag.has_value()) {
      ...
            "All PREFIX arguments must contain the same hash tag as the index");
    } else if (this_tag.has_value()) {
      return absl::InvalidArgumentError(
          "PREFIX argument(s) must not contain a hash tag");
```
**Fix:** Replace the sentence with: "Index names without a hash tag are cluster-wide indexes, and their prefixes must not contain a hash tag (`PREFIX argument(s) must not contain a hash tag`). Index names with a hash tag are single-slot indexes, and every prefix must contain the same hash tag (`All PREFIX arguments must contain the same hash tag as the index`)."

### F5 [minor] MISSING: Overview does not link the other Search topic pages (`topics/search.md:106`)
**Doc says:**
> See [Search Configurations](search-configurables.md) ... See [Search Info Fields](search-observables.md)
**Problem:** The overview is the entry point, but it links only the configurables and observables pages. topics/search-query.md, search-expressions.md, search-data-formats.md, and search-monitoring.md are unreachable from it. These cover query syntax and the vector data formats a reader needs next.
**Evidence:** `grep -n 'search-query\|search-expressions\|search-data-formats\|search-monitoring' topics/search.md` returns nothing.
**Fix:** Add a "Related pages" section that links each of the six search-*.md topics with a one-line description.

### F6 [minor] MISSING: No prerequisite for loading the module (`topics/search.md:6`)
**Doc says:**
> **Valkey-Search** (BSD-3-Clause), provided as a Valkey module, is a high-performance Search engine
**Problem:** The page never says how to get the module (valkey-bundle or `--loadmodule`) or which module version it describes. The commands fail with "unknown command" on a stock valkey-server 9.1.2, which loads no modules.
**Evidence:** topics/valkey-bundle.md and topics/bloomfilters.md cover loading their modules. topics/search.md has no `loadmodule` or bundle reference.
**Fix:** After the first paragraph, add: "Valkey Search is not built into valkey-server. Load it with `loadmodule /path/to/libsearch.so` or use [valkey-bundle](valkey-bundle.md). This page describes Valkey Search 1.3."

### F7 [nit] AI-NATIVE: Marketing heading and unsourced performance claims (`topics/search.md:7`)
**Doc says:**
> It delivers single-digit millisecond latency and high QPS, capable of handling billions of vectors with over 99% recall ... ## Use-Cases Where **Valkey-Search** Shines
**Problem:** The latency, scale, and recall figures have no conditions or source. The use-case list (lines 17-23) is generic filler any vector database page could carry, and it sits above the Valkey-specific material on consistency, backfill, and cluster behavior.
**Fix:** Drop the use-case section, or reduce it to one sentence. Either qualify the numbers (dataset, HNSW parameters, hardware) or remove them. If a heading is kept, rename it "Common use cases" and remove the bold.

### F8 [nit] STYLE: Weak front matter description (`topics/search.md:3`)
**Doc says:**
> description: Valkey Search Module Overview
**Fix:** `description: How Valkey Search indexes Hash and JSON keys, ingests and backfills data, and runs vector and full-text queries in standalone and cluster mode.`

### F9 [nit] CONFUSING: "Supported Commands" is a plaintext fence with no links (`topics/search.md:28`)
**Doc says:**
> ```plaintext
> FT.CREATE
> ...
**Problem:** The command names can't be clicked, even though a page exists for each one. FT.CREATE is the only command linked anywhere in the body (line 46).
**Fix:** Replace the fence with a bulleted list linking `../commands/ft.create.md` and the rest, each with a one-line purpose.

## Unverified
- topics/search.md:50: "The client which executed the mutation command is blocked until all indexes are updated." Not traced through the keyspace notification handler within the timebox.
- topics/search.md:94: Merkle-tree checksum and last-writer-wins resolution in the metadata manager. Not traced.
