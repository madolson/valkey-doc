VERDICT: NEEDS-WORK
COUNTS: blocker=0 minor=7 nit=8

# ACL (`acl`)

https://valkey.io/topics/acl/

Most examples still run on 9.1.2. The page has some Redis-era leftovers (module categories, "Valkey 7.0", an empty `+@all VS -@all` section), and the new database-permissions section gets a few details wrong. The most important fix is the Sentinel ACL rule: it lacks `+failover` and `+client|pause`/`unpause`, which coordinated failover needs, and it contradicts sentinel.md.

## Findings

### F1 [minor] INACCURATE: NOPERM error strings in alice example do not match 9.1.2 (`topics/acl.md:184`)
**Doc says:**
> (error) NOPERM this user has no permissions to access one of the keys used as arguments
> ...
> (error) NOPERM this user has no permissions to run the 'set' command
**Problem:** 9.1.2 prints different text for both errors. Readers who grep logs or match errors in client code will search for the wrong string.
**Evidence:** 9.1.2 run (`AUTH alice p1pp0`, then the commands):
```
NOPERM No permissions to access a key
NOPERM User alice has no permissions to run the 'set' command
```
**Fix:** Replace line 184 with `(error) NOPERM No permissions to access a key` and line 188 with `(error) NOPERM User alice has no permissions to run the 'set' command`.

### F2 [minor] OUTDATED: "command categories never include modules commands" (`topics/acl.md:266`)
**Doc says:**
> Note that command categories **never include modules commands** with the exception of +@all. ... However if you use the ACL rule +@read or any other, the modules commands are always excluded.
**Problem:** Modules can assign their commands to built-in categories, and can also register new categories. So `+@read` can grant module commands.
**Evidence:** valkey@9.1.2:src/module.c:1396 `ValkeyModule_SetCommandACLCategories to set ACL categories of the commands.` and src/module.c:1607 `Any module can register a command to any added categories using ValkeyModule_SetCommandACLCategories.`
**Fix:** Replace lines 266-274 with: "Modules can assign their commands to built-in or module-defined categories with `ValkeyModule_SetCommandACLCategories`. So `+@read` and other category rules can grant module commands, and `+@all` grants every module command, including ones loaded later. Before you load a module on an instance where users have category-based rules, check its categories with `ACL CAT <category>`."

### F3 [minor] OUTDATED: Valkey version references that never existed (`topics/acl.md:85`)
**Doc says:**
> Starting Valkey 7.0, it can be used with `|` for blocking subcommands

and lines 396-397:
> It is possible to use this mechanism in order to allow subcommands in Valkey versions prior to 7.0 (see above section).
**Problem:** Valkey's first release was 7.2.4, forked from Redis, so "Valkey 7.0" and "Valkey versions prior to 7.0" refer to nothing. The "above section" pointer at 397 also has no clear target.
**Evidence:** `git -C ~/git/valkey tag | sort -V | head -1` gives 7.2.4. The subcommand rules have worked in every Valkey release.
**Fix:** Line 85: drop "Starting Valkey 7.0,". Lines 396-397: delete.

### F4 [minor] BROKEN: "+@all VS -@all" section is an empty stub (`topics/acl.md:399`)
**Doc says:**
> ## +@all VS -@all
>
> In the previous section, it was observed how it is possible to define command ACLs based on adding/removing single commands.
**Problem:** The heading promises a comparison, and the section has one sentence that points backwards. The content the heading names is missing.
**Evidence:** Lines 399-402 are the whole section. The next line is `## Selectors`.
**Fix:** Delete the section, or replace the body with: "`+@all` grants every command, including module commands loaded later. Use `+@all -@dangerous` to start broad and subtract. `-@all` (the default for new users) grants nothing. Use `-@all +get +set` to grant only what a client needs. The allowlist form is safer, because new commands and modules never widen access by accident."

### F5 [minor] MISSING: database permissions have no version note (`topics/acl.md:454`)
**Doc says:**
> ## Database permissions
>
> In addition to commands, keys, and Pub/Sub channels, ACL rules can restrict which databases ...

and line 378:
> For versions without database permissions, we could alter the ACL of a user in the following way:
**Problem:** `db=`, `alldbs`, and `resetdbs` arrived in 9.1. The section never says so, and line 378 refers to "versions without database permissions" without naming them. On 9.0, `db=0,1` fails, and nothing on the page explains why.
**Evidence:** valkey@9.1.2 `git log 9.0.0..9.1.2 -- src/acl.c` includes `122070cf8 Database-level access control (#2309)`. Line 211 is the only place the page says 9.1 ("Valkey 9.1 added the `databases` field").
**Fix:** Add as the first sentence at line 456: "Database permissions are available in Valkey 9.1 and later." At line 378, change to "Before Valkey 9.1, you could alter the ACL of a user in the following way:". Also add "(Valkey 9.1 and later)" to the rule bullets at lines 107-114.

