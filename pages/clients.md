VERDICT: NEEDS-WORK
COUNTS: blocker=0 minor=5 nit=4

# Client handling (`clients`)

https://valkey.io/topics/clients/

Defaults (maxclients, output buffer limits, timeout, keepalive) are correct for 9.1.2, but several behavior claims are wrong or stale. The most important fix: the query buffer limit is configurable via `client-query-buffer-limit`, not a fixed 1 GB.

## Findings

### F1 [minor] INACCURATE: query buffer limit is configurable (`topics/clients.md:103`)
**Doc says:**
> This is a non-configurable hard limit that will close the connection when the client query buffer ... reaches 1 GB
**Problem:** The limit is a config, `client-query-buffer-limit`, default 1gb, min 1mb, settable at runtime. Readers who hit it will not know they can tune it.
**Evidence:** `valkey@9.1.2:src/config.c:3482`
> `createSizeTConfig("client-query-buffer-limit", NULL, DEBUG_CONFIG | MODIFIABLE_CONFIG, 1024 * 1024, LONG_MAX, server.client_max_querybuf_len, 1024 * 1024 * 1024, ...`

Live 9.1.2:
```
$ valkey-cli -p 20220 config set client-query-buffer-limit 2mb
OK
$ valkey-cli -p 20220 config get client-query-buffer-limit
client-query-buffer-limit
2097152
```
**Fix:** Rename heading to "Query Buffer Limit" and replace with: "Every client is also subject to a query buffer limit, set by `client-query-buffer-limit` (default `1gb`, minimum `1mb`). When a client's query buffer exceeds it, the connection is closed. The limit is a safety net against client or server bugs; raise it only if you send very large commands, such as big `MULTI`/`EXEC` batches." Keep the `#query-buffer-hard-limit` anchor working or update the link at line 117.

### F2 [minor] INACCURATE: maxclients check happens before the client is created, and counts cluster connections (`topics/clients.md:17`)
**Doc says:**
> After the client is initialized, Valkey checks if it is already at the limit configured for the number of simultaneous clients
**Problem:** Admission control runs before `createClient()`, so none of the setup at lines 13 to 15 happens for a rejected connection. In cluster mode, cluster bus connections count toward `maxclients`, and the error text differs. For TLS, no error is sent at all; the socket is dropped. The page says the error "will reach the client" (line 23) with no exception.
**Evidence:** `valkey@9.1.2:src/networking.c:1890-1914`
> `Admission control will happen before a client is created and connAccept() called`
> `if (listLength(server.clients) + getClusterConnectionsCount() >= server.maxclients) {`
> `err = "-ERR max number of clients + cluster connections reached\r\n";` ... `err = "-ERR max number of clients reached\r\n";`
> `Note that for TLS connections, no handshake was done yet so nothing is written and the connection will just drop.`
**Fix:** Move the limit check before the bullet list: "Before creating the client, Valkey checks whether the number of connected clients (plus cluster bus connections in cluster mode) has reached `maxclients`. If so, it writes `-ERR max number of clients reached` (in cluster mode, `-ERR max number of clients + cluster connections reached`) and closes the socket. TLS connections are dropped without an error because no handshake has happened yet. The rejection is counted in `rejected_connections` in `INFO stats`." Then describe the socket setup for accepted clients.

### F3 [minor] INACCURATE: timeout exemptions are incomplete (`topics/clients.md:154`)
**Doc says:**
> Note that the timeout only applies to normal clients and it **does not apply to Pub/Sub clients**
**Problem:** Blocked clients (BLPOP and similar), replicas, monitors, and the primary link are also exempt. A reader setting `timeout` to drop idle blocking consumers will see them stay connected.
**Evidence:** `valkey@9.1.2:src/timeout.c:56-61`
> `!c->flag.replica &&   /* No timeout for replicas and monitors */`
> `!mustObeyClient(c) && /* No timeout for primaries and AOF */`
> `!c->flag.blocked &&   /* No timeout for BLPOP */`
> `!c->flag.pubsub &&    /* No timeout for Pub/Sub clients */`
**Fix:** "The timeout does not apply to Pub/Sub clients (idle is normal for a push connection), clients blocked in a command such as `BLPOP`, replicas, `MONITOR` clients, or the connection to the primary. The default is `0`, meaning no timeout."

