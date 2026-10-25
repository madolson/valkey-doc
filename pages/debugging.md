VERDICT: MINOR-ISSUES
COUNTS: blocker=0 minor=4 nit=3

# Debugging (`debugging`)

https://valkey.io/topics/debugging/

The GDB workflow is sound but the page is Redis-era: the `DEBUG SEGFAULT` demo fails on a default 9.1.2 server, the screen detach key is wrong, and the sample output is from macOS with file names that no longer exist. Most important fix: tell readers where to send the report and not to attach a core file (full dataset) to a public GitHub issue.

## Findings

### F1 [minor] INACCURATE: DEBUG SEGFAULT is disabled by default (`topics/debugging.md:108`)
**Doc says:**
> Valkey has a command to simulate a segmentation fault (in other words a bad crash) using the `DEBUG SEGFAULT` command
**Problem:** On 9.1.2 `DEBUG` is blocked unless `enable-debug-command` is set. A reader trying the demo gets an error, not a crash.
**Evidence:** `valkey@9.1.2:src/config.c:3370`: `createEnumConfig("enable-debug-command", NULL, IMMUTABLE_CONFIG, ..., PROTECTED_ACTION_ALLOWED_NO, ...)`. Live run:
```
$ valkey-cli -p 20340 debug segfault
ERR DEBUG command not allowed. If the enable-debug-command option is set to "local", you can run it from a local connection, otherwise you need to set this option in the configuration file, and then restart the server.
```
**Fix:** Add after the sentence: "`DEBUG` is disabled by default. To try this on a test instance, start it with `enable-debug-command local` and run the command from a local connection." Also close the unbalanced parenthesis at line 109.

### F2 [minor] INACCURATE: Ctrl-a a does not detach screen or tmux (`topics/debugging.md:103`)
**Doc says:**
> Now it's time to detach your screen/tmux session, if you are running GDB using it, by pressing **Ctrl-a a** key combination.
**Problem:** In screen, `Ctrl-a a` sends a literal Ctrl-a to the window. Detach is `Ctrl-a d`. In tmux the default is `Ctrl-b d`. Following the page leaves the session attached.
**Evidence:** screen(1): "C-a a (meta) Send the command character (C-a) to window"; "C-a d (detach) Detach screen from this terminal." tmux(1): "d Detach the current client" under the default `C-b` prefix.
**Fix:** "Detach the session: press **Ctrl-a d** in screen or **Ctrl-b d** in tmux."

### F3 [minor] MISSING: no destination for the report, and core-file privacy advice is unsafe (`topics/debugging.md:177`)
**Doc says:**
> Valkey developers will make sure not to share the content with anyone else, and will delete the file as soon as it is no longer used for debugging purposes
> Finally you can send everything to the Valkey core team:
**Problem:** The page never says where to send anything. The server's own crash report sends users to public GitHub issues, where the project cannot promise confidentiality. A reader could attach a core file holding their full dataset to a public issue.
**Evidence:** `valkey@9.1.2:src/debug.c:2423-2425`: `"=== %s BUG REPORT END. Make sure to include from START to END. ===" ... "Please report the crash by opening an issue on github:" ... "https://github.com/valkey-io/valkey/issues"`.
**Fix:** Replace lines 179-182 with: "The core file contains all data that was in the Valkey instance. Do not attach it to a public GitHub issue. Open an issue at https://github.com/valkey-io/valkey/issues with the stack trace, registers, and crash log, and ask a maintainer how to share the core file privately." Add the full crash log (from `BUG REPORT START` to `BUG REPORT END`) to the list at line 188.

### F4 [minor] OUTDATED: sample GDB output is from Redis on macOS with nonexistent source files (`topics/debugging.md:115`)
**Doc says:**
> Program received signal EXC_BAD_ACCESS, Could not access memory.
> Reason: KERN_INVALID_ADDRESS at address: 0xffffffffffffffff
> debugCommand (c=0x7ffc32005000) at debug.c:220
> 220         *((char*)-1) = 'x';
> #1  0x000000010d246d63 in call (c=0x7ffc32005000) at valkey.c:1163
**Problem:** `EXC_BAD_ACCESS`/`KERN_INVALID_ADDRESS` are macOS Mach exceptions; Linux GDB prints `SIGSEGV, Segmentation fault`. `valkey.c` does not exist (the file is `src/server.c`), and `DEBUG SEGFAULT` no longer writes to `(char*)-1`. Same pattern in lines 90-91 (`Reading symbols for shared libraries + done` is old macOS GDB) and lines 131-138.
**Evidence:** `git -C ~/git/valkey ls-tree 9.1.2 src/` lists `src/server.c`, no `src/valkey.c`. `valkey@9.1.2:src/debug.c:538-543`:
```
} else if (!strcasecmp(objectGetVal(c->argv[1]), "segfault")) {
    /* Compiler gives warnings about writing to a random address
     * e.g "*((char*)-1) = 'x';". As a workaround, we map a read-only area
     * and try to write there to trigger segmentation fault. */
    char *p = mmap(NULL, 4096, PROT_READ, MAP_PRIVATE | MAP_ANON, -1, 0);
    *p = 'x';
```
**Fix:** Regenerate the attach, crash, `bt`, and `info registers` blocks from a Linux GDB session against a 9.1.2 `make noopt` build, or label them "illustrative output" and at minimum replace `valkey.c` with `server.c` and the signal line with `Program received signal SIGSEGV, Segmentation fault.`

### F5 [nit] STYLE: first-person voice, gendered pronoun, unbalanced parenthesis (`topics/debugging.md:36`)
**Doc says:**
> perform all the inspections in his or her computer
> So I'll use this command to crash my instance to show what happens in the GDB side:
> was even able to show me the file name
**Problem:** Single-author first person (lines 110, 120) from the original blog post; "his or her" (line 36); the parenthesis opened at line 108 is never closed.
**Fix:** Line 36: "on their own computer". Line 109-110: "(do not use it against a production instance). The following shows what GDB prints:". Line 120: "GDB detects the crash and shows the file name and line number".

### F6 [nit] AI-NATIVE: filler intro and vague headings (`topics/debugging.md:7`)
**Doc says:**
> Valkey is developed with an emphasis on stability. We do our best with every release to make sure you'll experience a stable product with no crashes.
> ## After the crash
> ## Thank you
**Problem:** The first paragraph carries no information; the page's subject (collecting a GDB backtrace, registers, and core file for a Valkey crash bug report) only appears at line 16. "After the crash" actually covers simulating a crash. "Thank you" is a heading with no content. Line 54-56 ("Valkey is not very CPU-bound") is an unqualified claim with no data.
**Fix:** Replace lines 7-17 with: "This guide is for operators who can reproduce a Valkey server crash. It shows how to attach GDB, capture a backtrace, registers, and a core file, and what to send to the Valkey developers." Rename "After the crash" to "Simulating a crash in GDB". Delete the "Thank you" section. Cut lines 54-56 or replace with "A `-O0` build is slower; use it only where you can tolerate that."

### F7 [nit] STYLE: code blocks are indented with no language tag (`topics/debugging.md:71`)
**Doc says:**
>     $ valkey-cli info | grep process_id
**Problem:** Every example (lines 71, 82, 86, 90, 98, 112, 130, 142, 174) is an indented block, so none carries a language tag.
**Fix:** Convert to fenced blocks: ` ```sh ` for shell commands and ` ```text ` for GDB sessions and output. Also use `valkey-cli info server` at line 71 to avoid dumping all sections.
