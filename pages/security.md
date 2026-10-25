VERDICT: NEEDS-WORK
COUNTS: blocker=1 minor=3 nit=3

# Security (`security`)

https://valkey.io/topics/security/

The page is a lightly renamed Redis-era page. The most important fix: it tells readers to report security issues in a public GitHub issue and points to a GPG key that does not exist, which contradicts the project's disclosure policy.

## Findings

### F1 [blocker] INACCURATE: Security contact sends vulnerabilities to public GitHub issues and a missing GPG key (`topics/security.md:11`)
**Doc says:**
> For security-related contacts, open an issue on GitHub, or when you feel it
> is really important to preserve the security of the communication, use the
> GPG key at the end of this document.

**Problem:** The project policy forbids filing vulnerabilities as issues. The page has no GPG key; it ends at line 138 with the "valkey user" sentence.
**Evidence:** `valkey@9.1.2:SECURITY.md`:
```
If you believe you've discovered a security vulnerability, please contact the Valkey team at security@lists.valkey.io.
Please *DO NOT* create an issue.
```
**Fix:** Replace lines 11-13 with: "To report a security vulnerability, email security@lists.valkey.io. Do not open a public GitHub issue. See the [security policy](https://github.com/valkey-io/valkey/blob/unstable/SECURITY.md)."

### F2 [minor] INACCURATE: Protected mode trigger described as "binding all interfaces", and binding all interfaces does not disable it (`topics/security.md:57`)
**Doc says:**
> Valkey enters a special mode called **protected mode** when it is
> executed with the default configuration (binding all the interfaces) and
> without any password in order to access it.

and line 66-67:
> the system administrator can still ignore the error given by Valkey and
> disable protected mode or manually bind all the interfaces.

**Problem:** Protected mode depends only on `protected-mode yes` and the default user having no password. `bind` plays no part, so binding all interfaces does not get around it. The shipped valkey.conf also binds loopback only, so "binding all the interfaces" is not the default config file behavior either.
**Evidence:** `valkey@9.1.2:src/networking.c:1827`:
```
    if (server.protected_mode && DefaultUser->flags & USER_FLAG_NOPASS) {
        if (connIsLocal(conn) != 1) {
            char *err = "-DENIED Running in protected mode because protected "
                        "mode is enabled and no password is set for the default user. "
```
`valkey@9.1.2:valkey.conf:88` `bind 127.0.0.1 -::1`, and valkey.conf:104-106: "When protected mode is on and the default user has no password, the server only accepts local connections from the IPv4 address (127.0.0.1), IPv6 address (::1) or Unix domain sockets." (The code comment above line 1827 still mentions bind; it is stale.)
**Fix:** Lines 57-62: "Protected mode is on by default (`protected-mode yes`). While it is on and the default user has no password, Valkey accepts connections only from 127.0.0.1, ::1, and Unix sockets, and replies to other clients with a `-DENIED` error explaining how to fix the configuration." Lines 66-67: "...can still ignore the error and set `protected-mode no`, or set a password for the default user." Drop "or manually bind all the interfaces".

### F3 [minor] CONFUSING: "AUTH is sent unencrypted" contradicts the TLS section (`topics/security.md:94`)
**Doc says:**
> Since the `AUTH` command, like every other Valkey command, is sent unencrypted, it
> does not protect against an attacker that has enough access to the network to
> perform eavesdropping.

**Problem:** Stated as absolute, then the next section says TLS covers all client connections. A reader concludes passwords always travel in clear text.
**Evidence:** topics/security.md:100-101: "Valkey has optional support for TLS on all communication channels, including client connections".
**Fix:** "Without TLS, `AUTH` and every other command are sent in clear text, so a password does not protect against an attacker who can eavesdrop on the network. Enable [TLS](tls.md) when traffic crosses an untrusted network."

### F4 [minor] MISSING: No coverage of hardened defaults or restricting dangerous commands (`topics/security.md:132`)
**Doc says:** Nothing. "Code security" (lines 132-138) is the last section.
**Problem:** Valkey ships configs whose sole purpose is attack-surface reduction, and no valkey-doc page explains them (`grep -rn 'enable-debug-command\|rename-command' topics/` returns nothing). Operators securing an instance need to know DEBUG, MODULE, and protected configs such as `dir`/`dbfilename` are blocked by default and can be opened to `local` only, and that ACLs (or `rename-command`) restrict CONFIG, FLUSHALL, and similar commands. Unix socket permissions (`unixsocketperm`) are also absent.
**Evidence:** `valkey@9.1.2:valkey.conf:114-135`:
```
# Commands that can increase the attack surface of the server and that aren't usually
# called by users are blocked by default.
...
# local - Allow only for local connections. Ones originating from the
#         IPv4 address (127.0.0.1), IPv6 address (::1) or Unix domain sockets.
#
# enable-protected-configs no
# enable-debug-command no
# enable-module-command no
```
Live 9.1.2: `CONFIG GET enable-debug-command` returns `no`.
**Fix:** Add a section "Restricting dangerous commands and configs": list `enable-protected-configs`, `enable-debug-command`, `enable-module-command` (default `no`; values `no`, `yes`, `local`), recommend ACL rules such as `-@dangerous` for application users with a link to acl.md, mention `rename-command` as the legacy option, and note `unixsocketperm` for socket access.

### F5 [nit] INACCURATE: requirepass "refuse any query" overstated (`topics/security.md:78`)
**Doc says:**
> When the `requirepass` setting is enabled, Valkey will refuse any query by
> unauthenticated clients. A client can authenticate itself by sending the
> **AUTH** command followed by the password.

**Problem:** `AUTH` and `HELLO ... AUTH` are accepted unauthenticated. `HELLO` with AUTH is not mentioned.
**Evidence:** Live 9.1.2 with `--requirepass secret`:
```
$ valkey-cli -p 21300 ping
NOAUTH Authentication required.
$ valkey-cli -p 21300 hello 3
NOAUTH HELLO must be called with the client already authenticated, otherwise the HELLO <proto> AUTH <user> <pass> option can be used to authenticate the client and select the RESP protocol version at the same time
```
**Fix:** "...Valkey rejects every command from unauthenticated clients with `NOAUTH` except `AUTH` and `HELLO` with the `AUTH` option, which clients use to send the password."

### F6 [nit] STYLE: "here" link text (`topics/security.md:73`)
**Doc says:**
> Read more about Access Control Lists [here](acl.md).

and line 101: "Read more about TLS [here](tls.md)."
**Problem:** Non-descriptive link text, poor for accessibility and for readers scanning links.
**Fix:** "See [Access Control Lists](acl.md)." and "See [TLS](tls.md)."

### F7 [nit] AI-NATIVE: Commodity filler and contradictory wording (`topics/security.md:6`)
**Doc says:**
> This document provides an introduction to the topic of security from the point of
> view of Valkey. It covers ... and other similar topics.

line 64: "We expect protected mode to seriously decrease the security issues caused by unprotected Valkey instances..."; line 105-106: "attacks that an attacker can trigger from the outside even without external access to the instance"; line 134: "Internally, Valkey uses all the well-known practices for writing secure code".
**Problem:** The intro does not say who the page is for; line 64 and line 134 carry no information; line 106 reads as a contradiction ("from the outside" vs "without external access").
**Fix:** Intro: "This page is for operators deploying Valkey. It covers the trust model, network exposure, protected mode, authentication, TLS, and attacks via crafted input." Cut line 64-65's first sentence and line 134-135. Line 106: "even without direct access to the instance, for example through a web application that stores user input in Valkey."
