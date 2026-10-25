VERDICT: NEEDS-WORK
COUNTS: blocker=2 minor=3 nit=6

# Installation (`installation`)

https://valkey.io/topics/installation/

Two of the install paths don't work as written. The Debian `apt install valkey` line names a package that does not exist, and the "Alpine" init-script walkthrough uses Debian's `update-rc.d`. Fix the package names first, then replace the init-script section with the systemd unit and OpenRC guidance that matches what 9.1.2 actually ships.

## Findings

### F1 [blocker] INACCURATE: Debian has no `valkey` or `valkey-redis-compat` package (`topics/installation.md:59`)
**Refuted on re-check:** Debian `valkey-server` declares `Provides: valkey` (9.1.2-1, 8.1.1+dfsg1-3+deb13u2, 8.0.1+dfsg1-1~bpo12+1), so `apt install valkey` works, and Ubuntu publishes `valkey-redis-compat` for noble and plucky. Keep this page's commands.

**Doc says:**
> sudo apt install valkey
> # For symlinked binaries to redis-cli and other redis-* tools (on Ubuntu)
> sudo apt install valkey-redis-compat
**Problem:** The Debian source package `valkey` builds the binary packages `valkey-server`, `valkey-tools` and `valkey-sentinel`. No binary package is named `valkey`, so `apt install valkey` fails on Debian, and the heading claims this works on Debian and five derivatives.
**Evidence:** packages.debian.org name search for "valkey" (all suites) returns only:
```
Package valkey-sentinel
Package valkey-server
Package valkey-tools
Package valkey-tools-dbgsym
Package python-valkey-doc
```
sources.debian.org lists source `valkey` 9.1.2-1 (forky, sid), 8.1.1 (trixie), 8.0.1 (bookworm-backports).
**Fix:** `sudo apt install valkey-server valkey-tools` (add `valkey-sentinel` if needed). Add a note that Debian 13 (trixie) or later, or bookworm-backports, is required. Keep the `valkey-redis-compat` line only if it is verified for a specific Ubuntu release, and name that release.
Tracked: valkey-io/valkey-doc#251

### F2 [blocker] INACCURATE: The "Alpine" init-script steps use Debian's `update-rc.d` (`topics/installation.md:196`)
**Doc says:**
> The remainder of this section explains how to set up Valkey using an init script, for distros like Alpine Linux that don't use systemd.
> ...
> sudo update-rc.d valkey_6379 defaults
**Problem:** Alpine uses OpenRC, which has no `update-rc.d` (that tool is Debian sysvinit). An Alpine reader gets "command not found" at the last step, and the instance won't start on boot. The Alpine `valkey` apk already ships an OpenRC service.
**Evidence:** `valkey@9.1.2:utils/valkey_init_script` is an LSB sysvinit script (`### BEGIN INIT INFO ... Default-Start: 2 3 4 5`). The OpenRC equivalent is `rc-update add <service> default`.
**Fix:** Change line 196 to "for distros that use sysvinit (for example Devuan)". Add: "On Alpine, the `valkey` package provides an OpenRC service: `sudo rc-update add valkey default && sudo rc-service valkey start`."

