VERDICT: MINOR-ISSUES
COUNTS: blocker=0 minor=4 nit=8

# Bloom Filters (`bloomfilters`)

https://valkey.io/topics/bloomfilters/

Defaults, config names, error strings, and INFO fields are correct. Two `BF.INFO` example outputs show numbers the module does not produce, and the page never says how to load the module. Fix the example outputs first.

Verified against valkey-bloom 1.0.1 (tag) and default branch 74d7eb1, both built from source and loaded into the 9.1.2 server. Both gave the same results for everything below.

## Findings

### F1 [minor] EXAMPLE: `BF.INFO usernames` Size and Max scaled capacity are wrong (`topics/bloomfilters.md:114`)
**Doc says:**
> 3) Size
> 4) (integer) 23912
> ...
> 15) Max scaled capacity
> 16) (integer) 26214300
**Problem:** For `BF.RESERVE usernames 0.001 10000` the module reports a different size and max scaled capacity. 26214300 is the max scaled capacity of a default filter (fp 0.01, capacity 100), copied in from the default example.
**Evidence:** 9.1.2 + valkey-bloom 1.0.1 and 74d7eb1, after running the page's commands:
```
> BF.INFO usernames
Size
18236
...
Max scaled capacity
20470000
```
**Fix:** Line 114: `4) (integer) 18236`. Line 126: `16) (integer) 20470000`.

### F2 [minor] EXAMPLE: default filter shows 2 items after one `BF.ADD` (`topics/bloomfilters.md:204`)
**Doc says:**
> 127.0.0.1:6379> BF.ADD default_filter item
> ...
> 7) Number of items inserted
> 8) (integer) 2
**Problem:** The example adds one item, so the count is 1.
**Evidence:**
```
> BF.ADD default_filter item
1
> BF.INFO default_filter
...
Number of items inserted
1
```
**Fix:** `8) (integer) 1`.

### F3 [minor] MISSING: no instructions for loading the module (`topics/bloomfilters.md:7`)
**Doc says:**
> Users will need to load this module onto their valkey server in order to use this feature.
**Problem:** The page never shows how to load it. The Tutorial (line 63) only links to Docker and a build guide, so a reader who already has the `.so` file has to go to GitHub to find out.
**Evidence:** valkey-bloom README.md:32 `valkey-server --loadmodule ./target/release/libvalkey_bloom.so` and README.md:62 `loadmodule /path/to/libvalkey_bloom.so`.
**Fix:** After line 65 add: "To load a built module, start the server with `valkey-server --loadmodule /path/to/libvalkey_bloom.so`, or add `loadmodule /path/to/libvalkey_bloom.so` to valkey.conf. Module configs such as `bf.bloom-capacity` can then be set in valkey.conf or with `CONFIG SET`."

### F4 [minor] CONFUSING: the table says `bf.bloom-expansion` controls scaling but never says how (`topics/bloomfilters.md:183`)
**Doc says:**
> | Scaling / Non Scaling | Scaling | BF.INSERT, BF.RESERVE | BF.BLOOM-EXPANSION |
**Problem:** The page never says that setting the config to 0 makes new filters non-scaling. Line 189 says the defaults "can be configured through configs" but names only `NONSCALING` as the way to get a non-scaling filter. The commands reject `EXPANSION 0`, so a reader cannot work this out from the command pages either.
**Evidence:** valkey-bloom@74d7eb1:src/lib.rs:130 `["bloom-expansion", ..., configs::BLOOM_EXPANSION_DEFAULT, 0, configs::BLOOM_EXPANSION_MAX as i64, ...]` (minimum 0). command_handler.rs:156 `match args.expansion == 0 { ... "NONSCALING"`. Live run:
```
> CONFIG SET bf.bloom-expansion 0
OK
> BF.RESERVE z 0.01 100 EXPANSION 0
ERR bad expansion
```
**Fix:** Add to line 223: "Setting `bf.bloom-expansion` to 0 makes new filters non-scaling by default."

### F5 [nit] EXAMPLE: example output is not formatted the way valkey-cli prints it (`topics/bloomfilters.md:264`)
**Doc says:**
> 1) (error) ERR operation exceeds bloom object memory limit

