VERDICT: NEEDS-WORK
COUNTS: blocker=2 minor=4 nit=4

# LDAP Authentication (`ldap`)

https://valkey.io/topics/ldap/

Mostly accurate against valkey-ldap 1.1.1 (tag 46c80e1) and main (6ad54c2), but the search+bind prose names a config that does not exist, and the "LDAP only" user setup leaves password login open for existing users. Fix the config name first.

## Findings

### F1 [blocker] INACCURATE: `ldap.search_base_dn` does not exist (`topics/ldap.md:47`)
**Doc says:**
> which is set in the module configuration under the `ldap.search_base_dn` parameter.
**Problem:** The config is `ldap.search_base`. A reader following the prose gets an unknown-option error from `CONFIG SET` and search+bind runs with an empty base. The table at line 200 has the right name, so the page contradicts itself.
**Evidence:** valkey-ldap@1.1.1:src/lib.rs (same on main, line 157):
```
            [
                "search_base",
                &*configs::LDAP_SEARCH_BASE,
                "",
```
**Fix:** Replace `ldap.search_base_dn` with `ldap.search_base`.

### F2 [blocker] INACCURATE: "only possible through the LDAP module" is false for users that already have a password (`topics/ldap.md:89`)
**Doc says:**
> To create a user without a password, simply omit the password when running the `ACL SETUSER` command ... `ACL SETUSER bob on +@hash` ... After creating the user `bob` in Valkey, authentication will only be possible through the LDAP module.
**Problem:** `ACL SETUSER` modifies an existing user in place, so omitting a password keeps any existing one. When LDAP rejects the credentials the module returns `AUTH_NOT_HANDLED`, so Valkey falls through to password auth. An operator moving existing users to LDAP leaves the old password working. The module README uses `resetpass` for this reason.
**Evidence:** Valkey 9.1.2 run:
```
> acl setuser bob '>secret'
OK
> acl setuser bob on +@hash
OK
> acl list
user bob on #2bb80d53... resetchannels -@all +@hash
> auth bob secret
OK
```
valkey-ldap@6ad54c2:src/auth.rs:
```
                if configs::get_return_auth_errors(ctx) {
                    Err(ValkeyError::String(err.to_string()))
                } else {
                    Ok(AUTH_NOT_HANDLED)
                }
```
**Fix:** Use `ACL SETUSER bob on resetpass ~* +@hash` and say: "`resetpass` removes any existing passwords so `bob` can't fall back to password authentication when the LDAP check fails."

### F3 [minor] EXAMPLE: example user gets no key access (`topics/ldap.md:94`)
**Doc says:**
> ACL SETUSER bob on +@hash
**Problem:** No key pattern is granted, so every hash command `bob` runs fails with NOPERM. A reader testing LDAP login will think the setup is broken.
**Evidence:** 9.1.2 `ACL LIST` after the command: `user bob on resetchannels -@all +@hash` (no `~` pattern).
**Fix:** Add a key pattern, for example `ACL SETUSER bob on resetpass ~* +@hash` (folds into the F2 fix).

### F4 [minor] MISSING: `ldap.return_auth_errors` undocumented, and so is the failure behavior (`topics/ldap.md:173`)
**Doc says:** General Options lists only `ldap.auth_mode` and `ldap.servers`.
**Problem:** Module 1.1.0 added `ldap.return_auth_errors` (bool, default `no`). By default a failed LDAP auth comes back as a generic `WRONGPASS`, and the real cause only shows up in the server log. Operators debugging a failed login need to know this. Turning it on also ends the auth chain when several auth modules are loaded.
**Evidence:** valkey-ldap@6ad54c2:src/lib.rs:216-221 `"return_auth_errors", ..., false`. CHANGELOG.md 1.1.0: "When `return_auth_errors` is disabled, the default behavior of returning `WRONGPASS` error is kept." Module README: "If enabled, will end the auth chain if using multiple authentication modules".
**Fix:** Add a row: `| ldap.return_auth_errors | boolean | no | Return the LDAP error text from AUTH instead of WRONGPASS. When enabled, a failure ends the auth chain if several auth modules are loaded. Added in valkey-ldap 1.1.0. |`. In Monitoring, add: "The cause of each failed LDAP authentication is written to the server log."

### F5 [minor] MISSING: no instructions for getting or loading the module (`topics/ldap.md:7`)
**Doc says:**
> To enable this feature, you must load the module on your Valkey server.
**Problem:** The page never says where the binary comes from or how to load it, so a reader can't get to the first configuration step.
**Evidence:** Module README: packages come from the "Build Packages" workflow artifacts, or a source build produces `target/release/libvalkey_ldap.so`. topics/valkey-bundle.md:16 ships it preloaded.
**Fix:** After line 7 add: "Load it with `loadmodule /path/to/libvalkey_ldap.so` in `valkey.conf` or `MODULE LOAD /path/to/libvalkey_ldap.so`. Packages and build instructions are in the [valkey-ldap README](https://github.com/valkey-io/valkey-ldap#installation). The module is preloaded in [valkey-bundle](valkey-bundle.md)."

