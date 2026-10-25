VERDICT: MINOR-ISSUES
COUNTS: blocker=0 minor=4 nit=3

# The Valkey server (`server`)

https://valkey.io/topics/server/

The page mirrors `valkey-server --help` accurately, but the stdin example states the wrong precedence, and every option description renders as a stray paragraph with a literal leading colon on valkey.io. Fix the precedence wording first.

## Findings

### F1 [minor] INACCURATE: stdin does not override command-line options (`topics/server.md:69`)
**Doc says:**
> Start with a config file, then some additional options overriding the ones in the config file, and finally some more options from stdin:
>
>     valkey-server /etc/myvalkey.conf --loglevel verbose -
**Problem:** "finally" implies stdin is applied last and wins. In fact the order is: config file, then stdin, then command-line options. A command-line option overrides stdin, wherever `-` appears. The page never states the precedence.
**Evidence:** valkey@9.1.2:src/config.c:700-711:
```
    /* Append content from stdin */
    if (config_from_stdin) {
    ...
    /* Append the additional options */
    if (options) {
```
and valkey@9.1.2:src/server.c:7560 `Precedence wise, File, stdin, explicit options -- last config is the one that matters.` Live run, config file sets `loglevel notice`:
```
$ echo 'loglevel warning' | valkey-server my.conf --loglevel verbose -
$ valkey-cli -p 21363 config get loglevel
loglevel
verbose
```
**Fix:** Replace the lead-in with "Start with a config file, read more configuration from stdin, and override both with command-line options:". Add to Options (line 24): "Configuration is applied in this order, later values win: the config file, then stdin, then command-line options."

### F2 [minor] BROKEN: option list renders as paragraphs with a literal leading colon (`topics/server.md:32`)
**Doc says:**
> \*\*\`-v\`\*\*, \*\*\`--version\`\*\*
> : Output version and exit.
**Problem:** The website's markdown renderer does not support definition lists. Each option renders as one paragraph with ": Output version and exit." run on after the option name. Applies to lines 32-45.
**Evidence:** `curl -sL https://valkey.io/topics/server/`:
```
<p><strong><code>-v</code></strong>, <strong><code>--version</code></strong>
: Output version and exit.</p>
...
<p><strong><code>--sentinel</code></strong>
: Start in <a href="/topics/sentinel">sentinel</a> mode</p>
```
**Fix:** Use a table (`| Option | Description |`) or a bullet list (`* **\`-v\`**, **\`--version\`**: Output version and exit.`).

### F3 [minor] MISSING: `-` and the special options only work in specific argument positions (`topics/server.md:24`)
**Doc says:**
> Use `-` to read configuration from stdin.
**Problem:** `-` is only recognized as the first or last argument. Anywhere else it is passed as a config value and startup fails. `-v`, `-h`, `--test-memory`, and `--check-system` are only recognized as the first argument. After any other option they are parsed as config directives and startup fails. The synopsis implies these rules but the prose does not state them, and the failure is a confusing FATAL CONFIG error.
**Evidence:** valkey@9.1.2:src/server.c:7539-7576: special options are checked only as `argv[1]`, and stdin needs `argv[j][0] == '-' && argv[j][1] == '\0' && (j == 1 || j == argc - 1)`. Live:
```
$ valkey-server --bind 127.0.0.1 --port 21362 - --loglevel verbose
>>> 'port "21362" "-"'
wrong number of arguments
$ valkey-server --port 21363 --version
>>> 'version'
Bad directive or wrong number of arguments
```
**Fix:** Line 24: "Use `-` as the first or last argument to read configuration from stdin." Line 30: "Additional options. These must be the first argument:" (covering `-v`, `-h`, `--test-memory`, `--check-system`). Leave `--sentinel` in its own list item, since it works in any position.

### F4 [minor] MISSING: `--sentinel` absent from the synopsis, with no mention of `valkey-sentinel` (`topics/server.md:9`)
**Doc says:**
> **`valkey-server`** [ _/path/to/valkey.conf_ ] [ _OPTIONS_ ] [**`-`**]
**Problem:** `--sentinel` is listed under Options but missing from Usage and Examples, and sentinel mode needs a config file. Readers also don't learn that `valkey-sentinel` (or any binary name containing `valkey-sentinel` or `redis-sentinel`) starts sentinel mode on its own. `--help` does show the sentinel form.
**Evidence:** valkey@9.1.2:src/server.c:7218-7225 `if (strstr(exec_name, "valkey-sentinel") != NULL) return 1;`. `valkey-server --help` prints `Sentinel mode:\n       ./valkey-server /etc/sentinel.conf --sentinel`. topics/sentinel.md:8-10 already documents both forms.
**Fix:** Add `**\`valkey-server\`** _/path/to/sentinel.conf_ **\`--sentinel\`**` to Usage. Change line 45 to "Start in [Sentinel](sentinel.md) mode. Requires a config file. Equivalent to running `valkey-sentinel`." Add the example `valkey-server /etc/sentinel.conf --sentinel`.

### F5 [nit] STYLE: typo and missing period (`topics/server.md:65`)
**Doc says:**
> Start as a replica of another Valkey server that can accessed at 127.0.0.1:8888:

and line 45 `: Start in [sentinel](sentinel.md) mode` (no period).
**Problem:** Missing "be". Line 45 is the only option description without a period.
**Fix:** "that can be accessed at". "Start in [Sentinel](sentinel.md) mode."

### F6 [nit] EXAMPLE: indented code blocks have no language tag (`topics/server.md:51`)
**Doc says:**
>     valkey-server
**Problem:** All seven examples (lines 51, 55, 59, 63, 67, 72, 77) are indented blocks. They render as `data-lang="plain"`.
**Evidence:** Live HTML: `<pre class="giallo" ...><code data-lang="plain">...valkey-server</span>`.
**Fix:** Convert each one to a fenced block tagged ```` ```sh ````.

### F7 [nit] AI-NATIVE: description says nothing beyond the title (`topics/server.md:17`)
**Doc says:**
> `valkey-server` is the Valkey database program.
>
> What is Valkey? See [Introduction](introduction.md).
**Problem:** The opening doesn't say what the page covers. The front matter description ("Manual for valkey-server, the Valkey server program") repeats the title.
**Fix:** "`valkey-server` starts a Valkey server. This page covers its command-line syntax: how to pass a config file, read configuration from stdin, override config directives with `--<directive>` options, and run the memory test, system check, or Sentinel mode." Use the same sentence for the front matter description.

## Unverified
- topics/server.md:38: `--test-memory` behavior. Source runs `memtest(megabytes, 50)` (50 passes, valkey@9.1.2:src/server.c:7548). Not run because it is interactive and long. The page could state the pass count, but I didn't confirm the output format.
