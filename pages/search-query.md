VERDICT: NEEDS-WORK
COUNTS: blocker=2 minor=5 nit=2

# Valkey Search - Queries (`search-query`)

https://valkey.io/topics/search-query/

The page documents valkey-search 1.3 features (VECTOR_RANGE, HYBRID_POLICY, FLOAT16/BFLOAT16) with no version note, but the released valkey-bundle for Valkey 9.1.2 ships search 1.2.1, where all of them are errors. The tag prefix examples also fail on the default config. Fix first: add a version note to the 1.3-only sections, and make every example run as written.

Verified against valkey-search tags 1.2.1 (788e8c9, latest GA) and 1.3.0-rc1 (e31273a), main at c672fd7, and a live `valkey/valkey-bundle:latest` container (valkey_version 9.1.2, `MODULE LIST` search ver 66049 = 1.2.1).

## Findings

### F1 [blocker] INACCURATE: 1.3-only query features documented with no version gate (`topics/search-query.md:10`)
**Doc says:**
> A non-vector query can also select keys by vector distance, with the [vector range matcher](#vector-range-match).

The same applies to lines 25, 42 (`FLOAT16` or `BFLOAT16`), 44-62 (`HYBRID_POLICY`), 164-205 (Vector Range Match), and 405-451 (Example Vector Range Queries).
**Problem:** None of these exist in valkey-search 1.2.1, the latest GA release and the version bundled with Valkey 9.1.2. They first appear in 1.3.0-rc1. The page never says so, so a reader on the current release gets parse errors.
**Evidence:** `git grep -l <pattern> 1.2.1 -- src` returns 0 files for `VECTOR_RANGE`, `HYBRID_POLICY`, `ADHOC_BF`, `BFLOAT16`, `yield_distance_as`. On 1.3.0-rc1, `src/commands/filter_parser.cc:356` parses `$yield_distance_as`. Live run on bundle 9.1.2 / search 1.2.1:
```
> FT.SEARCH idx "@v:[VECTOR_RANGE 5 $vec]=>{$YIELD_DISTANCE_AS: dist}" PARAMS 2 vec "\x00..." SORTBY dist RETURN 1 dist DIALECT 2
Invalid filter expression: `@v:[VECTOR_RANGE 5 $vec]`. `v` is not indexed as a numeric field
> FT.SEARCH idx "@category:{shoes}=>[KNN 10 @v $vec HYBRID_POLICY ADHOC_BF]" PARAMS 2 vec "\x00..." DIALECT 2
Error parsing vector similarity parameters: `[KNN 10 @v $vec HYBRID_POLICY ADHOC_BF]`. Unexpected argument `HYBRID_POLICY`
> FT.CREATE idx16 SCHEMA v VECTOR HNSW 6 TYPE FLOAT16 DIM 2 DISTANCE_METRIC L2
Invalid field type for field `v`: Error parsing value for the parameter `TYPE` - Unknown argument `FLOAT16`
```
**Fix:** Add a sentence at the top of each affected section, for example: "Available in valkey-search 1.3.0 and later. On 1.2.x the parser rejects `VECTOR_RANGE`." Use the same wording for `HYBRID_POLICY` and the 16-bit types on line 42. search-data-formats.md:148-151 already gives a 1.3.0 note for FLOAT16, so match that wording.

### F2 [blocker] EXAMPLE: Prefix and complex tag examples fail, and the complex output is wrong (`topics/search-query.md:348`)
**Doc says:**
> valkey-cli FT.SEARCH index @color:{b\*} RETURN 1 color

and line 367:
> valkey-cli FT.SEARCH index @color:{b*|green} RETURN 1 color
> 1) (integer) 2
> 2) "key3" ... 4) "key5"
**Problem:** The default `search.tag-min-prefix-length` is 2, so `b*` (one character before `*`) is rejected. In a shell, `b\*` reaches the server as `b*`. On line 367 the `|` is unquoted, so the shell pipes into a command named `green}`. Even with a valid prefix the documented result is wrong: `b*|green` must match key1, key2, key3, key4 and key5, not 2 keys. The prose at line 95 never mentions the minimum prefix length.
**Evidence:** valkey-search@1.2.1:src/indexes/tag.cc:99-102 (unchanged on main at tag.cc:198-201):
```
      // Prefix tags shorter than min length are rejected.
      if (tag.length() <= min_prefix_length) {
        return absl::InvalidArgumentError(absl::StrCat(
            "Tag string `", tag, "` is too short for prefix wildcard."));
```
valkey-search@main:src/valkey_search_options.cc:442 `constexpr uint32_t kDefaultTagMinPrefixLength{2};`. Live:
```
> CONFIG GET search.tag-min-prefix-length
search.tag-min-prefix-length
2
> FT.SEARCH index '@color:{b*}' RETURN 1 color
Invalid filter expression: `@color:{b*}`. Tag string `b*` is too short for prefix wildcard.
> FT.SEARCH index '@color:{b*|green}' RETURN 1 color
Invalid filter expression: `@color:{b*|green}`. Tag string `b*` is too short for prefix wildcard.
> FT.SEARCH index '@color:{bl*|green}' RETURN 1 color
4  key2 black, key3 green, key5 beige,green, key1 blue
```
**Fix:** Use a two-character prefix and quote every query: `valkey-cli FT.SEARCH index "@color:{be*}" RETURN 1 color` (key4, key5) and `valkey-cli FT.SEARCH index "@color:{bl*|green}" RETURN 1 color`, then paste the real output (4 results: key1, key2, key3, key5). After line 95 add: "The prefix must be longer than `search.tag-min-prefix-length` characters (default 2), or the query is rejected."

