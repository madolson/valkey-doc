VERDICT: NEEDS-WORK
COUNTS: blocker=2 minor=9 nit=6

# High availability with Valkey Sentinel (`sentinel`)

https://valkey.io/topics/sentinel/

The conceptual material is still sound, but the reference parts (pub/sub events, subcommands, config surface) have drifted from 9.1.2, and there is no security warning even though Sentinel turns protected mode off by default. Fix first: the Pub/Sub event list. Several event names are wrong, and two events do not exist, including `switch-master`, which the page calls "the message most external users are interested in".

## Findings

### F1 [blocker] MISSING: No warning that Sentinel disables protected mode by default (`topics/sentinel.md:63`)
**Doc says:**
> Sentinels by default run **listening for connections to TCP port 26379**, so for Sentinels to work, port 26379 of your servers **must be open** to receive connections from the IP addresses of the other Sentinel instances.
**Problem:** The page tells readers to open port 26379. It never says that Sentinel forces `protected-mode` off and, with no ACL or `requirepass`, accepts unauthenticated `SENTINEL SET`, `SENTINEL FAILOVER`, `SENTINEL REMOVE`, and `SHUTDOWN` from any reachable host. The authentication section, about 800 lines further down, presents auth as optional.
**Evidence:** `valkey@9.1.2:src/sentinel.c:496-498`
```
    server.port = VALKEY_SENTINEL_PORT;
    server.protected_mode = 0; /* Sentinel must be exposed. */
```
`valkey@9.1.2:sentinel.conf:3-6`: "By default protected mode is disabled in sentinel mode. Sentinel is reachable from interfaces different than localhost. Make sure the sentinel instance is protected from the outside world via firewalling or other means."
**Fix:** After line 70, add: "Sentinel disables `protected-mode` by default and listens on all interfaces. Restrict port 26379 to Sentinel peers and clients with a firewall, and configure authentication as described in [Configuring Sentinel instances with authentication](#configuring-sentinel-instances-with-authentication). An unauthenticated Sentinel lets anyone who can reach it trigger failovers or change its monitoring configuration." Add the same point to the "Fundamental things to know" list.

### F2 [blocker] INACCURATE: Pub/Sub event list has wrong and nonexistent channel names (`topics/sentinel.md:751`)
**Doc says:**
> * **+failover-detected** `<instance details>` ...
> * **-dup-sentinel** `<instance details>` ...
> * **no-good-slave** ... **selected-slave** ... **failover-state-send-slaveof-noone** ... **failover-end-for-timeout** ... **failover-end** ... **switch-master** `<primary name> <oldip> <oldport> <newip> <newport>` -- ... This is **the message most external users are interested in**.
**Problem:** The text says "The channel name is the same as the name of the event". A client that subscribes to `switch-master`, `failover-end`, or `selected-slave` as listed gets nothing, because the real channels start with `+`. `+failover-detected` and `-dup-sentinel` are never emitted. `no-good-slave` is really `-failover-abort-no-good-slave`. Events that are emitted are missing from the list: `+promoted-slave`, `+config-update-from`, `+convert-to-slave`, `+fix-slave-config`, `+failover-state-wait-promotion`, `-failover-abort-not-elected`, `-failover-abort-slave-timeout`, `-failover-abort-master-timeout`, `+vote-for-leader`, `+reboot`, `+monitor`/`-monitor`, `+set`, `+sentinel-address-switch`, `-slave-reconf-sent-timeout`, and the script events.
**Evidence:** Every literal event name in `valkey@9.1.2:src/sentinel.c`:
```
-cmd-link-reconnection +config-update-from +convert-to-slave +elected-leader -failover-abort-master-timeout -failover-abort-no-good-slave -failover-abort-not-elected -failover-abort-slave-timeout +failover-end +failover-end-for-timeout +failover-state-reconf-slaves +failover-state-select-slave +failover-state-send-slaveof-noone +failover-state-wait-promotion +fix-slave-config -monitor +monitor +new-epoch -odown +odown +promoted-slave -pubsub-link-reconnection +reboot +reset-master -script-child +script-child -script-error -script-timeout -sdown +sdown +selected-slave +sentinel +sentinel-address-switch +sentinel-address-update +sentinel-invalid-addr +set +slave +slave-reconf-done +slave-reconf-inprog +slave-reconf-sent +slave-reconf-sent-be -slave-reconf-sent-timeout +switch-master -tilt +tilt +try-failover +vote-for-leader
```
Live 9.1.2, `PSUBSCRIBE *` on a Sentinel during `SENTINEL FAILOVER mymaster COORDINATED`:
```
+new-epoch	1
+vote-for-leader	587ab93a6de6f6d5c35526a055d27a5fa202d8d0 1
+config-update-from	sentinel 587ab93a... 127.0.0.1 21342 @ mymaster 127.0.0.1 21340
+switch-master	mymaster 127.0.0.1 21340 127.0.0.1 21341
+slave	slave 127.0.0.1:21340 127.0.0.1 21340 @ mymaster 127.0.0.1 21341
```
**Fix:** Delete `+failover-detected` and `-dup-sentinel`. Rename the six entries to `+selected-slave`, `+failover-state-send-slaveof-noone`, `+failover-end-for-timeout`, `+failover-end`, `+switch-master`, and `-failover-abort-no-good-slave` ("No suitable replica to promote. The failover is aborted and retried after the failover timeout."). Add the missing events above, each with a one-line description, ideally as a table.

