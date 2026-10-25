VERDICT: NEEDS-WORK
COUNTS: blocker=2 minor=4 nit=3

# RDMA experimental support (`RDMA`)

https://valkey.io/topics/RDMA/

The setup instructions are wrong for 9.1.2. The module ignores the `bind=`/`port=` arguments shown in every example, so RDMA never listens, and the tuning section names a config that does not exist. Fix both by switching to the `rdma-bind`, `rdma-port`, and `rdma-completion-vector` configs.

## Findings

### F1 [blocker] INACCURATE: Module arguments `bind=`/`port=` are ignored, RDMA never listens (`topics/RDMA.md:27`)
**Doc says:**
> --loadmodule src/valkey-rdma.so bind=192.168.122.100 port=6379

(same pattern at lines 36 and 39)
**Problem:** The RDMA module's `OnLoad` ignores its arguments. The listener comes from the `rdma-bind` and `rdma-port` configs, and `rdma-port` defaults to 0. With 0 the server never creates an RDMA listener. If you follow the page, the server starts but serves no RDMA traffic.
**Evidence:** valkey@9.1.2:src/rdma.c:1910-1912
```
int ValkeyModule_OnLoad(void *ctx, ValkeyModuleString **argv, int argc) {
    UNUSED(argv);
    UNUSED(argc);
```
valkey@9.1.2:src/server.c:3148 `if (server.rdma_ctx_config.port != 0) {`. valkey@9.1.2:src/config.c:3428 `createIntConfig("rdma-port", ... server.rdma_ctx_config.port, 0, ...)`. valkey@9.1.2:README.md:242 uses the config form: `--loadmodule src/valkey-rdma.so --rdma-bind 192.168.122.100 --rdma-port 6379`.
**Fix:** Replace the examples with:
```sh
./src/valkey-server --loadmodule src/valkey-rdma.so \
     --rdma-bind 192.168.122.100 --rdma-port 6379
```
In valkey.conf: `loadmodule src/valkey-rdma.so`, `rdma-bind 192.168.122.100`, `rdma-port 6379`.

### F2 [blocker] INACCURATE: `rdma-comp-vector` does not exist (`topics/RDMA.md:61`)
**Doc says:**
> It is possible to tune the performance using `rdma-comp-vector`.
**Problem:** The config is named `rdma-completion-vector`. It is also IMMUTABLE, so you can only set it at startup. The page never says that.
**Evidence:** valkey@9.1.2:src/config.c:3430 `createIntConfig("rdma-completion-vector", NULL, IMMUTABLE_CONFIG, -1, 1024, server.rdma_ctx_config.completion_vector, -1, ...)`. Live 9.1.2:
```
$ valkey-cli -p 20020 config set rdma-comp-vector 1
ERR Unknown option or number of arguments for CONFIG SET - 'rdma-comp-vector'
$ valkey-cli -p 20020 config get 'rdma*'
rdma-bind / rdma-completion-vector -1 / rdma-port 0 / rdma-rx-size 1048576
```
**Fix:** "Tune this with `rdma-completion-vector` (range -1 to 1024, default -1, startup only)."

### F3 [minor] OUTDATED: Built-in RDMA build mode and build steps missing (`topics/RDMA.md:6`)
**Doc says:**
> Valkey supports the Remote Direct Memory Access (RDMA) connection type via a Valkey module that can be dynamically loaded on demand.

(and lines 249-251: "build Valkey Over RDMA into Valkey module")
**Problem:** Since 8.1.0 (commit 4695d118d, "RDMA builtin support #1209"), RDMA can also be compiled into the server with `BUILD_RDMA=yes`. The page never says how to build either mode or which libraries are required, so `src/valkey-rdma.so` appears from nowhere. CMake supports only module mode.
**Evidence:** valkey@9.1.2:README.md:40-49 "you'll need RDMA development libraries (e.g. librdmacm-dev and libibverbs-dev on Debian/Ubuntu). To build RDMA support as Valkey built-in: % make BUILD_RDMA=yes. To build RDMA as Valkey module: % make BUILD_RDMA=module". valkey@9.1.2:README.md:335 "`-DBUILD_RDMA=<no|module>` ... (only module mode supported)".
**Fix:** Add a "Building with RDMA" section before "Running manually" with the library prerequisites and both `make BUILD_RDMA=yes` and `make BUILD_RDMA=module`. Note that CMake supports only `-DBUILD_RDMA=module`. Add a built-in run example: `./src/valkey-server --rdma-bind 192.168.122.100 --rdma-port 6379`. Change line 6-7 to "via a built-in connection type (since 8.1) or a loadable module."

### F4 [minor] STYLE: Examples disable protected mode on a network-facing server (`topics/RDMA.md:26`)
**Doc says:**
> ./src/valkey-server --protected-mode no \

