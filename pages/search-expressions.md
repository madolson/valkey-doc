VERDICT: NEEDS-WORK
COUNTS: blocker=3 minor=5 nit=4

# Valkey Search - Aggregation Expressions (`search-expressions`)

https://valkey.io/topics/search-expressions/

The grammar, the operator table, and the timestamp table each contradict the parser and function code in valkey-search 1.2.1 (latest release tag) and on the default branch (c672fd7). Fix the precedence grammar first: as written, `+` binds tighter than `*`. Checked from source and unit tests only; no module build was run.

## Findings

### F1 [blocker] INACCURATE: Grammar inverts arithmetic precedence and misstates logical/power precedence (`topics/search-expressions.md:21`)
**Doc says:**
> `<relational> ::= <multiplicative> | <relational> <rel-op> <multiplicative>` ... `<multiplicative>::= <additive> | <multiplicative> <mul-op> <additive>` ... `<additive> ::= <unary> | <additive> <add-op> <unary>`

**Problem:** In this grammar `+`/`-` bind tighter than `*`/`/`, so `@a + @b * 2` would parse as `(@a+@b)*2`. The parser does the reverse, as the "C" claim at line 10 implies, so the page contradicts itself. The grammar also puts `&&` above `||` and `^` level with `*`. In 1.2.1, `&&` and `||` share one level and `^` is a left-associative peer of `*` and `/`. On the default branch (unreleased 1.3.0), `&&` binds tighter than `||` and `^` is a separate, higher, right-associative level. Both changes are gated by `search.emulate-release`.
**Evidence:** valkey-search@1.2.1:src/expr/expr.cc:504-524
```
LorOp: ops {"||", &FuncLor}, {"&&", &FuncLand}  -> DoDyadic(ctx, &Compiler::CmpOp, ops)
CmpOp  -> DoDyadic(ctx, &Compiler::AddOp, ops)
AddOp: {"+"},{"-"} -> DoDyadic(ctx, &Compiler::MulOp, ops)
MulOp: {"*"},{"/"},{"^"} -> DoDyadic(ctx, &Compiler::Primary, ops)
```
valkey-search@c672fd7:src/expr/expr.cc (comment above LorOp):
```
//  PowOp:   ^                 (Redisearch-compatible since 1.3.0)
//  MulOp:   * /
//  AddOp:   + -
//  CmpOp:   < <= == != > >=
//  LandOp:  &&                (Redisearch-compatible since 1.3.0)
//  LorOp:   ||
```
**Fix:** Rewrite the grammar so `<relational>` is built from `<additive>`, `<additive>` from `<multiplicative>`, and `<multiplicative>` from `<unary>`, with `<mul-op> ::= "*" | "/" | "^"` for 1.2.x. Add a note: "In valkey-search 1.3.0 and later, `^` binds tighter than `*` and `/` and is right-associative, and `&&` binds tighter than `||`. Set `search.emulate-release` to keep the 1.2 behavior."

### F2 [blocker] INACCURATE: `%` modulo operator does not exist (`topics/search-expressions.md:82`)
**Doc says:**
> `|   %    | Modulo         |` (also `<mul-op> ::= "*" | "/" | "^" | "%"` at line 29)

**Problem:** The parser has no `%` operator in any version, so an expression that uses it fails to compile.
**Evidence:** valkey-search@1.2.1:src/expr/expr.cc:522-523 `{"*", &FuncMul}, {"/", &FuncDiv}, {"^", &FuncPower}`. Default branch: `kFixedMulOps{{"*", &FuncMul}, {"/", &FuncDiv}}`, plus a separate `PowOp` for `^`. `git grep FuncMod` over src/expr returns nothing. Unconsumed input produces `"Extra characters at or near position "` (1.2.1 expr.cc:539).
**Fix:** Delete the `%` row and remove `"%"` from `<mul-op>`.

### F3 [blocker] INACCURATE: `minute`, `hour`, `day`, `month` return rounded timestamps, not field values (`topics/search-expressions.md:152`)
**Doc says:**
> `minute(timestamp) | Extract minute from timestamp. Range: 0-59`, `hour(timestamp) | ... Range: 0-23`, `day(timestamp) | ... Range: 1-31`, `month(timestamp) | ... Range: 0-11`

