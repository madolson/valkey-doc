VERDICT: NEEDS-WORK
COUNTS: blocker=1 minor=6 nit=4

# Serialization protocol specification (`protocol`)

https://valkey.io/topics/protocol/

The wire-format sections are accurate. The Client handshake section is not: the HELLO field values and all three example error replies differ from what 9.1.2 sends. Fix the HELLO field list first. Client authors code against it.

## Findings

### F1 [blocker] INACCURATE: HELLO reply fields do not match what 9.1.2 sends (`topics/protocol.md:568`)
**Doc says:**
> * **server**: "redis" (or other software name).
> ...
> * **role**: "primary" or "replica".
> * **modules**: list of loaded modules as an Array of Bulk Strings.

**Problem:** 9.1.2 sends `server` as `valkey` (`redis` only with `extended-redis-compatibility`), `role` as `master` or `replica`, and `modules` as an array of maps. It omits `role` in sentinel mode and adds `availability_zone` when that config is set. A client that matches `role == "primary"` never detects a primary. The same code is on unstable.
**Evidence:** valkey@9.1.2:src/networking.c:5920-5953
```
addReplyMapLen(c, 6 + !server.sentinel_mode + (sdslen(server.availability_zone) != 0));
addReplyBulkCString(c, server.extended_redis_compat ? "redis" : SERVER_NAME);
...
if (!server.sentinel_mode) {
    addReplyBulkCString(c, "role");
    addReplyBulkCString(c, server.primary_host ? "replica" : "master");
}
...
if (sdslen(server.availability_zone) != 0) {
    addReplyBulkCString(c, "availability_zone");
```
Output from 9.1.2 (`HELLO 3`):
```
%7\r\n$6\r\nserver\r\n$6\r\nvalkey\r\n$7\r\nversion\r\n$5\r\n9.1.2\r\n$5\r\nproto\r\n:3\r\n$2\r\nid\r\n:4\r\n$4\r\nmode\r\n$10\r\nstandalone\r\n$4\r\nrole\r\n$6\r\nmaster\r\n$7\r\nmodules\r\n*1\r\n%4\r\n$4\r\nname\r\n$3\r\nlua\r\n...
```
**Fix:** Replace lines 568-577 with:
> * **server**: "valkey" ("redis" when `extended-redis-compatibility` is enabled).
> * **version**: the server's version.
> * **proto**: the protocol version of the connection.
>
> Valkey also sends:
> * **id**: the connection's identifier (ID).
> * **mode**: "standalone", "sentinel" or "cluster".
> * **role**: "master" or "replica". Not sent in sentinel mode.
> * **modules**: an array of maps, one per loaded module, with `name`, `ver`, `path` and `args` fields.
> * **availability_zone**: only when the `availability-zone` config is set (Valkey 8.1 and later).
>
> See [`HELLO`](../commands/hello.md) for the full reply.

Also, `proto` at line 570 is described as "the highest supported version". The server sends `c->resp`, the connection's current version (`addReplyLongLong(c, c->resp)`). `HELLO 2` returns `proto 2`.
Tracked: valkey-io/valkey-doc#511 (role value only)

### F2 [minor] INACCURATE: NOPROTO error text is wrong (`topics/protocol.md:548`)
**Doc says:**
> Server: -NOPROTO sorry, this protocol version is not supported.

**Problem:** The text is wrong. Also, the server rejects versions below 2 as well as above 3, so "too big" at line 545 is incomplete.
**Evidence:** valkey@9.1.2:src/networking.c:5861 `addReplyError(c, "-NOPROTO unsupported protocol version");`. Live output: `HELLO 4` and `HELLO 1` both return `-NOPROTO unsupported protocol version\r\n`.
**Fix:** Line 548: `Server: -NOPROTO unsupported protocol version`. Line 545: "If the requested version is not supported by the server (Valkey supports 2 and 3), it replies with a `-NOPROTO` error."

### F3 [minor] INACCURATE: HELLO with a bad password returns WRONGPASS, not ERR (`topics/protocol.md:563`)
**Doc says:**
> Server: -ERR invalid password

**Problem:** The error prefix is `WRONGPASS`. A client that branches on the error prefix, as the Simple errors section recommends, would miss it. Line 560 also has a stray `!` in "`!AUTH` clause".
**Evidence:** 9.1.2 with `requirepass secret`, sending `HELLO 3 AUTH default mypassword`:
```
-WRONGPASS invalid username-password pair or user is disabled.\r\n
```
**Fix:** Line 563: `Server: -WRONGPASS invalid username-password pair or user is disabled.` Line 560: "`AUTH` clause".

### F4 [minor] MISSING: RESP3 attribute type (`|`) is absent from the type table (`topics/protocol.md:108`)
**Doc says:**
> The following table summarizes the RESP data types that Valkey supports:

