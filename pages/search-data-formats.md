VERDICT: NEEDS-WORK
COUNTS: blocker=0 minor=6 nit=4

# Valkey Search - Data Formats (`search-data-formats`)

https://valkey.io/topics/search-data-formats/

The page is detailed and mostly matches valkey-search main (c672fd7, post 1.3.0-rc1), but several of its stated facts are wrong: the English stop word list leaves out `the`, the tag separator "valid characters" list is a restriction the code does not enforce, and rejected values are said not to count as errors. Fix the stop word list first, since the page's own examples rely on `the` being a stop word.

## Findings

### F1 [minor] INACCURATE: Default English stop word list omits `the` (`topics/search-data-formats.md:371`)
**Doc says:**
> a, an, and, are, as, at, be, but, by, for, if, in, into, is, it, no, not, of, on, or, such, that, their, then, there, these, they, this, to, was, will, with
**Problem:** The source list has 33 words and includes `the`. The page lists 32. The examples at lines 382, 413-421, and 425-435 treat `the` as a stop word, so the page contradicts itself.
**Evidence:** valkey-search@c672fd7:src/indexes/text/languages/english.h:26-31
```
// English stop words (33 words) — sourced from Apache Lucene.
inline const std::vector<std::string> kEnglishStopWords{
    "a",    "is",   "the", "an",   "and",  "are",   "as",   "at",    "be",
```
**Fix:** Insert `the` between `that` and `their`: "... such, that, the, their, then, ...". Make the same fix in commands/ft.create.md:57.

### F2 [minor] INACCURATE: Tag SEPARATOR is not limited to the listed characters (`topics/search-data-formats.md:20`)
**Doc says:**
> The valid separator characters are: `,.<>{}[]"':;!@#$%^&*()-+=~`
**Problem:** The parser only checks that the separator is a single character. The page's own examples (lines 17 and 59) use `|`, which is not in the list, and the module tests use `|` too.
**Evidence:** valkey-search@c672fd7:src/commands/ft_create_parser.cc:462-467
```
  if (parameters.separator.length() != 1) {
    return absl::InvalidArgumentError(
        absl::StrCat("The separator must be a single character, but got `",
                     parameters.separator, "`"));
  }
  tag_index_proto->set_separator(parameters.separator);
