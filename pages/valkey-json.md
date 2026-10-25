VERDICT: MINOR-ISSUES
COUNTS: blocker=0 minor=3 nit=6

# Valkey JSON (`valkey-json`)

https://valkey.io/topics/valkey-json/

Checked against valkey-json tag 1.0.3 (latest release), built from source and loaded into Valkey 9.1.2. Every path and filter example, the ACL table, both config defaults, the INFO fields, and the listed error prefixes match 1.0.3. The most important fix: the page says the module must be loaded but never says how, or which module version it describes.

## Findings

### F1 [minor] MISSING: No instructions for loading the module or which module version applies (`topics/valkey-json.md:7`)
**Doc says:**
> Users will need to load this module onto their valkey server in order to use this feature.
**Problem:** The page never says how to load the module, and it doesn't link to a page that does. It also doesn't say which valkey-json release it describes. A reader can't get from this page to a working `JSON.SET`.
**Evidence:** No `loadmodule` text appears anywhere on the page. With no module loaded, 9.1.2 has no JSON commands. To test this page I had to run `valkey-server --loadmodule .../libjson.so`.
**Fix:** After line 7, add: "Load the module at startup with `valkey-server --loadmodule /path/to/libjson.so`, or add `loadmodule /path/to/libjson.so` to `valkey.conf`. The [valkey-bundle](valkey-bundle.md) container ships with it preloaded. This page describes valkey-json 1.0.x."

### F2 [minor] MISSING: Config limits are buried in prose, with no unit and no load-time form (`topics/valkey-json.md:28`)
**Doc says:**
> You can set this limit using the `CONFIG SET json.max-document-size <value>` command.
> ...
> * Max Depth - ... You can adjust this limit using the following command: `CONFIG SET json.max-path-limit <value>`
**Problem:** The two configs are the page's main Valkey-specific facts, but they sit inside bullet prose. The page doesn't say that `max-document-size` is in bytes and accepts memory units, or what error each limit returns. It also doesn't mention that both configs can be set at load time. The "Max Depth" label doesn't match the config name `max-path-limit`, so a reader searching for either name may miss the other.
**Evidence:** `valkey-json@1.0.3:src/json/json.cc:2625-2629`:
```
REGISTER_NUMERIC_CONFIG(ctx, "max-document-size", DEFAULT_MAX_DOCUMENT_SIZE, VALKEYMODULE_CONFIG_MEMORY, 0, LLONG_MAX, ...
REGISTER_NUMERIC_CONFIG(ctx, "max-path-limit", DEFAULT_MAX_PATH_LIMIT, VALKEYMODULE_CONFIG_DEFAULT, 0, INT_MAX, ...
```
Live run, 9.1.2 + json 1.0.3:
```
CONFIG GET json.*            -> json.max-document-size 0, json.max-path-limit 128
JSON.SET (129 nested arrays) -> LIMIT Document path nesting limit is exceeded
max-document-size 100, 200-byte string -> LIMIT Document size limit is exceeded
```
**Fix:** Rename the section to "Configuration limits" and replace the bullets with a table: `json.max-document-size` | `0` (no limit) | bytes, memory units accepted | `LIMIT Document size limit is exceeded`; `json.max-path-limit` | `128` | maximum nesting depth of objects and arrays | `LIMIT Document path nesting limit is exceeded`. Add: "Set these at runtime with `CONFIG SET`, or in `valkey.conf` / on the `loadmodule` line." valkey-bundle.md:168-171 already shows the config-file form, so link to it.

### F3 [minor] MISSING: Error prefix table omits prefixes the module returns (`topics/valkey-json.md:228`)
**Doc says:**
> | `ERR`            | A general error. |
> ...
> | `WRONGTYPE`      | Wrong value type.
**Problem:** The table leaves out `OVERFLOW`, `EMPTYVAL`, `OPFAIL`, and `ERROR`. `ERROR` is a separate prefix from `ERR`. `OVERFLOW` is what `JSON.NUMINCRBY` and `JSON.NUMMULTBY` return, and client code that matches on prefixes will miss it.
**Evidence:** `valkey-json@58faf1c:src/json/util.cc:31-66`:
```
case JSONUTIL_FAILED_TO_DELETE_VALUE: return "OPFAIL Failed to delete JSON value";
case JSONUTIL_MULTIPLICATION_OVERFLOW: return "OVERFLOW Multiplication would overflow";
case JSONUTIL_EMPTY_JSON_OBJECT: return "EMPTYVAL Empty JSON object";
case JSONUTIL_ALLOCATION_FAILURE: return "ERROR Memory Allocation failed ";
```
Live run: `JSON.SET n . 1.7e308` then `JSON.NUMMULTBY n . 10` returns `OVERFLOW Multiplication would overflow`.
**Fix:** Add these rows: `OVERFLOW` | Numeric result would overflow (JSON.NUMINCRBY, JSON.NUMMULTBY). `EMPTYVAL` | Operation on an empty object or array. `OPFAIL` | Internal operation failed, such as deleting a value. `ERROR` | Internal error, such as allocation failure or invalid RDB data. Also close the last row with a trailing `|`.

