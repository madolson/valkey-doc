VERDICT: NEEDS-WORK
COUNTS: blocker=0 minor=5 nit=3

# Valkey Search - Metrics and configurations (`search-monitoring`)

https://valkey.io/topics/search-monitoring/

Stale orphan. valkey-doc #500 (058a8fd1) replaced it with `search-observables.md` and `search-configurables.md`, which `search.md` links to, but left this file in place. The main fix is to delete it and redirect to those two pages. Until then it ships a wrong coordinator timeout default and an incomplete metric list. Verified against valkey-search 1.2.1 (latest stable tag) and main at c672fd7. The values below are the same on both unless noted.

## Findings

### F1 [minor] DUPLICATE: Whole page superseded by search-observables and search-configurables (`topics/search-monitoring.md:1`)
**Doc says:**
> title: "Valkey Search - Metrics and configurations"
**Problem:** Every metric table here repeats `topics/search-observables.md`, and the Configurations table repeats `topics/search-configurables.md`, nearly line for line. No page links here: `topics/search.md:108,112,220,224` link only to the other two. The newer pages have more content (section column, visibility rules, `hnswlib` metrics, `use-coordinator`), so readers who land here from search get the older, less complete copy.
**Evidence:** `git log -- topics/search-monitoring.md` shows a single commit, 2034f4d6 (#422). Commit 058a8fd1 (#500) added `search-configurables.md` and `search-observables.md` but left this file. `grep -rn search-monitoring topics/ commands/` and the same grep in the website repo both return nothing. The live page returns 200.
**Fix:** Delete `topics/search-monitoring.md`. Add a redirect from `/topics/search-monitoring/` to `/topics/search-observables/` in the website. Before deleting, copy this page's Default Value column into `search-configurables.md`, where that column is empty, using the corrected values from F2.

### F2 [minor] INACCURATE: coordinator-query-timeout-secs default is 25, not 120 (`topics/search-monitoring.md:159`)
**Doc says:**
> | search.coordinator-query-timeout-secs | Number | 120 | ...
**Problem:** The real default is 25 seconds. Operators who size client timeouts from this page will be off by almost 5x.
**Evidence:** `valkey-search@1.2.1:src/coordinator/client.cc:63` and `valkey-search@c672fd7:src/coordinator/client.cc:65`:
```
static constexpr int kCoordinatorQueryDefaultTimeout{25};
static constexpr int kCoordinatorQueryMinTimeout{1};
static constexpr int kCoordinatorQueryMaxTimeout{3600};
```
**Fix:** Change the default to `25` and add "Range 1 to 3600."

### F3 [minor] INACCURATE: Coordinator metrics depend on use-coordinator, not just cluster mode (`topics/search-monitoring.md:92`)
**Doc says:**
> These metrics are only available when cluster mode is enabled.
**Problem:** A cluster without `--use-coordinator` shows none of these metrics. The coordinator is created only when `use-coordinator` is set, and visibility depends on that object existing. Latency metrics also appear only after the first sample is recorded.
**Evidence:** `valkey-search@1.2.1:src/valkey_search.cc:1195`: `if (options::GetUseCoordinator().GetValue()) { ... coordinator_ = coordinator::ServerImpl::Create(...)`. `src/valkey_search.h:121`: `bool UsingCoordinator() const { return coordinator_ != nullptr; }`. `src/valkey_search.cc:533`: `.VisibleIf([]() -> bool { return ValkeySearch::Instance().UsingCoordinator(); })`.
**Fix:** "These metrics appear only when the module is loaded with `--use-coordinator`. Metrics ending in `_latency_usec` appear only after at least one sample has been recorded." This is the wording in `search-observables.md:11-12`.

### F4 [minor] MISSING: hnswlib metric section absent (`topics/search-monitoring.md:6`)
**Doc says:** (no `search_hnswlib` section)
**Problem:** Five application-visible metrics are not listed. These exception counters are the main signal for HNSW index failures.
**Evidence:** `valkey-search@1.2.1:src/valkey_search.cc:455-483` registers `hnsw_add_exceptions_count`, `hnsw_remove_exceptions_count`, `hnsw_modify_exceptions_count`, `hnsw_search_exceptions_count`, and `hnsw_create_exceptions_count`, all in section `"hnswlib"` with `.App()`.
**Fix:** Add a `### search_hnswlib` table with these five metrics (Count, "Exceptions during HNSW vector add/remove/modify/search/create"). Or resolve through F1, since `search-observables.md:37` already lists them.

### F5 [minor] MISSING: use-coordinator and info-developer-visible configs not listed (`topics/search-monitoring.md:121`)
**Doc says:** (the table has neither `search.use-coordinator` nor `search.info-developer-visible`)
**Problem:** `use-coordinator` is required for cluster-wide search and can only be set at load time. A reader who uses this page as the config reference cannot set up cluster search. `info-developer-visible` controls whether developer metrics show in INFO.
**Evidence:** `valkey-search@1.2.1:src/valkey_search_options.cc`: `config::BooleanBuilder(kUseCoordinator, false).Hidden() // can only be set during start-up`. `vmsdk/src/info.cc:91`: `vmsdk::config::Boolean("info-developer-visible", false);`.
**Fix:** Add `| search.use-coordinator | Boolean | false | Enables the cluster coordinator (gRPC). Load-time only; pass as a MODULE LOAD argument. |` and `| search.info-developer-visible | Boolean | false | Shows developer metrics in INFO SEARCH. |`.

### F6 [nit] STYLE: Typo "MODULE LODEX" (`topics/search-monitoring.md:119`)
**Doc says:**
> can be set on the `MODULE LODEX` command OR via the `CONFIG SET` command.
**Problem:** The typo is also visible on the live page. The sentence also says every config can be set with CONFIG SET, which is false for load-time-only configs such as `use-coordinator`.
**Fix:** "Each configuration below can be passed as an argument to `MODULE LOAD` (or `loadmodule` in valkey.conf), and most can also be changed at runtime with `CONFIG SET`."

### F7 [nit] CONFUSING: max-vector-attributes and max-attributes described identically (`topics/search-monitoring.md:150`)
**Doc says:**
> | search.max-vector-attributes | Number | 1000 | Controls the max number of attributes per index |
**Problem:** Two configs have the same description and no explanation of how they relate. In source, `max-vector-attributes` is a misnamed legacy alias that takes priority only if it was set explicitly.
**Evidence:** `valkey-search@1.2.1:src/commands/ft_create_parser.cc:615-619`: `// The incorrectly named max_vector_attributes is kept for backward ... options::GetMaxVectorAttributes().WasSet() ? options::GetMaxVectorAttributes().GetValue()`.
**Fix:** "Deprecated alias for `search.max-attributes`. If set, it overrides `search.max-attributes`."

### F8 [nit] AI-NATIVE: No intro, weak description (`topics/search-monitoring.md:3`)
**Doc says:**
> description: Valkey Search Module INFO SEARCH
**Problem:** The page opens directly on a table, and the description is a fragment. Nothing says that the metrics come from `INFO SEARCH`, that every field carries a `search_` prefix, or that developer metrics are hidden.
**Fix:** Moot if F1 is applied. Otherwise, set the description to "INFO SEARCH metrics and configuration parameters for the Valkey Search module" and add one sentence under the H1: "Run `INFO SEARCH` to see these fields. Each is prefixed with `search_`."
