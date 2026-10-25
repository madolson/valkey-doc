VERDICT: NEEDS-WORK
COUNTS: blocker=4 minor=9 nit=5

# Client Libraries (`client-libraries`)

https://valkey.io/clients/

The feature table, which is what the page exists for, is wrong in both directions: it credits three clients with features they lack (GLIDE C# connection pool, Redisson cluster scan, GLIDE PHP sharded pubsub restoration), it misses features four clients now ship, and every listed version is stale. Fix the table JSONs first. Then remove the duplicated feature definitions so only one copy exists.

## Findings

### F1 [blocker] INACCURATE: GLIDE C# marked as having a persistent connection pool (`clients/csharp/valkey-GLIDE.json:19`)
**Doc says:**
> "persistent_connection_pool": true
**Problem:** GLIDE C# 1.1.0 has no connection pool. Like every other GLIDE binding, which the table marks false, it uses glide-core's multiplexed connections.
**Evidence:** valkey-io/valkey-glide-csharp@v1.1.0:sources/Valkey.Glide/Abstract/ConnectionMultiplexer.cs:17 defines a `ConnectionMultiplexer` over glide-core. Grepping `connection pool|ConnectionPool` in `sources/` and `rust/src` matches only .NET `ThreadPool` and `ArrayPool<byte>`.
**Fix:** `"persistent_connection_pool": false`.

### F2 [blocker] INACCURATE: Redisson marked as supporting Cluster Scan (`clients/java/redisson.json:22`)
**Doc says:**
> "cluster_scan": true
**Problem:** The page defines Cluster Scan as a scan iterator that "remains valid even during failovers or cluster scaling". Redisson takes a one-time snapshot of the primaries, runs a separate SCAN cursor against each, and resets a node's cursor to 0 when that node disappears. Keys that move between nodes during the scan can be missed or returned twice, and the caller has no resumable cursor.
**Evidence:** redisson/redisson@redisson-4.8.0:redisson/src/main/java/org/redisson/RedissonKeys.java:137 `for (MasterSlaveEntry entry : ...getEntrySet())`. iterator/BaseIterator.java: `catch (RedisNodeNotFoundException e) { ... nextIterPos = initValue(); reset(); }`.
**Fix:** `"cluster_scan": false`.

### F3 [blocker] INACCURATE: GLIDE PHP marked as restoring PubSub state, but sharded PubSub is not implemented (`clients/php/valkey-GLIDE.json:13`)
**Doc says:**
> "pubsub_state_restoration": true

The feature definition (content/clients/_index.md:52) says: "for both legacy Pub/Sub and sharded Pub/Sub".
**Problem:** GLIDE PHP 1.1.2 has no `ssubscribe`, so it cannot restore sharded subscriptions, and the feature as the page defines it is not supported.
**Evidence:** valkey-io/valkey-glide-php@v1.1.2:valkey_glide.c:1222 `PHP_METHOD(ValkeyGlide, ssubscribe) { /* TODO: Implement */`. valkey_glide.stub.php:3231 `/* TODO public function ssubscribe(...) */`. The cluster stub has no `ssubscribe`.
**Fix:** `"pubsub_state_restoration": false` until `ssubscribe` ships, or add `"specific_properties": ["PubSub restoration for non-sharded channels only"]`.

### F4 [blocker] EXAMPLE: valkey-swift install command fails as written (`clients/swift/valkey-swift.json:5`)
**Doc says:**
> swift package add-dependency https://github.com/valkey-io/valkey-swift
**Problem:** SwiftPM's `add-dependency` requires a version requirement flag and errors without one.
**Evidence:** swiftlang/swift-package-manager@main:Sources/Commands/PackageCommands/AddDependency.swift:134-137
```
guard let firstRequirement = requirements.first else {
    throw StringError("must specify one of --exact, --branch, --revision, --from, or --up-to-next-minor-from")
}
```
**Fix:** `swift package add-dependency https://github.com/valkey-io/valkey-swift --from 1.5.0`

### F5 [minor] INACCURATE: Features that clients ship are marked unsupported (`clients/csharp/valkey-GLIDE.json:16`, `clients/java/redisson.json:24`, `clients/python/valkey-py.json:16`, `clients/swift/valkey-swift.json:15`)
**Doc says:**
> "AZ_based_read_from_replica": false (GLIDE C#, redisson, valkey-swift); "client_side_caching": false (valkey-py)
**Problem:** Each of these features exists in the client's current release. The valkey-py entry was already wrong at the 6.1.0 version the page lists.
**Evidence:**
- GLIDE C# 1.1.0: sources/Valkey.Glide/ConnectionConfiguration.cs:183,189 `AzAffinity,` / `AzAffinityReplicasAndPrimary,`. rust/src/ffi.rs:240 `ReadFromStrategy::AZAffinity => { coreReadFrom::AZAffinity(...)`.
- Redisson 4.8.0: CHANGELOG.md:21 "Valkey 8.0+ availability zone support through `AZ_AFFINITY`... read modes and `clientAvailabilityZone` setting".
- valkey-py v6.1.0 and v6.2.0: valkey/connection.py:455-458 `if self.client_cache: self.send_command("CLIENT", "TRACKING", "ON")`. Requires RESP3.
- valkey-swift 1.5.0: Sources/Valkey/ValkeyClientConfiguration.swift:222 `public static func az(_ zone: String, ...)`. Release notes: "Cluster readonly node selection based on AZ (Only works on Valkey 9.1 and later)".
**Fix:** Set these four values to `true`. Bump the versions per F7. For valkey-py add `"specific_properties": ["Client-side caching requires RESP3"]`. For valkey-swift add `"specific_properties": ["AZ-based reads in cluster mode, Valkey 9.1+"]`.
Tracked: valkey-io/valkey-doc#506 (Redisson AZ only)

### F6 [minor] INACCURATE: Feature claims that hold only in narrow cases, with no qualifier (`clients/php/predis.json:11`, `clients/go/valkey-go.json:12`)
**Doc says:**
> predis "smart_backoff_to_prevent_connection_storm": true (at "version":"v2.3.0"); valkey-go "pubsub_state_restoration": true
**Problem:** predis gained retry with backoff in v3.4.0, so the claim was false at the v2.3.0 the page lists. Even now it is off by default. valkey-go resubscribes after a reconnect, but slot migrations end sharded subscriptions and the caller has to resubscribe, which contradicts "after ... topology updates such as adding or removing shards".
**Evidence:** predis CHANGELOG v3.4.0 "Added optional retry support (#1616)". predis@v3.7.0:src/Connection/Parameters.php:53 `new Retry(new NoBackoff(), 0)`. valkey-go@v1.0.79:README.md:304 says `Receive()` returns nil on "`sunsubscribe` messages caused by slot migrations".
**Fix:** predis: bump the version to v3.7.0 and add `"specific_properties": ["Retry backoff is opt-in"]`. valkey-go: add `"specific_properties": ["Sharded subscriptions are not restored after slot migration"]`, or set the value to false.

### F7 [minor] OUTDATED: Every client's version and release date is stale (all `clients/*/*.json` `"version"` lines)
**Doc says:**
> GLIDE Python/Node/Java/Go v2.5.1, valkey-py 6.1.0, iovalkey v0.3.1, valkey-java v5.3.0, redisson v4.7.0, valkey-go 1.0.67, phpredis 6.1.0, predis v2.3.0, valkey-swift v1.0.0
**Problem:** Most entries are one or more releases behind, and some are two years old. Readers judge a client's maintenance status by these dates, so valkey-java (listed 2024-08-08) and predis look abandoned when they are not.
**Evidence:** `gh release list -L 3` latest stable: valkey-glide v2.5.3 (2026-09-25), valkey-py v6.2.0 (2026-10-12), iovalkey v0.4.0 (2026-07-27), valkey-java v5.5.0 (2025-10-10), redisson 4.8.0 (2026-10-02), valkey-go 1.0.79 (2026-10-15), phpredis 6.3.0 (2025-11-06), predis v3.7.0 (2026-10-08), valkey-swift 1.5.0 (2026-08-17). C#, PHP GLIDE and Ruby GLIDE are current.
**Fix:** Update `version` and `version_released` to these values. The hand-maintained fields will drift again, so either drop them or have the website build fill them from the GitHub releases API.

### F8 [minor] EXAMPLE: GLIDE Java install pins 1.x while the page lists v2.5.1 (`clients/java/valkey-GLIDE.json:9`, `:13`)
**Doc says:**
> `<version>[1.0.0,2.0.0)</version>` and `version: '1.+'`
**Problem:** Both snippets resolve to GLIDE 1.x, a different major version from the "v2.5.1" shown and from the 2.x quickstart linked as the website. The classifier list also leaves out platforms that 2.x ships.
**Evidence:** Maven Central io/valkey/valkey-glide maven-metadata.xml: latest stable 2.5.3. 2.5.3 also ships `osx-x86_64`, `linux_musl-aarch_64`, `linux_musl-x86_64`, `windows-x86_64`.
**Fix:** Maven `<version>[2.0.0,3.0.0)</version>`, Gradle `version: '2.+'`. Classifier comment: `osx-aarch_64 OR osx-x86_64 OR linux-aarch_64 OR linux-x86_64 OR linux_musl-aarch_64 OR linux_musl-x86_64 OR windows-x86_64`.

### F9 [minor] INACCURATE: GLIDE Ruby license is not established by the repo (`clients/ruby/valkey-GLIDE.json:10`)
**Doc says:**
> "license": "Apache-2.0"
**Problem:** valkey-glide-ruby has no LICENSE file. The gemspec declares no license, RubyGems shows `"licenses":[]`, and GitHub reports NOASSERTION. The only claim is a README line that links to the missing LICENSE file.
**Evidence:** `gh api repos/valkey-io/valkey-glide-ruby --jq .license.spdx_id` returns `NOASSERTION`. raw.githubusercontent.com/valkey-io/valkey-glide-ruby/main/LICENSE returns 404. README.md:355: "Apache-2.0 — see [LICENSE](...)".
**Fix:** Ask the GLIDE maintainers to add a LICENSE file and set `spec.license = "Apache-2.0"` in the gemspec. Keep the page value once that lands.

### F10 [minor] DUPLICATE: Feature definitions exist in two places and have already diverged (`content/clients/_index.md:45-66` vs `clients/advanced_client_features.md:1-20`)
**Doc says:**
> website _index.md:62: "please refer to [this link.](https://github.com/valkey-io/valkey/pull/325)"
> valkey-doc advanced_client_features.md:18: "please refer to [this link.](https://valkey.io/commands/client-capa/)"
**Problem:** The live page renders the website copy (templates/client-list.html renders `content_parts[1]` of the section content), so edits to valkey-doc's file never reach readers. The two copies already link to different targets for CAPA. clients/README.md:20 sends contributors to the copy that isn't rendered.
**Evidence:** The live page HTML contains `href="https://github.com/valkey-io/valkey/pull/325"`. Grepping the website repo for `advanced_client_features` finds no hits.
**Fix:** Make valkey-doc `clients/advanced_client_features.md` canonical and have the website load it, the way it loads the JSONs. Then delete lines 43-66 of website `_index.md`. Until that lands, make the CAPA link in both copies point to `/commands/client-capa/`.

### F11 [minor] INACCURATE: CAPA redirect description omits the READONLY exception and what the client must do (`content/clients/_index.md:62`, `clients/advanced_client_features.md:18`)
**Doc says:**
> When enabled, this feature allows the replica to redirect data access commands (both read and write operations) to the primary instance.
**Problem:** The replica does not forward anything. It replies `-REDIRECT host:port` and the client must reconnect to the primary. A client that has sent READONLY still gets read commands served locally. A reader who sets up read-from-replica with CAPA redirect would expect every read to be redirected.
**Evidence:** valkey@9.1.2:src/server.c:4435-4436
```
if (clientSupportStandAloneRedirect(c) && !obey_client &&
    (is_write_command || (is_read_command && !c->flag.readonly))) {
```
Live 9.1.2 replica on port 21601:
```
CLIENT CAPA redirect -> OK
GET k                -> REDIRECT 127.0.0.1:21600
SET k v              -> REDIRECT 127.0.0.1:21600
READONLY             -> OK
GET k                -> (nil)
```
**Fix:** "Introduced in Valkey 8.0 for standalone (non-cluster) deployments. A client that sends `CLIENT CAPA redirect` receives `-REDIRECT <primary-host>:<port>` from a replica for write commands, and for read commands unless the connection has sent `READONLY`. A client that supports this feature follows the redirect to the new primary, so a primary/replica switchover during an upgrade does not surface errors to the application. See [CLIENT CAPA](/commands/client-capa/)."

### F12 [minor] MISSING: AZ-based read never names the server config it depends on (`content/clients/_index.md:58`, `clients/advanced_client_features.md:14`)
**Doc says:**
> For more detailed information about this feature and its implementation, please refer to [this link.](https://github.com/valkey-io/valkey/pull/700)
**Problem:** AZ affinity only works if the operator sets `availability-zone` on every node. The page never says so and instead links to a 2024 PR titled "Upstream the availability zone info string from KeyDB".
**Evidence:** valkey@9.1.2:src/config.c:3357 `createSDSConfig("availability-zone", NULL, MODIFIABLE_CONFIG, ALLOW_EMPTY_STRING, server.availability_zone, "", ...)`. The default is empty, and the config has existed since 8.0.0 (8.0.0:src/config.c:3145). The 9.1.2 release notes: "`CLUSTER SHARDS`/`CLUSTER SLOTS` now include an `availability-zone` field (#3156)".
**Fix:** Replace the last sentence with: "Requires Valkey 8.0 or later with the `availability-zone` config set on each node (default: empty). The client must be configured with its own AZ."

### F13 [minor] BROKEN: "To add your client" links to a README with no instructions for adding a client (`content/clients/_index.md:41`, `clients/README.md:1-20`)
**Doc says:**
> To add your client to the list, please refer to [this link.](https://github.com/valkey-io/valkey-doc/blob/main/clients/README.md)
**Problem:** The README lists JSON fields but gives no submission process or acceptance criteria. That matters because the page says it lists "only clients which are regularly tested and recommended". The README also documents a `package_size` field that no JSON uses, and it omits the `version`, `version_released`, `license`, `website`, `name` and `specific_properties` fields that the template renders. It has typos: "contect", "despcription".
**Evidence:** clients/README.md:13 "**`package_size`** - the library's unpacked package size". `grep -l package_size clients/*/*.json` finds nothing. templates/client-list.html renders `client.version`, `client.version_released`, `client.license`, `client.website`.
**Fix:** In the README, add a "Submitting a client" section: open a PR adding `clients/<language>/<name>.json`, add the path to `recommended_clients_paths` in website `content/clients/_index.md`, and state the inclusion criteria. Replace `package_size` with the fields actually used, and fix the typos. Change the page sentence to "To propose a client, follow the [submission instructions](https://github.com/valkey-io/valkey-doc/blob/main/clients/README.md)."

### F14 [nit] OUTDATED: iovalkey description still says "Redis client" (`clients/node.js/iovalkey.json:3`)
**Doc says:**
> A robust, performance-focused and full-featured Redis client for Node.js.
**Evidence:** iovalkey@v0.4.0:package.json description: "A robust, performance-focused and full-featured Valkey/Redis client for Node.js."
**Fix:** Use the package.json text.

### F15 [nit] DUPLICATE: Cluster Scan definition restates the SCAN guarantees from commands/scan.md (`content/clients/_index.md:54`, `clients/advanced_client_features.md:10`)
**Doc says:**
> A complete iteration with SCAN retrieves all elements present in the collection from start to finish. If an element exists in the collection at the beginning and remains until the end of the iteration, SCAN will return it. Conversely, any element removed before the iteration begins...
**Evidence:** commands/scan.md:66-67 states the same two guarantees.
**Fix:** "**Cluster Scan** - A single cursor that iterates every key in a cluster with the same [SCAN guarantees](/commands/scan/) as a single node, and stays valid across failovers, resharding and scaling."

### F16 [nit] STYLE: Generic "this link" link text (`content/clients/_index.md:41`, `:58`, `:62`)
**Doc says:**
> please refer to [this link.](...)
**Fix:** Use descriptive link text, as in F11, F12 and F13.

### F17 [nit] AI-NATIVE: No meta description, and the intro doesn't say how clients are chosen (`content/clients/_index.md:1-4`, `:39`)
**Doc says:**
> This page offers an overview of recommended Valkey clients for various programming languages. A table of advanced features supported by the respective clients is provided, highlighting the unique advantages of one client over another.
**Problem:** The front matter has no `description`, and the live page has no `<meta name="description">`. The intro is generic and never says what "recommended" means or who maintains these clients.
**Fix:** Add `description = "Recommended Valkey client libraries by language, with install commands and a comparison of cluster, replica-read, and failover features."` to the front matter. Replace the intro with: "Client libraries the Valkey project recommends for Python, Node.js, Java, Go, PHP, Swift, C#, and Ruby, with install commands and a table comparing their cluster and high-availability features. Any client that supports Redis OSS 7.2 also works with Valkey 7.2 and later."

### F18 [nit] AI-NATIVE: "Latency Based Read From Replica" column is false for every client (`content/clients/_index.md:29`)
**Doc says:**
> "latency_based_read_from_replica",
**Problem:** All 15 rows show false, so the column tells the reader nothing and widens a table that is already wide.
**Evidence:** glide-core `ReadFrom` (glide-core/src/client/types.rs:154-165) has only Primary, PreferReplica, AZAffinity, AZAffinityReplicasAndPrimary, AllNodes and AZAffinityAllNodes. No other listed client implements latency-based selection.
**Fix:** Drop the column and its definition until a listed client implements it.

## Unverified
- clients/go/valkey-go.json:5: `go get github.com/valkey-io/valkey-go`. Confirmed the module path in go.mod. Could not run the command because proxy.golang.org is blocked from this host.
- clients/csharp/valkey-GLIDE.json:13 and clients/ruby/valkey-GLIDE.json: C# 1.1.0 and Ruby 1.0.0 pin older glide-core commits (bf74288, f1ce422). Confirmed through the GitHub API that `pubsub/synchronizer.rs` exists at both commits. Did not verify that the C# binding restores sharded subscriptions end to end.
- GLIDE `client_side_caching: false` for all bindings: correct under the CLIENT TRACKING definition. GLIDE 2.4+ (Java/Node/Go), C# 1.1.0 and PHP 1.1.2 do ship TTL-only client caches. The page does not say which kind of caching the column means.
