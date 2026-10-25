VERDICT: MINOR-ISSUES
COUNTS: blocker=0 minor=4 nit=3

# HyperLogLog (`hyperloglogs`)

https://valkey.io/topics/hyperloglogs/

Example and links are correct on 9.1.2. The main fix is the Performance section, which calls PFADD and PFCOUNT constant time when multi-key PFCOUNT is O(N) and sparse-encoded PFADD is O(N). The page also repeats itself and omits the sparse/dense encoding that only pfcount.md explains.

## Findings

### F1 [minor] INACCURATE: PFADD and PFCOUNT are not always constant time (`topics/hyperloglogs.md:82`)
**Doc says:**
> Writing (`PFADD`) to and reading from (`PFCOUNT`) the HyperLogLog is done in constant time and space.
**Problem:** PFCOUNT with multiple keys is O(N) in the number of keys with much larger constants, because it merges into a temporary HLL. PFADD on a sparse-encoded HLL is O(N) in the sparse representation size, which is why `hll-sparse-max-bytes` exists.
**Evidence:** `valkey@9.1.2:src/commands/pfcount.json`:
> "complexity": "O(1) with a very small average constant time when called with a single key. O(N) with N being the number of keys, and much bigger constant times, when called with multiple keys."

`valkey@9.1.2:valkey.conf:2373-2375`:
> The suggested value is ~ 3000 in order to have the benefits of the space efficient encoding without slowing down too much PFADD, which is O(N) with the sparse encoding.
**Fix:** "`PFADD` is O(1) per element. On small, sparse-encoded HLLs it is O(N) in the encoding size, bounded by `hll-sparse-max-bytes`. `PFCOUNT` on a single key is O(1) with a small constant (the result is cached in the key). `PFCOUNT` on multiple keys is O(N) in the number of keys with much higher constant time, because it merges the HLLs on the fly. `PFMERGE` is O(N) in the number of source HLLs."

### F2 [minor] MISSING: no sparse/dense encoding or `hll-sparse-max-bytes` (`topics/hyperloglogs.md:19`)
**Doc says:**
> constant amount of memory; 12k bytes in the worst case, or a lot less if your HyperLogLog ... has seen very few elements.
**Problem:** "A lot less" is never explained. The topic page does not name the sparse and dense encodings, the switch point, or the tuning config, so an operator sizing memory has nothing to go on. That content exists only on commands/pfcount.md:51-57, which is the wrong canonical home.
**Evidence:** `valkey@9.1.2:valkey.conf:2366-2378` defines `hll-sparse-max-bytes 3000` ("When a HyperLogLog using the sparse representation crosses this limit, it is converted into the dense representation"). Live 9.1.2: `CONFIG GET hll-sparse-max-bytes` returns `3000`. A 4-element HLL is 36 bytes (`STRLEN`). A ~20000-element HLL is 12304 bytes (12288 register bytes plus 16-byte header).
**Fix:** Add a "Memory usage and encoding" section: "Valkey stores small HLLs in a sparse run-length encoding and converts them to a dense 12,288-byte encoding (16,384 6-bit registers plus a 16-byte header) when the sparse form exceeds `hll-sparse-max-bytes` (default 3000). Raising it saves memory for many low-cardinality HLLs at the cost of slower `PFADD`." Move the encoding paragraphs from commands/pfcount.md:51-57 here and link to this page from pfcount.md.

### F3 [minor] CONFUSING: privacy note does not connect to HLL and overstates anonymity (`topics/hyperloglogs.md:59`)
**Doc says:**
> **Anonymous unique visits of a web page (SaaS, analytics tools)**
> Storing the IP address or any other kind of personal identifier is against the law in some countries, which makes it impossible to get unique visitor statistics on your website.
**Problem:** The note gives legal advice and never says what HLL does about it. The reader must infer that HLL does not store elements. "Anonymous" is also overstated: `PFADD` returns whether registers changed, so a caller can probe whether a guessed identifier (for example, an IPv4 address) was likely added.
**Evidence:** Live 9.1.2:
```
PFADD probe 1.2.3.4   -> (integer) 1
PFADD probe 1.2.3.4   -> (integer) 0
PFADD probe 5.6.7.8   -> (integer) 1
```
**Fix:** Replace the heading with "Unique visitors per page" and the note with: "An HLL stores only register state derived from element hashes, not the elements, so identifiers cannot be read back from the key. It is not anonymization: anyone who can run `PFADD` on the key can test guessed values, and your application still handles the raw identifiers. Hash or salt identifiers before adding them if that matters."

### F4 [minor] AI-NATIVE: same facts stated two or three times (`topics/hyperloglogs.md:9`)
**Doc says:**
> line 9: "uses up to 12 KB and provides a standard error of 0.81%." line 16: "is less than 1%." line 19: "12k bytes in the worst case"
> line 36: "If you need to merge two different HLLs, the `PFMERGE` command is available." line 54: "Valkey is also able to perform the union of HLLs" line 76: "`PFMERGE` combines two or more HyperLogLogs into one."
> line 51-52: "Some examples of use cases ... number of unique visitors to a web page" repeated by the "Use cases" section at lines 57-70.
> line 55 and line 78: two links to the same `../commands/#hyperloglog`.
**Problem:** Memory, error rate, merge, use cases, and the command list link each appear two or three times. Error is given as both "0.81%" and "less than 1%". Readers scan for facts and hit repetition.
**Fix:** Keep lines 7-9 as the summary. Delete lines 14-16 "which, in the case of the Valkey implementation for HyperLogLog, is less than 1%" (end the sentence at "standard error"). Delete lines 51-55. Keep the link at line 78 only.

### F5 [nit] STYLE: "sketches" undefined and inconsistent terminology (`topics/hyperloglogs.md:83`)
**Doc says:**
> Merging HLLs is O(n), where _n_ is the number of sketches.
**Problem:** The page uses "HyperLogLog" and "HLL" everywhere else. "Sketches" is not defined.
**Fix:** "where _n_ is the number of source HLLs."

### F6 [nit] EXAMPLE: fenced block has no language tag (`topics/hyperloglogs.md:38`)
**Doc says:**
> ```` ``` ````
**Problem:** The CLI session block has no language tag. Output itself matches 9.1.2 exactly.
**Fix:** Use ```` ```bash ```` or the tag other topic pages use for valkey-cli sessions.

### F7 [nit] STYLE: grammar and bold pseudo-headings (`topics/hyperloglogs.md:20`)
**Doc says:**
> line 20: "HyperLogLog (We'll just call them HLL from now) has seen very few elements."
> line 51: "Some examples of use cases for this data structure is counting"
> line 59 and 67: `**Anonymous unique visits ...**`, `**Note:**`
**Problem:** Mid-sentence capitalized aside, subject-verb disagreement, and bold text standing in for a heading.
**Fix:** Introduce "HLL" at first use on line 7: "HyperLogLog (HLL) is ...". Line 51 "is" to "are" (or delete per F4). Make line 59 a `###` heading.
