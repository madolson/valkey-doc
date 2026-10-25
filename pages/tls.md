VERDICT: NEEDS-WORK
COUNTS: blocker=1 minor=6 nit=5

# TLS (`tls`)

https://valkey.io/topics/tls/

The page describes unstable behavior as if it shipped: strict certificate validation was reverted from 9.1 before GA, so 9.1.2 loads expired certificates without complaint. Fix that first. The rest is missing version gates, missing config values, and a wrong description of `tls-client-cert-file`.

## Findings

### F1 [blocker] INACCURATE: Certificate validity check is not in 9.1.2 (`topics/tls.md:91`)
**Doc says:**
> Valkey performs validation on all TLS materials whenever they are loaded or reloaded. If any check fails, the load operation will fail immediately, preventing Valkey from using invalid TLS materials.
> - TLS files or directories are not empty or malformed
> - Certificates match their corresponding private keys
> - Certificates are within their valid time period

**Problem:** #2999 added fail-fast validation, and #3572 reverted it in 9.1.0-rc2, deferring it to the next major version. In 9.1.2 an expired or not-yet-valid certificate loads and is served, and an empty CA directory is not rejected. Readers who trust this section skip expiry monitoring and get a client-side outage instead of a startup error. The validity check exists only on unstable.
**Evidence:**
```
valkey@9.1.2:00-RELEASENOTES:127-128
### Behavior Changes
* Revert strict TLS certificate validation at config load as it is a breaking change, deferred to next major version (#3572)

$ git log --oneline 9.1.2 --grep=3572
699765702 Revert "Fail fast on invalid certificates at TLS config load (#2999)" (#3572)
```
`grep -n 'X509_cmp_current_time\|certificate is invalid'` finds nothing in 9.1.2:src/tls.c. On origin/unstable:src/tls.c:519 `if (X509_cmp_current_time(not_before) > 0 || X509_cmp_current_time(not_after) < 0) {` and :654 `"%s TLS certificate is invalid. Aborting TLS configuration."`. The empty-CA-dir check (13c66b82e) is not an ancestor of 9.1.2. Key/cert mismatch and unparseable files still fail in 9.1.2 because OpenSSL rejects them (9.1.2:src/tls.c:503-513).
**Fix:** Replace the section with: "Valkey fails to load TLS materials that cannot be parsed, or whose private key does not match the certificate. Valkey 9.1 does not reject expired or not-yet-valid certificates. Monitor expiry with the `INFO tls` fields `tls_server_cert_expires_in_seconds`, `tls_client_cert_expires_in_seconds`, and `tls_ca_cert_expires_in_seconds` (added in 9.1)." Add the expiry and empty-directory checks back, marked with the release that ships them, once that release is out.

### F2 [minor] INACCURATE: Client certificate is for outgoing connections, not mTLS with connecting peers (`topics/tls.md:73`)
**Doc says:**
> When mutual TLS (mTLS) is enabled, Valkey can be configured with a client certificate and private key to authenticate itself to connecting peers:

**Problem:** `tls-client-cert-file` is used only when Valkey opens a connection (replica to primary, cluster bus, Sentinel). It does not depend on mTLS, and Valkey does not present it to peers that connect to it. If it is unset, Valkey uses `tls-cert-file` for outgoing connections too.
**Evidence:** valkey@9.1.2:valkey.conf:218-228:
```
# Normally the server uses the same certificate for both server functions (accepting
# connections) and client functions (replicating from a primary, establishing
# cluster bus connections, etc.).
...
# different certificates for incoming (server) and outgoing (client)
# connections. To do that, use the following directives:
```
**Fix:** Rename the heading to "Client certificate for outgoing connections" and write: "By default Valkey presents `tls-cert-file` on both incoming and outgoing connections. If your CA issues server-only and client-only certificates, set a separate certificate for the connections Valkey opens (replication, cluster bus, Sentinel):"

### F3 [minor] INACCURATE: Sentinel `tls-replication` does not control what Sentinel listens on (`topics/tls.md:148`)
**Doc says:**
> the very same `tls-replication` directive will determine whether Sentinel's port, that accepts connections from other Sentinels, will support TLS as well. That is, Sentinel will be configured with `tls-port` if and only if `tls-replication` is enabled.

