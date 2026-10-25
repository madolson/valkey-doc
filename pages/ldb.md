VERDICT: MINOR-ISSUES
COUNTS: blocker=0 minor=4 nit=4

# Debugging Lua scripts (`ldb`)

https://valkey.io/topics/ldb/

The debugger model and sync/fork behavior are described correctly, but every transcript on the page shows pre-9.0 output, and two documented commands (`m`, `b 1 2 3 4`) fail on 9.1.2. Most important fix: replace the help screen and session transcripts with real 9.1.2 output.

## Findings

### F1 [minor] EXAMPLE: Help screen does not match 9.1.2, and `m` no longer abbreviates `maxlen` (`topics/ldb.md:64`)
**Doc says:**
> Valkey Lua debugger help:
> ...
> [r]edis <cmd>        Execute a Valkey command.
> [m]axlen [len]       Trim logged Valkey replies and Lua var dumps to len.
> ...
> Debugger functions you can call from Lua scripts:
> server.debug()        Produce logs in the debugger console.

and at line 134:
> every Valkey debugger command starts with a different character, and you can use the single initial character in order to refer to the command.

**Problem:** In 9.1.2 the help title is `LUA debugger help:`, `maxlen` is listed first as `[max]len`, `[r]edis` is now `[v]alkey`, `list`/`print`/`break` are each one entry, and the "Debugger functions" footer is gone. `maxlen` needs the 3-char prefix `max`, so `m 10` returns an error. The one-character claim at line 134 is false for `maxlen`. The help output changed with the scripting engine refactor in 9.0. 8.1.0 still printed `Lua debugger help:` (`valkey@8.1.0:src/lua/debug_lua.c:875`).
**Evidence:** `valkey@9.1.2:src/scripting_engine.c:944-947`:
```
        .name = "maxlen",
        .prefix_len = 3,
```
`valkey@9.1.2:src/modules/lua/debug_lua.c:675-676` (`redis` is a hidden alias):
```
VALKEYMODULE_SCRIPTING_ENGINE_DEBUGGER_COMMAND("valkey", 1, valkey_params, 1, "Execute a command.", 0, valkeyCommandHandler),
VALKEYMODULE_SCRIPTING_ENGINE_DEBUGGER_COMMAND("redis", 1, valkey_params, 1, NULL, 1, valkeyCommandHandler),
```
Live 9.1.2:
```
lua debugger> m 10
<error> Unknown debugger command or wrong number of arguments.
lua debugger> max
Current maxlen is 256
```
`help` prints:
```
LUA debugger help:
[h]elp               Show this help.
[max]len [len]       Trim logged replies to len. Specifying zero as
                     <len> means unlimited. If no <len> is specified,
                     the current value is shown. Usage: maxlen [len]
[s]tep               Run current line and stop again.
[n]ext               Alias for step.
[c]ontinue           Run till next breakpoint.
[l]ist [line] [ctx]  List source code around a specific line. If no
                     line is specified the list is printed around the
                     current line. [ctx] specifies how many lines to
                     show before/after [line].
[w]hole              List all source code. Alias for 'list 1 1000000'.
[p]rint [var]        Show the value of the specified variable [var].
                     Can also show global vars KEYS and ARGV. If no
                     [var] is specidied, shows the value of all local
                     variables.
[b]reak [line|-line] Add/Remove a breakpoint to the specified line.
                     If no [line] is specified, it shows all
                     breakpoints. When line = 0, it removes all
                     breakpoints.
[t]race              Show a backtrace.
[e]val <code>        Execute some Lua code (in a different callframe).
[v]alkey <cmd>       Execute a command.
[a]bort              Stop the execution of the script. In sync mode
                     dataset changes will be retained.
```
**Fix:** Replace the block at lines 64-96 with the 9.1.2 output above (prefixed with `lua debugger> help`). Move `server.debug()` and `server.breakpoint()` into a sentence below it, since the help no longer lists them. Change line 134-136 to: "Most debugger commands can be abbreviated to their first character, shown in brackets in the help output. For example, type `s` instead of `step`. `maxlen` needs at least `max`."

### F2 [minor] EXAMPLE: Session transcripts show pre-9.0 tags and formatting (`topics/ldb.md:108`)
**Doc says:**
> <redis> ping