### F3 [minor] INACCURATE: SDOWN uses a config name that does not exist (`topics/sentinel.md:919`)
**Doc says:**
> does not receive a valid reply to PING requests for the number of seconds specified in the configuration as `is-master-down-after-milliseconds` parameter.
**Problem:** No such parameter exists. The parameter is `down-after-milliseconds`, and it is in milliseconds, not seconds.
**Evidence:** `valkey@9.1.2:sentinel.conf` defines only `sentinel down-after-milliseconds <master-name> <milliseconds>`. A grep for `is-master-down-after` in `valkey@9.1.2:src/` finds nothing.
**Fix:** "...for the number of milliseconds specified by the `down-after-milliseconds` option of the monitored primary (default 30000)."

### F4 [minor] OUTDATED: Subcommand list uses only the deprecated `MASTER` names and omits subcommands that exist in 9.1.2 (`topics/sentinel.md:601`)
**Doc says:**
> * **SENTINEL GET-MASTER-ADDR-BY-NAME `<primary name>`** ...
> * **SENTINEL INFO-CACHE** Return cached `INFO` output from primaries and replicas.
> * **SENTINEL IS-MASTER-DOWN-BY-ADDR** ... **SENTINEL MASTER** ... **SENTINEL MASTERS** ...

and at line 1245:
> Unfortunately in this command these words are part of the protocol, so we'll be able to remove such occurrences only when this API will be naturally deprecated.
**Problem:** Since 8.0, `SENTINEL PRIMARY`, `PRIMARIES`, `GET-PRIMARY-ADDR-BY-NAME`, and `IS-PRIMARY-DOWN-BY-ADDR` exist, and the `MASTER` forms are deprecated. The page never mentions the new names, and its closing note says no alternative exists. `SENTINEL DEBUG` (tunes TILT and timing parameters) and `SENTINEL HELP` are missing. `INFO-CACHE` requires a `<primary-name>` argument. The restricted client ACL at line 877 grants only `+sentinel|get-master-addr-by-name +sentinel|master`, so a client that uses the 8.0 names gets NOPERM. No `commands/sentinel-*.md` pages exist, so this list is the only reference.
**Evidence:** `valkey@9.1.2:src/commands/sentinel-master.json`: `"deprecated_since": "8.0.0", "replaced_by": "`SENTINEL PRIMARY`"`. Same for `sentinel-get-master-addr-by-name.json` (replaced by `SENTINEL GET-PRIMARY-ADDR-BY-NAME`). `sentinel-primary.json`: `"since": "8.0.0"`. Live 9.1.2:
```
$ valkey-cli -p 21342 sentinel info-cache
ERR wrong number of arguments for 'sentinel|info-cache' command
$ valkey-cli -p 21342 sentinel help
...
DEBUG [<param> <value> ...]
    Show a list of configurable time parameters and their values (milliseconds).
GET-PRIMARY-ADDR-BY-NAME <primary-name>
...
IS-PRIMARY-DOWN-BY-ADDR <ip> <port> <current-epoch> <runid>
PRIMARY <primary-name>
PRIMARIES
```
**Fix:** List `SENTINEL PRIMARY`, `PRIMARIES`, `GET-PRIMARY-ADDR-BY-NAME`, and `IS-PRIMARY-DOWN-BY-ADDR` as the primary entries, each marked "(Valkey 8.0+; `MASTER` form is a deprecated alias)". Use the new names in the tutorial (lines 486, 566, 592) and at lines 648, 693, and 704. Add `SENTINEL DEBUG [<param> <value> ...]` and `SENTINEL HELP`. Change `INFO-CACHE` to `SENTINEL INFO-CACHE <primary name>`. Add `+sentinel|primary +sentinel|get-primary-addr-by-name` to the line 877 ACL. Rewrite line 1245 as: "Event names and some reply fields still use 'master' and 'slave' for compatibility. Valkey 8.0 added `PRIMARY` aliases for the commands."