**Problem:** Sentinel listens on whatever `port` and `tls-port` you configure. `tls-replication` decides whether Sentinel uses TLS on outgoing links, and whether the port it announces to other Sentinels in hello messages is `tls-port` or `port`.
**Evidence:** valkey@9.1.2:src/sentinel.c:2936-2941:
```
    if (sentinel.announce_port)
        announce_port = sentinel.announce_port;
    else if (server.tls_replication && server.tls_port)
        announce_port = server.tls_port;
    else
        announce_port = server.port;
```
and :2370 `server.tls_replication && (instanceLinkNegotiateTLS(link->cc) == C_ERR)`.
**Fix:** "`tls-replication` also controls the port Sentinel announces to other Sentinels. With `tls-replication yes` and `tls-port` set, Sentinel advertises `tls-port`, so other Sentinels connect over TLS. Otherwise it advertises `port`. `sentinel announce-port` overrides both."

### F4 [minor] OUTDATED: Version-gated features with no version stated (`topics/tls.md:81`, `topics/tls.md:115`)
**Doc says:**
> Valkey can automatically reload TLS materials at a specified interval

> To let clients authenticate as a user with one of the identities in the client certificate, configure `tls-auth-clients-user`.

**Problem:** `tls-auto-reload-interval` is new in 9.1. `tls-auth-clients-user` arrived in 9.0 with CN only, and URI was added in 9.1. A 8.x or 9.0 user will hit unknown-config or unknown-value errors.
**Evidence:** `git grep '"tls-auto-reload-interval"'` matches 9.1.2:src/config.c:3495 and nothing in 9.0.0. `"tls-auth-clients-user"` is present in 9.0.0. valkey@9.1.2:00-RELEASENOTES:163-164: `* Support automatic TLS reload by @yang-z-o (#3020)`, `* Support TLS authentication using SAN URI by @yang-z-o (#3078)` (9.1.0-rc1).
**Fix:** Add "Added in Valkey 9.1." to the auto reload paragraph. For `tls-auth-clients-user`, add "Added in Valkey 9.0 (`CN`). `URI` was added in 9.1."

### F5 [minor] MISSING: Accepted values for `tls-auth-clients` and `tls-auth-clients-user` (`topics/tls.md:105`, `topics/tls.md:116`)
**Doc says:**
> You may use `tls-auth-clients no` to disable client authentication.

> configure `tls-auth-clients-user`. When enabled, Valkey automatically authenticates TLS clients as Valkey ACL users by extracting a field from the client certificate

**Problem:** The page leaves out `optional` and never says which certificate field is used or how to choose it. A reader cannot configure `tls-auth-clients-user` from this page.
**Evidence:** valkey@9.1.2:src/config.c:121-131: `{"no", ...}, {"yes", ...}, {"optional", ...}` and `{"CN", ...}, {"URI", ...}, {"off", ...}`. The default is `TLS_CLIENT_FIELD_OFF` (:3499). valkey.conf:263-272 says URI uses "the first one matching an enabled Valkey user".
**Fix:** Add a table:

| Config | Values | Default |
|---|---|---|
| `tls-auth-clients` | `yes` (certificate required), `optional` (validated if presented), `no` (not requested) | `yes` |
| `tls-auth-clients-user` | `CN` (subject Common Name), `URI` (first SAN URI that names an enabled user), `off` | `off` |

### F6 [minor] MISSING: Unmatched certificates fall through to the default user (`topics/tls.md:119`)
**Doc says:**
> If no match is found, the client remains as the default user.

**Problem:** With stock config the `default` user is `on nopass +@all`. A client whose certificate matches no user, or matches a disabled user, gets full access. Readers who rely on certificate identity for access control need to be told to lock down `default`.
**Evidence:** valkey@9.1.2:src/tls.c:1307-1312 returns NULL for a missing or disabled user (`if (!result || !(result->flags & USER_FLAG_ENABLED)) { ... result = NULL; }`). valkey.conf:258-259: "If no match is found, the client remains unauthenticated as the default user."
**Fix:** Append: "If you use certificate identity for access control, restrict or disable the default user (for example `ACL SETUSER default off`) so unmatched certificates get no access. Failed matches are counted in `acl_access_denied_tls_cert` in `INFO stats`."

### F7 [minor] MISSING: TLS module build needs `--loadmodule`, and Sentinel cannot use it (`topics/tls.md:6`, `topics/tls.md:17`)
**Doc says:**
> SSL/TLS is supported by Valkey as an optional feature that needs to be enabled at compile time.