**Problem:** These four functions return a Unix timestamp truncated to the start of the minute, hour, day, or month. A reader who filters on `hour(@ts) == 13` gets no matches.
**Evidence:** valkey-search@c672fd7:testing/expr/value_test.cc:303-326
```
Value ts(double(1739565015));
EXPECT_EQ(FuncMinute(ts), Value(1739565000));
EXPECT_EQ(FuncHour(ts), Value(1739563200));
EXPECT_EQ(FuncDay(ts), Value(1739491200));
...
EXPECT_EQ(FuncMonth(ts), Value(1738368000))   // emulate-release >= 1.2.1: Feb 1 2025 00:00 UTC
```
valkey-search@1.2.1:src/expr/value.cc:543-568 `TIME_ROUND(FuncMonth, true, true, true) ... return Value(double(::mktime(&tm)));`. 1.2.1 sets `tm_mday = 0`, so `month()` returns the last day of the *previous* month at 00:00. The default branch fixes this under emulate-release >= 1.2.1.
**Fix:** Change the rows to: `minute(timestamp)`: timestamp rounded down to the start of the minute. `hour(timestamp)`: rounded down to the start of the hour. `day(timestamp)`: rounded down to 00:00 UTC of the day. `month(timestamp)`: 00:00 UTC on the first day of the month. Point readers to `hour`-of-day extraction via `timefmt(@ts, '%H')`.

### F4 [minor] INACCURATE: `substr` negative length and offset semantics are wrong (`topics/search-expressions.md:136`)
**Doc says:**
> `A length of -1 means the remainder of the string`

**Problem:** A negative length is added to the remaining length, so `-1` drops the last byte. `substr('hello', 0, -1)` returns `hell`. A negative offset counts from the end, which the page does not mention. Offsets and lengths are bytes, not characters.
**Evidence:** valkey-search@1.2.1:src/expr/value.cc:434-445 (same on c672fd7)
```
int64_t offset = *offset_p >= 0 ? *offset_p : *offset_p + ls.size();
...
int64_t len = (ls.size() - offset) + *length_p;
```
**Fix:** "The substring of s starting at byte `offset` for `length` bytes. A negative offset counts from the end of the string. A negative length stops that many bytes before the end, so -1 omits the last byte. Use a length at least `strlen(s)` to take the remainder."

### F5 [minor] INACCURATE: Logical operator truthiness is misdescribed (`topics/search-expressions.md:107`)
**Doc says:**
> The logical operators `&&`, `||` and `!` generate a numeric 0 if the input can be converted to a numeric 0, otherwise they generate a numeric 1.

**Problem:** `!` inverts, so it returns 1 for a 0 input. Strings are not converted to numbers. In 1.2.1 every string, including `"1"`, is false. On the default branch, non-empty strings are true (so `"0"` is true) and empty strings are false. Nil is false. NaN is true.
**Evidence:** valkey-search@1.2.1:src/expr/value.cc:81-91
```
} else if (auto result = std::get_if<double>(&value_)) {
  if (IsNan(*result)) { return true; }
  return !(*result == 0.0);
} else {
  return false;
```
valkey-search@c672fd7:src/expr/value.cc `asbool_string_truthy ... [&] { return !sv.empty(); }`.
**Fix:** "`&&` and `||` return 1 or 0. `!` returns 1 if its operand is false and 0 otherwise. A number is false when it equals 0. Nil is false. In 1.2.x every string is false. From 1.3.0, a non-empty string is true."

### F6 [minor] MISSING: `timefmt` and `parsetime` format argument is optional with a default (`topics/search-expressions.md:156`)
**Doc says:**
> `timefmt(timestamp, fmt)` ... `parsetime(time_string, fmt)`

**Problem:** Both functions accept one or two arguments. With one argument they use ISO 8601 `%FT%TZ`. The page shows `fmt` as required and gives no default.
**Evidence:** valkey-search@1.2.1:src/expr/expr.cc:221-222 `{"timefmt", {1, 2, &ProxyTimefmt}}, {"parsetime", {1, 2, &ProxyParsetime}}`. c672fd7 expr.cc ProxyTimefmt: `Value fmt("%FT%TZ"); if (params.size() > 1) { fmt = ... }`.
**Fix:** Change the syntax to `timefmt(timestamp [, fmt])` and `parsetime(time_string [, fmt])`, and add "`fmt` defaults to `%FT%TZ` (for example `2025-02-14T20:30:15Z`)."

### F7 [minor] INACCURATE: Bare `inf` is not a valid numeric constant (`topics/search-expressions.md:62`)
**Doc says:**
> plus _inf_, _+inf_ and _-inf_.

**Problem:** The parser hands only input starting with `+`, `-`, `.`, or a digit to the number parser. Input starting with `inf` goes to the function-call parser, which returns nothing because no `(` follows, and compilation fails. `+inf` and `-inf` work through `strtod`.
**Evidence:** valkey-search@1.2.1:src/expr/expr.cc:382-409: `case '+': case '-': case '.': case '0'...'9': return Number(ctx);` ... `default: return ParseFunctionCall(ctx);`. ParseFunctionCall: `if (!s_.SkipWhiteSpacePopByte('(')) { s_ = s; return nullptr; }`.
**Fix:** "plus `+inf` and `-inf`. Write positive infinity as `+inf`. A bare `inf` is not accepted."

