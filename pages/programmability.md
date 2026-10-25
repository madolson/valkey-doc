VERDICT: MINOR-ISSUES
COUNTS: blocker=0 minor=3 nit=4

# Programmability (`programmability`)

https://valkey.io/topics/programmability/

Mostly accurate for 9.1.2. The biggest problem is the claim that Lua is the only scripting engine: since 9.0 engines plug in through a module API, and since 9.1 Lua itself ships as a module.

## Findings

### F1 [minor] OUTDATED: "single scripting engine" ignores the 9.x scripting engine module API (`topics/programmability.md:26`)
**Doc says:**
> Presently, Valkey supports a single scripting engine, the [Lua 5.1](https://www.lua.org/) interpreter.
**Problem:** Starting in 9.0, modules can register scripting engines. In 9.1, Lua was moved into a module (statically linked by default), and INFO has a `Scripting Engines` section. Readers will think the engine is fixed and built in.
**Evidence:** `valkey@9.1.2:src/valkeymodule.h:2304`: `VALKEYMODULE_API int (*ValkeyModule_RegisterScriptingEngine)(ValkeyModuleCtx *module_ctx,` (also in 9.0.0). `valkey@9.1.2:00-RELEASENOTES`, 9.1.0-rc1: `* Move Lua scripting engine into a Valkey module by @rjd15372 (#2858)` and `* New INFO section for scripting engines by @rjd15372 (#2738)`. The 9.1.2 startup log:
```
23409:M 24 Oct 2026 00:56:44.650 * Static Module 'lua' successfully loaded
```
`INFO everything` sections include `# Scripting Engines`.
**Fix:** "Valkey ships with one scripting engine, the [Lua 5.1](https://www.lua.org/) interpreter, which is loaded as a built-in module. Since Valkey 9.0, modules can register more engines with `ValkeyModule_RegisterScriptingEngine` (see the [modules API reference](modules-api-ref.md)). `INFO scriptingengines` lists the loaded engines."

### F2 [minor] INACCURATE: "only command allowed is SHUTDOWN NOSAVE" after a write contradicts the previous bullet (`topics/programmability.md:117`)
**Doc says:**
> If the script had already performed even a single write operation, the only command allowed is `SHUTDOWN NOSAVE` that stops the server without saving the current data set on disk
**Problem:** The busy gate checks only the command's `ALLOW_BUSY` flag. It does not care whether the script has written. Once a script writes, `SCRIPT KILL` returns UNKILLABLE, but `AUTH`, `HELLO`, `CLIENT ID`, `SELECT`, and the rest of the line 115 list still work. Line 115 and line 117 contradict each other.
**Evidence:** `valkey@9.1.2:src/server.c:4606`: `if (isInsideYieldingLongCommand() && !(c->cmd->flags & CMD_ALLOW_BUSY)) {` (there is no write check). Live 9.1.2, script running `server.call('set','k','v') while true do end`:
```
UNKILLABLE Sorry the script already executed write commands against the dataset. You can either wait the script termination or kill the server in a hard way using the SHUTDOWN NOSAVE command.
```
**Fix:** "If the script has already performed a write, `SCRIPT KILL` and `FUNCTION KILL` fail with an `UNKILLABLE` error. The only way to stop the script is `SHUTDOWN NOSAVE`, which stops the server without saving the current data set to disk. The connection-management commands listed above still work."

### F3 [minor] INACCURATE: Functions attributed to "v7.0" of Valkey (`topics/programmability.md:41`)
**Doc says:**
> Secondly, added in v7.0, Valkey Functions are essentially scripts that are first-class database elements.
**Problem:** Valkey's first release was 7.2. Functions came from Redis OSS 7.0. Line 80 of this same page already says "Redis OSS 7.0", so the page is inconsistent with itself.
**Evidence:** `topics/programmability.md:80`: "Read-only scripts and read-only script commands were introduced in Redis OSS 7.0". `eval-intro.md:266` also uses "Redis OSS 7.0".
**Fix:** "Secondly, Valkey Functions (introduced in Redis OSS 7.0, available in all Valkey releases) are scripts that are first-class database elements."

### F4 [nit] STYLE: Heading level skips from h2 to h4 (`topics/programmability.md:78`)
**Doc says:**
> #### Read-only script history
**Problem:** It sits under `## Read-only scripts` with no h3 in between. The live page renders `<h4 id="read-only-script-history">`.
**Fix:** Change it to `### Read-only script history`.

### F5 [nit] CONFUSING: The recommendation comes after the history list, away from the features it refers to (`topics/programmability.md:87`)
**Doc says:**
> The recommended approach is to use the standard scripting commands with the `no-writes` flag unless you need one of the previously mentioned features.
**Problem:** "Previously mentioned features" means the ACL and replica-routing bullets at lines 75-76, but the sentence sits after the Redis OSS history list, so it reads as if it points to the history items.
**Fix:** Move the sentence directly after line 76 and make it explicit: "...unless you need ACL restriction to read-only scripts or client routing of `*_RO` commands to replicas."

### F6 [nit] AI-NATIVE: The Background section is commodity filler, and the timeout facts are spread across sentences (`topics/programmability.md:13`)
**Doc says:**
> Valkey is a _"domain-specific language for abstract data types"_. ... Developers can use this capability for implementing robust, application-specific APIs.
and at line 100:
> Scripts are subject to a maximum execution time (set by default to five seconds).
**Problem:** Lines 13-23 carry no Valkey-specific information, and "Background" does not say what the section answers. The quotable fact (config name and default) is split between line 100 and line 106.
**Fix:** Cut lines 13-16 and 21-23. Rename "Background" to "What scripts are". Rewrite line 100 as: "Scripts are subject to a maximum execution time, set by `busy-reply-threshold` (alias `lua-time-limit`), default 5000 milliseconds." Delete line 106. Default verified in `valkey@9.1.2:src/config.c:3454`: `createLongLongConfig("busy-reply-threshold", "lua-time-limit", ..., 5000, ...)`.

### F7 [nit] STYLE: Typo (`topics/programmability.md:15`)
**Doc says:**
> Most the commands specialize at manipulating core [data types]
**Fix:** "Most of the commands specialize in manipulating core data types". This goes away if F6's cut is applied.

## Unverified
- topics/programmability.md:68-69: read-only scripts never fail with OOM and are not blocked during write pauses. Not tested live within the timebox. The `no-writes` OOM and pause behavior is documented in lua-api.md#script_flags but was not checked against source.