### F4 [nit] BROKEN: ECMA-404 link points to RFC 7159 (`topics/valkey-json.md:9`)
**Doc says:**
> [ECMA-404](https://www.ietf.org/rfc/rfc7159.txt)
**Problem:** The ECMA-404 link goes to the IETF RFC. RFC 7159 has also been obsoleted by RFC 8259.
**Evidence:** The link target is the same URL as the RFC7159 link on that line.
**Fix:** `[RFC 8259](https://www.rfc-editor.org/rfc/rfc8259)` and `[ECMA-404](https://ecma-international.org/publications-and-standards/standards/ecma-404/)` (returns 200).

### F5 [nit] EXAMPLE: Restricted-syntax path `.address.city` does not exist in the sample document (`topics/valkey-json.md:220`)
**Doc says:**
> | `'.address.city'`                     | City name.                         |
**Problem:** Every other row in this table uses the store document. This one returns an error against it.
**Evidence:**
```
JSON.GET s '.address.city'
NONEXISTENT JSON path does not exist
```
**Fix:** `| '.store.bicycle.color' | The color of the bicycle. |` (returns `"red"`).

### F6 [nit] EXAMPLE: One reply is shown in a different format from the rest (`topics/valkey-json.md:187`)
**Doc says:**
> [{"price":15,"sold":false,"title":"abc"}]
**Problem:** Every other reply in this block uses interactive valkey-cli quoting, for example `"[{\"price\":5,...}]"`. This one is raw.
**Evidence:** `valkey-cli --no-raw JSON.GET k1 '$.books[?(@.title == "abc")]'` prints `"[{\"price\":15,\"sold\":false,\"title\":\"abc\"}]"`.
**Fix:** Replace line 187 with `"[{\"price\":15,\"sold\":false,\"title\":\"abc\"}]"`.

### F7 [nit] STYLE: Fenced code blocks have no language tag (`topics/valkey-json.md:107`, `topics/valkey-json.md:177`)
**Doc says:**
> ```
> { "store": {
**Problem:** Neither block has a language tag.
**Evidence:** Lines 107 and 177 both open with a bare fence.
**Fix:** Use ```` ```json ```` at line 107 and ```` ```text ```` at line 177. Also fix line 105, which says "example XML data" when the data is JSON.

### F8 [nit] AI-NATIVE: Weak description, vague headings, and filler that overclaims (`topics/valkey-json.md:4`, `topics/valkey-json.md:11`, `topics/valkey-json.md:24`)
**Doc says:**
> description: Introduction to Valkey JSON
> ## Example Valkey JSON Commands
> Its key benefits include fast search and filtering capabilities. ... making it an ideal choice for applications that require dynamic and flexible data storage.
**Problem:** The description tells the reader nothing. Line 24 is generic marketing copy. "Fast search" also misleads: JSONPath filters evaluate inside a single document. Indexed search across keys is valkey-search (topics/search.md).
**Fix:** Description: "Store, query, and update JSON documents in Valkey with the valkey-json module: JSONPath syntax, config limits, ACL categories, and error prefixes." Rename the heading to "Core JSON commands". Replace line 24 with: "JSONPath queries and in-place updates operate on a single document. To index and search across many JSON keys, use [valkey-search](search.md)."

### F9 [nit] STYLE: Garbled ACL sentence and first-person editorial aside (`topics/valkey-json.md:44`, `topics/valkey-json.md:76`)
**Doc says:**
> For each of these categories and a row for each command, if the cell contains a “y” then that command is added into that category.
> We've reordered and modified the descriptions in the table for clarity.
**Problem:** Line 44 is hard to parse. Line 76 is an aside about how the doc was edited, not information about the module.
**Fix:** Line 44: "A `y` means the command belongs to that ACL category." Delete the second sentence of line 76.