**Problem:** Valkey can build TLS in (`BUILD_TLS=yes`) or as a module (`BUILD_TLS=module`). With a module build, the quick start command fails unless the module is loaded, and Sentinel does not support the module.
**Evidence:** valkey@9.1.2:README.md:30-38:
```
To build TLS support as Valkey built-in:
    % make BUILD_TLS=yes
To build TLS as Valkey module:
    % make BUILD_TLS=module
Note that sentinel mode does not support TLS module.
```
and README.md:204-208 shows `--loadmodule src/valkey-tls.so` in the module-mode run command.
**Fix:** After line 9 add: "Build with `make BUILD_TLS=yes` to compile TLS in, or `make BUILD_TLS=module` to build `valkey-tls.so` and load it with `--loadmodule /path/to/valkey-tls.so`. Sentinel requires the built-in build."

### F8 [nit] STYLE: Code blocks without a language tag (`topics/tls.md:17`, `:25`, `:40`, `:55`, `:66`, `:76`, `:87`, `:122`)
**Doc says:**
>     ./src/valkey-server \

**Problem:** Two indented blocks and six untagged fences. Line 122 is a command but looks like config.
**Fix:** Turn the indented blocks into ```` ```bash ```` fences. Tag the config blocks as `conf`. Show line 122 as `valkey-cli ACL SETUSER client-user on allcommands allkeys` in a `bash` fence, or tag it `text` with "Run:" before it.

### F9 [nit] AI-NATIVE: "Getting Started" H2 holds every section (`topics/tls.md:11`)
**Doc says:**
> ## Getting Started

**Problem:** Replication, Cluster, Sentinel, and Performance all sit under "Getting Started" as H3s, and that is the only H2. The heading says nothing about the content below it.
**Fix:** Drop "Getting Started" and promote each H3 to H2 ("Quick start", "TLS listening port", "Certificates and keys", "Client certificate authentication", "TLS for replication", "TLS for the cluster bus", "TLS with Sentinel", ...).

### F10 [nit] MISSING: Cluster section omits TLS announce ports (`topics/tls.md:137`)
**Doc says:**
> When Valkey Cluster is used, use `tls-cluster yes` in order to enable TLS for the cluster bus and cross-node connections.

**Problem:** Behind NAT or with mixed TLS/plain clients, operators need `cluster-announce-tls-port`. topics/cluster-tutorial.md:68-70 documents it, and this page does not link there.
**Fix:** Append: "To advertise TLS ports to clients, see `cluster-announce-tls-port` and `cluster-announce-client-tls-port` in the [cluster tutorial](cluster-tutorial.md)."

### F11 [nit] AI-NATIVE: Performance section is generic (`topics/tls.md:160`)
**Doc says:**
> TLS adds a layer to the communication stack with overheads due to writing/reading to/from an SSL connection, encryption/decryption and integrity checks.

**Problem:** This is true of any TLS server. The Valkey-specific advice is that I/O threads handle TLS connections and absorb much of the cost.
**Evidence:** valkey@9.1.2:00-RELEASENOTES:56 "Skip unnecessary post-read processing with I/O threads on socket and TLS connections".
**Fix:** Replace with: "TLS lowers per-instance throughput because of encryption and integrity checks. Enable `io-threads` to spread TLS read and write work across threads."

### F12 [nit] STYLE: Weak front matter description and unlinked `valkey.conf` reference (`topics/tls.md:3`, `topics/tls.md:156`)
**Doc says:**
> description: Valkey TLS support

> Please consult the self documented `valkey.conf` for more information.

**Problem:** The description carries no information. The `valkey.conf` reference has no link, and `tls-key-file-pass`, `tls-protocols` (default TLSv1.2 and TLSv1.3), and `tls-ciphers` are never named.
**Fix:** Description: "Enable TLS and mTLS in Valkey for clients, replication, cluster bus, and Sentinel, including certificate reload and certificate-based ACL authentication." Line 154-156: "Use `tls-key-file-pass` for encrypted keys. `tls-protocols` (default `TLSv1.2 TLSv1.3`), `tls-ciphers`, and `tls-ciphersuites` control protocol and cipher choice. See [valkey.conf](https://github.com/valkey-io/valkey/blob/unstable/valkey.conf)."

## Unverified
- topics/tls.md:84: "Reloading is performed in a background thread, so it does not block the main server thread." 9.1.2:src/tls.c:919 queues `bioCreateTlsReloadJob()`, and :687 says "Updated serially by BIO thread, applied atomically by main thread". I did not trace whether the main thread's apply step blocks on file I/O. The server binary is built without TLS, so I could not run it.
