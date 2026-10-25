VERDICT: MINOR-ISSUES
COUNTS: blocker=0 minor=3 nit=3

# ARM support (`ARM`)

https://valkey.io/topics/ARM/

The page is a Redis 4.0-era Raspberry Pi note. Its main build claim (libc allocator by default on ARM) has been false since 2022, and it says nothing about aarch64 servers, which is how most people run Valkey on ARM today. Fix the allocator paragraph first.

## Findings

### F1 [minor] INACCURATE: ARM does not default to libc malloc (`topics/ARM.md:36`)
**Doc says:**
> The only difference is that by default, Valkey uses the `libc` allocator instead of defaulting to `jemalloc` as it does in other Linux based environments. This is because we believe that for the small use cases inside embedded devices, memory fragmentation is unlikely to be a problem. Moreover `jemalloc` on ARM may not be as tested as the `libc` allocator.

**Problem:** The Makefile picks jemalloc on every Linux build, whatever the architecture. The ARM exception was removed in Redis #11407 (2022), and that commit notes the old check never matched aarch64 or armv8l anyway.
**Evidence:** `valkey@9.1.2:src/Makefile:74-78`
```
# Default allocator defaults to Jemalloc on Linux and libc otherwise
MALLOC=libc
ifeq ($(uname_S),Linux)
	MALLOC=jemalloc
endif
```
`valkey@3e112d461` "Use jemalloc by default also on ARM (#11407)": "Till now Redis attempted to avoid using jemalloc on ARM, but didn't do that properly (missing armv8l and aarch64), so in fact we did you jemalloc on these without a problem."
**Fix:** Replace lines 36-41 with: "There is nothing ARM-specific in the build. On Linux, Valkey uses jemalloc by default on ARM, as on every other architecture. To use the libc allocator instead, build with `make MALLOC=libc`."

### F2 [minor] OUTDATED: Benchmarks are Redis OSS 4.0 numbers on Raspberry Pi 1 and 3 (`topics/ARM.md:45`)
**Doc says:**
> Performance testing of Valkey was performed on the Raspberry Pi 3 and Pi 1 model B. [...] The following numbers were obtained using Redis OSS 4.0.

**Problem:** The heading and first sentence present these as Valkey results, but they come from a different product, released about eight years ago, on hardware that is now obsolete. Readers will take them as current Valkey ARM performance.
**Evidence:** The page says so itself at line 47-48 ("obtained using Redis OSS 4.0"). No Valkey version is named anywhere on the page.
**Fix:** Delete lines 43-64. If numbers are wanted, replace them with one sentence pointing readers to [valkey-benchmark](benchmark.md) to measure their own device, or with Valkey 9.x numbers that state the hardware, version, and `valkey-benchmark` command line used.

### F3 [minor] MISSING: No coverage of aarch64 servers or the jemalloc page-size pitfall (`topics/ARM.md:7`)
**Doc says:**
> Valkey supports the ARM processor, for example the Raspberry Pi, as a main platform.

**Problem:** The page treats ARM as an embedded and IoT platform only. Most ARM deployments today are 64-bit servers (for example AWS Graviton), and Valkey's own release benchmarks run on arm64. The one ARM-specific trap is left out: jemalloc bakes in the build host's page size, so a binary built on a 4K-page kernel fails on a 64K-page kernel.
**Evidence:** `valkey@9.1.2:.github/workflows/benchmark-release.yml:41-42` (`arch: arm64`, `machine: ec2-al-2023-pr-benchmarking-arm64`). `valkey@3e112d461` commit message: "jemalloc uses the page size of the build machine as the maximum page size to be supported by the build. [...] one can use: `JEMALLOC_CONFIGURE_OPTS=\"--with-lg-page=16\" make`". `valkey@9.1.2:README.md:144-148`: CNTVCT monotonic clock "enabled by default on supported architectures (x86_64 Linux and aarch64)".
**Fix:** Change the first sentence to "Valkey supports 32-bit and 64-bit ARM (aarch64) on Linux, from Raspberry Pi boards to ARM servers." Then add a section "Building portable ARM binaries" with: "jemalloc uses the build machine's page size as its maximum. If you distribute a binary or container image across ARM hosts with different page sizes (4K and 64K), build with `JEMALLOC_CONFIGURE_OPTS=\"--with-lg-page=16\" make`."

### F4 [nit] INACCURATE: "SSD cards" should be "SD cards" (`topics/ARM.md:19`)
**Doc says:**
> The append-only file storage of Valkey is well suited for SSD cards.

**Problem:** On a Raspberry Pi the storage is SD cards. "SSD cards" is not a thing.
**Fix:** "Valkey's append-only file writes sequentially, which suits SD card storage."

### F5 [nit] STYLE: Grammar and stale wording (`topics/ARM.md:9`, `:24`, `:28`, `:31`)
**Doc says:**
> While Valkey does run on Android, in the future we look forward to extend our testing efforts to Android to also make it an officially supported platform.

> Linux on ARM allows to trap unaligned accesses

> Even when kernel alignment fixing set as disabled Valkey should run as expected.

> ## Building Valkey in the Pi

**Problem:** Ungrammatical phrasing, and a forward-looking Android promise with no date or tracking issue. Nothing in the 9.1.2 source or CI targets Android.
**Fix:** Line 9-10: "Valkey may build and run on Android, but Android is not tested or officially supported." Line 24: "Linux on ARM can trap unaligned accesses". Line 28-29: "Valkey runs correctly even with kernel alignment fixing disabled." Line 31: "## Building Valkey on a Raspberry Pi".

### F6 [nit] AI-NATIVE: Weak description and generic headings (`topics/ARM.md:4`, `:43`)
**Doc says:**
> Exploring Valkey on the ARM CPU Architecture

> ## Performance

**Problem:** The description says nothing concrete. "Performance" does not say what is measured or on what hardware. The intro bullet list (lines 15-20) is generic IoT marketing that comes before the build facts readers need.
**Fix:** Description: "Supported ARM platforms, build notes, and allocator defaults for running Valkey on ARM Linux." If F2's section stays, rename it "Benchmark results on Raspberry Pi". Cut lines 12-20 down to one sentence, or move them below the build section.

## Unverified
- topics/ARM.md:26: "Valkey avoids any kind of unaligned access". `valkey@9.1.2:src/config.h:322-323` defines `USE_ALIGNED_ACCESS` only for `__sparc__`/`__arm__` (32-bit), so on aarch64 the code may rely on hardware unaligned access support. I did not audit every code path, and could not test on ARM hardware. The safer wording is "On 32-bit ARM, Valkey avoids unaligned accesses".
