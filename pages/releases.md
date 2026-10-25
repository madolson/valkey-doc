VERDICT: MINOR-ISSUES
COUNTS: blocker=0 minor=3 nit=2

# Releases and versioning schema (`releases`)

https://valkey.io/topics/releases/

Support table dates match the release tags and all links resolve. The most important fix is the replication compatibility claim in Major versions, which a 9.x primary with an 8.x replica contradicts.

## Findings

### F1 [minor] INACCURATE: Replication stream is not backward compatible across majors (`topics/releases.md:56`)
**Doc says:**
> Although Major versions can introduce breaking changes, data replicated from primaries to replicas will always be sent in a backward compatible format.
**Problem:** A 9.x primary sends an RDB stamped `VALKEY080`. Older replicas reject it, and by default even a 9.x replica rejects any RDB version newer than its own. The next sentence (upgrade replicas first) is right only because this one is false. A reader who trusts line 56 may upgrade the primary first.
**Evidence:** `valkey@9.1.2:src/rdb.h:52` `#define RDB_VERSION 80`. `valkey@8.1.0:src/rdb.h:53` `#define RDB_VERSION 11`. `valkey@9.1.2:src/rdb.c:89-90`:
```
    if (server.rdb_version_check == RDB_VERSION_CHECK_STRICT) {
        if (rdbver > RDB_VERSION) return false; /* future version */
```
`valkey@9.1.2:valkey.conf:614` `rdb-version-check strict`, and the comment at 607-613 says "This also applies to replication".
**Fix:** Replace with: "A replica can always replicate from a primary running the same or an older version. A replica running an older version may not be able to load data from a newer primary (see the `rdb-version-check` config), so always upgrade replicas before primaries."

### F2 [minor] CONFUSING: Table assigns 9.1 extended security support before the policy allows it (`topics/releases.md:99`)
**Doc says:**
> | 9.1 | 2026-05-19 | 2029-05-19 | 2031-05-19 |
**Problem:** Line 90 says the minor that gets 5-year security support "will be decided once the next major version has been launched". Valkey 10 has not launched, and `9.2.0-rc1` is already tagged, so 9.1 is unlikely to be the last 9.x minor. The table promises 2031 anyway.
**Evidence:** Line 90: "The minor version to be used for this extended security support will be decided once the next major version has been launched." `git -C ~/git/valkey tag` lists `9.2.0-rc1`.
**Fix:** Set the 9.1 security support end to "TBD (2031-05-19 if 9.1 is the last 9.x minor)", or match the 9.0 row (2029-05-19). Add 9.2 when it goes GA.
Tracked: valkey-io/valkey-io.github.io#275 (manual table maintenance)

### F3 [minor] CONFUSING: Minor-version guidance contradicts rolling upgrades (`topics/releases.md:40-44`)
**Doc says:**
> Upgrading from a previous minor version should be safe, and will not introduce incompatibilities between servers in the cluster when default server configurations are used.
> ...
> For this reason, it is not recommended to run a Valkey cluster with servers running on different minor versions.
**Problem:** A rolling upgrade runs mixed minor versions by definition. The reader cannot tell whether mixed versions are fine during the upgrade or must be avoided entirely. "This type of incompatibility" also refers back two sentences.
**Fix:** Replace lines 42-44 with: "Running servers on different minor versions is supported only while a rolling upgrade is in progress. New commands, data types, and config options can break compatibility between servers, so don't use features added in the new minor version until every server in the cluster runs it."

### F4 [nit] AI-NATIVE: Opening is filler and doesn't say what the page covers (`topics/releases.md:6-7`)
**Doc says:**
> Valkey is usually among the most critical pieces of a software stack.
> For this reason, Valkey's release cycle prioritizes highly stable releases at the cost of slower release cycles.
**Problem:** The first sentence could describe any database. The paragraph doesn't say what the page covers or who it's for.
**Fix:** Replace with: "This page describes how Valkey versions are numbered, what compatibility each patch, minor, and major release guarantees, how release candidates become stable releases, and how long each version gets bug and security fixes. Read it before you plan an upgrade."

### F5 [nit] STYLE: Wording and grammar (`topics/releases.md:13`, `topics/releases.md:52`)
**Doc says:**
> Valkey stable releases will generally follow `major.minor.patch` [semantic versioning schema](https://semver.org/).

> Examples of significant functionality includes altering the behavior
**Problem:** "Schema" should be "scheme", and the title has the same error. "Will generally" hedges a guarantee that line 14 then states as explicit. On line 52, "Examples ... includes" has a subject-verb agreement error.
**Fix:** Line 13: "Valkey stable releases follow the `major.minor.patch` [semantic versioning scheme](https://semver.org/)." Line 52: "Examples of significant functionality include ...". Title: "Releases and versioning scheme".
