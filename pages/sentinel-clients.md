VERDICT: NEEDS-WORK
COUNTS: blocker=0 minor=6 nit=3

# Sentinel client spec (`sentinel-clients`)

https://valkey.io/topics/sentinel-clients/

The protocol is still correct, but the page tells client authors to implement commands that have been deprecated since 8.0.0, and it claims there is no primary-named alternative. Fix that first: make `SENTINEL GET-PRIMARY-ADDR-BY-NAME` the primary instruction and keep `GET-MASTER-ADDR-BY-NAME` only as a fallback for older servers.

## Findings

### F1 [minor] OUTDATED: Spec mandates deprecated `SENTINEL GET-MASTER-ADDR-BY-NAME` (`topics/sentinel-clients.md:51`)
**Doc says:**
> SENTINEL get-master-addr-by-name master-name

The same command also appears at lines 101, 123, and 129.
**Problem:** Valkey 8.0.0 added `GET-PRIMARY-ADDR-BY-NAME` and deprecated the master form. A new client written from this spec will be built on a deprecated command.
**Evidence:** valkey@9.1.2:src/commands/sentinel-get-master-addr-by-name.json:10-11
```
"deprecated_since": "8.0.0",
"replaced_by": "`SENTINEL GET-PRIMARY-ADDR-BY-NAME`",
```
valkey@9.1.2:src/commands/sentinel-get-primary-addr-by-name.json:6 `"since": "8.0.0"`. On a live 9.1.2 sentinel:
```
$ valkey-cli -p 21321 sentinel get-primary-addr-by-name mymaster
127.0.0.1
21320
```
**Fix:** At lines 51 and 101, use `SENTINEL GET-PRIMARY-ADDR-BY-NAME <primary-name>`. At lines 123 and 129, use `GET-PRIMARY-ADDR-BY-NAME`. After line 53, add: "`SENTINEL GET-PRIMARY-ADDR-BY-NAME` is available since Valkey 8.0.0. To support older Valkey or Redis OSS Sentinels, fall back to `SENTINEL GET-MASTER-ADDR-BY-NAME`, which returns the same reply."

### F2 [minor] OUTDATED: Closing note says master/slave are unavoidable in the protocol (`topics/sentinel-clients.md:152`)
**Doc says:**
> Unfortunately in the given commands these words are part of the protocol, so we’ll be able to remove such occurrences only when this API will be naturally deprecated.
**Problem:** This is false for 9.1.2. Every Sentinel command this page uses has a primary-named form, and the master forms are already deprecated.
**Evidence:** valkey@9.1.2:src/sentinel.c:3904-3905 accepts both `get-primary-addr-by-name` and `get-master-addr-by-name`. Line 3823 accepts both `primaries` and `masters`. In `SENTINEL HELP` on 9.1.2, every subcommand is listed as `GET-PRIMARY-ADDR-BY-NAME <primary-name>`, `PRIMARIES`, `REPLICAS <primary-name>`, and so on.
**Fix:** Once F1 is applied, delete the note. If you keep a note, replace it with: "The `ROLE` reply still reports `master` and `slave` for backward compatibility."

### F3 [minor] INACCURATE: Sentinel also kills Pub/Sub clients, not only `type normal` (`topics/sentinel-clients.md:87`)
**Doc says:**
> it sends a `CLIENT KILL type normal` command to the instance in order to make sure all the clients are disconnected
**Problem:** Sentinel sends both `CLIENT KILL TYPE normal` and `CLIENT KILL TYPE pubsub`. Client authors need to know that subscriber connections are dropped too, and must resolve the primary address again on reconnect.
**Evidence:** valkey@9.1.2:src/sentinel.c:4922-4924 (sentinelSendReplicaOf). The same pattern is at 4835-4838 (failover reconfiguration).
```
for (int type = 0; type < 2; type++) {
    retval = valkeyAsyncCommand(ri->link->cc, sentinelDiscardReplyCallback, ri, "%s KILL TYPE %s",
                                sentinelInstanceMapCommand(ri, "CLIENT"), type == 0 ? "normal" : "pubsub");
```
**Fix:** Replace the text with: "it sends `CLIENT KILL TYPE normal` and `CLIENT KILL TYPE pubsub` to the instance, so all regular and Pub/Sub clients are disconnected from the reconfigured instance."