### F6 [minor] MISSING: search must match exactly one entry (`topics/ldap.md:47`)
**Doc says:**
> The module will then attempt to match the username specified in the `AUTH` command against the value of a configurable entry attribute
**Problem:** Authentication fails if the search returns zero entries or more than one, and if the entry lacks `ldap.search_dn_attribute`. Directories with duplicate `uid` values under the base hit this. `entryDN` is also an operational attribute that some servers, including Active Directory, don't provide (AD uses `distinguishedName`).
**Evidence:** valkey-ldap@6ad54c2:src/vkldap/connection.rs:335-351:
```
        if rs.len() == 0 {
            return Err(VkLdapError::NoLdapEntryFound(search_filter));
        }
        if rs.len() > 1 {
            return Err(VkLdapError::MultipleEntryFound(search_filter));
        }
        ...
        if !sentry.attrs.contains_key(dn_attribute) {
            return Err(VkLdapError::InvalidDNAttribute(dn_attribute.clone()));
```
**Fix:** After line 49 add: "The search must return exactly one entry, and that entry must contain the `ldap.search_dn_attribute` attribute. Otherwise authentication fails. Use `ldap.search_filter`, `ldap.search_base`, and `ldap.search_scope` to narrow the search."

### F7 [nit] INACCURATE: anonymous search happens when either credential is missing (`topics/ldap.md:42`)
**Doc says:**
> If no username and password is configured for search bind account, an anonymous bind will be attempted
**Problem:** The admin bind is skipped unless both `search_bind_dn` and `search_bind_passwd` are set. Setting only the DN still gives an unbound search.
**Evidence:** valkey-ldap@6ad54c2:src/vkldap/connection.rs:286-287: `if let Some(bind_dn) = &settings.search_bind_dn { if let Some(bind_passwd) = &settings.search_bind_passwd {`
**Fix:** "If `ldap.search_bind_dn` or `ldap.search_bind_passwd` is empty, the module skips the bind and searches without credentials."

### F8 [nit] INACCURATE: `CONFIG GET ldap.*` does not show all options (`topics/ldap.md:101`)
**Doc says:**
> To view the current values of all module-related options, use the command `CONFIG GET ldap.*`.
**Problem:** `ldap.search_bind_passwd` is registered `SENSITIVE | HIDDEN`, so the glob leaves it out.
**Evidence:** valkey-ldap@6ad54c2:src/lib.rs:185-188: `"search_bind_passwd", ..., ConfigurationFlags::SENSITIVE | ConfigurationFlags::HIDDEN`
**Fix:** Append: "`ldap.search_bind_passwd` is hidden from the pattern match."

### F9 [nit] STYLE: fenced blocks without a language tag (`topics/ldap.md:25`)
**Doc says:** bare fences at lines 25, 55, 93, 140.
**Problem:** No language tag on any example block.
**Evidence:** `topics/ldap.md:25` "```", same at 55, 93, 140.
**Fix:** Use ```` ```ldif ```` for lines 25 and 55, and ```` ```text ```` for 93 and 140 (or put a `>` prompt in a ```` ```bash ```` block for 93).

### F10 [nit] AI-NATIVE: weak front matter description, and line 19 is confusing (`topics/ldap.md:4`)
**Doc says:**
> Introduction to LDAP / Active Directory Authentication

> Typically, the prefix is set to `CN=` or `DOMAIN\` for Active Directory environments, while the suffix defines the remainder of the DN in non-Active Directory setups.
**Problem:** The description doesn't name the module or the use case. Line 19 mixes AD and non-AD in one sentence, and the default prefix is lowercase `cn=` (line 191).
**Fix:** description: "Authenticate Valkey users against LDAP or Active Directory with the valkey-ldap module: bind and search+bind modes, TLS, and configuration reference." Line 19: "For Active Directory, set the prefix to `DOMAIN\` and leave the suffix empty. For other directories, set the prefix to the RDN attribute (default `cn=`) and the suffix to the rest of the DN."

## Unverified
- topics/ldap.md:7: "compatible with Valkey versions 7.2.x and above". CHANGELOG 1.0.0-rc1 says 7.2.X, but main ships Dockerfiles only for 8.0 to 9.1. I didn't build and load it on 7.2.
- topics/ldap.md:42-45: anonymous search over pooled connections. Pooled connections are rebound as the last authenticated user and never unbound (connection.rs `bind`, `close`), so a later "anonymous" search may run as that user. I didn't confirm whether the pool resets connections between requests.
