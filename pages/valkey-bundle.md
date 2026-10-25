VERDICT: NEEDS-WORK
COUNTS: blocker=3 minor=4 nit=3

# Valkey Bundle Getting Started Guide (`valkey-bundle`)

https://valkey.io/topics/valkey-bundle/

The quick start commands work against the current image (valkey/valkey-bundle:latest, Valkey 9.1.2). The sample configuration file and the LDAP example do not. The most important fix is the sample valkey.conf: run as documented, the container exits at startup. It fails on four separate lines, and once those are fixed the server is still unreachable from the host.

## Findings

### F1 [blocker] EXAMPLE: Sample valkey.conf prevents the container from starting (`topics/valkey-bundle.md:181`)
**Doc says:**
> bf.bloom-memory-usage-limit 134217728  # 128MB
> ldap.servers "ldap://primary:389,ldap://backup:389"
> search.use-coordinator no
> logfile "/var/log/valkey/valkey.log"

**Problem:** I copied the sample verbatim (lines 160-236) and ran it with the documented `docker run -v ... valkey-server /usr/local/etc/valkey/valkey.conf`. The server exits at startup. It hit four separate fatal errors, which I found by fixing them one at a time:
1. `/var/log/valkey/` does not exist in the image.
2. Valkey has no inline comments. For module configs, config.c joins the extra tokens into the value, so the value becomes `134217728 # 128MB`.
3. valkey-ldap 1.1.1 splits `ldap.servers` on spaces, not commas.
4. `search.use-coordinator` in a config file aborts startup. I did not trace why.

**Evidence:**
```
*** FATAL CONFIG FILE ERROR (Version 9.1.2) ***
Can't open the log file: No such file or directory
```
After setting `logfile ""`:
```
# Issue during loading of configuration bf.bloom-memory-usage-limit : argument couldn't be parsed into an integer
# Module Configurations were not set, likely a missing LoadConfigs call. Unloading the module.
=== VALKEY BUG REPORT START: Cut & paste starting from here ===
```
After removing `  # 128MB`:
```
# Issue during loading of configuration ldap.servers : invalid port number
# Can't load module from /usr/lib/valkey/libvalkey_ldap.so: server aborting
```
After switching to a space-separated list:
```
# Module Configuration detected without loadmodule directive or no ApplyConfig call: aborting
```
Startup succeeded only after `search.use-coordinator` was also removed.

valkey@9.1.2:src/config.c:585-586:
```
sds val = sdsdup(argv[1]);
for (int i = 2; i < argc; i++) val = sdscatfmt(val, " %S", argv[i]);
```
valkey-ldap@1.1.1:src/configs.rs:153: `let urls = server_list.split(" ");`. topics/ldap.md:105 and :176 already say the list is space-separated.

**Fix:**
- Put the comment on its own line: `# 128MB` above `bf.bloom-memory-usage-limit 134217728`.
- Use `ldap.servers "ldap://primary:389 ldap://backup:389"`.
- Delete the `# Enable cluster mode` / `search.use-coordinator no` pair. Its comment also contradicts its value.
- Replace the logfile line with `logfile ""` and the comment `# Log to stdout so docker logs works`. This also matches line 133.

### F2 [blocker] EXAMPLE: Sample config binds to loopback, so the server is unreachable from outside the container (`topics/valkey-bundle.md:162`)
**Doc says:**
> bind 127.0.0.1
> protected-mode yes

and line 244:
> docker run -v /myvalkey/conf:/usr/local/etc/valkey --name my-valkey-bundle valkey/valkey-bundle valkey-server /usr/local/etc/valkey/valkey.conf

**Problem:** Inside a container, `bind 127.0.0.1` accepts only connections from within the container. The run command also has no `-p` flag. Even with a port published, host clients cannot connect.

**Evidence:** I started the container with the F1 lines fixed and `-p 127.0.0.1:21541:6379`:
```
$ valkey-cli -p 21541 -a strong_password ping
Error: Connection reset by peer
$ docker exec dr-conf valkey-cli -a strong_password ping
PONG
```
**Fix:** Change the sample to `bind 0.0.0.0` (keep `requirepass`) with the comment `# Listen on all container interfaces; Docker port mapping controls exposure`. Change the run command to `docker run -d -p 6379:6379 -v /myvalkey/conf:/usr/local/etc/valkey --name my-valkey-bundle valkey/valkey-bundle valkey-server /usr/local/etc/valkey/valkey.conf`.