### F6 [minor] INACCURATE: database evaluation rules are wrong for FLUSHALL and COPY, and CLUSTER FLUSHSLOT is missing (`topics/acl.md:530`)
**Doc says:**
> * [`COPY`](../commands/copy.md) - the destination database id when the `DB` option is used.
>
> Commands that do not specify a database id but operate across the entire keyspace require the user to have `alldbs`:
> * FLUSHALL * CLUSTER CANCELSLOTMIGRATIONS * CLUSTER MIGRATESLOTS
**Problem:** Three errors. (1) The all-database commands do not need the `alldbs` flag. They need access to every database id, and an explicit `db=0,...,15` passes. (2) `CLUSTER FLUSHSLOT` also has the `ALL_DBS` flag and is not listed. (3) Like MOVE, COPY also requires access to the current (source) database. This check was added in 9.1.2.
**Evidence:** valkey@9.1.2:src/acl.c:1883-1888:
```
} else if ((cmd->flags & CMD_ALL_DBS) && !(selector->flags & SELECTOR_FLAG_ALLDBS)) {
    for (int i = 0; i < server.dbnum; i++) {
        if (!ACLSelectorCanAccessDb(selector, i)) {
```
src/acl.c:1870-1878: "Commands such as MOVE and COPY also read from / operate on the keys in the current database ... the current database must therefore be authorized as well". `git grep -l '"ALL_DBS"' 9.1.2 -- src/commands/` lists cluster-flushslot.json. 9.1.2 run:
```
> ACL SETUSER frank on >pw +@all ~* db=0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15
OK
(as frank) > FLUSHALL
OK
```
**Fix:** Line 530: "`COPY`: the current (source) database, and the destination database id when the `DB` option is used." Line 534: "Commands that operate across every database require access to all databases, either `alldbs` or a `db=` list that names every database id:". Add `CLUSTER FLUSHSLOT` to the list.

### F7 [minor] INACCURATE: Sentinel ACL rule breaks coordinated failover and contradicts sentinel.md (`topics/acl.md:647`)
**Doc says:**
> * AUTH, CLIENT, SUBSCRIBE, SCRIPT, PUBLISH, PING, INFO, MULTI, SLAVEOF, CONFIG, CLIENT, EXEC.
>
>     ACL SETUSER sentinel-user on >somepassword allchannels +multi +slaveof +ping +exec +subscribe +config|rewrite +role +publish +info +client|setname +client|kill +script|kill
**Problem:** Since 9.0, Sentinel's coordinated failover (`SENTINEL FAILOVER <name> COORDINATED`) sends `FAILOVER TO`, `FAILOVER ABORT`, `CLIENT PAUSE`, and `CLIENT UNPAUSE`. This rule allows none of them. sentinel.md:820-822 says `+failover` and `+client` are required since 9.0 and gives a different rule. The list at line 649 also repeats CLIENT, omits ROLE (which the rule grants), and omits FAILOVER.
**Evidence:** valkey@9.1.2:src/sentinel.c:4780 `"%s PAUSE %d WRITE", sentinelInstanceMapCommand(ri, "CLIENT")`, :4787 `"%s TO %s %s TIMEOUT %d", sentinelInstanceMapCommand(ri, "FAILOVER")`, :4845 `"%s UNPAUSE"`, :4902 `"%s ABORT", ... "FAILOVER"`. topics/sentinel.md:820: "Since Valkey Sentinel 9.0, the sentinel user requires the `+failover` and `+client` permissions for proper operation."
**Fix:** Line 649: "AUTH, CLIENT, CONFIG, EXEC, FAILOVER, INFO, MULTI, PING, PUBLISH, ROLE, SCRIPT, SLAVEOF, SUBSCRIBE." Add `+failover +client|pause +client|unpause` to the rule at line 653. Make sentinel.md the canonical Sentinel ACL reference, and replace lines 647-653 here with a short pointer to it.

### F8 [nit] CONFUSING: "commands which do not require authentication cannot be removed" (`topics/acl.md:85`)
**Doc says:**
> Note that commands which do not require authentication cannot be removed, e.g. AUTH, HELLO, and RESET.
**Problem:** `-auth` is accepted and shows up in `ACL LIST`, but has no effect. A reader auditing `ACL LIST` output will think AUTH is blocked.
**Evidence:** 9.1.2 run:
```
> ACL SETUSER frank -auth -hello
OK
> ACL LIST
user frank on #30c9... ~* ... +@all -auth -hello
```
src/acl.c:1895 skips the command check when `cmd->flags & CMD_NO_AUTH`.
**Fix:** "Commands that do not require authentication, such as AUTH, HELLO, and RESET, are always allowed. `-auth` is accepted and appears in `ACL LIST`, but has no effect."

