VERDICT: NEEDS-WORK
COUNTS: blocker=0 minor=5 nit=2

# Valkey Search - Configuration (`search-configurables`)

https://valkey.io/topics/search-configurables/

The table gives names but leaves the Default column empty for 41 of 46 rows. It also says every config can be set at load or with `CONFIG SET`, which is false for hidden and developer configs, and it is out of sync with a second, partly wrong copy of the same table in search-monitoring.md. Most important fix: fill in defaults and ranges from source, then make this page the only config table. Verified against valkey-search tags 1.2.1 (latest stable) and 1.3.0-rc1, and main at c672fd7.

## Findings

### F1 [minor] MISSING: Default column empty for 41 of 46 configs (`topics/search-configurables.md:12`)
**Doc says:**
> | search.query-string-bytes                     | Number  |               | Controls the length of the query string for FT.SEARCH command
**Problem:** Only enable-partial-results, enable-consistent-results, max-nonvector-search-results-fetched, cursor-max-count and cursor-max-idle-ms have a default (lines 22, 23, 38, 53, 54). Defaults and valid ranges are the main reason anyone opens this page. No row gives a min/max, and none says the thread defaults depend on the host's physical core count.
**Evidence:** valkey-search@c672fd7, same values in 1.2.1:
```
src/valkey_search_options.cc:67   kDefaultQueryStringBytes{10240}, min 1
src/valkey_search_options.cc:21   kHNSWDefaultBlockSize{10240}
src/valkey_search_options.cc:105  kDefaultThreadsCount = vmsdk::GetPhysicalCPUCoresCount(); max 1024   (reader/writer)
src/valkey_search_options.cc:139  utility-threads default 1
src/valkey_search_options.cc:160  max-worker-suspension-secs 60, range 0..3600
src/valkey_search_options.cc:23   ft-info-timeout-ms 5000 (100..300000); :26 ft-info-rpc-timeout-ms 2500
src/commands/ft_create_parser.cc:83  kTimeoutMs{50000}   (default-timeout-ms)
src/commands/ft_search_parser.cc:37  kDefaultKnnLimit{10000}; max-timeout-ms default/max query::kMaxTimeoutMs = 60000
src/query/response_generator.cc:54   max-search-result-record-size = 10MB/2 = 5242880; :67 fields-count = 1000/2 = 500
src/coordinator/client.cc:65      kCoordinatorQueryDefaultTimeout{25}
vmsdk/src/cluster_map.cc:28       cluster-map-expiration-ms 250 (0..3600000)
src/schema_manager.cc:56,60       max-indexes 1000; backfill-batch-size 10240
```
**Fix:** Fill the column: query-string-bytes 10240; hnsw-block-size 10240; reader-threads and writer-threads = physical core count (1..1024); utility-threads 1; max-worker-suspension-secs 60; use-coordinator no; skip-rdb-load no; skip-corrupted-internal-update-entries no; log-level notice; search-result-background-cleanup no; high-priority-weight 100 (0..100); ft-info-timeout-ms 5000; ft-info-rpc-timeout-ms 2500; local-fanout-queue-wait-threshold 50; thread-pool-wait-time-samples 100 (10..10000); max-term-expansions 200 (1..100000); tag-min-prefix-length 2; search-result-buffer-multiplier 1.5 (1.0..1000.0); drain-mutation-queue-on-save no; query-string-depth 1000; query-string-terms-count 1000 (max 10000); fuzzy-max-distance 3 (1..50); max-vector-knn 10000 (max 100000); proximity-inorder-compat-mode no; max-prefixes 8; max-tag-field-length 256; max-numeric-field-length 128; max-vector-attributes 1000; max-vector-dimensions 32768; max-vector-m 2000000; max-vector-ef-construction 1000000; max-vector-ef-runtime 1000000; default-timeout-ms 50000; max-timeout-ms 60000; max-search-result-record-size 5242880; max-search-result-fields-count 500; backfill-batch-size 10240; coordinator-query-timeout-secs 25; max-indexes 1000; cluster-map-expiration-ms 250. Replace `INT64_MAX` on line 54 with "9223372036854775807 (no limit)". Add a Range column.

### F2 [minor] INACCURATE: Not every config can be set at load or with CONFIG SET (`topics/search-configurables.md:8`)
**Doc says:**
> Thus each of the named configuration below can be set on the module load command OR via the `CONFIG SET` command.
**Problem:** Hidden configs are never registered with Valkey, so `CONFIG SET search.use-coordinator` and `search.use-coordinator` in valkey.conf do not work. It can only be passed as a module load argument. `ft-info-timeout-ms` and `ft-info-rpc-timeout-ms` are developer configs: changing them at runtime fails unless `search.debug-mode` is `yes`. Line 18's "can only be set at startup" doesn't say which startup mechanism works.
**Evidence:** valkey-search@1.2.1 and main:
```
src/valkey_search_options.cc:  static auto use_coordinator = config::BooleanBuilder(kUseCoordinator, false)
                                  .Hidden()  // can only be set during start-up
vmsdk/src/module_config.cc (Init):  if (entry == debug_mode.get() || entry->IsHidden()) { continue; }
src/valkey_search_options.cc:  ft_info_timeout_ms ... .Dev()  // can only be set in debug mode
vmsdk/src/module_config.h:220:  if (IsDeveloperConfig() && !IsDebugModeEnabled() && !IsLoadingConfigFile() ...
      "Modification of '%s' requires '%s' to be enabled."
```
**Fix:** Replace line 8 with: "Most settings can be passed as module load arguments, set in valkey.conf as `search.<name>`, or changed at runtime with `CONFIG SET search.<name>`. Exceptions: `use-coordinator` can only be passed as a module load argument (`--use-coordinator yes`). `ft-info-timeout-ms` and `ft-info-rpc-timeout-ms` are developer settings that can only be changed while `search.debug-mode` is `yes`." Change line 18 to "Module load argument only; not visible to CONFIG GET/SET."

