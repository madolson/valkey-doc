VERDICT: MINOR-ISSUES
COUNTS: blocker=0 minor=2 nit=1

# License (`license`)

https://valkey.io/topics/license/

The page is one sentence that links out. It never names the license, and it promises trademark information it does not have. Fix: state "BSD 3-Clause" in the page body and add the trademark policy link.

## Findings

### F1 [minor] MISSING: Page never names the license (`topics/license.md:8`)
**Doc says:**
> Please refer to [license](https://github.com/valkey-io/valkey?tab=License-1-ov-file#readme) for Valkey licensing information.
**Problem:** "What license is Valkey under?" is the one question this page exists to answer, and the reader has to leave the site to get the answer. The page also leaves out that the code carries two BSD-3-Clause grants (Valkey contributors from 2024, Redis Ltd. for 2006-2020). The only place the docs name the license is a parenthetical in topics/introduction.md:6 ("open source (BSD licensed)").
**Evidence:** valkey@9.1.2:COPYING:1-5 and :13-16:
```
# License 1
SPDX-License-Identifier: BSD-3-Clause
BSD 3-Clause License
Copyright (c) 2024-present, Valkey contributors
...
# License 2
BSD 3-Clause License
Copyright (c) 2006-2020, Redis Ltd.
```
The modules use the same license: valkey-bloom/LICENSE, valkey-json/LICENSE, and valkey-search/LICENSE all start with "BSD 3-Clause License".
**Fix:** Replace line 8 with:
> Valkey is released under the [BSD 3-Clause License](https://github.com/valkey-io/valkey/blob/unstable/COPYING) (SPDX: `BSD-3-Clause`). The code is covered by two copyright notices under that license: one for Valkey contributors (2024 to present) and one for Redis Ltd. (2006 to 2020), which covers code inherited from Redis before the fork. The official modules (valkey-bloom, valkey-json, valkey-search) are also BSD 3-Clause.

### F2 [minor] MISSING: Front matter promises trademark information the page lacks (`topics/license.md:4`)
**Doc says:**
> License and trademark information
**Problem:** The body says nothing about trademarks. The Linux Foundation trademark policy link shows up only in the site footer on every page (`lfprojects.org/policies/trademark-policy/`, HTTP 200). The page also leaves out that the documentation is licensed CC BY-SA 4.0, which only topics/index.md:8 states. Issue #296 uses the same "License and trademark information" text for the menu entry, so readers arrive expecting trademark content.
**Evidence:** The live page's only trademark reference is the footer link `href="https://lfprojects.org/policies/trademark-policy/"`. topics/index.md:7-8: "It's released under the [Creative Commons Attribution-ShareAlike 4.0 International license](https://creativecommons.org/licenses/by-sa/4.0/)."
**Fix:** Add:
> ## Trademark
> Valkey is a trademark of LF Projects, LLC. Use of the name and logo is governed by the [Linux Foundation trademark policy](https://lfprojects.org/policies/trademark-policy/).
>
> ## Documentation license
> This documentation is released under the [Creative Commons Attribution-ShareAlike 4.0 International license](https://creativecommons.org/licenses/by-sa/4.0/).
Tracked: valkey-io/valkey-doc#296 (menu text only, not the missing content)

### F3 [nit] STYLE: Link uses a fragile GitHub UI tab URL and vague link text (`topics/license.md:8`)
**Doc says:**
> [license](https://github.com/valkey-io/valkey?tab=License-1-ov-file#readme)
**Problem:** `?tab=License-1-ov-file` is a GitHub repo-page UI parameter, not a stable file URL. The link text "license" doesn't say where it goes. "Please refer to" is filler.
**Evidence:** The URL returns 200 today, but it depends on GitHub's tab naming. The stable file is `COPYING` at the valkey repo root (valkey@9.1.2:COPYING).
**Fix:** Link to `https://github.com/valkey-io/valkey/blob/unstable/COPYING` with the text "BSD 3-Clause License", as in the F1 replacement.