### F3 [minor] INACCURATE: The security defaults leave out protected mode and the packaged `bind` (`topics/installation.md:158`)
**Doc says:**
> By default Valkey binds to **all the interfaces** and has no authentication at all.
**Problem:** This is only half right. Without a config file, the compiled default is `bind * -::*`, but `protected-mode yes` is also on by default and refuses non-loopback clients when no password is set. The shipped `valkey.conf`, which packages and the steps on this page use, binds to loopback only. A reader is told the server is wide open when it isn't, and never learns which setting actually protects it.
**Evidence:**
```
valkey@9.1.2:src/server.h:159: #define CONFIG_DEFAULT_BINDADDR {"*", "-::*"}
valkey@9.1.2:src/config.c:3272: createBoolConfig("protected-mode", NULL, MODIFIABLE_CONFIG, server.protected_mode, 1, NULL, NULL),
valkey@9.1.2:valkey.conf:88: bind 127.0.0.1 -::1
valkey@9.1.2:valkey.conf:112: protected-mode yes
```
**Fix:** "Started without a config file, Valkey listens on all interfaces, but protected mode (`protected-mode yes`, the default) rejects connections from non-loopback addresses until you set a password or an explicit `bind`. The example `valkey.conf` binds to `127.0.0.1 -::1`. Before exposing Valkey on a network:" Then keep the list, and link [Security](security.md#protected-mode) instead of explaining it again (topics/security.md:42-67 already covers firewalling, `bind` and protected mode).

### F4 [minor] CONFUSING: The persistence advice says to run SAVE by hand every time (`topics/installation.md:178`)
**Doc says:**
> If you want your database to persist and be reloaded after a restart, make sure to call the [SAVE](../commands/save.md) command manually every time you want to force a data set snapshot.
**Problem:** The text implies data won't survive a restart unless you call SAVE yourself. In fact the default save points already snapshot automatically, and SAVE blocks the server. A reader trying to make the data durable is pointed at a blocking manual command when the real answer is AOF.
**Evidence:** Live 9.1.2:
```
$ valkey-cli -p 20560 config get save
save
3600 1 300 100 60 10000
```
**Fix:** "By default Valkey writes an RDB snapshot after 3600 s if at least 1 key changed, 300 s if at least 100 changed, or 60 s if at least 10000 changed (`save 3600 1 300 100 60 10000`). Writes made since the last snapshot are lost on a crash. For stronger durability, enable AOF (`appendonly yes`); see [Persistence](persistence.md). `SHUTDOWN` saves before exiting when save points are configured."

### F5 [minor] OUTDATED: The systemd setup is skipped even though 9.1.2 ships unit files (`topics/installation.md:193`)
**Doc says:**
> Valkey supports systemd, but this document was written for init scripts, before systemd was widely adapted.
> There are many guides online for how to set up a systemd service.
**Problem:** Most readers run systemd, and the page sends them to "guides online". The repository ships ready-made units and an install script, and systemd notify needs a build flag the page never mentions.
**Evidence:** `valkey@9.1.2:utils/systemd-valkey_server.service`, `utils/systemd-valkey_multiple_servers@.service`, `utils/install_server.sh`. README at 9.1.2: "To build with systemd support ... run: `make USE_SYSTEMD=yes`".
**Fix:** Replace lines 193-194 with: "For systemd, build with `make USE_SYSTEMD=yes`, set `supervised systemd` in your config, and install `utils/systemd-valkey_server.service` to `/etc/systemd/system/valkey.service`. Then run `sudo systemctl enable --now valkey`. Distribution packages already include a unit." Move the sysvinit walkthrough under its own "Using a sysvinit script" heading.

### F6 [nit] OUTDATED: The tarball example and `make install` description are stale (`topics/installation.md:20`)
**Doc says:**
> Unpack the tarball (e.g. `tar -xzvf valkey-8.0.1.tar.gz`)
> (line 198) By default, `make install` will copy the `valkey-server` and `valkey-cli` binaries to `/usr/local/bin`.
**Problem:** The latest release is 9.1.2. `make install` also installs `valkey-benchmark`, the `valkey-sentinel`/`valkey-check-rdb`/`valkey-check-aof` symlinks, and `redis-*` compatibility symlinks by default.
**Evidence:** `valkey@9.1.2:src/Makefile:845-858` (install target), `src/Makefile:425: USE_REDIS_SYMLINKS?=yes`.
**Fix:** Use `tar -xzvf valkey-9.1.2.tar.gz`. Change line 198 to: "`make install` installs `valkey-server`, `valkey-cli`, `valkey-benchmark` and related symlinks (including `redis-*` names unless `USE_REDIS_SYMLINKS=no`) to `/usr/local/bin` (override with `PREFIX=`)."

### F7 [nit] STYLE: Typos and casing (`topics/installation.md:7`)
**Doc says:**
> This is a an installation guide.
> (line 193) before systemd was widely adapted.
> (line 26) ### MacOS
**Problem:** "a an", "adapted" should be "adopted", "MacOS" should be "macOS".
**Fix:** Correct all three.

### F8 [nit] EXAMPLE: Fenced blocks have no language tag (`topics/installation.md:141`)
**Doc says:**
> ```
> $ valkey-cli ping
**Problem:** The blocks at lines 141, 150, 181, 202, 209, 215, 224, 230, 243 and 249 have no language tag.
**Fix:** Tag the shell commands as `bash`. Use `text` for the interactive sessions at 141 and 150.

### F9 [nit] BROKEN: The OpenPKG link doesn't resolve (`topics/installation.md:123`)
**Doc says:**
> and openpkg on [OpenPKG](http://www.openpkg.org/).
**Problem:** Two `curl -sIL` attempts to http://www.openpkg.org/ failed to connect (HTTP 000).
**Fix:** Remove the OpenPKG mention, or replace it with a working package URL.

### F10 [nit] STYLE: The headings embed links and end in colons (`topics/installation.md:28`)
**Doc says:**
> #### Using [Homebrew](https://brew.sh/) to install and run Valkey:
**Problem:** The headings at lines 28 and 40 have links inside them and trailing colons, which makes the anchors and the table of contents messy.
**Fix:** Use `#### Homebrew` and `#### MacPorts`, and put the links in the body text.

### F11 [nit] AI-NATIVE: Generic opening and filler sentences (`topics/installation.md:7`)
**Doc says:**
> This is a an installation guide. You'll learn how to install, run, and experiment with the Valkey server process.
> (line 13) These are some ways to install Valkey.
> (line 137) However, to make hacking with Valkey simpler, ...
> (line 169) Of course using Valkey just from the command line interface is not enough ...
**Problem:** The opening paragraph doesn't say which platforms or methods the page covers, or who it's for. Lines 13, 137 and 169 add no information.
**Fix:** Change line 7 to "How to install Valkey from source, containers, or OS packages (macOS, Linux, BSD; Windows via WSL), verify it with `valkey-cli`, secure it, and run it as a service." Delete line 13. Cut line 137 down to its last sentence. Change line 169 to "To use Valkey from an application, install a [client library](../clients/)."

## Unverified
- topics/installation.md:64-118: Alpine/Kali/Wolfi, yum/dnf (`valkey-compat-redis`, `valkey-doc`), ALT, Arch, FreeBSD, NixOS, openSUSE, Solus, Void, Exherbo package names. I checked only Debian. repology.org and pkgs.org returned 000 and 403 to curl.
- topics/installation.md:122: The SlackBuilds URL returns 200, but I did not confirm it hosts a current version.
