VERDICT: MINOR-ISSUES
COUNTS: blocker=0 minor=4 nit=4

# Pipelining (`pipelining`)

https://valkey.io/topics/pipelining/

The concept explanation is sound and the netcat example reproduces on 9.1.2. The Ruby example is broken on current redis-rb, and the page gives no Valkey-specific limits or caveats (reply buffer limits, non-atomicity) a reader needs before sending large pipelines.

## Findings

### F1 [minor] EXAMPLE: Ruby benchmark uses the `pipelined` form redis-rb 5.0 removed (`topics/pipelining.md:117`)
**Doc says:**
> ```ruby
>   r.pipelined do
>     10_000.times do
>       r.ping
> ```
**Problem:** Since redis-rb 5.0, commands inside `pipelined` must be called on the block argument. Calling them on the outer client is no longer supported, so a reader on any current gem does not get the pipelined benchmark the page describes.
**Evidence:** redis/redis-rb CHANGELOG.md:162 (5.0.0):
```
- Removed the deprecated `pipelined` and `multi` signature. Commands now MUST be called on the block argument, not the original redis instance.
```
CHANGELOG.md:195 (4.6.0): "Deprecate calling commands on `Redis` inside `Redis#pipelined`."
**Fix:** Replace lines 117-121 with:
```ruby
  r.pipelined do |pipeline|
    10_000.times do
      pipeline.ping
    end
  end
```
Line 96 should also say the `redis` gem is a Redis client that works against Valkey, or the example should use a client listed in clients/ruby/ (valkey-GLIDE).

### F2 [minor] INACCURATE: SCRIPT LOAD does not guarantee EVALSHA will succeed (`topics/pipelining.md:146`)
**Doc says:**
> Valkey explicitly supports it with the [SCRIPT LOAD](../commands/script-load.md) command (it guarantees that `EVALSHA` can be called without the risk of failing).
**Problem:** The script cache is emptied by `SCRIPT FLUSH`, a restart, or a failover to a replica that never got the script. Any of these, or another client's flush between the load and the call, makes a pipelined EVALSHA return NOSCRIPT. topics/eval-intro.md:180-184 says exactly this and tells clients to fall back to EVAL in pipelines, so the two pages contradict each other.
**Evidence:** 9.1.2 run:
```
$ valkey-cli -p 21000 script load "return 1"
e0e1f9fabfc9d4800c877a703b823ac0578ff8db
$ valkey-cli -p 21000 script flush
OK
$ valkey-cli -p 21000 evalsha e0e1f9fabfc9d4800c877a703b823ac0578ff8db 0
NOSCRIPT No matching script.
```
topics/eval-intro.md:182: "Because of that, the `NOSCRIPT` error can return from a pipelined request but can't be handled."
**Fix:** Replace with: "Use [SCRIPT LOAD](../commands/script-load.md) before the pipeline so `EVALSHA` normally finds the script. The script cache can still be emptied (by `SCRIPT FLUSH`, a restart, or a failover), and a pipelined `EVALSHA` then returns `NOSCRIPT`. See [EVALSHA in the context of pipelining](eval-intro.md#evalsha-in-the-context-of-pipelining)."

