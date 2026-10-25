# valkey.io documentation review, October 2026

Every valkey.io topic page, the clients page, the docs entry pages, and the site structure, reviewed against Valkey 9.1.2 by one AI agent per page. 83 reports, 35 blockers, 405 minor, 371 nits. Agents checked claims against 9.1.2 source and a live 9.1.2 build. A second pass re-verified every blocker, and a sample was checked by hand. Treat findings as well-evidenced leads, not as truth: one blocker was refuted on re-check, and some proposed fixes needed correcting (noted inline).

Reviewed commits: valkey-doc `a95bf25ca`, valkey-io.github.io `5900f06`, valkey `9.1.2`. Line numbers refer to those commits.

## Summary

### Fix first

1. Security page tells reporters to open a public GitHub issue and points to a GPG key the page does not contain. SECURITY.md at 9.1.2 says to email security@lists.valkey.io and "*DO NOT* create an issue". security F1 [CONFIRMED]
2. Sentinel turns protected mode off (`src/sentinel.c:498`, "Sentinel must be exposed"), and the page never says so or tells readers to firewall it. sentinel F1 [CONFIRMED; the "listens on all interfaces" part of the fix comes from the default bind, not sentinel.c]
3. The sample valkey.conf on the bundle page stops the container from starting: inline `# 128MB` comments become part of module config values (`config.c:585-586`), and /var/log/valkey does not exist. valkey-bundle F1 [CONFIRMED]
4. The bundle sample config binds to loopback inside the container, so the published port does not connect. valkey-bundle F2 [CONFIRMED]
5. The first quickstart examples fail in order: `SET user:1000` then `HSET user:1000` returns WRONGTYPE. Reproduced on 9.1.2. quickstart F1 [CONFIRMED]
6. CLUSTER SETSLOT TIMEOUT is parsed in milliseconds (`cluster_legacy.c:7766`), but the doc says seconds. commands/cluster-setslot.md:96 has the same error. cluster-spec F1 [CONFIRMED]
7. The cluster migration procedure assumes one AOF file. 9.x writes the multi-part `appendonlydir/`. cluster-tutorial F1 [CONFIRMED; "won't load it" is too strong, because `aof.c:1788-1800` still loads a legacy single file. The fix should say to copy the whole directory]
8. The script cache sections leave out EVAL LRU eviction (`eval.c:343-358`, 500 entries). Clients that rely on EVALSHA get unexpected NOSCRIPT errors. eval-intro F1 [CONFIRMED]
9. The search page says a query timeout returns an error, but `prefer_partial_results` defaults to true, so it returns partial results. search F1 [CONFIRMED]
10. Query features that exist only in 1.3 (VECTOR_RANGE, HYBRID_POLICY, ADHOC_BF, BFLOAT16, yield_distance_as) appear with no version gate, while 1.2.1 is the GA release. search-query F1 [CONFIRMED]
11. The expression grammar swaps additive and multiplicative precedence for 1.2.1 (`expr.cc:504-524`). Separate `&&` precedence and right-associative `^` exist only in 1.3. search-expressions F1 [CONFIRMED]
12. The blocking-ops example module is broken in three ways: the `free_privdata` signature has one argument instead of two, it replies with the pointer instead of the value, and `IsBlockedTimeoutRequest` is never called. modules-blocking-ops F1, F2, F3 [CONFIRMED]
13. The HELLO reply fields do not match 9.1.2. The live reply has `server`=valkey, `role`=master, and `modules` as an array of maps. protocol F1 [CONFIRMED]
14. `server.SERVER_VERSION` and `SERVER_VERSION_NUM` return nil on 9.1.2. Only VALKEY_*, REDIS_* and SERVER_NAME exist (`script_lua.c:1694-1717`), so the source comment at `:1789` is misleading too. migration.md:42 repeats the error. lua-api F1 [CONFIRMED]
15. The Sentinel Pub/Sub event list includes `failover-detected` and `dup-sentinel`, which do not exist, and drops `+` prefixes. sentinel F2 [CONFIRMED]
16. RDMA module arguments `bind=`/`port=` are ignored (`rdma.c:1910-1912`), and `rdma-port` defaults to 0, so RDMA never listens. RDMA F1 [CONFIRMED]
17. The TLS page describes a certificate validity-period check that was reverted before 9.1.0 (#3572 reverted #2999). tls F1 [CONFIRMED]
18. The client feature matrix is wrong: GLIDE C# has no connection pool, Redisson's scan iterator resets on failover, and GLIDE PHP `ssubscribe` is a TODO. client-libraries F1, F2, F3 [CONFIRMED]

### Other confirmed blockers

- search-expressions F2 (no `%` operator in any version), F3 (`minute`/`hour`/`day`/`month` return truncated timestamps). [CONFIRMED; `mktime` uses local time, so the proposed "UTC" wording is only right on UTC hosts]
- search-query F2: `b*` fails the default min prefix length of 2, and `b*|green` matches 5 keys, not 2. [CONFIRMED]
- valkey-bundle F3: the LDAP example uses a host that does not resolve and never creates the ACL user. [CONFIRMED]
- ldap F1: the config is `ldap.search_base`, not `search_base_dn` (`src/lib.rs:157`). [CONFIRMED]
- ldap F2: users that already have a password still authenticate if LDAP fails (`auth.rs:29-33`). [CONFIRMED; the sentence is true for new users, so this is minor, not a blocker]
- installation F2: the Alpine steps use `update-rc.d`. Alpine uses OpenRC, and the apk ships /etc/init.d/valkey. [CONFIRMED]
- RDMA F2: the config is `rdma-completion-vector`. `rdma-comp-vector` does not exist (`config.c:3430`). [CONFIRMED]
- cli F1: `VALKEYCLI_HOST`/`VALKEYCLI_PORT` are not in 9.1.2. [CONFIRMED; the adding commit baa5a48b5 is also in tag 8.0.8, so the report's "Since Valkey 9.2" fix wording is wrong]
- command-arguments F1: nested args are under `arguments`, and `value` is never sent (`server.c:5140-5186`). [CONFIRMED]
- modules-api-ref F1: the page is stale against 9.1.2 `module.c` and is missing the `ScriptingEngineDebuggerLog` ownership rule (`module.c:14555`) and the type-255 note (`module.c:9619`). [CONFIRMED]
- client-libraries F4: `swift package add-dependency` with no version flag fails. [CONFIRMED]

### Refuted

- installation F1 [REFUTED on manual check: Debian valkey-server declares `Provides: valkey` in 9.1.2-1, 8.1.1+dfsg1-3+deb13u2 and 8.0.1+dfsg1-1~bpo12+1 (sources.debian.org debian/control), so `apt install valkey` selects valkey-server. Ubuntu publishes valkey-redis-compat for noble and plucky (Launchpad). The compile agent's CONFIRMED was wrong.]

### Cross-cutting patterns

- **Untagged code blocks.** 1111 of 1268 rendered `<pre>` blocks are `plain` (site-structure F12). Pages: acl cli cluster-spec cluster-tutorial command-arguments debugging distlock eval-intro functions-intro geospatial indexing ldap lists lru-cache lua-api mass-insertion modules-native-types notifications performance-on-cpu persistence protocol replication search-data-formats search-query sets sorted-sets streams-intro tls valkey-json. Sweep: enable markdownlint MD040 and MD046 (fenced style) in valkey-doc CI. Use `console` for valkey-cli sessions and `ini` for config.
- **Redis-era version notes and names.** Version gates like "Redis OSS 7.0" or "Valkey 3.2" read as Valkey versions, and legacy names remain (`redis.call`, `REDISCLI_*`, `gen_redis_proto`, redis-rb-cluster). Pages: acl ARM cli clients cluster-spec cluster-tutorial command-arguments eval-intro functions-intro hashes lua-api mass-insertion modules-api-ref modules-intro problems protocol pubsub replication sentinel sets streams-intro valkey.conf. Sweep: grep `Redis OSS|Redis [0-9]|version [0-9]\.[0-9]|redis\.call|REDISCLI_`. Delete gates for anything that predates the 7.2.0 fork, and rewrite the rest as "Since Valkey X.Y".
- **Features with no version label.** These are the inverse of the previous pattern: 9.x-only or search-1.3-only behavior shown as always present. Pages: acl geospatial replication search-configurables search-data-formats search-query sentinel streams-intro tls. Sweep: adopt one "Since Valkey X.Y" (or "Since valkey-search 1.3") convention in the valkey-doc README. Check each `00-RELEASENOTES` feature against the pages.
- **Non-inclusive or gendered prose.** "master"/"slave" in prose where aliases exist, plus gendered pronouns. Pages: cli cluster-spec cluster-tutorial debugging memory-optimization migration sentinel twitter-clone. Sweep: use primary/replica in prose and keep the legacy token only inside literal command or config names, with the alias named next to it.
- **Broken heading hierarchy.** Extra H1s from `#` or setext `===` under the template's H1, plus skipped levels (site-structure F6). Pages: cli cluster-tutorial index indexing lists lua-api modules-intro modules-native-types notifications programmability search-data-formats search-expressions search-query search-configurables search-observables sentinel sentinel-clients sorted-sets strings valkey-bundle. Sweep: markdownlint MD001, MD003 (atx), and MD025 with `front_matter_title`. Delete `# Valkey documentation` from topics/index.md.
- **Weak or unused front matter descriptions.** They are never emitted as `<meta name="description">` (site-structure F1). Where they exist, they often restate the title. Pages: ARM atomic-slot-migration benchmark bitfields bitmaps bloomfilters cli client-libraries data-types distlock eval-intro functions-intro hashes indexing introduction ldap lists lru-cache lua-api mass-insertion modules-api-ref problems quickstart search search-configurables search-expressions search-monitoring server sets site-pages sorted-sets streams-intro strings tls valkey-bundle valkey.conf valkey-json. Sweep: fix the template first (docs-page.html plus head.html, site-structure F1), then rewrite each description to say what the reader learns.
- **First-person and blog voice.** Pages: atomic-slot-migration benchmark debugging faq latency mass-insertion memory-optimization modules-native-types pipelining sets streams-intro strings valkey-json. Sweep: a style-guide rule ("second person, present tense, no author asides") applied page by page.
- **Stale, unsourced benchmark numbers.** These include 2010-era Redis figures, Raspberry Pi 1 runs and old EC2 generations. Pages: ARM benchmark faq memory-optimization RDMA search streams-intro. Sweep: either date and source each number with the Valkey version, or delete it.
- **Definition-list syntax renders as literal colons.** Zola does not render `term\n: definition`. Pages: benchmark cli server. Sweep: grep `^: ` in topics/ and convert the matches to bullet lists.
- **"Single scripting engine" is stale for 9.x.** Pages: eval-intro functions-intro programmability. Sweep: one shared sentence about the scripting engine module API, linked to modules-api-ref.
- **Commands and configs not linked, or generic link text.** Pages: bitfields client-libraries data-types faq geospatial index license modules-blocking-ops modules-native-types security sentinel-clients signals valkey.conf. Sweep: link the first mention of each command to `../commands/<cmd>.md` and ban "here"/"this link".
- **Typos and grammar.** Found on about 50 pages. valkey-doc already runs a spellcheck (see the wordlist commit a95bf25c), so these are mostly grammar. Sweep: a single copy-edit pass, not per-page PRs.

### Duplicate content

| Concept | Pages and lines | Canonical | Others |
|---|---|---|---|
| Search metrics and configs | search-monitoring (whole page), search-observables.md:14, search-configurables.md:10. Reported from three sides: search-monitoring F1, search-observables F4, search-configurables F4 | search-observables (metrics), search-configurables (configs) | Copy search-monitoring's Default Value column into search-configurables, delete search-monitoring, and redirect it to search-observables |
| Client feature definitions | website content/clients/_index.md:45-66 vs valkey-doc clients/advanced_client_features.md:1-20, which have already diverged (client-libraries F10, F15) | valkey-doc advanced_client_features.md | Have the website load that file the way it loads the JSONs, delete the website copy, and link Cluster Scan to SCAN instead of restating its guarantees |
| Intrinsic latency test | cli.md:962-995 vs latency.md:75-111. The two already drift (5 s vs 100 s) (cli F8) | latency | Cut cli to a usage paragraph and a link |
| Primary without persistence | admin.md:38 vs replication.md:42 (admin F7) | replication | One line in admin plus a link |
| min-replicas mitigation | sentinel.md:285-290 and 1166-1174, plus replication.md (sentinel F10) | replication for the configs, sentinel Example 2 for the scenario | Cut sentinel:1166-1174 to one line linking to Example 2 |
| THP and EC2 host advice, both outdated | admin.md:16 and 44-48 vs latency.md:23-24 and 248 (admin F3, F6; latency F2) | admin (OS setup) | Fix once in admin (`disable-thp yes` is the default, PV/HVM advice is obsolete), and have latency link to it |
| 128-bit big-endian index | indexing.md:403 vs sorted-sets.md:174-182 (indexing F12) | indexing | Cut sorted-sets to one sentence plus a link |
| RAM test | problems.md:10 vs quickstart.md:93. Both lack the size argument (problems F6, quickstart F3) | problems | Quickstart links to it. Fix the argument in problems |
| SCAN and KEYS semantics | keyspace.md:108-140, copied from commands/scan.md:8-10 and keys.md:8-29 (keyspace F2) | command pages | One paragraph plus links |
| Shutdown sequence | signals.md:22-24 and 36-39 vs commands/shutdown.md:3-5 and 69-72 (signals F2) | commands/shutdown.md | Keep the signal-specific parts and link |
| Single-instance lock | distlock.md:81 vs commands/set.md:54-75 (distlock F4) | distlock | Cut set.md to two sentences plus a link |
| SETSLOT replication and TIMEOUT | cluster-spec.md:534-555 vs commands/cluster-setslot.md (cluster-spec F9). Both carry the seconds error from cluster-spec F1 | commands/cluster-setslot.md | Keep the rationale in the spec and link the syntax |
| HELLO reply fields | protocol.md:566 vs commands/hello.md (protocol F7). protocol's copy is the wrong one (protocol F1) | commands/hello.md | Keep the 3 mandatory fields and link |
| XREAD `$`, blocking, `~` trimming | streams-intro.md:329 and 802 vs xread.md and xtrim.md (streams-intro F13) | command pages for options | Keep the tutorial narrative and link the options |
| Stop words and separators | search-data-formats.md:20 vs ft.create.md (search-data-formats F6) | search-data-formats | ft.create states the defaults in one line and links |
| Active expire algorithm | latency.md:530-546 vs commands/expire.md:140-169. The latency copy is wrong (latency F1) | commands/expire.md | Replace with a short summary and link |
| Repeats within a page | replication.md:443 (replication F6), introduction.md:23 (introduction F3), search.md:113 rendered twice (search F2), streams-intro front section (streams-intro F19) | n/a | Delete the second copy |

### Making the docs AI-native

Google's guide treats AI search as ordinary SEO. Unique, non-commodity content matters most. Pages must be indexable with a snippet, content must be crawlable without depending on JS, duplicates waste crawl, and headings help people. Semantic HTML is "not required" but "generally a good idea". The changes that matter, in order:

1. **Make snippets come from the page's own summary.** No topic page emits a meta description, although all 79 sources have one (site-structure F1). On 9 pages, including faq and history, the first text in `<main>` is the "likely incorrect" banner (site-structure F2). Fix the template, move the banner after the content or into `<aside>`, then rewrite the weak descriptions listed above.
2. **Lead with Valkey-specific facts and cut commodity filler.** About 35 pages have no scope paragraph (for example admin, cli, latency, persistence, modules-intro). About 19 pages have marketing or generic filler (quickstart F5, lists, strings, sorted-sets, valkey-json, security). Several pages describe tunables without the config name or default (admin, cluster-spec, faq, lru-cache, sentinel, atomic-slot-migration). Put config names, defaults and version behavior in the prose of the first paragraph. That is the content an LLM cannot produce on its own.
3. **Descriptive headings and one H1.** Vague headings ("Example", "Notes", "Imaging") on about 21 pages, and multiple H1s on 11+ pages (site-structure F6). Google says headings help, and that content does not need to be chunked further.
4. **Cut duplication and send clear duplicate signals.** Apply the duplicate table above. Add `rel="canonical"` (site-structure F4). End internal links with a slash so they don't cost a 301 (399 of 509, site-structure F5). Noindex or publish the /topics/license/ soft 404 (site-structure F3). Restore the /docs/topics/tls/ alias (site-structure F9).
5. **Server-render navigation.** The topic index on every topic page is fetched by JS from a headless fragment that is in the sitemap (site-structure F7). Include it at build time.
6. **Reader-trust signals (not a guide claim).** Add a version and "last updated" line, sitemap `lastmod` (site-structure F8), and code fence languages (site-structure F12). These help readers and parsers. The guide does not name them as AI ranking factors.

Things the guide says do not help Google: llms.txt, markdown copies of pages, and special AI markup or schema. Google Search ignores them, and they neither help nor harm it. They may help non-Google agents. The cheapest version is a public "View source" link to the raw markdown (site-structure F14): no new artifact to maintain, and the current edit link sends anonymous clients to a login page. BreadcrumbList JSON-LD (site-structure F15) is optional. It can enable rich results, but the guide says AI features need no structured data.

### Pages with no report

None. All 83 rows in index.tsv have a report and a verdict. None are NO-REPORT or MALFORMED.

## Scoreboard

| Page | Verdict | Blocker | Minor | Nit | Live page |
|---|---|---|---|---|---|
| [client-libraries](pages/client-libraries.md) | NEEDS-WORK | 4 | 9 | 5 | https://valkey.io/clients/ |
| [search-expressions](pages/search-expressions.md) | NEEDS-WORK | 3 | 5 | 4 | https://valkey.io/topics/search-expressions/ |
| [modules-blocking-ops](pages/modules-blocking-ops.md) | NEEDS-WORK | 3 | 4 | 4 | https://valkey.io/topics/modules-blocking-ops/ |
| [valkey-bundle](pages/valkey-bundle.md) | NEEDS-WORK | 3 | 4 | 3 | https://valkey.io/topics/valkey-bundle/ |
| [sentinel](pages/sentinel.md) | NEEDS-WORK | 2 | 9 | 6 | https://valkey.io/topics/sentinel/ |
| [search-query](pages/search-query.md) | NEEDS-WORK | 2 | 5 | 2 | https://valkey.io/topics/search-query/ |
| [RDMA](pages/RDMA.md) | NEEDS-WORK | 2 | 4 | 3 | https://valkey.io/topics/RDMA/ |
| [ldap](pages/ldap.md) | NEEDS-WORK | 2 | 4 | 4 | https://valkey.io/topics/ldap/ |
| [installation](pages/installation.md) | NEEDS-WORK | 2 | 3 | 6 | https://valkey.io/topics/installation/ |
| [cluster-spec](pages/cluster-spec.md) | NEEDS-WORK | 1 | 10 | 5 | https://valkey.io/topics/cluster-spec/ |
| [cli](pages/cli.md) | NEEDS-WORK | 1 | 9 | 5 | https://valkey.io/topics/cli/ |
| [cluster-tutorial](pages/cluster-tutorial.md) | NEEDS-WORK | 1 | 9 | 10 | https://valkey.io/topics/cluster-tutorial/ |
| [modules-api-ref](pages/modules-api-ref.md) | NEEDS-WORK | 1 | 7 | 3 | https://valkey.io/topics/modules-api-ref/ |
| [protocol](pages/protocol.md) | NEEDS-WORK | 1 | 6 | 4 | https://valkey.io/topics/protocol/ |
| [tls](pages/tls.md) | NEEDS-WORK | 1 | 6 | 5 | https://valkey.io/topics/tls/ |
| [search](pages/search.md) | NEEDS-WORK | 1 | 5 | 3 | https://valkey.io/topics/search/ |
| [command-arguments](pages/command-arguments.md) | NEEDS-WORK | 1 | 4 | 2 | https://valkey.io/topics/command-arguments/ |
| [eval-intro](pages/eval-intro.md) | NEEDS-WORK | 1 | 4 | 3 | https://valkey.io/topics/eval-intro/ |
| [lua-api](pages/lua-api.md) | NEEDS-WORK | 1 | 4 | 10 | https://valkey.io/topics/lua-api/ |
| [quickstart](pages/quickstart.md) | NEEDS-WORK | 1 | 4 | 4 | https://valkey.io/topics/quickstart/ |
| [security](pages/security.md) | NEEDS-WORK | 1 | 3 | 3 | https://valkey.io/topics/security/ |
| [modules-intro](pages/modules-intro.md) | NEEDS-WORK | 0 | 10 | 6 | https://valkey.io/topics/modules-intro/ |
| [site-structure](pages/site-structure.md) | NEEDS-WORK | 0 | 9 | 7 | https://valkey.io/topics/ |
| [streams-intro](pages/streams-intro.md) | NEEDS-WORK | 0 | 9 | 10 | https://valkey.io/topics/streams-intro/ |
| [admin](pages/admin.md) | NEEDS-WORK | 0 | 8 | 4 | https://valkey.io/topics/admin/ |
| [persistence](pages/persistence.md) | NEEDS-WORK | 0 | 8 | 5 | https://valkey.io/topics/persistence/ |
| [acl](pages/acl.md) | NEEDS-WORK | 0 | 7 | 8 | https://valkey.io/topics/acl/ |
| [indexing](pages/indexing.md) | NEEDS-WORK | 0 | 7 | 6 | https://valkey.io/topics/indexing/ |
| [notifications](pages/notifications.md) | NEEDS-WORK | 0 | 7 | 5 | https://valkey.io/topics/notifications/ |
| [replication](pages/replication.md) | NEEDS-WORK | 0 | 7 | 6 | https://valkey.io/topics/replication/ |
| [benchmark](pages/benchmark.md) | NEEDS-WORK | 0 | 6 | 6 | https://valkey.io/topics/benchmark/ |
| [client-side-caching](pages/client-side-caching.md) | NEEDS-WORK | 0 | 6 | 4 | https://valkey.io/topics/client-side-caching/ |
| [lists](pages/lists.md) | NEEDS-WORK | 0 | 6 | 6 | https://valkey.io/topics/lists/ |
| [search-data-formats](pages/search-data-formats.md) | NEEDS-WORK | 0 | 6 | 4 | https://valkey.io/topics/search-data-formats/ |
| [sentinel-clients](pages/sentinel-clients.md) | NEEDS-WORK | 0 | 6 | 3 | https://valkey.io/topics/sentinel-clients/ |
| [twitter-clone](pages/twitter-clone.md) | NEEDS-WORK | 0 | 6 | 7 | https://valkey.io/topics/twitter-clone/ |
| [bitmaps](pages/bitmaps.md) | NEEDS-WORK | 0 | 5 | 3 | https://valkey.io/topics/bitmaps/ |
| [clients](pages/clients.md) | NEEDS-WORK | 0 | 5 | 4 | https://valkey.io/topics/clients/ |
| [faq](pages/faq.md) | NEEDS-WORK | 0 | 5 | 4 | https://valkey.io/topics/faq/ |
| [functions-intro](pages/functions-intro.md) | NEEDS-WORK | 0 | 5 | 5 | https://valkey.io/topics/functions-intro/ |
| [mass-insertion](pages/mass-insertion.md) | NEEDS-WORK | 0 | 5 | 4 | https://valkey.io/topics/mass-insertion/ |
| [migration](pages/migration.md) | NEEDS-WORK | 0 | 5 | 4 | https://valkey.io/topics/migration/ |
| [modules-native-types](pages/modules-native-types.md) | NEEDS-WORK | 0 | 5 | 5 | https://valkey.io/topics/modules-native-types/ |
| [pubsub](pages/pubsub.md) | NEEDS-WORK | 0 | 5 | 4 | https://valkey.io/topics/pubsub/ |
| [search-configurables](pages/search-configurables.md) | NEEDS-WORK | 0 | 5 | 2 | https://valkey.io/topics/search-configurables/ |
| [search-monitoring](pages/search-monitoring.md) | NEEDS-WORK | 0 | 5 | 3 | https://valkey.io/topics/search-monitoring/ |
| [sets](pages/sets.md) | NEEDS-WORK | 0 | 5 | 4 | https://valkey.io/topics/sets/ |
| [valkey.conf](pages/valkey.conf.md) | NEEDS-WORK | 0 | 5 | 4 | https://valkey.io/topics/valkey.conf/ |
| [bloomfilters](pages/bloomfilters.md) | MINOR-ISSUES | 0 | 4 | 8 | https://valkey.io/topics/bloomfilters/ |
| [debugging](pages/debugging.md) | MINOR-ISSUES | 0 | 4 | 3 | https://valkey.io/topics/debugging/ |
| [distlock](pages/distlock.md) | MINOR-ISSUES | 0 | 4 | 5 | https://valkey.io/topics/distlock/ |
| [hyperloglogs](pages/hyperloglogs.md) | MINOR-ISSUES | 0 | 4 | 3 | https://valkey.io/topics/hyperloglogs/ |
| [key-specs](pages/key-specs.md) | MINOR-ISSUES | 0 | 4 | 4 | https://valkey.io/topics/key-specs/ |
| [keyspace](pages/keyspace.md) | MINOR-ISSUES | 0 | 4 | 4 | https://valkey.io/topics/keyspace/ |
| [latency](pages/latency.md) | MINOR-ISSUES | 0 | 4 | 5 | https://valkey.io/topics/latency/ |
| [latency-monitor](pages/latency-monitor.md) | MINOR-ISSUES | 0 | 4 | 5 | https://valkey.io/topics/latency-monitor/ |
| [ldb](pages/ldb.md) | MINOR-ISSUES | 0 | 4 | 4 | https://valkey.io/topics/ldb/ |
| [lru-cache](pages/lru-cache.md) | MINOR-ISSUES | 0 | 4 | 4 | https://valkey.io/topics/lru-cache/ |
| [memory-optimization](pages/memory-optimization.md) | MINOR-ISSUES | 0 | 4 | 4 | https://valkey.io/topics/memory-optimization/ |
| [performance-on-cpu](pages/performance-on-cpu.md) | MINOR-ISSUES | 0 | 4 | 6 | https://valkey.io/topics/performance-on-cpu/ |
| [pipelining](pages/pipelining.md) | MINOR-ISSUES | 0 | 4 | 4 | https://valkey.io/topics/pipelining/ |
| [search-observables](pages/search-observables.md) | MINOR-ISSUES | 0 | 4 | 2 | https://valkey.io/topics/search-observables/ |
| [server](pages/server.md) | MINOR-ISSUES | 0 | 4 | 3 | https://valkey.io/topics/server/ |
| [signals](pages/signals.md) | MINOR-ISSUES | 0 | 4 | 5 | https://valkey.io/topics/signals/ |
| [site-pages](pages/site-pages.md) | MINOR-ISSUES | 0 | 4 | 6 | https://valkey.io/docs/ |
| [sorted-sets](pages/sorted-sets.md) | MINOR-ISSUES | 0 | 4 | 5 | https://valkey.io/topics/sorted-sets/ |
| [transactions](pages/transactions.md) | MINOR-ISSUES | 0 | 4 | 3 | https://valkey.io/topics/transactions/ |
| [ARM](pages/ARM.md) | MINOR-ISSUES | 0 | 3 | 3 | https://valkey.io/topics/ARM/ |
| [atomic-slot-migration](pages/atomic-slot-migration.md) | MINOR-ISSUES | 0 | 3 | 4 | https://valkey.io/topics/atomic-slot-migration/ |
| [bitfields](pages/bitfields.md) | MINOR-ISSUES | 0 | 3 | 4 | https://valkey.io/topics/bitfields/ |
| [geospatial](pages/geospatial.md) | MINOR-ISSUES | 0 | 3 | 3 | https://valkey.io/topics/geospatial/ |
| [hashes](pages/hashes.md) | MINOR-ISSUES | 0 | 3 | 3 | https://valkey.io/topics/hashes/ |
| [index](pages/index.md) | MINOR-ISSUES | 0 | 3 | 5 | https://valkey.io/topics/ |
| [introduction](pages/introduction.md) | MINOR-ISSUES | 0 | 3 | 4 | https://valkey.io/topics/introduction/ |
| [programmability](pages/programmability.md) | MINOR-ISSUES | 0 | 3 | 4 | https://valkey.io/topics/programmability/ |
| [releases](pages/releases.md) | MINOR-ISSUES | 0 | 3 | 2 | https://valkey.io/topics/releases/ |
| [strings](pages/strings.md) | MINOR-ISSUES | 0 | 3 | 5 | https://valkey.io/topics/strings/ |
| [valkey-json](pages/valkey-json.md) | MINOR-ISSUES | 0 | 3 | 6 | https://valkey.io/topics/valkey-json/ |
| [command-tips](pages/command-tips.md) | MINOR-ISSUES | 0 | 2 | 4 | https://valkey.io/topics/command-tips/ |
| [data-types](pages/data-types.md) | MINOR-ISSUES | 0 | 2 | 5 | https://valkey.io/topics/data-types/ |
| [history](pages/history.md) | MINOR-ISSUES | 0 | 2 | 4 | https://valkey.io/topics/history/ |
| [license](pages/license.md) | MINOR-ISSUES | 0 | 2 | 1 | https://valkey.io/topics/license/ |
| [problems](pages/problems.md) | MINOR-ISSUES | 0 | 2 | 4 | https://valkey.io/topics/problems/ |

This was generated by AI but verified, with love, by a human.