**Problem:** Valkey emits the attribute type. Modules emit it through `ValkeyModule_ReplyWithAttribute` (documented in topics/modules-api-ref.md:1634), and `DEBUG PROTOCOL attrib` emits it too. A RESP3 parser written from this table fails on `|`.
**Evidence:** valkey@9.1.2:src/networking.c:1453 `void addReplyAttributeLen(client *c, long length) {`. valkey@9.1.2:src/module.c:3325 `case COLLECTION_REPLY_ATTRIBUTE: addReplyAttributeLen(c, len); break;`. valkey@9.1.2:src/debug.c:854-864 (`attrib` emits an attribute map followed by "Some real reply following the attribute").
**Fix:** Add the row `| [Attributes](#attributes) | RESP3 | Aggregate | `|` |` and an "### Attributes" section: the encoding is `|<number-of-entries>\r\n<key-1><value-1>...`, it is the same as a map, and it precedes the reply it annotates. Clients that don't use attributes must still parse and skip them.
Tracked: valkey-io/valkey-doc#510

### F5 [minor] OUTDATED: version history is stated only in Redis OSS terms (`topics/protocol.md:39`)
**Doc says:**
> Up to and including Redis OSS 7, both RESP2 and RESP3 clients can invoke all core commands.

**Problem:** The reader runs Valkey. "Up to and including Redis OSS 7" leaves open whether Valkey 8 or 9 dropped this, and they did not. Line 33 ("RESP2, early became the standard") is also ungrammatical.
**Evidence:** 9.1.2 accepts `HELLO 2` and `HELLO 3` (networking.c:5861 range check), and every core command runs under both protocols.
**Fix:** Line 39: "In all Valkey versions, both RESP2 and RESP3 clients can invoke all core commands." Line 33: "The next version, RESP2, became the standard communication method for clients." Keep line 36 as history ("Redis OSS 6.0 introduced...").
Tracked: valkey-io/valkey-doc#462 (touches this file; may cover it)

### F6 [minor] BROKEN: `#nil-reply` anchor lands on Arrays instead of Nulls (`topics/protocol.md:216`)
**Doc says:**
> `<a name="nil-reply"></a>`
>
> `### Arrays`

**Problem:** topics/transactions.md:201 and :288 link "[Null reply](protocol.md#nil-reply)" for EXEC's null reply. The anchor sits directly above the Arrays heading, so the reader lands on Arrays.
**Evidence:** topics/protocol.md:216-218. The live page renders `<a name="nil-reply">` immediately before `<h3 id="arrays">`.
**Fix:** Move `<a name="nil-reply"></a>` to just above `### Nulls` (line 284).

### F7 [minor] DUPLICATE: HELLO reply fields are documented here and in commands/hello.md, and this page doesn't link to hello.md (`topics/protocol.md:566`)
**Doc says:**
> A successful reply to the `HELLO` command is a map reply. ... certain fields are mandatory ...

**Problem:** commands/hello.md:16-48 also documents the field list. The two copies have already drifted (F1: `primary` here, `master` there; `availability_zone` only there). `HELLO` is never linked from this page.
**Evidence:** commands/hello.md:16-19: "`HELLO` always replies with a list of current server and connection properties ... The `availability_zone` field only shows up when it is set in the configs".
**Fix:** Make commands/hello.md canonical for the reply fields. At line 539 here, link the first mention as [`HELLO`](../commands/hello.md). Cut the field list to the three mandatory fields plus "See [`HELLO`](../commands/hello.md) for the full reply."

### F8 [nit] CONFUSING: type table calls bulk types "Aggregate", contradicting the page's own three categories (`topics/protocol.md:115`)
**Doc says:**
> We categorize every RESP data type as either _simple_, _bulk_ or _aggregate_.

but the table has `| [Bulk strings](#bulk-strings) | RESP2 | Aggregate | `$` |`.
**Problem:** The table has no "Bulk" row, although line 97 defines that category. Bulk strings (115), bulk errors (121) and verbatim strings (122) are listed as Aggregate. Line 635 repeats the error: "aggregate types (for example, bulk strings or arrays)".
**Fix:** Set Category to `Bulk` for lines 115, 121 and 122. Line 635: "Reading the length of bulk and aggregate types (for example, bulk strings or arrays)".

### F9 [nit] EXAMPLE: C parser example has a stale comment and a compiler warning (`topics/protocol.md:643`)
**Doc says:**
> unsigned char *p = "$123\r\n";
> ...
> /* Now p points at '\r', and the len is in bulk_len. */

**Problem:** No variable is named `bulk_len`. The initializer warns under `-Wall`.
**Evidence:**
```
t.c:4:24: warning: pointer targets in initialization differ in signedness [-Wpointer-sign]
     unsigned char *p = "$123\r\n";
```
**Fix:** `const char *p = "$123\r\n";` and `/* Now p points at '\r', and the length is in len. */`

### F10 [nit] EXAMPLE: unknown-command error example is not what 9.1.2 prints (`topics/protocol.md:155`)
**Doc says:**
> -ERR unknown command 'asdf'

**Evidence:** 9.1.2 output for inline `asdf`: `-ERR unknown command 'asdf', with args beginning with: \r\n`
**Fix:** `-ERR unknown command 'asdf', with args beginning with: `

### F11 [nit] STYLE: typo and an untagged code fence (`topics/protocol.md:572`, `topics/protocol.md:668`)
**Doc says:**
> In Valkey' RESP3 implementation

and line 668 opens a fence with ```` ``` ```` and no language tag.
**Fix:** Line 572: "In Valkey's RESP3 implementation". Line 668: tag the fence ```` ```text ````. The EVAL example is correct: on 9.1.2 it returns `,1\r\n`.