### F8 [minor] MISSING: No module version stated, and behavior differs across versions (`topics/search-expressions.md:6`)
**Doc says:**
> The `FILTER`, `APPLY`, `SORTBY` and `GROUPBY` stages of `FT.AGGREGATE` use expressions to compute values.

**Problem:** Operator precedence, string truthiness, `lower`/`upper` on non-strings, pre-epoch date handling, and `month()` all differ between 1.2.1 and the default branch. Each change is gated by `search.emulate-release`. The page names no version and does not mention the config, and search-configurables.md does not document `emulate-release` either. The first paragraph also does not say who the page is for or link to `FT.AGGREGATE`.
**Evidence:** valkey-search@c672fd7:src/expr/value.cc comments: "1.2.1 fix: refuse non-string inputs (matches Redisearch — lower(0) → Nil)", "1.2.1 fix shared by all date functions: pre-epoch (negative) timestamps return Nil", and expr.cc "Both fixes are gated by search.emulate-release per COMPATIBILITY.md". `grep emulate-release topics/search-configurables.md` returns nothing.
**Fix:** Open with: "This page describes the expression language used by the `APPLY`, `FILTER`, `SORTBY`, and `GROUPBY ... REDUCE` stages of [`FT.AGGREGATE`](../commands/ft.aggregate.md), as of valkey-search 1.2." Add version notes to F1, F3, and F5, and link to the `search.emulate-release` config once it is documented. PR #444 adds array values (an unreleased feature) to this page and should carry the same version labeling.
Tracked: valkey-io/valkey-doc#444 (arrays only)

### F9 [nit] INACCURATE: Grammar `<unary>` and `<arg-list>` rules do not match the parser (`topics/search-expressions.md:37`)
**Doc says:**
> `<unary> ::= <primary> | <unary> <unary-op> <primary>` and `<arg-list> ::= | <primary> | <arg-list> "," <primary>`

**Problem:** The `<unary>` rule as written makes `!` a binary operator (`a ! b`). The parser treats `!` as prefix-only over a primary. Function arguments are full expressions, not primaries, so `concat(@a, @b + 1)` is valid.
**Evidence:** valkey-search@1.2.1:src/expr/expr.cc Invert: `CHECK(s_.PopByte('!')); ... Primary(ctx)`. ParseFunctionCall: `VMSDK_ASSIGN_OR_RETURN(auto param, ParseExpression(ctx));`.
**Fix:** `<unary> ::= <primary> | "!" <unary>` and `<arg-list> ::= | <expression> | <arg-list> "," <expression>`.

### F10 [nit] STYLE: `monthofyear` described as "day of month" (`topics/search-expressions.md:150`)
**Doc says:**
> `monthofyear(timestamp) | Extract day of month from timestamp. Range: 0-11`

**Fix:** "Extract month of year from timestamp (0 = January). Range: 0-11".

### F11 [nit] EXAMPLE: Grammar code fence has no language tag (`topics/search-expressions.md:12`)
**Doc says:**
> ```` ``` ```` (opening the BNF block)

**Fix:** Use ```` ```text ```` (or ```` ```ebnf ````).

### F12 [nit] AI-NATIVE: Weak front matter description and generic table lead-ins (`topics/search-expressions.md:3`)
**Doc says:**
> `description: Valkey Search Module Aggregation Expression Language` and, at line 127, `String processing functions. Inputs are converted to strings as appropriate.`; line 143 `Supported time functions include:`

**Problem:** The description restates the title. The lead-ins carry no information.
**Fix:** `description: Operators, type conversion rules, and functions for FT.AGGREGATE APPLY, FILTER, and SORTBY expressions in valkey-search.` Cut line 143. Replace line 127 with "String functions convert their inputs to strings. `upper` and `lower` change ASCII letters only."

## Unverified
- topics/search-expressions.md:89-94: The relational conversion rules match `Compare()` in value.cc. On the default branch, `FILTER` uses separate `FilterFunc*` comparators, and a missing field in `APPLY` drops the record (`MissingPropagates`). I did not determine whether these change the documented Nil rules in released 1.2.1, and no module build was run.
- topics/search-expressions.md:6: `SORTBY` and `GROUPBY` are listed as expression users. ft_aggregate_exec.cc:946/963 on the default branch compiles expressions for sort and group keys. I did not check whether 1.2.1 accepts arbitrary expressions there or only `@field`.
