VERDICT: MINOR-ISSUES
COUNTS: blocker=0 minor=4 nit=2

# Valkey Search - INFO SEARCH Metrics (`search-observables`)

https://valkey.io/topics/search-observables/

The metric list matches valkey-search main (c672fd7). Every Application-visible field is present and no extra ones are listed. The most important fix: the page omits the `search_` prefix that INFO adds to every field and section, so the names shown are not what `INFO SEARCH` prints. It also says nothing about which module version the metrics apply to. Checked against valkey-search main c672fd7 and tags 1.2.1 and 1.3.0-rc1.

## Findings

### F1 [minor] INACCURATE: Metric and section names lack the `search_` prefix INFO actually prints (`topics/search-observables.md:14`)
**Doc says:**
> | `number_of_indexes` | `index_stats` | Count | Total number of search indexes |

**Problem:** Valkey prefixes every module INFO field and section with the module name. `INFO SEARCH` prints `# search_index_stats` and `search_number_of_indexes:...`, not `number_of_indexes`. The bare names apply to every row in the table and to `compatibility-*` at lines 83-85. Anyone writing an exporter or alert from this table gets the wrong keys.
**Evidence:** valkey@9.1.2:src/module.c:11067-11068, 11086, 11139:
```
sds full_name = sdsdup(ctx->module->name);
if (name != NULL && strlen(name) > 0) full_name = sdscatfmt(full_name, "_%s", name);
ctx->info = sdscatfmt(ctx->info, "# %S\r\n", full_name);
ctx->info = sdscatfmt(ctx->info, "%s_%s:%s\r\n", ctx->module->name, field, value);
```
The module's own tests use the prefixed name: valkey-search@c672fd7:integration/test_number_of_indexes_on_replica_fullsync.py:16 `replica_client.info("search").get("search_number_of_indexes")`.
**Fix:** Add after line 12: "`INFO SEARCH` prefixes every field and section with `search_`. For example, `number_of_indexes` in section `index_stats` prints as `search_number_of_indexes` under `# search_index_stats`. To fetch one section, run `INFO search_index_stats`." The other option is to print the full names in the table, as topics/search-monitoring.md:12 already does.

### F2 [minor] MISSING: No module version, and several listed metrics are not in the latest stable valkey-search (`topics/search-observables.md:8`)
**Doc says:**
> Only Application metrics are listed here. These are kept compatible across releases.

**Problem:** The page tracks main and never says so. `num_cursors` (line 44), `compatibility-invalid_data_drops_key` (line 16), all six `vector_registry_*` rows (lines 74-79), and the `search.emulate-release` config (lines 16, 83) do not exist in 1.2.1, the latest stable tag. They first appear in 1.3.0-rc1. `search.info-developer-visible` exists in both. Someone on 1.2.x will look for fields that are not there.
**Evidence:** Exact-name `git grep` in valkey-search src:
```
== 1.2.1
num_cursors                          (no match)
compatibility-invalid_data_drops_key (no match)
vector_registry_sharing_active       (no match)
vector_registry_pending_unshare_cnt  (no match)
emulate-release                      (no match)
== 1.3.0-rc1
num_cursors 1.3.0-rc1:src/cursor.cc
compatibility-invalid_data_drops_key 1.3.0-rc1:src/index_schema.cc
vector_registry_sharing_active 1.3.0-rc1:src/valkey_search.cc
emulate-release 1.3.0-rc1:src/commands/filter_parser.cc
```
**Fix:** Add a "Since" column, or a sentence after line 9: "`num_cursors`, `compatibility-*`, and `vector_registry_*` metrics were added in valkey-search 1.3.0."

### F3 [minor] INACCURATE: `compatibility-tag_rejects_non_utf8` is also always reported (`topics/search-observables.md:85`)
**Doc says:**
> `compatibility-invalid_data_drops_key` is always reported; the others appear in `INFO SEARCH` only after the code path is first reached.

