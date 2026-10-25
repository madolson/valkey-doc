VERDICT: MINOR-ISSUES
COUNTS: blocker=0 minor=2 nit=4

# Command tips (`command-tips`)

https://valkey.io/topics/command-tips/

The tip names, the command examples, and the `COMMAND INFO PING` output all match 9.1.2. Most important fix: `agg_logical_or` is described as a logical AND.

## Findings

### F1 [minor] INACCURATE: agg_logical_or described as logical AND (`topics/command-tips.md:86`)
**Doc says:**
> **agg_logical_or:** the client should return the result of a logical _AND_ operation on all replies
**Problem:** Copy-paste from `agg_logical_and`. A client author implementing from this text would aggregate OR replies with AND.
**Evidence:** The line is identical to line 82 except for the tip name. The tip name says OR.
**Fix:** "**agg_logical_or:** the client should return the result of a logical _OR_ operation on all replies (only applies to integer replies, usually from commands that return either _0_ or _1_)."

### F2 [minor] INACCURATE: response default refers to `request_policy` (`topics/command-tips.md:61`)
**Doc says:**
> The default behavior for commands without a `request_policy` tip only applies to replies with of nested types (i.e., an array, a set, or a map).
**Problem:** This sits in the `response_policy` section and describes the default aggregation. The relevant tip is `response_policy`. As written, it reads as if `request_policy` controls how replies are aggregated.
**Evidence:** Commands such as `KEYS` (`REQUEST_POLICY:ALL_SHARDS`, no response policy) and `MGET` (`REQUEST_POLICY:MULTI_SHARD`, no response policy) in valkey@9.1.2:src/commands/keys.json and mget.json are the default-aggregation examples given at lines 65 and 68. Both have a request_policy and lack a response_policy.
**Fix:** "The default behavior for commands without a `response_policy` tip only applies to replies of nested types (an array, a set, or a map)."

### F3 [nit] STYLE: typos and missing words (`topics/command-tips.md:6`, `:61`, `:66`)
**Doc says:**
> This page documents a small part of the reply of the [`COMMAND`](../commands/command.md).

> replies with of nested types

> These should be packed in a single in no particular order.
**Problem:** Missing noun "command" (line 6), stray "with" (line 61), missing noun "array" (line 66).
**Fix:** "the reply of the [`COMMAND`](../commands/command.md) command", "replies of nested types", "These should be packed in a single array, in no particular order."

### F4 [nit] BROKEN: mixed bullet markers split the request_policy list (`topics/command-tips.md:43`)
**Doc says:**
> - **all_nodes:** ...
> * **all_shards:** ...
> - **multi_shard:** ...
**Problem:** Switching between `-` and `*` starts a new list in CommonMark. The four request_policy values render as three separate `<ul>` elements (the live page has 4 `<ul>`: 3 here plus 1 for response_policy). The response_policy list uses `*` throughout.
**Fix:** Use `*` for all items at lines 43, 50, 56.

### F5 [nit] EXAMPLE: fenced block has no language tag (`topics/command-tips.md:97`)
**Doc says:**
> ```
> 127.0.0.1:6379> command info ping
**Problem:** The block has no language tag. Its output does match 9.1.2 (verified against a live server). The heading "Example" also does not say what it shows.
**Fix:** Tag the fence `bash`, matching other CLI transcripts in valkey-doc, and rename the heading to "## Example: tips in the `COMMAND INFO` reply".

### F6 [nit] AI-NATIVE: intro does not say what tips are for or who reads them (`topics/command-tips.md:6`)
**Doc says:**
> This page documents a small part of the reply of the [`COMMAND`](../commands/command.md).
> In the reply of the COMMAND command, each command is represented by an array.
**Problem:** The audience (client and proxy authors routing commands in cluster mode) and the purpose only appear in the second paragraph. The opening sentence is about the page, not the subject.
**Fix:** Open with: "Command tips are strings that tell Valkey clients and proxies how to route a command in a cluster and how to combine replies from multiple shards. They appear as the 8th element of each command's entry in the [`COMMAND`](../commands/command.md) reply. This page is for client library authors." Then cut lines 6 to 12 down to whatever that opening does not already say.