### F3 [blocker] EXAMPLE: LDAP example shows `AUTH ... OK` but skips the required ACL user and uses a server that does not exist (`topics/valkey-bundle.md:102-113`)
**Doc says:**
> CONFIG SET ldap.servers "ldap://ldap.valkey.io:389"
> ...
> AUTH valkey "ldap_password"
> OK

**Problem:** valkey-ldap only authenticates users that already exist in the Valkey ACL. The example never creates user `valkey`, so `AUTH` fails even against a real LDAP server. `ldap.valkey.io` does not resolve, so the `OK` output is invented.

**Evidence:** valkey-ldap@1.1.1:README.md:5: "the user must already exist in Valkey before LDAP can be used for authentication." Run against the bundle image:
```
$ valkey-cli CONFIG SET ldap.servers "ldap://ldap.valkey.io:389"
OK
$ valkey-cli AUTH valkey ldap_password
WRONGPASS invalid username-password pair or user is disabled.
$ getent hosts ldap.valkey.io
(no result)
```
**Fix:** Before `AUTH`, add `> ACL SETUSER valkey on resetpass +@all ~*` with output `OK`. Name the server as a placeholder (`ldap://<your-ldap-host>:389`) and add the sentence "Replace the server and DN values with your directory's; see [LDAP authentication](ldap.md#setting-up-valkey-users)."

### F4 [minor] EXAMPLE: INFO modules output is stale and does not match the current image (`topics/valkey-bundle.md:60-66`)
**Doc says:**
> module:name=bf,ver=10000,...
> module:name=search,ver=10000,...
> module:name=json,ver=10010,...
> module:name=ldap,ver=16777471,...

**Problem:** Every version number differs from the current image, and the module order is different. The real output also lists the built-in `lua` module and dozens of module INFO sections (`json_core_metrics`, `search_*`, and others), so a reader comparing the two will think something is wrong.

**Evidence:** `docker exec <c> valkey-cli INFO modules` on valkey/valkey-bundle:latest (VALKEY_VERSION=9.1.2):
```
module:name=json,ver=10003,api=1,filters=0,usedby=[search],using=[],options=[handle-io-errors|handle-repl-async-load]
module:name=ldap,ver=16843263,api=1,filters=0,usedby=[],using=[],options=[handle-atomic-slot-migration]
module:name=bf,ver=10001,api=1,filters=0,usedby=[],using=[],options=[handle-io-errors]
module:name=search,ver=66049,api=1,filters=0,usedby=[],using=[json],options=[handle-io-errors|handle-repl-async-load|no-implicit-signal-modified]
module:name=lua,ver=1,api=1,filters=0,usedby=[],using=[],options=[handle-repl-async-load|handle-atomic-slot-migration]
```
**Fix:** Use `MODULE LIST`, which prints only the module entries, or paste the five lines above and add "Version numbers vary by image tag." Also state which module versions each image ships (for example, 9.1 ships JSON 1.0.3, Bloom 1.0.1, Search 1.2.1, LDAP 1.1.1, per valkey-bundle `versions.json`), or link to that file.

### F5 [minor] MISSING: "Persistent storage" example persists nothing beyond the container's life (`topics/valkey-bundle.md:125-131`)
**Doc says:**
> Persistent storage allows you to save data snapshots locally. ...
> docker run --name my-valkey-bundle -d valkey/valkey-bundle valkey-server --save 60 1 --loglevel warning

**Problem:** The image declares no volume (`docker image inspect` shows `Volumes: map[]` and `WorkingDir: /data`). Snapshots go to /data in the container's writable layer and are lost when the container is removed. "Locally" suggests the host.

**Evidence:** `docker image inspect valkey/valkey-bundle --format '{{.Config.Volumes}} {{.Config.WorkingDir}}'` gives `map[] /data`.

**Fix:** `docker run --name my-valkey-bundle -d -v valkey-data:/data valkey/valkey-bundle valkey-server --save 60 1 --loglevel warning`, plus the sentence "Snapshots are written to `/data` in the container; mount a volume there to keep them after the container is removed."

### F6 [minor] OUTDATED: Example tags point at superseded releases (`topics/valkey-bundle.md:30`, `:43`)
**Doc says:**
> using a version like 8.1-bookworm
> docker run --name my-valkey-bundle -d -p 6379:6379 valkey/valkey-bundle:8.1.0-alpine