### F4 [minor] MISSING: Step 3 does not say what `ROLE` returns (`topics/sentinel-clients.md:66`)
**Doc says:**
> call the `ROLE` command in order to verify the role of the instance is actually a primary.
**Problem:** The page uses "primary" and "replica" everywhere, but on 9.1.2 `ROLE` returns the literal strings `master` and `slave`. A client author who matches on "primary" will reject every valid primary and loop forever on Step 1.
**Evidence:** Live 9.1.2:
```
$ valkey-cli -p 21320 role
1) "master"
2) (integer) 0
...
```
**Fix:** Add after line 67: "The first element of the `ROLE` reply is the string `master` for a primary and `slave` for a replica." Make the same reference at line 109.

### F5 [minor] MISSING: Replica discovery gives no guidance on which replicas to use (`topics/sentinel-clients.md:105`)
**Doc says:**
> SENTINEL replicas primary-name
>
> In order to retrieve a list of replica instances.
**Problem:** The page does not say the reply is a list of field/value maps, which fields hold the address, or that a client should skip replicas Sentinel already considers down. A naive client will connect to failed replicas.
**Evidence:** valkey@9.1.2:src/sentinel.c:3324 emits the `ip` field. Lines 3337 and 3342 emit `s_down` and `disconnected` flags:
```
if (ri->flags & SRI_S_DOWN) flags = sdscat(flags, "s_down,");
if (ri->link->disconnected) flags = sdscat(flags, "disconnected,");
```
**Fix:** After line 107, add: "Each entry is a map of fields. Use `ip` and `port` for the address. Skip entries whose `flags` field contains `s_down` or `disconnected`, or whose `master-link-status` is not `ok`."

### F6 [minor] STYLE: Setext `===` headings render as H1 and break the heading hierarchy (`topics/sentinel-clients.md:71`)
**Doc says:**
> Handling reconnections
> ===

The same problem is at lines 81, 96, 113, 118, 126, and 136. The Step 1, 2, and 3 headings use `---` (lines 37, 46, 62).
**Problem:** On the live page, seven sections render as `<h1>` alongside the page title, while the Step headings render as `<h2>` at the same level as their parent section. The outline is inverted, so readers and outline tools see three top-level steps followed by seven new documents.
**Evidence:** `curl -sL https://valkey.io/topics/sentinel-clients/`:
```
<h2 id="valkey-service-discovery-via-sentinel">Valkey service discovery via Sentinel
<h2 id="step-1-connect-to-the-first-sentinel">Step 1: connect to the first Sentinel
...
<h1 id="handling-reconnections">Handling reconnections
<h1 id="sentinel-failover-disconnection">Sentinel failover disconnection
```
**Fix:** Convert the `===` headings to `## `. Convert the Step headings to `### `.

### F7 [nit] CONFUSING: Placeholder name changes between `master-name`, `primary-name`, and `<master-name>` (`topics/sentinel-clients.md:53`)
**Doc says:**
> Where *master-name* should be replaced with the actual service name

Line 105 uses `SENTINEL replicas primary-name`. Line 131 uses `SENTINEL sentinels <master-name>`.
**Problem:** Three spellings for one argument. Line 101 and line 105 are a before/after pair and still disagree.
**Fix:** Use `<primary-name>` everywhere, to match `SENTINEL HELP`.

### F8 [nit] STYLE: Grammar errors (`topics/sentinel-clients.md:74`)
**Doc says:**
> For instance Sentinel should contacted again the following cases:
**Problem:** Several sentences are broken: line 74 as quoted; line 13 "Valkey clients developers ... in their clients implementation"; line 49 "the client should retry to execute"; line 92 "If the client will contact a Sentinel with yet not updated information"; line 143 "speedup".
**Fix:** Line 74: "Sentinel should be contacted again in the following cases:". Line 13: "Valkey client developers ... in their client implementation". Line 49: "the client should execute". Line 92: "If the client contacts a Sentinel that has not yet updated its configuration". Line 143: "speed up".

### F9 [nit] AI-NATIVE: Generic link text for the Sentinel doc (`topics/sentinel-clients.md:18`)
**Doc says:**
> please check the [Valkey Documentation](sentinel.md)
**Problem:** The link text does not name its target, and the sentence buries the page's audience statement under qualifiers.
**Fix:** "This page is for client library authors and assumes familiarity with [Valkey Sentinel](sentinel.md)."