### F4 [minor] EXAMPLE: CLIENT LIST output is from an old Redis version (`topics/clients.md:170-172`)
**Doc says:**
> `addr=127.0.0.1:52555 fd=5 name= age=855 idle=0 flags=N db=0 sub=0 psub=0 multi=-1 qbuf=0 qbuf-free=32768 obl=0 oll=0 omem=0 events=r cmd=client`
**Problem:** 9.1.2 output starts with `id=`, includes `laddr`, `ssub`, `tot-mem`, `user`, `resp`, `lib-name`, `tot-net-in`, and others, and shows `cmd=client|list`. The prompt also uses port 6379 while describing `cmd=client`. The example does not match what readers see.
**Evidence:** Live 9.1.2:
```
id=3 addr=127.0.0.1:53344 laddr=127.0.0.1:20220 fd=9 name= age=0 idle=0 flags=N capa= db=0 sub=0 psub=0 ssub=0 multi=-1 watch=0 qbuf=0 qbuf-free=0 argv-mem=10 multi-mem=0 rbs=16384 rbp=16384 obl=0 oll=0 omem=0 tot-mem=17050 events=r cmd=client|list user=default redir=-1 resp=2 lib-name= lib-ver= tot-net-in=26 tot-net-out=0 tot-cmds=0
```
**Fix:** Replace the example with current 9.1.2 output (as above, two clients, with `6379` in the prompt), tag the fence as `text`, add `id` (used by `CLIENT KILL ID`) and `tot-mem` (what client eviction measures) to the field list at lines 177-184.

### F5 [minor] OUTDATED: version notes reference Redis OSS (`topics/clients.md:142`, `topics/clients.md:190`)
**Doc says:**
> Client eviction is available from Redis OSS 7.0.

> Starting with Redis OSS 4.0, the client name is shown in the `SLOWLOG` output
**Problem:** Every Valkey release has both features. The notes send readers to Redis version history that does not apply to Valkey.
**Evidence:** Valkey forked from Redis 7.2.4; `maxmemory-clients` and `CLIENT NO-EVICT` exist in all Valkey releases (`valkey@9.1.2:src/config.c:3483`).
**Fix:** Delete line 142. At line 190, change to "The client name is also shown in `SLOWLOG` output, to help identify clients that cause latency."

### F6 [nit] CONFUSING: CLIENT KILL by address only (`topics/clients.md:188`)
**Doc says:**
> you can close a client's connection using the `CLIENT KILL` command, specifying the client address as its argument.
**Problem:** The old `CLIENT KILL addr:port` form is legacy. `CLIENT KILL ID <id>` is unambiguous and the filter forms (`USER`, `ADDR`, `LADDR`, `TYPE`, `MAXAGE`) are what operators need.
**Fix:** "You can close a connection with `CLIENT KILL ID <id>`, using the `id` field from `CLIENT LIST`. See [`CLIENT KILL`](../commands/client-kill.md) for other filters such as `ADDR`, `USER`, and `TYPE`."

### F7 [nit] STYLE: CLIENT NO-EVICT shown as two inline code spans, and typos (`topics/clients.md:134`, `topics/clients.md:138`, `topics/clients.md:84`, `topics/clients.md:11`)
**Doc says:**
> `CLIENT NO-EVICT` `on`

> For this reason by default Sets limits to the output buffer size

> Valkey accepts clients connections
**Problem:** Renders as two separate code spans (live page: `<code>CLIENT NO-EVICT</code> <code>on</code>`). "by default Sets" is missing the subject. "clients connections" is a typo.
**Fix:** Use a fenced block: ```` ```text\nCLIENT NO-EVICT ON\n``` ```` (and `OFF`), link to [`CLIENT NO-EVICT`](../commands/client-no-evict.md). Line 84: "For this reason Valkey sets limits by default on the output buffer size". Line 11: "client connections".

### F8 [nit] EXAMPLE: fenced blocks have no language tag (`topics/clients.md:55`, `topics/clients.md:169`)
**Doc says:**
> ```` ``` ````
**Problem:** Both code fences are untagged.
**Fix:** Tag as `text` (or `console` for the shell transcript at line 55).

### F9 [nit] AI-NATIVE: output buffer section omits the config name and syntax (`topics/clients.md:99`)
**Doc says:**
> It is possible to change the limit at runtime using the `CONFIG SET` command ... See the example `valkey.conf` in the Valkey distribution for more information about how to set the limit.
**Problem:** The section never names the directive or its syntax, so a reader must leave the page to apply it. The defaults are in prose but not quotable as config.
**Evidence:** `valkey@9.1.2:valkey.conf:2422,2449-2451`
> `client-output-buffer-limit <class> <hard limit> <soft limit> <soft seconds>`
> `client-output-buffer-limit normal 0 0 0` / `replica 256mb 64mb 60` / `pubsub 32mb 8mb 60`
**Fix:** Replace line 99 with: "Set limits with `client-output-buffer-limit <class> <hard> <soft> <soft-seconds>`, in `valkey.conf` or with `CONFIG SET`. Defaults:" followed by a `text` block containing the three default lines above.
