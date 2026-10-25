VERDICT: NEEDS-WORK
COUNTS: blocker=1 minor=4 nit=2

# Command arguments (`command-arguments`)

https://valkey.io/topics/command-arguments/

The page describes the wrong reply format. It documents a `value` field that the server never sends and leaves out the real `arguments` key for nested arguments. The XADD example is a stale Redis 7.0 dump that doesn't match 9.1.2 output. Fix the nested-argument key and regenerate the example first.

## Findings

### F1 [blocker] INACCURATE: Nested arguments are under `arguments`, not `value`, and `value` is never sent (`topics/command-arguments.md:44`)
**Doc says:**
> * **value:** the argument's value.
>   For arguments types other than _oneof_ and _block_, this is a string that describes the value in the command's syntax.
>   For the _oneof_ and _block_ types, this is an array of nested arguments, each being a map as described in this section.

**Problem:** 9.1.2 never sends a `value` key. Nested arguments of `oneof` and `block` come back under `arguments`. A client written from this page finds no nested arguments and no value strings. The example at lines 70, 88, 114 and others uses `"value"` in the same wrong way.
**Evidence:** `valkey@9.1.2:src/server.c:5149-5186` sends only `name`, `type`, `display_text`, `key_spec_index`, `token`, `summary`, `since`, `deprecated_since`, `flags`, and then:
```
        if (args[j].type == ARG_TYPE_ONEOF || args[j].type == ARG_TYPE_BLOCK) {
            addReplyBulkCString(c, "arguments");
            addReplyCommandArgList(c, args[j].subargs, args[j].num_args);
        }
```
Live `COMMAND DOCS XADD` on 9.1.2 shows `arguments` nested under `trim` and has no `value` key anywhere (see F2).
**Fix:** Delete the `value` bullet and add: `* **arguments:** present only for _oneof_ and _block_ arguments. An array of nested arguments, each a map as described in this section.` Update line 30/32 wording to match.

### F2 [minor] EXAMPLE: XADD example doesn't match 9.1.2 output (`topics/command-arguments.md:63`)
**Doc says:**
> ```
> 1) 1) "name"
>    2) "key"
>    3) "type"
>    4) "key"
>    5) "value"
>    6) "key"
> ...
> 4) 1) "name"
>    2) "id_or_auto"
> ...
>          2) "auto_id"
> ...
> 5) 1) "name"
>    2) "field_value"
> ```
**Problem:** The dump has no `display_text` or `key_spec_index` (both are documented at lines 17 and 33). It uses `value` instead of `display_text`/`arguments`, shows `since` as `6.2` instead of `6.2.0`, and uses old argument names (`id_or_auto`, `auto_id`, `field_value`). The flags are shown unquoted (`optional`), which doesn't match the quoted strings around them.
**Evidence:** `valkey-cli -p 20280 COMMAND DOCS XADD` on 9.1.2 (raw output, abridged):
```
arguments
name
key
type
key
display_text
key
key_spec_index
0
name
nomkstream
type
pure-token
display_text
nomkstream
token
NOMKSTREAM
since
6.2.0
flags
optional
name
trim
type
block
flags
optional
arguments
...
name
id-selector
type
oneof
arguments
name
auto-id
...
name
data
type
block
flags
multiple
arguments
```
**Fix:** Replace the block with fresh output of `COMMAND DOCS XADD` from valkey-cli 9.1.2 (the `arguments` value only), tag the fence as `text`, and change "Here's `XADD`'s arguments array" to "Here's the `arguments` array that `COMMAND DOCS XADD` returns on Valkey 9.1".

### F3 [minor] INACCURATE: Flag is `multiple_token`, not `multiple-token` (`topics/command-arguments.md:43`)
**Doc says:**
> - **multiple-token:** denotes the possible repetition of the argument with its preceding token (see `SORT`'s `GET pattern` clause).
**Problem:** The server sends the flag with an underscore. A client matching the hyphenated string never matches.
**Evidence:** `valkey@9.1.2:src/server.c:5125`: `{CMD_ARG_MULTIPLE_TOKEN, "multiple_token"}`. Live `COMMAND DOCS SORT` shows `flags` → `optional`, `multiple`, `multiple_token` on `get-pattern`.
**Fix:** `- **multiple_token:** ...`

### F4 [minor] OUTDATED: `since` and `deprecated_since` are described as Redis OSS versions (`topics/command-arguments.md:37`)
**Doc says:**
> * **since:** the debut Redis OSS version of the argument (or for module commands, the module version).
> * **deprecated_since:** the Redis OSS version that deprecated the command (or for module commands, the module version).
**Problem:** Many arguments were added in Valkey releases, so these values are Valkey versions. For example, CLIENT KILL filters have `"since": "9.0.0"` and BGSAVE has an argument with `"since": "8.1.0"`. commands/command-docs.md:35 already says "the Valkey version". Line 38 also says "the command" when this field describes the argument.
**Evidence:** `valkey@9.1.2:src/commands/client-kill.json:178`: `"since": "9.0.0"`. `valkey@9.1.2:src/commands/bgsave.json:40`: `"since": "8.1.0"`.
**Fix:** `* **since:** the Valkey version that introduced the argument (or for module commands, the module version).` / `* **deprecated_since:** the Valkey version that deprecated the argument (or for module commands, the module version).`

### F5 [minor] AI-NATIVE: Intro doesn't say who the page is for or how to call the command (`topics/command-arguments.md:6`)
**Doc says:**
> The `COMMAND DOCS` command returns documentation-focused information about available Valkey commands.
**Problem:** The intro never says that this format is for client library and tooling authors who render command syntax or validate input. It also doesn't link to the command page, so readers can't see how the `arguments` key fits into the full reply.
**Fix:** Replace lines 6-8 with: "The [`COMMAND DOCS`](../commands/command-docs.md) reply includes an _arguments_ key for each command: an array that describes the command's syntax as a tree. Client libraries and tools use it to render syntax or validate input. This page describes each argument map's fields and walks through `XADD` as an example."

### F6 [nit] STYLE: Example fence has no language tag (`topics/command-arguments.md:65`)
**Doc says:**
> ```` ``` ````
**Problem:** The fenced block has no language tag.
**Fix:** Use ```` ```text ````.

### F7 [nit] AI-NATIVE: Generic heading "Example" (`topics/command-arguments.md:50`)
**Doc says:**
> ## Example
**Problem:** The heading doesn't say what the example shows.
**Fix:** `## Example: XADD's trimming clause and arguments array`