### F9 [nit] STYLE: typo "is valid matches" (`topics/acl.md:16`)
**Doc says:**
> If the password is valid matches, the connection will be authenticated
**Problem:** Ungrammatical.
**Evidence:** Line 16 as quoted.
**Fix:** "If the password matches, the connection is authenticated as `<username>`."

### F10 [nit] CONFUSING: ACL GETUSER description contradicts itself (`topics/acl.md:191`)
**Doc says:**
> it is possible to use an alternative to `ACL LIST` which is designed to be more suitable for computers to read, while `ACL GETUSER` is more human readable.
**Problem:** The sentence introduces `ACL GETUSER` as the computer-friendly alternative, then calls it the human-readable one. Line 211 then calls it "more parsable".
**Fix:** "To inspect alice's configuration (user names are case sensitive), use `ACL GETUSER`, which returns a field-value reply that is easier for programs to parse than `ACL LIST`."

### F11 [nit] EXAMPLE: myuser ACL LIST output order differs from 9.1.2 (`topics/acl.md:253`)
**Doc says:**
> 1) "user default on nopass ~* &* +@all"
> 2) "user myuser off resetchannels -@all +get +set"
**Problem:** 9.1.2 lists rules in the order they were applied, and still lists alice from the earlier example.
**Evidence:** 9.1.2 run of the page's sequence:
```
user alice on #2d9c75... ~cached:* ~objects:* ~items:* ~public:* resetchannels -@all +get
user default on nopass ~* &* +@all
user myuser off resetchannels -@all +set +get
```
**Fix:** Change line 255 to `"user myuser off resetchannels -@all +set +get"`.

### F12 [nit] EXAMPLE: ACL GETUSER default output omits the `databases` field (`topics/acl.md:560`)
**Doc says:**
> 9) "channels"
> 10) "&*"
> 11) "selectors"
> 12) (empty array)
**Problem:** 9.1.2 returns a `databases` field before `selectors`, as the page shows at line 211.
**Evidence:** 9.1.2 `ACL GETUSER default` returns `... channels &* databases alldbs selectors`.
**Fix:** Insert `11) "databases"` / `12) "alldbs"` and renumber `selectors` to 13-14.

### F13 [nit] STYLE: fenced blocks with no language tag (`topics/acl.md:473`)
**Doc says:**
> ```
> > ACL SETUSER alice on +@all ~* db=0,1 nopass
**Problem:** Code fences at lines 473, 483, 493, and 504 have no language tag. The rest of the page uses indented blocks, so the style is inconsistent too.
**Fix:** Tag them ` ```text ` (or convert to the indented style used elsewhere).

### F14 [nit] INACCURATE: ACL GENPASS output described as "alphanumerical" (`topics/acl.md:597`)
**Doc says:**
> The command outputs a 32-byte (256-bit) pseudorandom string converted to a 64-byte alphanumerical string.
**Problem:** The output is lowercase hex. The page also never mentions the optional bits argument.
**Evidence:** 9.1.2: `ACL GENPASS` returns `49d90767efbb19d22e17d435c81ca87ca54d382e55a9af7abdcfadb58d38cda1`, and `ACL GENPASS 64` returns `b3c91f7013e29af3`.
**Fix:** "By default the command returns 256 random bits as a 64-character hex string. Pass a bit count, for example `ACL GENPASS 128`, for a shorter one."

### F15 [nit] AI-NATIVE: generic managed-service filler (`topics/acl.md:33`)
**Doc says:**
> Another typical usage of ACLs is related to managed Valkey instances. Valkey is often provided as a managed service both by internal company teams ... we want to be sure that configuration commands are excluded for the customers.
**Problem:** Six lines that say only "managed services block CONFIG". No Valkey-specific detail.
**Fix:** Cut to one sentence appended to item 2: "Managed-service operators use the same mechanism to keep `CONFIG` and other `@admin` commands away from tenants."

## Unverified
- topics/acl.md:659: Atomic slot migration needs the replication user to have `+cluster|syncslots`. src/cluster_migrateslots.c:1074 shows an AUTH step on the export side. I did not trace which credentials it uses (`primaryuser` or another config) within the timebox. Related: valkey-io/valkey-doc#486.
