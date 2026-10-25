VERDICT: NEEDS-WORK
COUNTS: blocker=1 minor=4 nit=4

# Quick start guide (`quickstart`)

https://valkey.io/topics/quickstart/

The examples reuse one key for a string and then a hash, so a reader who types them in order hits WRONGTYPE on the second block. Fix the key names first. The page also never shows how to start a server or connect with valkey-cli, despite being the quick start.

## Findings

### F1 [blocker] EXAMPLE: Hash example fails with WRONGTYPE after string example (`topics/quickstart.md:38`)
**Doc says:**
> 127.0.0.1:6379> SET user:1000 "Alice"
...
> 127.0.0.1:6379> HSET user:1000 name "Alice" email "alice@example.com" age "30"
> (integer) 3
**Problem:** Line 27 makes `user:1000` a string. Line 38 runs HSET on the same key, which errors instead of returning 3.
**Evidence:** Valkey 9.1.2, run in page order:
```
$ valkey-cli SET user:1000 Alice
OK
$ valkey-cli HSET user:1000 name Alice email alice@example.com age 30
WRONGTYPE Operation against a key holding the wrong kind of value
```
**Fix:** Use a different key for the string example, for example `SET user:1000:name "Alice"` / `GET user:1000:name` at lines 27-30, and update the prose at line 33 to match.

### F2 [minor] MISSING: Quick start never shows how to start the server or connect (`topics/quickstart.md:20`)
**Doc says:**
> Once connected with Valkey, you can interact by issuing commands
**Problem:** The page jumps from "see the Installation Guide" to `127.0.0.1:6379>` prompts with no step for running `valkey-server` or `valkey-cli`. Line 99 then says "Now that you have Valkey running", which the page never covers.
**Evidence:** No occurrence of `valkey-server` (other than `--test-memory`) or a `valkey-cli` connect step on lines 1-96.
**Fix:** Add before "Data Operations": "Start a server with `valkey-server` (listens on port 6379 by default), then open another terminal and run `valkey-cli`. You should see the `127.0.0.1:6379>` prompt used in the examples below."

### F3 [minor] EXAMPLE: `valkey-server --test-memory` requires a size argument (`topics/quickstart.md:93`)
**Doc says:**
> Run `valkey-server --test-memory` to perform a memory test of your system.
**Problem:** Without a megabyte argument the command prints an error and exits.
**Evidence:**
```
$ valkey-server --test-memory
Please specify the amount of memory to test in megabytes.
Example: ./valkey-server --test-memory 4096
```
valkey@9.1.2:src/server.c:7546-7552.
**Fix:** "Run `valkey-server --test-memory <megabytes>` (for example `valkey-server --test-memory 4096`) to test your system's RAM."

### F4 [minor] MISSING: admin.md sends readers here for security guidance that does not exist (`topics/quickstart.md:78`)
**Doc says:** (topics/admin.md:42)
> Please check our [security page](security.md) and the [quick start](quickstart.md) for information about how to secure Valkey.
**Problem:** This page has no security content: no mention of protected mode, `bind`, `requirepass`, or ACLs. The troubleshooting bullet at line 90 tells readers to open port 6379 through firewalls without any warning about exposure.
**Evidence:** grep of topics/quickstart.md for `security|protected|bind|requirepass|ACL` matches only line 115, a passing mention of "security" in the link to index.md.
**Fix:** Add a best-practices bullet: "**Don't expose Valkey to untrusted networks:** by default Valkey has no password. Keep `bind` limited to trusted interfaces, leave `protected-mode` on, and configure [ACLs](acl.md) before opening port 6379. See [Security](security.md)." Or drop the quickstart link from admin.md:42.

### F5 [minor] AI-NATIVE: Marketing filler in caching section and closing (`topics/quickstart.md:76`)
**Doc says:**
> E-commerce platforms use Valkey to serve personalized recommendations instantly. Gaming companies rely on it ... Valkey is built to keep up — no matter how demanding the workload.
and line 117:
> Happy caching with Valkey! ... Good luck on your Valkey journey!
also line 95: "We want to ensure that Valkey runs smoothly in your environment."
**Problem:** Unverifiable, generic claims that add length and no instruction. Line 117 also refers back to "**Next steps** above", which the reader just read.
**Fix:** Cut line 76 after the first sentence, delete line 117 and the second sentence of line 95.

### F6 [nit] BROKEN: Stray `****` renders literally (`topics/quickstart.md:22`)
**Doc says:**
> Let’s walk through some fundamental operations **** with two
**Evidence:** Live page HTML: `fundamental operations **** with two of the most commonly used data types`
**Fix:** "Let’s walk through some fundamental operations with two of the most commonly used data types in Valkey: **strings** and **hashes**."

### F7 [nit] EXAMPLE: Fenced blocks have no language tag (`topics/quickstart.md:26`)
**Doc says:** bare ``` at lines 26, 37, 68.
**Fix:** Use the tag other valkey-doc pages use for CLI sessions (for example ```` ```bash ```` / ```` ```text ````), consistently across all three.

### F8 [nit] STYLE: Typos, grammar, and odd Redis comparison (`topics/quickstart.md:101`)
**Doc says:**
> Use [Try Valkey](https://valkey.io/try-valkey/) allows you to try Valkey
> Take a look at few of our official modules
> chat systems o live notifications
> **[Valkey Json](valkey-json.md)**
> append-only file logging (AOF) similar to Redis (line 86)
**Fix:** "[Try Valkey](https://valkey.io/try-valkey/) lets you run Valkey in your browser with no installation." / "a few of our official modules" / "chat systems or live notifications" / "Valkey JSON" / drop "similar to Redis" and link [Persistence](persistence.md). Lines 101, 108, 113, 109, 86.

### F9 [nit] STYLE: Front matter and headings misdescribe the page (`topics/quickstart.md:3`)
**Doc says:**
> description: Understand how to use basic Valkey data types
> ## Getting Started with Valkey
**Problem:** The page covers install pointers, basic commands, caching, best practices, and troubleshooting, not only data types. The H2 duplicates the title and every real section sits under it as H3. "Data Operations" says little about what the section answers.
**Fix:** description: "Run your first Valkey commands, cache data with a TTL, and avoid common setup mistakes." Remove the `## Getting Started with Valkey` wrapper and promote sections to H2. Rename "Data Operations" to "Store and read data with strings and hashes".