### F3 [minor] MISSING: memory warning omits the Valkey limits that apply (`topics/pipelining.md:71`)
**Doc says:**
> the server will be forced to queue the replies, using memory. So if you need to send a lot of commands with pipelining, it is better to send them as batches each containing a reasonable number, for instance 10k commands
**Problem:** The warning is generic. It does not say that normal clients have no reply buffer limit by default, so a client that pipelines without reading can grow server memory without bound. It also does not mention the query buffer limit that caps unread input. These are the Valkey-specific facts an operator needs.
**Evidence:** 9.1.2 defaults:
```
client-output-buffer-limit
normal 0 0 0 slave 268435456 67108864 60 pubsub 33554432 8388608 60
client-query-buffer-limit
1073741824
```
**Fix:** Append to the note: "By default, `client-output-buffer-limit` is `normal 0 0 0`, so Valkey does not limit the reply buffer of a normal client. Unread replies keep growing until memory runs out. Pending input is capped by `client-query-buffer-limit` (default 1gb). Use `INFO clients` or `CLIENT LIST` (`omem`, `qbuf` fields) to watch these buffers." Also link [valkey-benchmark -P](benchmark.md#using-pipelining) as the way to measure the right batch size.

### F4 [minor] MISSING: no statement that a pipeline is not atomic (`topics/pipelining.md:58`)
**Doc says:**
> This time we don't pay the cost of RTT for every call, but just once for the three commands.
**Problem:** Nothing on the page says that commands in a pipeline run in order but can be interleaved with other clients' commands, or that one command failing does not stop the rest. Readers often confuse pipelining with transactions. eval-intro.md:181 states the interleaving, but only in the scripting context.
**Evidence:** topics/eval-intro.md:181: "The commands in a pipelined request run in the order they are sent, but other clients' commands may be interleaved for execution between these."
**Fix:** Add after line 69: "Valkey runs pipelined commands in the order they were sent, but a pipeline is not atomic. Commands from other clients can run between them, and an error in one command does not stop the commands after it. For atomic execution, use [MULTI/EXEC](transactions.md) or a [script](eval-intro.md)."

### F5 [nit] CONFUSING: the throughput graph has no axis labels and no context (`topics/pipelining.md:89`)
**Doc says:**
> eventually reaches 10 times the baseline obtained without pipelining, as shown in this figure.
**Problem:** The PNG has no axis titles. Its only legend is the gnuplot artifact `"/tmp/pipelining.txt"`. The y-axis starts at 100000, so the "10 times" claim cannot be read off the figure. The hardware and server version are also unknown.
**Fix:** Add a caption under the image: "X axis: pipeline length (commands per batch). Y axis: operations per second." Change "eventually reaches 10 times the baseline" to "levels off at roughly ten times the unpipelined rate in this test". Or regenerate the graph with labeled axes using `valkey-benchmark -P`.

### F6 [nit] AI-NATIVE: weak heading and filler (`topics/pipelining.md:73`)
**Doc says:**
> ## It's not just a matter of RTT

and line 46:
> For instance many POP3 protocol implementations already support this feature, dramatically speeding up the process of downloading new emails from the server.
**Problem:** The heading does not name what the section covers (syscall overhead). The POP3 sentence is trivia that adds no Valkey information.
**Fix:** Rename the heading to "Pipelining reduces syscall overhead, not just RTT". Delete line 46.

### F7 [nit] STYLE: first-person, informal, and imprecise wording (`topics/pipelining.md:83`, `:132`, `:172`)
**Doc says:**
> The context switch is a huge speed penalty.

> yields the following figures on my MacOS system

> Basically a busy loop benchmark is the silliest thing that can be done
**Problem:** A `read()`/`write()` syscall costs a user-to-kernel mode switch, not a process context switch. Line 132 says "my" with no author identified. "Silliest" is tone, not guidance.
**Fix:** Line 83: "Each syscall switches from user mode to kernel mode, which is expensive compared to the command itself." Line 132: "on a macOS machine". Lines 172-174: "A single-connection busy loop measures scheduler and round-trip latency, not server throughput. Use multiple connections or pipelining instead (see [benchmark](benchmark.md))."

### F8 [nit] EXAMPLE: code fence language tags (`topics/pipelining.md:134`, `:155`)
**Doc says:** Line 134 opens the output block with a bare fence. Line 155 tags pseudo-code as `sh`.
**Problem:** The output block has no language tag. The pseudo-code is not shell, so `sh` highlighting is wrong.
**Fix:** Use `text` for both fences.

## Unverified
- topics/pipelining.md:135-138: "1.185238 seconds" vs "0.250783 seconds" timings. These are from an old, unknown setup. I did not rerun them because Ruby and the gem are not installed here.