### F5 [minor] INACCURATE: Says replicas behind Docker port mapping cannot be monitored (`topics/sentinel.md:391`)
**Doc says:**
> so there is currently no way to monitor with Sentinel a set of primary and replica instances deployed with Docker, **unless you instruct Docker to map the port 1:1**.
>
> For the first problem, in case you want to run a set of Sentinel instances using Docker ...
**Problem:** Problem 2 (a wrong replica address in the primary's `INFO`) is solved by `replica-announce-ip` and `replica-announce-port` on each replica. The page gives a fix only for problem 1 and says problem 2 has none.
**Evidence:** `topics/replication.md:420-421` documents `replica-announce-ip 5.5.5.5` and `replica-announce-port 1234` for exactly this NAT and port-forwarding case.
**Fix:** Replace "so there is currently no way ... map the port 1:1**." with: "To fix this, set `replica-announce-ip` and `replica-announce-port` on each replica to the externally reachable address (see [Replication: configuring replication in Docker and NAT](replication.md)), or map ports 1:1."

### F6 [minor] INACCURATE: SENTINEL MONITOR is said to reject hostnames (`topics/sentinel.md:636`)
**Doc says:**
> with the difference that you can't use a hostname in as `ip`, but you need to provide an IPv4 or IPv6 address.
**Problem:** With `resolve-hostnames yes`, `SENTINEL MONITOR` accepts a hostname, as the page's own DNS section (line 415) implies.
**Evidence:** Live 9.1.2:
```
$ valkey-cli -p 21342 sentinel monitor h1 localhost 21340 2
ERR Invalid IP address or hostname specified
$ valkey-cli -p 21342 sentinel config set resolve-hostnames yes
OK
$ valkey-cli -p 21342 sentinel monitor h1 localhost 21340 2
OK
```
`valkey@9.1.2:src/sentinel.c:3989`: `anetResolve(..., sentinel.resolve_hostnames ? ANET_NONE : ANET_IP_ONLY)`.
**Fix:** "...identical to the `sentinel monitor` configuration directive. A hostname is accepted only when `resolve-hostnames` is enabled. Otherwise provide an IPv4 or IPv6 address."

### F7 [minor] INACCURATE: SENTINEL SET is said to accept every sentinel.conf parameter (`topics/sentinel.md:638`)
**Doc says:**
> All the configuration parameters that can be configured via `sentinel.conf` are also configurable using the SET command.
**Problem:** By default `notification-script` and `client-reconfig-script` are rejected (`deny-scripts-reconfig yes`), and global options (`announce-ip`, `resolve-hostnames`, and others) go through `SENTINEL CONFIG SET`, not `SENTINEL SET`. Line 644 repeats the claim.
**Evidence:** Live 9.1.2:
```
$ valkey-cli -p 21342 sentinel set mymaster notification-script /bin/true
ERR Reconfiguration of scripts path is denied for security reasons. Check the deny-scripts-reconfig configuration directive in your Sentinel configuration
```
`valkey@9.1.2:src/sentinel.c:106`: `#define SENTINEL_DEFAULT_DENY_SCRIPTS_RECONFIG 1`.
**Fix:** "All per-primary options (`down-after-milliseconds`, `failover-timeout`, `parallel-syncs`, `quorum`, `auth-user`, `auth-pass`, `primary-reboot-down-after-period`, `rename-command`, `notification-script`, `client-reconfig-script`) can be changed with `SENTINEL SET`. Script paths can be changed only if `sentinel deny-scripts-reconfig no` is set in the configuration file." Remove "As already stated, ... startup configuration file" at line 644.

### F8 [minor] MISSING: Notification and client-reconfig scripts, and other per-primary options, are not documented anywhere (`topics/sentinel.md:23`)
**Doc says:**
> * **Notification**. Sentinel can notify the system administrator, or other computer programs, via the API, that something is wrong with one of the monitored Valkey instances.
**Problem:** The page mentions `SENTINEL PENDING-SCRIPTS`, `sentinel_running_scripts`, and line 856 says clients can rely on "a script that performs a transparent redirection". It never names or explains `notification-script`, `client-reconfig-script`, or `deny-scripts-reconfig`, or what arguments the scripts receive. The same is true of `sentinel rename-command` (needed when monitored instances rename `CONFIG`, `REPLICAOF`, and similar commands) and `primary-reboot-down-after-period`. No other valkey-doc page covers these.
**Evidence:** `grep -rln 'notification-script\|client-reconfig-script' topics commands` returns nothing. `valkey@9.1.2:sentinel.conf` defines `notification-script`, `client-reconfig-script`, `deny-scripts-reconfig`, `rename-command`. Script limits are at `valkey@9.1.2:src/sentinel.c:133-135,96-97` (max 16 running, queue 256, 10 retries, 60 s max runtime, 30 s retry delay).
**Fix:** Add a "Notification and client reconfiguration scripts" section under "Other Sentinel options". Cover the arguments the client-reconfig script receives (`<primary-name> <role> <state> <from-ip> <from-port> <to-ip> <to-port>`), the limits above, and the fact that `deny-scripts-reconfig` defaults to `yes`. Add one-line entries for `rename-command` and `primary-reboot-down-after-period`.

### F9 [minor] EXAMPLE: Failover test command fails on a default 9.1.2 primary (`topics/sentinel.md:551`)
**Doc says:**
>     valkey-cli -p 6379 DEBUG sleep 30
**Problem:** `enable-debug-command` defaults to `no`, so the tutorial's only failover test step returns an error, and the reader sees no failover.
**Evidence:** Live 9.1.2, default primary:
```
$ valkey-cli -p 21340 debug sleep 0
ERR DEBUG command not allowed. If the enable-debug-command option is set to "local", you can run it from a local connection, otherwise you need to set this option in the configuration file, and then restart the server.
$ valkey-cli -p 21340 config get enable-debug-command
enable-debug-command
no
```
**Fix:** Change the tutorial's primary setup to say "start the primary with `--enable-debug-command local`". Alternatively, replace the step with `valkey-cli -p 5000 SENTINEL FAILOVER mymaster`, or stop the primary process (`kill -STOP <pid>`, then `kill -CONT <pid>`).

### F10 [minor] DUPLICATE: min-replicas mitigation explained twice on the page and again in replication.md (`topics/sentinel.md:1172`)
**Doc says:**
> With the above configuration (please see the self-commented `valkey.conf` example in the Valkey distribution for more information) a Valkey instance, when acting as a primary, will stop accepting writes if it can't write to at least 1 replica.
**Problem:** The code block and this paragraph appear word for word at lines 285-290 and 1169-1174. `topics/replication.md:378` documents `min-replicas-to-write` and `min-replicas-max-lag` as the canonical reference.
**Fix:** Keep the Example 2 explanation (lines 285-290), but replace "please see the self-commented `valkey.conf`" with a link to [replication.md](replication.md). At lines 1166-1174 keep only: "Bound the divergence with `min-replicas-to-write` and `min-replicas-max-lag` as shown in [Example 2](#example-2-basic-setup-with-three-boxes). With `min-replicas-max-lag 10`, Valkey 3 stops accepting writes after 10 seconds."

### F11 [minor] AI-NATIVE: Defaults are not stated in prose (`topics/sentinel.md:136`)
**Doc says:**
> * `down-after-milliseconds` is the time in milliseconds an instance should not be reachable ... for a Sentinel starting to think it is down.
**Problem:** The defaults a reader would want to quote do not appear anywhere: `down-after-milliseconds` 30000, `failover-timeout` 180000, `parallel-syncs` 1, `replica-priority` 100, `resolve-hostnames`/`announce-hostnames` no. The page's examples use 60000, 5000, and 10000, so a reader cannot tell what applies when the option is omitted. `failover-timeout` is used but never defined.
**Evidence:** `valkey@9.1.2:src/sentinel.c:90` `sentinel_default_down_after = 30000`, `:98` `sentinel_default_failover_timeout = 60 * 3 * 1000`, `:102` `SENTINEL_DEFAULT_PARALLEL_SYNCS 1`, `:107-108` hostnames 0. `valkey@9.1.2:src/config.c:3402` `replica-priority` default 100.
**Fix:** Replace the bullet list in "Other Sentinel options" with a table: option | default | meaning. Include `down-after-milliseconds`, `failover-timeout` (with its four roles from `sentinel.conf`), `parallel-syncs`, `primary-reboot-down-after-period`, `auth-user`, and `auth-pass`. State `replica-priority` default 100 in "Replicas priority".

### F12 [nit] INACCURATE: Global parameter list omits `loglevel` and multi-parameter CONFIG (`topics/sentinel.md:655`)
**Doc says:**
> Global parameters that can be manipulated include:
**Problem:** `SENTINEL CONFIG GET *` on 9.1.2 also returns `loglevel`. Since 7.2, `SENTINEL CONFIG GET` and `SET` accept multiple parameters, but the syntax shown at lines 595-596 and 650-651 allows one.
**Evidence:** Live `sentinel config get '*'` returns `resolve-hostnames announce-hostnames announce-ip announce-port sentinel-user sentinel-pass loglevel`. `valkey@9.1.2:src/commands/sentinel-config.json` history: `"7.2.0", "Added the ability to set and get multiple parameters in one call."`
**Fix:** Add "`loglevel`" to the list. Change the syntax to `SENTINEL CONFIG GET <name> [<name> ...]` and `SENTINEL CONFIG SET <name> <value> [<name> <value> ...]`.

### F13 [nit] EXAMPLE: INFO sentinel output is missing `sentinel_total_tilt` (`topics/sentinel.md:1229`)
**Doc says:**
>     sentinel_tilt:0
>     sentinel_tilt_since_seconds:-1
>     sentinel_running_scripts:0
**Problem:** 9.1.2 prints `sentinel_total_tilt` between these fields. The intro sentence "In the Sentinel TILT mode, if we send the INFO command" also does not match output showing `sentinel_tilt:0` (not in TILT).
**Evidence:** Live 9.1.2:
```
# Sentinel
sentinel_masters:1
sentinel_tilt:0
sentinel_tilt_since_seconds:-1
sentinel_total_tilt:0
sentinel_running_scripts:0
```
**Fix:** Insert `sentinel_total_tilt:0` and document it: "number of times this Sentinel entered TILT since startup". Change the intro to "The INFO command reports TILT state:".

### F14 [nit] OUTDATED: Redis-era version gates (`topics/sentinel.md:595`)
**Doc says:**
> (`>= 6.2`) ... Starting with version 6.2, Sentinel has *optional* support for host names. ... Starting with Redis OSS 6.2, the [Access Control List (ACL)](acl.md) is available, whereas older versions support password-only authentication.
**Problem:** Every Valkey release has these features, so the gates at lines 407, 595, 596, 607, 618, 621, 622, and 855 are noise. Meanwhile the gates that matter for Valkey users (8.0 `PRIMARY` aliases, 9.0 `COORDINATED`) are missing or scattered.
**Fix:** Delete the `>= 6.x` markers and the two "Starting with ... 6.2" sentences. Line 407 becomes "Sentinel has optional support for host names." Line 855 becomes "Sentinel instances can require clients to authenticate with [ACL](acl.md) users or a single `requirepass` password."

### F15 [nit] STYLE: Setext `---` headings break the heading hierarchy (`topics/sentinel.md:468`)
**Doc says:**
> Asking Sentinel about the state of a primary
> ---
**Problem:** Lines 468, 533, 785, 897, 958, 979, and 1186 use `---` underlines, which render as `<h2>`. Subsections then appear as siblings of their parents ("Asking Sentinel..." at the same level as "A quick tutorial", "Replicas priority" as a sibling of "Sentinel API"). Line 853 (`### Sentinel clients implementation` followed by `---`) renders a stray `<hr />`.
**Evidence:** Live page: `<h2 id="asking-sentinel-about-the-state-of-a-primary">`, `<h2 id="replicas-priority">`, `<h3 id="sentinel-clients-implementation">Sentinel clients implementation</h3>\n<hr />`.
**Fix:** Convert each to an ATX `###` heading at the correct level, and delete the `---` on line 853.

### F16 [nit] EXAMPLE: Example names and labels contradict the prose (`topics/sentinel.md:640`)
**Doc says:**
> configuration of a primary called `objects-cache`:
>
>     SENTINEL SET objects-cache-master down-after-milliseconds 1000

and line 1147:
> Sentinels 1 and 2 started a failover promoting Sentinel 1 as the new primary.
**Problem:** The prose names the primary `objects-cache`, but the command uses `objects-cache-master`. Sentinels are never promoted. Valkey 1 is. Line 1183 "it's predecessor" should be "its". The box at line 179 (`| Valkey primary M1   |`) is misaligned.
**Fix:** Use `objects-cache` in both commands (lines 642 and 646). Line 1147: "promoting Valkey 1 as the new primary". Line 1183: "its predecessor". Remove two spaces in line 179.

### F17 [nit] STYLE: Non-inclusive terms outside literal names (`topics/sentinel.md:155`)
**Doc says:**
> * Master-specific configuration parameters are modified using `SENTINEL SET`.

and line 993:
> * Masters failed over are reconfigured as replicas when they return available.
**Problem:** The rest of the page uses "primary". These are prose, not protocol names.
**Fix:** "Primary-specific configuration parameters..." and "Primaries that were failed over are reconfigured as replicas when they become available again." Line 1123 `milliseconds_since_master_is_in_SDOWN_state` becomes `milliseconds_since_primary_is_in_SDOWN_state`.

## Unverified
- topics/sentinel.md:820: "Since Valkey Sentinel 9.0, the sentinel user requires the `+failover` and `+client` permissions". Confirmed that 9.1.2 sends `FAILOVER` and `CLIENT KILL`/`SETNAME` (`src/sentinel.c:4780-4924`). Did not check whether `CLIENT` was already required before 9.0, which would make the "since 9.0" for `+client` wrong.
