VERDICT: MINOR-ISSUES
COUNTS: blocker=0 minor=4 nit=4

# key-specs (`key-specs`)

https://valkey.io/topics/key-specs/

Structurally sound, but several stated facts contradict what 9.1.2 actually returns: MIGRATE's startfrom, which commands carry the incomplete flag, and the SET example output. Fix the incomplete section first, since a client author would key logic off it.

## Findings

### F1 [minor] INACCURATE: MIGRATE startfrom given as -1, contradicts line 66 and server (`topics/key-specs.md:177`)
**Doc says:**
> In the case of `MIGRATE`, the search begins at the end (_startfrom_ has the value of _-1_).

**Problem:** MIGRATE's keyword spec uses startfrom -2 (line 66 of the same page says -2). The page contradicts itself.
**Evidence:** `COMMAND INFO MIGRATE` on 9.1.2, second key spec:
```
flags RW access delete incomplete begin_search type keyword spec keyword KEYS startfrom -2 find_keys type range spec lastkey -1 keystep 1 limit 0
```
**Fix:** "In the case of `MIGRATE`, the search begins near the end (_startfrom_ has the value of _-2_) and scans backwards for the `KEYS` token."

### F2 [minor] INACCURATE: SORT does not carry the incomplete flag (`topics/key-specs.md:187`)
**Doc says:**
> the only commands with _incomplete_ key specifications are `SORT` and `MIGRATE`.

**Problem:** Only MIGRATE has the `incomplete` flag. SORT signals the problem with `begin_search`/`find_keys` of type `unknown`, which is a different mechanism. A client that checks for the `incomplete` flag to decide when to fall back to `COMMAND GETKEYS` would miss SORT. Lines 181-182 ("Another case of incompleteness is the `SORT` command") reinforce the confusion.
**Evidence:** Scanning `COMMAND INFO` for every command in `COMMAND LIST` on 9.1.2 for the `incomplete` flag:
```
INCOMPLETE: migrate
```
`git -C ~/git/valkey grep -ln '"INCOMPLETE"' 9.1.2 -- src/commands/` returns only `src/commands/README.md` and `src/commands/migrate.json`. SORT's specs as returned: `begin_search type unknown ... find_keys type unknown` with no `incomplete` flag.
**Fix:** Replace line 187 with: "`MIGRATE` is the only command with the _incomplete_ flag. `SORT` instead uses `begin_search` and `find_keys` of type _unknown_ for its `BY`/`GET` and `STORE` keys, so clients must also treat _unknown_ specs as a signal to fall back to `COMMAND GETKEYS`." Change line 181 to "A related case is the `SORT` command."

### F3 [minor] EXAMPLE: SET example output omits `notes` and `variable_flags` (`topics/key-specs.md:201`)
**Doc says:**
> ```
>   1) 1) "flags"
>      2) 1) RW
>         2) access
>         3) update
>      3) "begin_search"
> ```

**Problem:** 9.1.2 returns a `notes` field and a `variable_flags` flag for SET. The page cites SET as the `variable_flags` example at line 193, then shows output without that flag. The ZUNION example matches 9.1.2. Neither block shows the command that produced it or has a language tag.
**Evidence:** `valkey-cli --no-raw COMMAND INFO SET`, element 9:
```
    9) 1) 1) "notes"
          2) "RW and ACCESS due to the optional `GET` argument"
          3) "flags"
          4) 1) RW
             2) access
             3) update
             4) variable_flags
          5) "begin_search"
          6) 1) "type"
             2) "index"
             3) "spec"
             4) 1) "index"
                2) (integer) 1
          7) "find_keys"
          8) 1) "type"
                ...
```
**Fix:** Replace the SET block with the output above (full), add a line before each block such as "Element 9 of `COMMAND INFO SET`:", and tag the fences as `text`.

### F4 [minor] CONFUSING: example lists are misfiled and use an undefined term (`topics/key-specs.md:62`)
**Doc says:**
> More examples of the _keyword_ search type include:
> * `SET` has a `begin_search` specification of type _index_ with a value of _1_.

and line 98-103, under `### keynum`:
> * The `SET` command has a _range_ of _0_, _1_ and _0_.
> * The `ZUNION` command has a _start_search_ type _index_ ...

**Problem:** The "keyword" examples list opens with an _index_ example. The examples under `### keynum` are mostly _range_ examples. `start_search` (lines 66, 103) is not a field name; the field is `begin_search`. Line 31 uses "BS", never defined. The range triples ("_0_, _1_ and _0_") don't say which value is lastkey, keystep, or limit. A reader has to reverse-engineer each.
**Fix:** Retitle line 62 "Examples of `begin_search`:". Move the line 98-103 list to the end of the `## find_keys` section, retitled "Examples of `find_keys`:", and spell out fields, e.g. "`MSET`: _range_ with lastkey _-1_, keystep _2_, limit _0_." Replace `start_search` with `begin_search` (lines 66, 103) and "the BS" with "`begin_search`" (line 31).

### F5 [nit] STYLE: typos and formatting (`topics/key-specs.md:30`)
**Doc says:** line 30 "**begin_search:**:" (double colon); line 85 "if _lastkey_ is has the value"; line 96 "Tthe number"; line 193 "optional  _GET_" (double space); line 5 "What are command key specification".
**Problem:** Typos.
**Fix:** "**begin_search:**", "if _lastkey_ has the value", "The number", single space, and description "What command key specifications are and how clients use them to extract key names."

### F6 [nit] OUTDATED: anachronistic "Valkey clients" before key specs existed (`topics/key-specs.md:15`)
**Doc says:**
> Before introducing these specifications, Valkey clients had no trivial programmatic means to extract key names for all commands.

**Problem:** Key specs shipped in Redis OSS 7.0, before Valkey existed (line 14 says so). Valkey has always had them.
**Fix:** "Before Redis OSS 7.0 introduced these specifications, clients had no trivial programmatic means to extract key names for all commands. Valkey has supported key specifications since its first release."

### F7 [nit] BROKEN: movablekeys link points at the general flags section (`topics/key-specs.md:21`)
**Doc says:**
> [_movablekeys_ flag](../commands/command.md#flags)

**Problem:** commands/command.md has a dedicated `### Movablekeys` heading (line 84).
**Fix:** Link to `../commands/command.md#movablekeys`.

### F8 [nit] AI-NATIVE: opening doesn't say who the page is for (`topics/key-specs.md:10`)
**Doc says:**
> Many of the commands in Valkey accept key names as input arguments.

**Problem:** The audience (client and proxy authors who route commands in cluster mode) is only implied, at line 17.
**Fix:** Replace lines 10-11 with: "This page is for client and proxy authors who need to find key names in command arguments, for example to route commands in Valkey Cluster. Element 9 of each `COMMAND` / `COMMAND INFO` reply is an array of the command's key specifications."