**Problem:** Both tags still exist on Docker Hub, but 8.1.0 is the first 8.1 release with older modules, and the bundle now builds on Debian trixie (`versions.json`: `"debian": {"version": "trixie"}` for 8.1, 9.0 and 9.1). Readers copying the example get an old server.

**Evidence:** valkey-bundle@d5b6592:versions.json: `"9.1": {"version": "9.1.3", "valkey-server": {"version": "9.1.2"} ...`. 9.1/debian/Dockerfile: `FROM docker.io/valkey/valkey:9.1.2-trixie`.

**Fix:** Line 30: "using a version like 9.1-trixie". Line 43: `valkey/valkey-bundle:9.1-alpine`.

### F7 [minor] CONFUSING: Inconsistent heading hierarchy; advanced setup has no heading (`topics/valkey-bundle.md:6`, `:18`, `:123`, `:249`)
**Doc says:**
> ## What is Valkey Bundle
> # Quick Start to Using the Bundle
> Valkey Bundle supports more advanced setup options too including:
> ## Next Steps

**Problem:** The live page renders an h2, then an h1, then an h2. Persistence, environment flags, and the custom config file sit as numbered items under a plain sentence inside Quick Start, so they cannot be linked to or found in the outline.

**Evidence:** The live page has `<h2 id="what-is-valkey-bundle">`, `<h1 id="quick-start-to-using-the-bundle">`, `<h2 id="next-steps">`.

**Fix:** Make line 18 `## Quick start with Docker`. Replace line 123 with `## Configure persistence, flags, and a custom config file`. Make items 1-3 `###` headings: "Persist data with snapshots", "Pass flags with VALKEY_EXTRA_FLAGS", "Use a custom configuration file". Rename line 6 to `## What is Valkey Bundle?`.

### F8 [nit] EXAMPLE: Module command examples show no output, and the CLI sessions are tagged `bash` (`topics/valkey-bundle.md:59-113`, `:153`, `:159`)
**Doc says:**
> ```bash
> > JSON.SET test $ '{"hello": "world"}'
> ```

**Problem:** The JSON, Bloom, and Search blocks show commands with no replies, while the LDAP block shows replies (fabricated, see F3). The prompts also differ: `my-valkey-bundle:6379>` at line 60, `>` elsewhere. valkey-cli sessions and the config file are tagged `bash`.

**Evidence:** Real replies from the image: JSON.SET `OK`, JSON.GET `{"hello":"world"}`, BF.RESERVE `OK`, BF.ADD `1`, BF.EXISTS `1`, FT.CREATE `OK`, each HSET `1`, FT.SEARCH `3` followed by p1/p2/p3 with scores 0, 2, 2.

**Fix:** Merge each module's commands into one block with replies, use one prompt (`127.0.0.1:6379>`), and tag the config blocks `text` or `conf`.

### F9 [nit] STYLE: "Some of the modules" undersells the list (`topics/valkey-bundle.md:11`)
**Doc says:**
> Some of the modules included in the bundle are:

**Problem:** The list names all four modules the image ships.

**Evidence:** valkey-bundle@d5b6592:9.1/alpine/Dockerfile clones exactly json, bloom, search, and ldap.

**Fix:** "The bundle includes these modules:"

### F10 [nit] AI-NATIVE: Generic filler and weak front matter (`topics/valkey-bundle.md:3`, `:20`, `:251`)
**Doc says:**
> description: How to get started with Valkey Bundle
> The following steps will guide you through launching and interacting with your first instance!
> Once you've set up the Valkey Bundle, it's time to start exploring the modules. Check out the documentation for each one to learn what they can do and how to use them effectively.

**Problem:** These sentences carry no information. The description does not say what the bundle is.

**Evidence:** Quoted text above.

**Fix:** Description: "Run Valkey with the JSON, Bloom, Search, and LDAP modules preloaded using the valkey/valkey-bundle Docker image." Cut the line 20 sentence after the Docker Hub link, and cut line 251 entirely (the list stands on its own).

## Unverified
- topics/valkey-bundle.md:195: why `search.use-coordinator` in a config file aborts startup with "Module Configuration detected without loadmodule directive". I confirmed the abort by removing lines one at a time. I did not trace the cause in valkey-search 1.2.1 (the config is registered `.Hidden()` at src/valkey_search_options.cc:143-146).
- topics/valkey-bundle.md:206-209: whether `ldap.use_starttls yes` with nonexistent cert paths fails at startup or only at AUTH. Once the F1 lines were fixed, the server started with these lines present, but I did not exercise AUTH.