```
**Fix:** "The separator must be exactly one character (one byte). The default is `,`." Remove the character list here and in commands/ft.create.md:71.

### F3 [minor] INACCURATE: Rejected values are counted in `hash_indexing_failures` (`topics/search-data-formats.md:85`)
**Doc says:**
> When a value is rejected, the key is not indexed for that field. This is not treated as an error; the key simply does not appear in numeric range queries.
**Problem:** For any field type, a rejected value (`kInvalidData`, for example NaN, a non-numeric string, or a vector blob of the wrong size) or an empty one (`kMissing`, for example a tag value with no tags) increments the skipped counter, and `FT.INFO` reports that counter as `hash_indexing_failures`. The page names this counter only for invalid UTF-8 text (line 361), so a reader will not know that numeric, tag, and vector rejections show up there too.
**Evidence:** valkey-search@c672fd7:src/index_schema.cc:568-570 and 1368-1370
```
// Overload for Add/Modify, which return a RecordResult. kAdded counts as a
// success; kMissing and kInvalidData both count as skipped (the field produced
// nothing to index).
...
  ValkeyModule_ReplyWithSimpleString(ctx, "hash_indexing_failures");
  ValkeyModule_ReplyWithString(...  stats_.subscription_add.skipped_cnt ...
```
**Fix:** Replace the sentence with: "The write itself succeeds and no error is returned to the client. The key is left out of that field's index, and the rejection is counted in the `hash_indexing_failures` field of `FT.INFO`." Add the same note to the vector size-mismatch paragraph (lines 224-227).

### F4 [minor] MISSING: 1.3 features have no version stated (`topics/search-data-formats.md:390`)
**Doc says:**
> Supported: `ENGLISH` (default), `ARABIC`, `DUTCH`, `FRENCH`, `GERMAN`, `INDONESIAN`, `ITALIAN`, `PORTUGUESE`, `RUSSIAN`, `SPANISH`, `SWEDISH`, `TURKISH`.
**Problem:** Release 1.2.1 accepts only `ENGLISH`. The other languages, Unicode whitespace splitting (line 332), per-language punctuation (line 340), and `FLOAT16`/`BFLOAT16` (lines 119-159) first appear in 1.3.0, which exists only as 1.3.0-rc1. Lines 147-151 give 1.3.0 as the index's minimum version but never say that 1.3.0 is when the types were introduced. commands/ft.create.md:59 already gates the punctuation change with "before 1.3".
**Evidence:** `git -C ~/git/valkey-search grep -n 'kLanguageByStr' 1.2.1 -- src/commands/ft_create_parser.cc`
```
1.2.1:src/commands/ft_create_parser.cc:218:    kLanguageByStr({{"ENGLISH", data_model::LANGUAGE_ENGLISH}});
```
`git ls-tree 1.2.1 src/indexes/text/languages/` returns nothing. At 1.3.0-rc1, `"FRENCH"` and `"ARABIC"` are at ft_create_parser.cc:226 and :236.
**Fix:** Add "Requires valkey-search 1.3.0 or later; earlier versions support only `ENGLISH`." after the language list. Add "Available from valkey-search 1.3.0." under "Supported Data Types" and at the start of the Unicode whitespace sentence.

### F5 [minor] MISSING: Tag case-insensitivity folds ASCII only (`topics/search-data-formats.md:38`)
**Doc says:**
> When `CASESENSITIVE` is not set, tags like `"Blue"` and `"blue"` are treated as equivalent during search.
**Problem:** Tag normalization lowercases byte by byte with `ascii_tolower`, so `"Ä"` and `"ä"` remain distinct tags. Text fields use ICU case folding (line 365), and a reader will expect tags to behave the same way.
**Evidence:** valkey-search@c672fd7:src/indexes/tag.cc:108-116
```
std::string Tag::Normalize(absl::string_view tag) const {
  if (case_sensitive_) {
    return std::string(tag);
  }
  std::string out(tag);
  for (auto &c : out) {
    c = absl::ascii_tolower(static_cast<unsigned char>(c));
```
**Fix:** Append: "Only ASCII letters are folded. Non-ASCII characters such as `Ä` and `ä` are always compared exactly."

### F6 [minor] DUPLICATE: Stop word list, punctuation set, and separator list are duplicated in FT.CREATE (`topics/search-data-formats.md:20`)
**Doc says:** Line 20 (separator list), lines 336-338 (default punctuation), line 371 (stop words).
**Problem:** commands/ft.create.md:57, :59, and :71 carry the same three lists word for word, and both pages have the errors described in F1 and F2. Two copies drift apart over time.
**Evidence:** commands/ft.create.md:71: "`SEPARATOR <sep>` (optional): One of these characters `,.<>{}[]\"':;!@#$%^&*()-+=~` used to delimit individual tags." commands/ft.create.md:57 has the same 32-word stop word list.
**Fix:** Make this page canonical for the default lists, since it explains the processing. In ft.create.md, state the defaults in one line and link to `../topics/search-data-formats.md#stop-word-removal` and `#lexical-analysis`.

### F7 [nit] CONFUSING: "tracked but not indexed" contradicts "untracked" (`topics/search-data-formats.md:28`)
**Doc says:**
> If the field value produces no valid tags after processing, the key is tracked but not indexed for that field.
**Problem:** Line 54 says "key is untracked". The code adds the key to `untracked_keys_` (valkey-search@c672fd7:src/indexes/tag.cc:147-151).
**Fix:** "If the field value produces no valid tags after processing, the key is not indexed for that field."

### F8 [nit] CONFUSING: BFLOAT16 described as truncation right after rounding is required (`topics/search-data-formats.md:177`)
**Doc says:**
> which is what makes the conversion a truncation of the top two bytes rather than a change of layout.
**Problem:** Lines 203-204 and the Python code round to nearest-even before keeping the high half. "Truncation" suggests that dropping the low bytes is enough.
**Fix:** "...which is why the conversion keeps the top two bytes (after rounding) rather than changing the layout."

### F9 [nit] STYLE: Fenced blocks without a language tag (`topics/search-data-formats.md:16`)
**Doc says:** Bare ``` fences at lines 16, 34, 48, 61, 97, 141, 170, 236, 278, 287, 295, 303, 309, 336, 375.
**Problem:** The blocks get no highlighting, and readers cannot tell commands from pseudo-output.
**Fix:** Tag command blocks `bash` (or `text` for the `-->` annotated listings, the byte layout at 170, and the error at 278).

### F10 [nit] STYLE: Section headings are H1 under the page-title H1 (`topics/search-data-formats.md:8`)
**Doc says:** `# Tag Fields`, `# Numeric Fields` (line 66), `# Vector Fields` (line 109), `# Text Fields` (line 315).
**Problem:** The rendered page has five `<h1>` elements, including the site's page title.
**Fix:** Demote each heading by one level (`#` to `##`, `##` to `###`, `###` to `####`).

## Unverified
- topics/search-data-formats.md:382: removed stop words take no position. Did not find the position-assignment code in the timebox. snowball_language.cc:128 filters stop words, but I did not trace how positions are assigned. No module build was available to run it.
- topics/search-data-formats.md:231-244: the claim that the native JSON array and string forms are equivalent, and that the string is "the older form". The 1.2.1 and main parsers both strip `[`/`"`, but I did not confirm when native array ingestion was added.