(also line 35)
**Problem:** Both examples copy `--protected-mode no` with no password onto a server bound to a routable address. Readers who copy them get an unauthenticated server on the network. The page gives no reason for the flag.
**Evidence:** valkey@9.1.2:README.md:236 and 242 show the RDMA invocation without `--protected-mode no`.
**Fix:** Drop `--protected-mode no` from both examples. Add: "Protected mode rejects non-loopback clients when no password is set. Configure `requirepass` or ACL users (see [ACL](acl.md)) rather than disabling protected mode."

### F5 [minor] MISSING: No config reference for `rdma-bind`, `rdma-port`, `rdma-rx-size` (`topics/RDMA.md:29`)
**Doc says:**
> Bind address/port of RDMA can be modified at runtime using the following command:
**Problem:** The page never lists the four RDMA configs, their defaults, or what `rdma-port 0` means. `rdma-rx-size` is not mentioned at all, though it sets the per-connection RX buffer. Readers cannot find these values anywhere in valkey-doc.
**Evidence:** valkey@9.1.2:src/config.c:3428-3430, 3522: `rdma-port` 0-65535 default 0, MODIFIABLE. `rdma-rx-size` 64KB-16MB default 1MB, IMMUTABLE. `rdma-completion-vector` -1-1024 default -1, IMMUTABLE. `rdma-bind` MODIFIABLE, multi-arg. Without RDMA built: `CONFIG SET rdma-port 6380` returns `ERR CONFIG SET failed (possibly related to argument 'rdma-port') - No RDMA building support.`
**Fix:** Add a table under "Running manually":

| Config | Default | Range | Runtime | Meaning |
|---|---|---|---|---|
| `rdma-bind` | empty | addresses | yes | RDMA listen addresses |
| `rdma-port` | 0 | 0-65535 | yes | RDMA listen port; 0 disables RDMA |
| `rdma-rx-size` | 1048576 | 65536-16777216 | no | Per-connection RX buffer bytes |
| `rdma-completion-vector` | -1 | -1-1024 | no | CQ completion vector; -1 random |

### F6 [minor] CONFUSING: "How does Valkey use RDMA" prose is garbled (`topics/RDMA.md:249`)
**Doc says:**
> Enabling RDMA support needs to link additional libraries, rather than valkey-server's additional dependence on the shared libraries, build Valkey Over RDMA into Valkey module, Then a user starts valkey-server with RDMA module, valkey-server loads the additional shared libraries on demand.
**Problem:** This is a run-on sentence and hard to parse. It is also outdated because it describes module mode only (see F3). Line 247-248 implies TLS is module-only, but TLS can also be built in.
**Fix:** "Valkey abstracts listen, accept, connect, read, and write behind connection types. TCP and Unix sockets are always built in. TLS and RDMA can be built in or built as modules. Module mode keeps libibverbs and librdmacm out of valkey-server's dependencies. The libraries load only when you load `valkey-rdma.so` at startup."

### F7 [nit] AI-NATIVE: Headings don't describe their content (`topics/RDMA.md:9`)
**Doc says:**
> ## Getting Started

and `### Example 1` (line 65), `### Example 2` (line 81)
**Problem:** "Getting Started" holds an RDMA definition and a performance claim, not setup steps. "Example 1/2" don't say what each layout achieves.
**Fix:** Rename to "What RDMA provides", "Example: isolate Valkey from interrupt CPUs", and "Example: one CPU per server, interrupt, and completion vector".

### F8 [nit] AI-NATIVE: Unsourced performance claim (`topics/RDMA.md:15`)
**Doc says:**
> Test results indicate that Valkey Over RDMA achieves approximately 2 times higher QPS and lower latency.
**Problem:** The page doesn't say which test, hardware, or workload produced this figure, so readers cannot reproduce or quote it.
**Fix:** Link the benchmark source (PR or blog) and state the hardware and workload, or delete the sentence.

### F9 [nit] EXAMPLE: Commands mixed with annotations and inconsistent with README (`topics/RDMA.md:53`)
**Doc says:**
>     rdma dev show (a new version iproute2 package)
>
>     ibv_devices (ibverbs-utils package of Debian/Ubuntu)
**Problem:** The parenthetical sits inside the code block, so copy-paste fails. The indented blocks here and at lines 26-47 can't carry a language tag. README uses `rdma res show` (valkey@9.1.2:README.md:259).
**Fix:** Use fenced ```sh blocks throughout. Write: "Check that the interface supports RDMA with `rdma dev show` (iproute2) or `ibv_devices` (ibverbs-utils on Debian/Ubuntu)."

## Unverified
- topics/RDMA.md:255: "experimental" status. Tracked: valkey-io/valkey-doc#496 says RDMA is GA, but valkey@9.1.2:README.md:229 still says "Valkey Over RDMA is an experimental feature." I did not resolve which is correct.
- topics/RDMA.md:258: replication unsupported over RDMA. src/rdma.c has no replication handling, but I did not trace connTypeOfReplication.
- topics/RDMA.md:239: dl.acm.org links return 403 to curl (likely bot protection). They are probably fine in a browser.
