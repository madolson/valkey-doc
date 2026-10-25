VERDICT: MINOR-ISSUES
COUNTS: blocker=0 minor=2 nit=4

# Troubleshooting Valkey (`problems`)

https://valkey.io/topics/problems/

A three-bullet link hub. The one command it gives fails as written, and it never says where to report a problem. Fix the `--test-memory` invocation first.

## Findings

### F1 [minor] EXAMPLE: `valkey-server --test-memory` requires a size argument (`topics/problems.md:10`)
**Doc says:**
> Please test your RAM using **valkey-server --test-memory** in case Valkey is not stable in your system.
**Problem:** Run as written, the command exits with a usage error and tests nothing. It needs a size in megabytes.
**Evidence:** valkey@9.1.2:src/server.c:7552 `"Example: ./valkey-server --test-memory 4096\n\n"`. Live 9.1.2 run:
```
$ valkey-server --test-memory
Please specify the amount of memory to test in megabytes.
Example: ./valkey-server --test-memory 4096
```
**Fix:** "Test your RAM with `valkey-server --test-memory <megabytes>`, for example `valkey-server --test-memory 4096`. See [valkey-server](server.md) for details." Put the command in backticks instead of bold.

### F2 [minor] MISSING: no place to report a problem or get help (`topics/problems.md:6`)
**Doc says:**
> Part of the Valkey project is helping people that are experiencing problems because we don't like to leave people alone with their issues.
**Problem:** The page promises help but gives no channel. A reader with an issue that is not latency, a crash, or bad RAM has nowhere to go.
**Evidence:** The page has three bullets (lines 8 to 10). None links to an issue tracker or community channel.
**Fix:** Add a bullet: "To report a bug, open an issue at https://github.com/valkey-io/valkey/issues and include the output of `INFO` and the server log. For questions, see the community channels at https://valkey.io/connect/." Verify the connect URL before publishing.

### F3 [nit] OUTDATED: "long history" of crashes is Redis-era history (`topics/problems.md:10`)
**Doc says:**
> We have a long history of users experiencing crashes with Valkey that actually turned out to be servers with **broken RAM**.
**Problem:** Valkey forked in 2024. The history was inherited from Redis. The sentence also reads as an admission that Valkey crashes often.
**Fix:** "Crashes that look like Valkey bugs are often caused by faulty RAM."

### F4 [nit] CONFUSING: garbled latency bullet (`topics/problems.md:8`)
**Doc says:**
> If you have **latency problems** with Valkey, that in some way appears to be idle for some time, read our [Valkey latency troubleshooting guide](latency.md).
**Problem:** "that in some way appears to be idle for some time" has no clear subject and is hard to parse.
**Fix:** "If Valkey responds slowly or stalls intermittently, read the [latency troubleshooting guide](latency.md)."

### F5 [nit] AI-NATIVE: intro is filler and front matter description is weak (`topics/problems.md:3`, `topics/problems.md:6`)
**Doc says:**
> description: Problems with Valkey? Start here.

> This page tries to help you with what to do if you have issues with Valkey. Part of the Valkey project is helping people...
**Problem:** Neither says what the page covers. The intro carries no information.
**Fix:** description: "Where to start when Valkey has latency problems, crashes, or unstable memory." Intro: "This page points to the right guide for common Valkey problems: latency, crashes, and faulty RAM."

### F6 [nit] DUPLICATE: RAM test advice repeated in quickstart (`topics/problems.md:10`)
**Doc says:**
> Please test your RAM using **valkey-server --test-memory**
**Problem:** topics/quickstart.md:93 repeats the same advice ("Run `valkey-server --test-memory` to perform a memory test") with the same missing size argument. It already links here at quickstart.md:95.
**Fix:** Keep problems.md canonical. In quickstart.md:93, cut the command and link to problems.md (or fix the argument there too).

## Unverified
- topics/problems.md:10: memtest86 link https://memtest86.com. curl returns 403 with and without a browser user agent (redirects to https://www.memtest86.com/), likely bot protection. Not confirmed broken.