(line 108). Line 113: "The `<redis>` and `<reply>` lines show...". Line 201: `<debug> line 3: {1; 2; 3}, false`. Line 228: `<retval> "0beec7b5ea3f0fdbc95d0dd47f3c5bc275da8a33"`.
**Problem:** 9.1.2 logs executed commands with `<command>`, not `<redis>`. Tables print with a trailing `; `, and `eval` return strings are unquoted. A reader matching the page against their terminal sees different output on every example.
**Evidence:** Live 9.1.2, script `server.call('ping')` / `local a = {1,2,3}` / `local b = false` / `server.debug(a,b)`:
```
* Stopped at 1, stop reason = step over
-> 1   server.call('ping')
<command> ping
<reply> "+PONG"
* Stopped at 2, stop reason = step over
...
<retval> 0beec7b5ea3f0fdbc95d0dd47f3c5bc275da8a33
...
<debug> line 4: {1; 2; 3; }, false
```
**Fix:** Line 108: `<command> ping`. Line 113: "The `<command>` and `<reply>` lines show...". Line 201: `<debug> line 3: {1; 2; 3; }, false`. Line 228: `<retval> 0beec7b5ea3f0fdbc95d0dd47f3c5bc275da8a33`.

### F3 [minor] INACCURATE: `b 1 2 3 4` is rejected; `break` takes one line (`topics/ldb.md:143`)
**Doc says:**
> Just use `b 1 2 3 4` to add a breakpoint in line 1, 2, 3, 4.

**Problem:** In 9.1.2 `break` accepts at most one argument. Passing several returns an error and sets no breakpoints.
**Evidence:** `valkey@9.1.2:src/modules/lua/debug_lua.c:653`: `{.name = "line|-line", .optional = 1},` (not variadic). `checkCommandParameters` rejects `args_count > cmd->params_len` (`valkey@9.1.2:src/scripting_engine.c`). Live:
```
lua debugger> b 1 2 3 4
<error> Unknown debugger command or wrong number of arguments.
lua debugger> b 3 4
<error> Unknown debugger command or wrong number of arguments.
lua debugger> b
No breakpoints set. Use 'b <line>' to add one.
```
**Fix:** "Use `b <line>` to add a breakpoint, for example `b 3`. Repeat the command to add more."

### F4 [minor] MISSING: No mention of `SCRIPT DEBUG`, or that only `EVAL` scripts can be debugged (`topics/ldb.md:233`)
**Doc says:**
> any [client](../clients/) can be used for debugging as long as it meets one of the following conditions: ... The client provides an interface for sending arbitrary commands over RESP.

**Problem:** The page never names the command a client sends to start a session, so the "Debugging clients" section can't be acted on. It also never says LDB works only for `EVAL` scripts, not Functions (`FCALL`). That limit is stated elsewhere (topics/functions-intro.md:62 "with the only exception being the Valkey Lua scripts debugger"; commands/script-debug.md:1 "Set the debug mode for subsequent scripts executed with `EVAL`").
**Fix:** Append to line 233: "A client starts a session by sending [`SCRIPT DEBUG YES`](../commands/script-debug.md) (forked) or `SCRIPT DEBUG SYNC` (synchronous), then running `EVAL`. Debugger commands are sent as RESP arrays on the same connection." Add to the intro after line 7: "LDB debugs scripts run with `EVAL`. It does not support Valkey Functions (`FCALL`)."

### F5 [nit] BROKEN: Start command renders as plain paragraph, not code (`topics/ldb.md:43`)
**Doc says:**
>     ./valkey-cli --ldb --eval /tmp/script.lua

**Problem:** The 4-space indented code inside the list item renders on the live page as `<p>./valkey-cli --ldb --eval /tmp/script.lua</p>`.
**Fix:** Use a fenced block indented under the list item: ```` ```bash ```` ... ```` ``` ````. Also add a language tag (`bash` or `text`) to the untagged blocks at lines 47, 64, 104, 159 (`lua`), 178, 195, 226.

### F6 [nit] CONFUSING: `help` is listed among commands "not passed to the Valkey debugger" (`topics/ldb.md:62`)
**Doc says:**
> The only commands which are not passed to the Valkey debugger are: ... `help` -- this command is passed to the Valkey Lua debugger

**Problem:** The list says it covers commands that are not passed through, then says `help` is passed through.
**Fix:** Remove `help` from the list and say after it: "Type `help` to have the debugger list its commands:".

### F7 [nit] STYLE: Typos and grammar (`topics/ldb.md:21`)
**Doc says:** line 21 "Support from logging", line 120 "When the scripts terminates", line 168 "but default LDB uses", line 169 "while it has being debugged", line 184 "taking the changes operated to the dataset", line 191 "The `server.debug()` command".
**Fix:** "Support for logging", "When the script terminates", "by default LDB uses", "while it is being debugged", "keeping the changes made to the dataset", "The `server.debug()` function".

### F8 [nit] STYLE: Video iframe has no title (`topics/ldb.md:32`)
**Doc says:**
> <iframe width="560" height="315" src="https://www.youtube.com/embed/IMvRfStaoyM" frameborder="0" allowfullscreen></iframe>

**Problem:** No `title` attribute, so screen readers announce an unnamed frame.
**Fix:** Add `title="Introduction to the Lua debugger (LDB)"`.