**Problem:** On main, `compatibility-tag_rejects_non_utf8` is a static App counter with no `VisibleIf`, so it shows from startup just like `invalid_data_drops_key`. Only the counters created by the `VALKEY_SEARCH_COMPATIBILITY_FIX` macro are lazy. This counter is unreleased (added in c47ae7b, not in 1.3.0-rc1), so the sentence is correct for 1.3.0-rc1 and wrong for main.
**Evidence:** valkey-search@c672fd7:src/indexes/tag.cc:40-46:
```
// INFO counter for the non-UTF-8 TAG compatibility defect (see
// COMPATIBILITY.md). Statically constructed because AddRecord/ModifyRecord run
// on worker threads, where VALKEY_SEARCH_COMPATIBILITY_FIX's lazily
// constructed counter is unsafe.
vmsdk::info_field::Integer tag_rejects_non_utf8_compat_counter(
    "compatibility", "compatibility-tag_rejects_non_utf8",
    vmsdk::info_field::IntegerBuilder().App());
```
vmsdk/src/info.cc:257-258: `return visible_func_ ? (*visible_func_)() : true;`
**Fix:** Replace with: "`compatibility-invalid_data_drops_key` and `compatibility-tag_rejects_non_utf8` are always reported. The others appear in `INFO SEARCH` only after the code path is first reached." Also add a `compatibility-tag_rejects_non_utf8` row to the table next to line 16.

### F4 [minor] DUPLICATE: topics/search-monitoring.md has a second, older copy of this metric list (`topics/search-observables.md:14`)
**Doc says:** (search-observables.md:43)
> | `number_of_indexes` | `index_stats` | Count | Total number of search indexes |

**Problem:** topics/search-monitoring.md:6-57 and its later sections list the same metrics with different names (prefixed) and wording. It has drifted: it includes a `search_vector_externing` section (search-monitoring.md:72) that this page and main lack, and it has no `cursors`, `compatibility`, or `vector_registry` sections. Both pages are published (https://valkey.io/topics/search-monitoring/ returns 200). topics/search.md:112 and :224 link only to search-observables.md.
**Evidence:** topics/search-monitoring.md:13 `| search_number_of_indexes | Count | Total number of search indexes |`. search-monitoring.md:72 `### search_vector_externing`.
**Fix:** Make search-observables.md canonical. Replace the "Metrics - INFO SEARCH" section of search-monitoring.md with one line: "See [INFO SEARCH metrics](search-observables.md)." Keep only its Configurations section, or merge that section into search-configurables.md.

### F5 [nit] AI-NATIVE: Opening has no scope statement, uses "Application metrics" without defining it, and has no example output (`topics/search-observables.md:3`)
**Doc says:**
> description: Valkey Search Module INFO SEARCH Metrics
> Only Application metrics are listed here.

**Problem:** The description repeats the title. The first paragraph never says what the page covers or how to get the data, and "Application" vs "Developer" is internal vmsdk terminology. No sample output shows the format of the `_latency_usec` string fields.
**Fix:** Description: "Reference for every INFO SEARCH field reported by the valkey-search module: section, unit, meaning, and when it appears." Opening line: "Run `INFO SEARCH` to get valkey-search metrics. This page lists the stable fields, which keep their names across releases. Internal debugging fields are hidden unless `search.info-developer-visible` is `yes`, and they can change in any release." Add a short `INFO search_index_stats` output block, fenced as `text`.

### F6 [nit] STYLE: Table order is neither alphabetical nor grouped by section (`topics/search-observables.md:44`)
**Doc says:**
> | `num_cursors` | `cursors` | ...

**Problem:** Rows are sorted by section, except `num_cursors` (`cursors`) sits between `index_stats` rows (lines 42-46). Within `query` (lines 53-60), rows are not alphabetical.
**Fix:** Move line 44 above line 42, or split the table into one table per section with `###` headings named for the actual INFO section (e.g. `search_index_stats`). That also fixes F1 on the section side.

## Unverified
- topics/search-observables.md:20 (all `_latency_usec` rows): the format of the "Latency distribution" string from `GetStatsString()`. I did not locate the formatter within the timebox, and no live server with the module was available.