### F3 [minor] EXAMPLE: Backslash in the negation example changes the result (`topics/search-query.md:275`)
**Doc says:**
> @genre: {comedy} \-@year:[2015 2024]
**Problem:** The `\-` is a markdown escape that ended up inside a code block. Typed as shown, the query returns nothing instead of the comedy books outside 2015-2024.
**Evidence:** Live, with d:1 (comedy, 2020), d:3 (comedy, 2010):
```
> FT.SEARCH t '@genre: {comedy} \-@year:[2015 2024]' NOCONTENT
0
> FT.SEARCH t '@genre: {comedy} -@year:[2015 2024]' NOCONTENT
1
d:3
```
**Fix:** `@genre:{comedy} -@year:[2015 2024]`. The same stray escapes appear in prose on line 129 (`\+inf, \-inf`). Write those as `` `+inf`, `-inf` ``.

### F4 [minor] INACCURATE: Suffix search default and error behavior misstated (`topics/search-query.md:236`)
**Doc says:**
> Note that suffix searching will only locate words in fields that have `WITHSUFFIXTRIE` specified, i.e., fields declared with `NOSUFFIXTRIE` will not be searched.
**Problem:** The page never says that `NOSUFFIXTRIE` is the default. A field-less `*hello` on an index where no text field has `WITHSUFFIXTRIE` returns an error, not an empty result. commands/ft.create.md:79 says "The default is `WITHSUFFIXTRIE`", which is wrong and contradicts this page.
**Evidence:** valkey-search@1.2.1:src/commands/ft_create_parser.cc:506 (main:560) `field_params.with_suffix_trie = false;`. Live, `FT.CREATE s PREFIX 1 s: SCHEMA t TEXT`:
```
> FT.SEARCH s '@t:*hello' NOCONTENT
Invalid filter expression: `@t:*hello`. Field does not support suffix search
> FT.SEARCH s '*hello' NOCONTENT
Invalid filter expression: `*hello`. No fields support suffix search
```
**Fix:** Replace with: "Suffix search requires `WITHSUFFIXTRIE` on the field. The default is `NOSUFFIXTRIE`. Without a field specifier, only `WITHSUFFIXTRIE` fields are searched, and the query fails with `No fields support suffix search` if there are none. With a field specifier on a `NOSUFFIXTRIE` field, the query fails with `Field does not support suffix search`." Fix commands/ft.create.md:79 to say the default is `NOSUFFIXTRIE`.

### F5 [minor] EXAMPLE: Fuzzy error text does not match the server (`topics/search-query.md:261`)
**Doc says:**
> %%hello%               Error: leading and trailing count % count are different
**Problem:** The server returns a different message, so a reader searching for the documented text finds nothing.
**Evidence:**
```
> FT.SEARCH t '%%hello%' NOCONTENT
Invalid filter expression: `%%hello%`. Invalid fuzzy '%' markers
```
**Fix:** `%%hello%               Error: Invalid fuzzy '%' markers (leading and trailing counts differ)`.

