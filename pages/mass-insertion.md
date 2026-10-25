VERDICT: NEEDS-WORK
COUNTS: blocker=0 minor=5 nit=4

# Bulk loading (`mass-insertion`)

https://valkey.io/topics/mass-insertion/

The page is a lightly renamed Redis-era doc. Its `--pipe` mechanics are right, but it says errors go to stdout (they go to stderr), its RESP template is wrong, and it says nothing about cluster mode. Fix the stdout/stderr claim and the RESP template first.

## Findings

### F1 [minor] INACCURATE: pipe mode writes errors to stderr, not stdout (`topics/mass-insertion.md:55`)
**Doc says:**
> The valkey-cli utility will also make sure to only redirect errors received
> from the Valkey instance to the standard output.

**Problem:** Error replies go to stderr. Stdout gets the progress lines and the `errors:/replies:` summary. A reader who captures stdout to collect failures gets no error text. The page also never says that the process exits 1 when any error occurs, which is what scripts need.

**Evidence:** valkey@9.1.2:src/valkey-cli.c:8994-8996, 9087-9091
```c
if (reply->type == VALKEY_REPLY_ERROR) {
    fprintf(stderr, "%s\n", reply->str);
    errors++;
...
printf("errors: %lld, replies: %lld\n", errors, replies);
if (errors) exit(1); else exit(0);
```
Run on 9.1.2 (`SET`, `SET`, `INCR` on a string), with stdout and stderr captured separately:
```
exit=1
OUT
All data transferred. Waiting for the last reply...
Last reply received from server.
errors: 1, replies: 3
ERR
ERR value is not an integer or out of range
```
**Fix:** "valkey-cli prints each error reply from the server to standard error, and progress and the final `errors: N, replies: N` summary to standard output. It exits with status 1 if any command returned an error, and 0 otherwise."

### F2 [minor] INACCURATE: RESP template shows one `$<len>` for all arguments (`topics/mass-insertion.md:65`)
**Doc says:**
```
*<args><cr><lf>
$<len><cr><lf>
<arg0><cr><lf>
<arg1><cr><lf>
...
<argN><cr><lf>
```
**Problem:** In RESP, every argument needs its own `$<len>` line in front of it. The template shows one `$<len>` for all of them, which contradicts the `SET key value` example right below it (lines 76-82). Someone who writes a generator from the template will produce invalid protocol.

**Evidence:** The page's own example at lines 76-82 has a `$N` line before each argument. The Ruby generator at lines 97-99 also writes `$`+bytesize for each argument.

**Fix:**
```
*<number of arguments><cr><lf>
$<length of arg0><cr><lf>
<arg0><cr><lf>
$<length of arg1><cr><lf>
<arg1><cr><lf>
...
$<length of argN><cr><lf>
<argN><cr><lf>
```

### F3 [minor] EXAMPLE: Ruby snippets combined into `proto.rb` produce an error (`topics/mass-insertion.md:104`)
**Doc says:**
> puts gen_redis_proto("SET","mykey","Hello World!").inspect

and then at line 119: `$ ruby proto.rb | valkey-cli --pipe` with `errors: 0, replies: 1000`.

**Problem:** `proto.rb` needs the function from the first snippet. If the reader copies that whole snippet, the `puts ... .inspect` debug line writes a quoted string with literal backslashes to stdout. The server rejects it as an unknown inline command, so the run does not give the output the page shows.

**Evidence:** 9.1.2 run with both snippets in one `proto.rb`:
```
ERR unknown command '*3  $3  SET  $5  mykey  $12  Hello World!  ', with args beginning with:
All data transferred. Waiting for the last reply...
Last reply received from server.
errors: 1, replies: 1001
exit=1
```
With the `puts` line removed: `errors: 0, replies: 1000`, exit 0.

**Fix:** Drop line 104 from the function snippet. Show `proto.rb` as one complete file (function plus the loop), and keep the `.inspect` demo out of it.

### F4 [minor] CONFUSING: plain text commands called "Valkey protocol format" (`topics/mass-insertion.md:24`)
**Doc says:**
> I will create a file containing the
> following commands in the Valkey protocol format:
>
>     SET Key0 Value0

**Problem:** The block is in inline command format, not RESP. Lines 20-22 and 58-89 say the file should contain RESP, and `valkey-cli --help` describes `--pipe` as "Transfer raw RESP protocol from stdin". Inline commands happen to work for this example. They break on values that contain spaces, quotes, newlines, or binary data, and the page never says so.

