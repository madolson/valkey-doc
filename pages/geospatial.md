VERDICT: MINOR-ISSUES
COUNTS: blocker=0 minor=3 nit=3

# Geospatial (`geospatial`)

https://valkey.io/topics/geospatial/

All three examples reproduce exactly on 9.1.2. The page is thin. It never says geo indexes are sorted sets or what the coordinate limits are, it leaves out that BYPOLYGON is 9.0+, and the "closest stations" example returns unsorted results. Fix the example first: add `ASC`.

## Findings

### F1 [minor] EXAMPLE: "Closest stations" example returns results unsorted (`topics/geospatial.md:34`)
**Doc says:**
> Suppose you're building a mobile app that lets you find all of the bike rental stations closest to your current location.
> `GEOSEARCH bikes:rentable FROMLONLAT -122.2612767 37.7936847 BYRADIUS 5 km WITHDIST`
**Problem:** Without `ASC`, GEOSEARCH does not sort by distance. The output lists station:1 (1.8523) before the closer station:2 (1.4979). A reader copying this into a "closest" feature gets the wrong order.
**Evidence:** 9.1.2 run with `ASC` added:
```text
station:2
1.4979
station:1
1.8523
station:3
2.2441
```
**Fix:** Change the command to `GEOSEARCH bikes:rentable FROMLONLAT -122.2612767 37.7936847 BYRADIUS 5 km ASC WITHDIST`, update the output to the order above, and change the lead-in at line 32 to "...and return the distance to each location, nearest first:".

### F2 [minor] MISSING: BYPOLYGON shown with no version requirement (`topics/geospatial.md:8`)
**Doc says:**
> This data structure is useful for finding nearby points within a given radius, bounding box, or polygon shaped area.
**Problem:** BYPOLYGON arrived in 9.0.0. Lines 8, 13, and 43 to 50 present it as if every version has it. On 8.x the same command fails with a syntax error.
**Evidence:** valkey@9.1.2:src/commands/geosearch.json:14-17:
```text
"9.0.0",
"Added support for the BYPOLYGON option."
```
**Fix:** At line 43, add: "`BYPOLYGON` requires Valkey 9.0 or later. The polygon needs at least 3 vertices, given as longitude/latitude pairs after the vertex count."

### F3 [minor] MISSING: No mention of the sorted-set storage or coordinate limits (`topics/geospatial.md:7`)
**Doc says:**
> Geospatial indexes let you store coordinates and search for them.
**Problem:** The page never says a geo index is a sorted set. That fact is what tells a reader how to delete members (`ZREM`, since there is no GEODEL), count them (`ZCARD`), and why `TYPE` returns `zset`. The page also leaves out the valid latitude range, so out-of-range inputs return an error the reader does not expect. Both facts are only in commands/geoadd.md:1-12.
**Evidence:** commands/geoadd.md:8: "Valid latitudes are from -85.05112878 to 85.05112878 degrees." commands/geoadd.md:12: "there is no **GEODEL** command because you can use [`ZREM`](zrem.md) to remove elements. The Geo index structure is just a sorted set."
**Fix:** After line 8, add: "A geospatial index is a [sorted set](sorted-sets.md) whose scores are 52-bit geohashes, so sorted set commands such as `ZREM` and `ZCARD` work on it. Longitudes range from -180 to 180 and latitudes from -85.05112878 to 85.05112878. See [`GEOADD`](../commands/geoadd.md) for details."

### F4 [nit] STYLE: Wording errors (`topics/geospatial.md:12`)
**Doc says:**
> `GEOADD` adds a location to a given geospatial index
> `GEOSEARCH` returns locations with a given radius
> Searching for all stations in a custom defined area or an irregularly shaped area by using the `BYPOLYGON` search option.
**Problem:** GEOADD takes one or more members. Line 13 says "with" where it means "within". Line 43 is a sentence fragment. Line 47 has a double space in `BYPOLYGON  5`.
**Fix:** Line 12: "adds one or more locations". Line 13: "within a given radius". Line 43: "Use the `BYPOLYGON` option to search an irregularly shaped area." Line 47: `BYPOLYGON 5`.

### F5 [nit] STYLE: Code fences have no language tag (`topics/geospatial.md:23`)
**Doc says:**
> ```` ``` ```` (lines 23, 33, 46)
**Problem:** None of the three fenced blocks has a language tag.
**Fix:** Tag all three as ` ```text `.

### F6 [nit] STYLE: Command names are not linked to their reference pages (`topics/geospatial.md:12`)
**Doc says:**
> * `GEOADD` adds a location ...
> * `GEOSEARCH` returns locations ...
**Problem:** These are the only two commands the page covers, and neither links to its reference page. Readers need those pages for the options (COUNT, ASC/DESC, WITHCOORD, BYBOX, FROMMEMBER).
**Fix:** Use [`GEOADD`](../commands/geoadd.md) and [`GEOSEARCH`](../commands/geosearch.md), and mention [`GEOSEARCHSTORE`](../commands/geosearchstore.md) for saving results.