### F6 [minor] INACCURATE: Filter complexity claim of "no worse than O(log N)" (`topics/search-query.md:68`)
**Doc says:**
> Determining the computational complexity of a particular filter is difficult, but generally no worse than O(log N) and sometimes as fast as O(1).
**Problem:** A filter cannot be sub-linear when its result is linear in N. A negation returns every key that does not match (line 268), and a short prefix or `[-inf +inf]` range can match most of the index. Readers who size workloads from this sentence will underestimate cost.
**Evidence:** Live, `-@genre:{comedy}` returned every key in the index without genre `comedy` (d:2, d:4), including keys with no genre field. The module caps non-vector result fetching at `search.max-nonvector-search-results-fetched` (default 100000, valkey-search@main:src/valkey_search_options.cc:589-590) precisely because filters can produce large candidate sets.
**Fix:** Replace the sentence with: "Cost depends on how many keys each matcher selects. Selective tag, numeric, and term matchers use their indexes. Negations, short prefixes, and wide ranges can touch a large fraction of the index."

### F7 [minor] MISSING: Defaults of the configs that govern query behavior are not stated (`topics/search-query.md:255`)
**Doc says:**
> The maximum allowed edit distance is control by the configuration setting `search.fuzzy-max-distance`.
**Problem:** The default is not given here or for tag prefixes (see F2), so readers must guess at the limits. Line 199 does give a default for `search.max-nonvector-search-results-fetched`, so the page is inconsistent.
**Evidence:** Live `CONFIG GET search.fuzzy-max-distance` returns `3`. `%%%hello%%%` is accepted.
**Fix:** "The maximum edit distance is set by `search.fuzzy-max-distance` (default 3); see [configurables](search-configurables.md)." Add the tag prefix default as in F2.

### F8 [nit] CONFUSING: Text operator combination sentence contradicts OR (`topics/search-query.md:209`)
**Doc says:**
> Unlike the other search operators. The text search operators do not require that a field be specified. ... Regardless, if multiple text search operators are combined in an expression, then only keys which have all of the text search operators will satisfy the query.
**Problem:** This is only true for AND. `hello | world` matches keys with either word. The sentence also starts with a fragment.
**Fix:** "Unlike other operators, text search operators do not require a field. Without one, all text fields in the index are searched. Text operators combine with `AND`, `OR`, and negation like any other matcher."

### F9 [nit] STYLE: Typos and missing code-block language tags (`topics/search-query.md:68`)
**Doc says:**
> It is not the case that the of filtering terminology implies an O(N) scan

Also line 95 "If a match string end with", line 255 "is control by", line 255 "damerau-levenshtein" (proper noun: Damerau-Levenshtein), and blank padding lines inside the code blocks at 383-387 and 391-395. None of the fenced blocks on the page has a language tag (lines 16, 24, 32, 72, 98, 108, 114, 120, 134, 158, 170, 218, 227, 239, 248, 257, 274, 284, 296, 312, 324, 334, 347, 366, 383, 391, 400, 409, 418, 431, 440).
**Problem:** Typos, and code blocks with no syntax tag.
**Fix:** "It is not the case that filtering implies an O(N) scan", "ends with", "is controlled by", "Damerau-Levenshtein". Tag CLI blocks as `bash` and query-only and BNF blocks as `text`. Remove the blank lines inside code blocks.

## Unverified
- topics/search-query.md:177-203: vector range semantics (IP/COSINE radius behavior, NaN/inf handling, OR-branch distance, WITHSCORES = 0, HNSW approximation via `search.max-nonvector-search-results-fetched`). The parser behavior matches valkey-search@main:src/commands/filter_parser.cc:354-534, including the `$epsilon` HNSW-only and >0 rule. I could not run these because no 1.3 build or image was available, and building the module was out of timebox.
- topics/search-query.md:405-451: vector range example output. Not run, for the same reason. The data setup was confirmed: KNN on the same data gave `__v_score` 1, 4, 25 for p1, p2, p3 on search 1.2.1.
- topics/search-query.md:246: exact phrase "not subject to stop word removal nor stemming". Not checked in source.