### F3 [minor] MISSING: Module load argument syntax not shown (`topics/search-configurables.md:8`)
**Doc says:**
> can be set on the module load command OR via the `CONFIG SET` command.
**Problem:** Module arguments use `--<name> <value>` with no `search.` prefix, while CONFIG SET uses `search.<name>`. The table only shows the `search.` form, so `loadmodule libsearch.so search.reader-threads 8` looks right but is rejected.
**Evidence:** valkey-search@c672fd7:vmsdk/src/module_config.cc:187-194:
```
  if (!key.starts_with("--")) {
    return absl::InvalidArgumentError(absl::StrFormat(
        "Command line argument: '%s' must start with '--'", key));
  }
  // remove the "--"
  key = key.substr(2);
```
**Fix:** Add after line 8:
````
```
# valkey.conf
loadmodule /path/to/libsearch.so --reader-threads 8 --use-coordinator yes
search.max-indexes 2000
```
```
CONFIG SET search.max-indexes 2000
```
````

### F4 [minor] DUPLICATE: Second config table in search-monitoring.md has drifted (`topics/search-configurables.md:10`)
**Doc says:** This page's table (lines 10-57) is repeated as "## Configurations" in topics/search-monitoring.md:117-167, with the same intro sentence (with a "MODULE LODEX" typo at line 119).
**Problem:** The two tables differ. The monitoring copy has defaults this page lacks, and it lists async-fanout-threshold, max-attributes and max-mutation-queue-size-on-restore, which this page doesn't. Some of its values are also wrong, e.g. `search.coordinator-query-timeout-secs | Number | 120` (search-monitoring.md:163), while source is 25. Readers get different answers depending on which page they land on.
**Evidence:** valkey-search@1.2.1:src/coordinator/client.cc:63 `static constexpr int kCoordinatorQueryDefaultTimeout{25};`
**Fix:** Make this page canonical: merge in the monitoring table's defaults and extra rows, using F1's values. Then replace search-monitoring.md:117-167 with "See [Search Configuration](search-configurables.md)."

### F5 [minor] MISSING: Unreleased and newer configs listed with no version, user-facing configs absent (`topics/search-configurables.md:43`)
**Doc says:**
> | search.max-vector-attributes                  | Number  |               | Controls the max number of attributes per index
**Problem:** `max-timeout-ms`, `cursor-max-count` and `cursor-max-idle-ms` (lines 49, 53, 54) are not in 1.2.1, the latest stable release. They first appear in 1.3.0-rc1, and the page doesn't say so. `max-vector-attributes` is deprecated in favor of `max-attributes`, which isn't listed. Several user-facing (non-dev) configs are missing: `max-attributes`, `async-fanout-threshold`, `max-mutation-queue-size-on-restore` (in 1.2.1); `max-query-queue-depth`, `max-group-key-expansion`, `vector-unshare-batch-size`, `default-scorer` (from 1.3.0-rc1); and `debug-mode`, which F2's dev configs depend on.
**Evidence:**
```
$ for n in max-timeout-ms cursor-max-count cursor-max-idle-ms; do git grep -q "\"$n\"" 1.2.1 -- src vmsdk/src || echo missing $n; done
missing max-timeout-ms
missing cursor-max-count
missing cursor-max-idle-ms
```
valkey-search@c672fd7:src/commands/ft_create_parser.cc (max_vector_attributes):
```
/// This is a legacy incorrectly named configuration. left intact for backward
/// compatibility. But it's deprecated.
```
src/valkey_search_options.cc: `max-query-queue-depth` default 100000 ("0 = unlimited"), `max-group-key-expansion` default 65536, `vector-unshare-batch-size` default 10240, `async-fanout-threshold` default 30, `max-mutation-queue-size-on-restore` default 10000, `default-scorer` default BM25STD. None use `.Dev()` or `.Hidden()`.
**Fix:** Add rows for the missing configs with those defaults. Mark `max-vector-attributes` "Deprecated; use `search.max-attributes`". Add "Since" notes (for example "1.3.0") to rows not in 1.2.1.

### F6 [nit] INACCURATE: search-result-buffer-multiplier typed as String (`topics/search-configurables.md:32`)
**Doc says:**
> | search.search-result-buffer-multiplier        | String  |
**Problem:** It is a decimal number with a range. The module only stores it as a string because the module API has no double type. "String" suggests free text.
**Evidence:** valkey-search@c672fd7:src/valkey_search_options.cc: `config::DoubleBuilder(kSearchResultBufferMultiplierConfig, 1.5, 1.0, 1000.0)`; vmsdk/src/module_config.h:570 "`double` -> Double configuration (stored as string under the hood)".
**Fix:** Type "Decimal (set as a string, e.g. `"1.5"`)", default 1.5, range 1.0 to 1000.0.

### F7 [nit] AI-NATIVE: H1, description and descriptions carry little information (`topics/search-configurables.md:3`)
**Doc says:**
> description: Valkey Search Module Configurable Settings
> # Configurables
**Problem:** The heading and description don't say what the page answers. Most rows open with "Controls the ...", which repeats the name and gives no units, effect or tradeoff (e.g. line 13 "HNSW resize configuration parameter").
**Fix:** H1 "Valkey Search configuration parameters". Description: "Names, defaults, ranges, and how to set every valkey-search module configuration parameter." Rewrite rows to state effect and unit, e.g. hnsw-block-size: "Number of vectors added to an HNSW index each time it grows."
