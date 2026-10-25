VERDICT: MINOR-ISSUES
COUNTS: blocker=0 minor=2 nit=4

# History (`history`)

https://valkey.io/topics/history/

The page reads as current in March 2024 and has not been updated since. The most important fix is the licensing section: Redis added AGPLv3 as a license option in May 2025, so "The end of open source Redis" no longer holds without qualification.

## Findings

### F1 [minor] OUTDATED: Licensing section omits Redis 8 AGPLv3 relicense (`topics/history.md:138`)
**Doc says:**
> The end of open source Redis
> ...
> In 2024, Redis Ltd. changed the license of Redis from the open source BSD license to dual source-available licenses.
**Problem:** The heading and section present the 2024 license change as the final state. Since May 2025, Redis 8 is also offered under the OSI-approved AGPLv3. A reader comes away with an outdated picture of the Redis licensing situation.
**Evidence:** https://redis.io/blog/agplv3/ ("Redis is now available under the AGPLv3 open source license", datePublished 2025-05-01):
> Adding the OSI-approved AGPL as an additional licensing option for Redis, starting with Redis 8 (available now)
**Fix:** Rename the heading to "The 2024 Redis license change" and add a closing paragraph: "In May 2025, with Redis 8, Redis Ltd. [added AGPLv3](https://redis.io/blog/agplv3/) as a third licensing option. Valkey remains under the BSD 3-clause license."

### F2 [minor] MISSING: History stops at Valkey's first release (`topics/history.md:179`)
**Doc says:**
> Three weeks later, a [second press release][lf-press-2] announced seven more companies joining and the first release, Valkey 7.2.5.
**Problem:** The page ends in April 2024. Two and a half years of the project's own history are missing, including every major release. A reader looking for "Valkey history" finds the Redis backstory and nothing about Valkey itself.
**Evidence:** Release tag dates in valkey@9.1.2: 8.0.0 2024-09-15, 8.1.0 2025-03-31, 9.0.0 2025-10-16, 9.1.0 2026-05-19.
**Fix:** Add a short final section, "Valkey releases", with one line per major release (8.0 Sep 2024, 8.1 Mar 2025, 9.0 Oct 2025, 9.1 May 2026) linking to the release notes or the corresponding valkey.io blog posts.

### F3 [nit] INACCURATE: The second press release announced 7.2.5-rc1, not 7.2.5 (`topics/history.md:179`)
**Doc says:**
> a [second press release][lf-press-2] announced seven more companies joining and the first release, Valkey 7.2.5.
**Problem:** The linked press release is titled "Valkey Community Announces Release Candidate" and announces 7.2.5-rc1. 7.2.5 GA did ship the same day, but the release notes are the source for that, not the press release.
**Evidence:** lf-press-2 page text: "April 16, 2024 – The Linux Foundation is excited to announce the Valkey 7.2.5-rc1 release candidate". valkey@7.2.5:00-RELEASENOTES:
> Valkey 7.2.5 GA  -  Released Tue 16 Apr 2024
> Upgrade urgency LOW: This is the first stable release of Valkey 7.2.5.
**Fix:** "Three weeks later, on April 16, 2024, a [second press release][lf-press-2] announced seven more companies joining, and the project shipped its first stable release, Valkey 7.2.5."

### F4 [nit] STYLE: Typos and grammar errors (`topics/history.md:22`)
**Doc says:** a recurring pattern:
- 21-22: "To improve it, he created an in-memory database called called" (doubled word, and "it" is ambiguous)
- 37-39: "Slicehost ... provided Virtual Machines for testing in a virtualized environment and Linode provided virtual machines for testing in a virtualized environment" (repeated clause)
- 110: "also clarified in in the company's blog"
- 126: "the projects [Governance]"
- 133: "the Redis Labs changed its name"
- 171: "It was therefore and easy decision"
**Fix:** 21-22 "To improve LLOOGG, he created an in-memory database called". 37-39 "In 2010, Slicehost (acquired by Rackspace) and Linode provided virtual machines for testing." 110 "in the". 126 "the project's". 133 "Redis Labs changed its name". 171 "therefore an easy decision".

### F5 [nit] AI-NATIVE: Title and opening do not say the page is about Valkey's origins (`topics/history.md:2`)
**Doc says:**
> title: "History"
> description: How the Valkey project started
**Problem:** The bare title "History" (rendered "Documentation: History") does not name its subject, and the first paragraph opens with antirez rather than stating what the page covers.
**Fix:** Title "History of Valkey". Open with: "This page covers how Valkey came to exist: the Redis project it forked from, the companies that sponsored it, the 2024 license change, and the launch of Valkey under the Linux Foundation."

### F6 [nit] STYLE: Missing link from the Documentation by Topic index
**Problem:** The page is not linked from any section of the topic index, so readers can only reach it by URL or alias.
**Evidence:** Issue #303 lists "Documentation: History | https://valkey.io/topics/history/ | **What is Valkey?** page".
**Fix:** Link the page from the "What is Valkey?" section, as #303 proposes.
Tracked: valkey-io/valkey-doc#303

## Unverified
- topics/history.md:13: Instagram engineering link. It resolves (52.6.3.192), but curl gets no HTTP response (000) from this host. The medium.com mirror returns 403 to curl. Could not tell bot blocking from a dead link.
- topics/history.md:34-46, 63-68: 2009-2015 sponsorship details and dates (Engine Yard, Hitmeister, VMware until May 2013, Pivotal to June 2015). No primary source in the available checkouts.