and line 356:
> []
**Problem:** valkey-cli adds no `1)` prefix to a single error and prints an empty array as `(empty array)`.
**Evidence:**
```
> bf.add full_filter ne_item
(error) ERR operation exceeds bloom object memory limit
> BF.INSERT vv VALIDATESCALETO 1000
(empty array)
```
**Fix:** Line 264: `(error) ERR operation exceeds bloom object memory limit`. Line 356: `(empty array)`.

### F6 [nit] MISSING: `bf.bloom-defrag-enabled` is not documented (`topics/bloomfilters.md:179`)
**Doc says:** The defaults table and the config section list six configs. `bf.bloom-defrag-enabled` is not among them.
**Problem:** `CONFIG GET bf.*` returns seven configs, and the page shows defrag metrics (lines 320 to 324) without mentioning the config that turns defrag on or off.
**Evidence:** `CONFIG GET bf.*` output includes `bf.bloom-defrag-enabled` `yes`. configs.rs:31 `BLOOM_DEFRAG_DEFAULT: bool = true`.
**Fix:** Under "Bloom filter defrag metrics" add: "Defrag of bloom objects is controlled by `bf.bloom-defrag-enabled` (default `yes`)."

### F7 [nit] CONFUSING: the ad example mixes purchased products with ads shown (`topics/bloomfilters.md:30`)
**Doc says:**
> use a Bloom filter to store all the products they have purchased... If the product is already in the filter, it means the ad has already been shown to the user
**Problem:** The filter stores purchases, but the bullets read it as a record of ads shown.
**Fix:** "For each user, use a Bloom filter to store the ads already shown to them. Before showing an ad, check the filter. If the ad is absent, show it and add it to the filter. If it is present, pick a different ad."

### F8 [nit] CONFUSING: garbled memory comparison (`topics/bloomfilters.md:155`)
**Doc says:**
> Also, non scaling filters in general use less memory for a scaling filter that has scaled out several times to hold the same capacity.
**Fix:** "For the same total capacity, a non-scaling filter also uses less memory than a scaling filter that has scaled out several times."

### F9 [nit] STYLE: config names in two different cases (`topics/bloomfilters.md:181`)
**Doc says:** `BF.BLOOM-CAPACITY` (lines 181 to 186, 332) and `bf.bloom-capacity` (lines 219 to 251).
**Problem:** `CONFIG GET` returns the names in lowercase.
**Fix:** Use lowercase `bf.bloom-*` everywhere.

### F10 [nit] CONFUSING: the memory limit is explained twice on the page (`topics/bloomfilters.md:332`)
**Doc says:** Lines 245 to 265 ("Memory usage limit") and lines 330 to 332 ("Handling Large Bloom Filters", item 1) both describe `bf.bloom-memory-usage-limit`, its 128 MB default, and what happens when it is hit.
**Fix:** Cut lines 330 to 332 down to "1. Memory usage: see [Memory usage limit](#memory-usage-limit)." Keep the 245 to 265 section as the one full explanation.

### F11 [nit] AI-NATIVE: front matter description and opening paragraph are weak (`topics/bloomfilters.md:4`)
**Doc says:**
> Introduction to Bloom Filters
**Problem:** The description says nothing specific. The first paragraph does not say what the page covers.
**Fix:** Description: "Use Bloom filters in Valkey with the valkey-bloom module: commands, scaling behavior, default configs, memory limits, and metrics." At the end of line 7 add: "This page covers use cases, a CLI walkthrough, scaling vs. non-scaling filters, default properties and configs, memory limits, and monitoring."

### F12 [nit] STYLE: typos and grammar errors
**Doc says:** Line 90 "Check if as single user", line 153 "using a scaling bloom filters", line 328 "validations bloom filters faces", line 346 "to the reach the specified capacity without hitting either limits", line 147 sentence has no final period.
**Fix:** "Check if a single user", "using a scaling bloom filter", "validations bloom filters face", "to reach the specified capacity without hitting either limit", and add the period.
