VERDICT: MINOR-ISSUES
COUNTS: blocker=0 minor=4 nit=6

# Documentation entry pages (`site-pages`)

https://valkey.io/docs/

The download artifacts, checksums, and Docker tags for 9.1.2 are correct and resolve. The main gaps: /docs/ gives a new reader no path to installing or starting Valkey, and the download page shows wrong release dates for older lines and hides two supported release lines.

## Findings

### F1 [minor] MISSING: /docs/ has no getting-started entry and no intro (`content/docs/index.md:10`)
**Doc says:**
> * [Command Reference](/commands/) A categorized listing of all Valkey commands
> * [Documentation by topic](/topics/) In-depth documentation covering a wide variety of operational and usage subjects
> * [Client Libraries](/clients/) ...
> * [Valkey Admin](http://valkey-admin.valkey.io/) ...
**Problem:** The documentation landing page lists only reference indexes. A new reader cannot reach introduction, quickstart, or installation without guessing that they sit inside /topics/. The page also has no sentence saying what it covers or who it is for.
**Evidence:** `topics/introduction.md`, `topics/quickstart.md`, and `topics/installation.md` exist in valkey-doc@a95bf25c. None are linked from content/docs/index.md:10-13 or anywhere in the rendered /docs/ body (checked all hrefs in the live HTML).
**Fix:** Add a one-line intro and put getting-started entries first:
```markdown
Start here to install Valkey, run your first commands, and find reference material.

* [Introduction](/topics/introduction/) What Valkey is and what it is used for.
* [Quick start](/topics/quickstart/) Install Valkey, start a server, and run your first commands.
* [Installation](/topics/installation/) Packages, containers, binaries, and building from source.
* [Command Reference](/commands/) ...
```

### F2 [minor] INACCURATE: Download page shows the newest release's date for every older line (`templates/download.html:43`)
**Doc says (live /download/):**
> Version 7.x.x
> Release Date: 2026-09-01
> GitHub Release: 7.2.14
**Problem:** 7.2.14 was released 2026-07-21. The loop for older releases resets `release` but not `release_date`, so every past-release block reuses the date of 9.1.2. 8.1.10 shows the right date only by coincidence.
**Evidence:** templates/download.html:19 `{% set release_date = most_recent_release_page[0].date %}` is set once. Lines 41-45 set `major` and `release` only. content/download/releases/v7-2-14.md:3 `date: 2026-07-21`.
**Fix:** In templates/download.html after line 43 add `{% set release_date = older_release.date %}`.
Tracked: valkey-io/valkey-io.github.io#662 (rewrites download.html and includes/release.html; confirm the rewrite sets the date per release)

### F3 [minor] MISSING: Download page hides supported 9.0 and 8.0 release lines (`templates/download.html:38`)
**Doc says:**
> Latest supported version of past releases
**Problem:** The page groups by major version, so only 8.1.10 and 7.2.14 appear. 9.0.6 and 8.0.11 are missing even though both lines are supported and both just shipped security fixes. A 9.0 or 8.0 user who reads "latest supported version" concludes their line has no current patch or is unsupported.
**Evidence:** templates/download.html:39-42 compares only `nth(n=0)` of the tag. valkey-doc topics/releases.md:100 `| 9.0 | 2025-10-21 | 2028-10-21 | 2028-10-21 |` and :102 `| 8.0 | 2024-09-15 | 2027-09-15 | 2027-09-15 |`. content/download/releases/v9-0-6.md and v8-0-11.md exist.
**Fix:** Group by `major.minor` instead of major (compare the first two split parts), and change the heading to "Latest version of each supported release line". Link to /topics/releases/#list-of-supported-versions.
Tracked: valkey-io/valkey-io.github.io#662

### F4 [minor] OUTDATED: Try Valkey runs 9.0.4 and does not say so (`content/try-valkey/_index.md:5`)
**Doc says:**
> state_file = "https://download.valkey.io/try-me-valkey/9.0.4/states/state.bin.gz"
**Problem:** The emulator ships 9.0.4, two minors and several security patches behind 9.1.2. The page text never states the version, so a reader trying a 9.1 feature from the docs gets an unknown-command or syntax error with no explanation. The page also gives no next step after the prompt loads.
**Evidence:** `curl -sI https://download.valkey.io/try-me-valkey/9.1.2/states/state.bin.gz` returns 200, `last-modified: Tue, 01 Sep 2026`, and `.../9.1.2/fs/alpine-fs.json` returns 200, so a 9.1.2 image appears to be published but is not wired up.
**Fix:** Point the three front matter URLs at 9.1.2 after confirming the image boots. Add body text to content/try-valkey/_index.md: "This runs Valkey 9.1.2 in your browser. New to Valkey? Follow the [quick start](/topics/quickstart/) commands here."
Tracked: valkey-io/valkey-io.github.io#526 (guidance only, not the version)

### F5 [nit] STYLE: Broken sentence and reversed repo name (`content/connect.md:12`)
**Doc says:**
> If you want to suggest a **new feature** for Valkey, you feature request issue on [valkey/valkey-io](...)
**Problem:** Missing verb, and the link text names a repo that does not exist; the link goes to valkey-io/valkey.
**Fix:** "If you want to suggest a **new feature** for Valkey, file a feature request on [valkey-io/valkey](...)."

### F6 [nit] STYLE: Outdated and inconsistent platform names (`content/connect.md:13`)
**Doc says:**
> connect with other developers on our [slack](https://valkey.io/slack)
> * [Valkey on Twitter](https://twitter.com/valkey_io)
**Problem:** "slack" is lowercase. The site footer on the same page links `https://x.com/valkey_io`; the body says Twitter. The footer also links YouTube, which the "social media" list omits.
**Fix:** "[Slack](https://valkey.io/slack)", "[Valkey on X](https://x.com/valkey_io)", and add "[Valkey on YouTube](https://www.youtube.com/@valkeyproject)" (line 16).

### F7 [nit] STYLE: Grammar errors in performance methodology (`_data/perf.toml:22`)
**Doc says:**
> generated using the valkey-perf-benchmark tool an AWS `c8g.metal-48xl` instance
> It helps compare key releases side by side, highlight performance gains from new features.
**Fix:** "tool on an AWS `c8g.metal-48xl` instance" (line 22). "It compares key releases side by side and highlights performance gains from new features." (line 21).

### F8 [nit] STYLE: No meta description on any entry page (`content/docs/index.md:1`)
**Problem:** None of /docs/, /connect/, /try-valkey/, /performance/, /download/ emit `<meta name="description">` or `og:description` (grepped live HTML). Front matter has only `title`. Same for content/connect.md:1, content/try-valkey/_index.md:1, content/performance.md:1, content/download/index.md:1.
**Fix:** Add `description` to each front matter, for example /download/: `description = "Download Valkey 9.1.2 and the latest patch of each supported release: binaries, checksums, and container images."`

### F9 [nit] STYLE: Stray `<title>` element inside page body (`templates/valkey-try-me.html:17`)
**Doc says:**
> `<title>Try Valkey</title>`
**Problem:** Because content/try-valkey/_index.md has no body, the template's fallback emits a second `<title>` inside `<main>` (live HTML line 134). Invalid HTML; the page already has an `<h1>`.
**Fix:** Delete templates/valkey-try-me.html:17, or move the fallback paragraphs into content/try-valkey/_index.md and drop the `{% else %}` branch.

### F10 [nit] BROKEN: Valkey Admin linked over plain HTTP (`content/docs/index.md:13`)
**Doc says:**
> [Valkey Admin](http://valkey-admin.valkey.io/)
**Problem:** Redirects to HTTPS. Link directly.
**Fix:** `https://valkey-admin.valkey.io/`