**Evidence:** valkey@9.1.2:src/valkey-cli.c:3048 `"  --pipe             Transfer raw RESP protocol from stdin to server.\n"`. The 9.1.2 run of the plain text `data.txt` above was accepted as inline commands.

**Fix:** "For example, to load keys of the form `KeyN -> ValueN`, the file must contain these commands, encoded in RESP (see [Generating RESP protocol](#generating-resp-protocol)):" and keep the block as an illustration of the logical commands.

### F5 [minor] MISSING: no guidance for cluster mode or `--pipe-timeout` (`topics/mass-insertion.md:45`)
**Doc says:**
> cat data.txt | valkey-cli --pipe

**Problem:** `--pipe` sends raw bytes over a single connection and does not follow `MOVED`/`ASK` redirects. Against a cluster, every key not owned by the target node fails with a MOVED error, and `-c` has no effect on this. The page also leaves out `--pipe-timeout` (default 30 seconds, after which the run fails if no reply arrives), which matters for slow or large loads.

**Evidence:** valkey@9.1.2:src/valkey-cli.c:10192-10194 connects once (`cliConnect(0)`, then `pipeMode()`). The reply loop at 8994-9009 only counts and prints errors, with no redirect handling. Timeout: src/valkey-cli.c:85 `#define CLI_DEFAULT_PIPE_TIMEOUT 30`, :9081-9083 `"No replies for %d seconds: exiting."`.

**Fix:** Add a short section: "In cluster mode, `--pipe` does not follow redirects. Split the input by hash slot and pipe each part to the primary that owns those slots. Keys sent to the wrong node fail with `MOVED`." Also add: "If no reply arrives for `--pipe-timeout` seconds (default 30, `0` waits forever) after all input is sent, valkey-cli exits with an error." Link to [valkey-cli](cli.md) for the flag reference.

### F6 [nit] OUTDATED: Ruby function named `gen_redis_proto` (`topics/mass-insertion.md:94`)
**Doc says:**
> def gen_redis_proto(*cmd)

**Problem:** This is Redis naming on a Valkey page that has nothing to do with compatibility or migration. The name also appears at lines 104 and 112.

**Fix:** Rename it `gen_resp` in all three places.

### F7 [nit] STYLE: broken inline code and untagged code blocks (`topics/mass-insertion.md:25`)
**Doc says:**
> keys in the form: `keyN -> ValueN' I will create

**Problem:** The code span opens with a backtick and closes with an apostrophe, so it renders literally on the live page (`` `keyN -&gt; ValueN' ``). The page also writes `keyN` but uses `Key0` in the examples. Every non-Ruby example (lines 28, 37, 47, 51, 65, 76, 86, 119) is an indented block with no language tag. "[Documented here]" (line 61) has a stray capital.

**Fix:** `` `KeyN -> ValueN` ``. Convert the indented blocks to fenced blocks tagged `bash` (shell commands) or `text` (output and protocol). Change line 61 to "is [documented here](protocol.md)".

### F8 [nit] STYLE: first person voice (`topics/mass-insertion.md:24`)
**Doc says:**
> For instance if I need to generate a large data set ... I will create a file

**Problem:** The page switches between "I" (lines 24-25), "we" (lines 116, 136, 142, 145), and "you". The rest of the docs use second person.

**Fix:** "For example, to generate a data set of billions of keys of the form `KeyN -> ValueN`, create a file ...". Line 116: "Pipe the program's output to valkey-cli:". Lines 142-147: "valkey-cli does not need to parse the protocol it sends. It only parses replies, and counts them to report how many commands were transferred."

### F9 [nit] AI-NATIVE: intro and description undersell the page; netcat history is filler (`topics/mass-insertion.md:4`)
**Doc says:**
> description: Writing data in bulk using the RESP protocol

and line 7: "This document describes some strategies for bulk loading data in Valkey."

**Problem:** There is one strategy, `valkey-cli --pipe`, and neither the description nor the intro names it. Lines 33-40 describe the old netcat method ("In the past ...") only to reject it.

**Fix:** description: "Load large data sets into Valkey quickly by generating RESP and streaming it with valkey-cli --pipe." Intro: "This page shows how to bulk load data into Valkey by generating a RESP command file and streaming it with `valkey-cli --pipe`, which reports errors and confirms that every command was processed." Cut lines 33-40 down to one sentence: "Piping the file with `nc` is not reliable: it cannot tell when the server has processed the last command and does not report errors."
