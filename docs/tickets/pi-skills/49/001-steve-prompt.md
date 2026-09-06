I discoverered a medium severity security bug in the pi-skills open source project and raised:

https://github.com/badlogic/pi-skills/issues/49

I didn't submit a fix at that time, but now I would like to:
- fully understand the problem
- weigh up and understand the possible fixes
- submit a potential fix that I can be reasonably confident is one that they could/should merge.

The process for submitting bugs/fixes and contributing isn't documented at:
https://github.com/badlogic/pi-skills

but the person who owns this repo also wrote Pi at:
https://github.com/earendil-works/pi

which has a very specific process for contributing at:
https://github.com/earendil-works/pi/blob/main/CONTRIBUTING.md

which I suspect may apply to this pi-skills repo, and so I'd like to respect that.

This is my first time contributing, so I will need to spend some time working to understand how the project works. I am a developer familiar with Typescript, but not an expert, and so will need some of the basics explained to me please when considering the options for fixing this.

Please investigate the bug, the way the two repos are set up for dependency management (how they avoid security issues like this in dependencies), how this is normally handled by open source projects like this (include references to a couple or a few web pages pls), and then write a report for me to read that includes this and your recommendations (not a final fix, yet) at:

002-claude-report.md

I'm hoping to be able to read this report in approx 15 minutes (or less?) so it can't be too detailed.
Please include a TLDR Summary at the top and clickable TOC.

Any questions before you start? Please put them here with a placeholder for my answer and wait for me to fill them in.

Thanks.
---

## Claude's questions before writing the report (added 2026-09-06)

Context from the investigation so far, so the questions make sense (full detail will go in `002-claude-report.md`):

- `npm install` in `brave-search` does **not** error. It succeeds and prints a warning: `1 high severity vulnerability`. The only change `npm audit fix` makes is bumping `ws` 8.18.3 → 8.21.3 in `package-lock.json` (one package).
- npm says **high**, the issue says **medium**, because two advisories hit `ws` 8.18.3 and npm reports the worst one: [GHSA-58qx-3vcg-4xpx](https://github.com/advisories/GHSA-58qx-3vcg-4xpx) (medium, CVSS 4.4, memory disclosure, fixed in 8.20.1) and [GHSA-96hv-2xvq-fx4p](https://github.com/advisories/GHSA-96hv-2xvq-fx4p) (high, CVSS 7.5, memory-exhaustion DoS, fixed in 8.21.0).
- `browser-tools` has the same problem: `jsdom@27.0.1 → ws@8.18.3`, and `puppeteer-core` pulls `ws` in as well.
- `jsdom` 28+ dropped `ws` (it uses `undici` instead), so upgrading `jsdom` removes the vulnerable package entirely. jsdom 29.1.1 needs Node ≥ 20.19 / 22.13; jsdom 30.0.1 needs Node ≥ 22.22.2 / 24.15. pi itself requires Node ≥ 22.19.
- The vulnerable `ws` code only runs when a page script opens a WebSocket. `brave-search` never enables `runScripts`, so the bug is not reachable in practice. The real-world problem is the audit warning every user sees, and pi's own `SECURITY.md` asks dependency reports to say whether the issue is reachable.

HUMAN: This is a very good point. https://github.com/earendil-works/pi/blob/main/SECURITY.md says "For dependency reports, include evidence that the shipped dependency is affected and that the issue is reachable through Pi." - and so we can change the wording on this issue from "security issue" to lower priority "disturbing to users" issue - easily fixed by changing versions.

- `pi-skills` has no CI, no `CONTRIBUTING.md`, no `.github` folder and no auto-close bot. The maintainer last pushed on 2026-06-06 and has not commented on any pi-skills PR or issue in 2026; 25 PRs are open with no review. The `pi` repo's process requires an `lgtm` reply from a maintainer before opening a PR, and you are not on its approved-contributors list.

### Q1. Scope: fix `brave-search` only, or `browser-tools` as well?

`browser-tools` has the identical `jsdom`/`ws` problem plus `ws` via `puppeteer-core`.

**Claude's recommended answer:** The report covers both skills. The first PR changes only `brave-search`, and its description mentions that `browser-tools` has the same issue and offers a follow-up PR.

**Why:** Issue #49 is about `brave-search`, so a PR that fixes exactly that is easy to review and merge. `browser-tools` has a much bigger tree (211 packages vs 51, including puppeteer), needs Chrome to test, and has three separate `ws` requirements, so it deserves its own small PR. A quiet maintainer is far more likely to merge a one-file, one-package change than a two-skill change, and pi's "you must understand your code" rule is easier to honour on a small diff.

**Steve's answer:** I agree.

### Q2. Contribution process: how do you want to handle the `lgtm` gate?

pi's `CONTRIBUTING.md` says do not open a PR until a maintainer replies `lgtm`. pi-skills has no gate bot, but the maintainer has been silent there since June. Options: (a) you post a short comment on issue #49 in your own voice offering the PR and asking for `lgtm`, then wait; (b) open the PR anyway, referencing the issue; (c) ask on the pi Discord first. I will draft any comment for you but will not post anything myself.

**Claude's recommended answer:** (a). Post a short, own-voice comment on #49 saying you would like to submit the fix as a PR and asking for `lgtm`. Wait about a week. If there is no reply, open the PR anyway with one line saying you followed pi's CONTRIBUTING guidance and got no response.

**Why:** It costs nothing, it respects the process you want to honour, and it leaves a visible record that you asked. pi-skills has no auto-close workflow, so a PR will not be closed by a bot either way. The only PR ever merged on pi-skills (#1) was reviewed within five days, so a week is a fair wait. Discord is for urgent problems, and this is not urgent because the vulnerable code is not reachable from the skill.

**Steve's answer:** I agree - option (a) (after editing the issue to massively reduce it's size, but point to this public conversation for any further details reqd)

### Q3. Which fix flavour, and which Node versions must it support?

Candidates, from smallest to largest: (1) lockfile-only bump of `ws` to 8.21.3, which is exactly what `npm audit fix` did; (2) bump `jsdom` within `^27` to 27.4.0, which still keeps `ws` but at 8.21.3; (3) upgrade `jsdom` to 29.1.1, which removes `ws` and keeps pi's Node ≥ 22.19 floor; (4) upgrade `jsdom` to 30.0.1, which removes `ws` but raises the floor to Node 22.22.2.

**Claude's recommended answer:** Option (1) for the PR, with no change to supported Node versions. Mention option (3) in the PR description as the longer-term fix the maintainer may prefer, and let the report present the tested results for both.

**Why:** Option (1) is a three-line lockfile change with zero behaviour risk. jsdom 27 already allows `ws` 8.21.3 through its `^8.18.3` range, so no `overrides` entry is needed, and it is exactly what Dependabot would open. Option (3) is a major-version upgrade of the library that does all of the skill's HTML parsing, and it swaps `ws` for `undici`, a much larger package with its own advisory history. That is a bigger review for a maintainer who is not currently reviewing anything, and a bigger thing for a first-time contributor to vouch for. Option (4) would drop Node 20 and part of Node 22, which some pi, Claude Code and Codex users are still on. Option (2) gives no security benefit beyond (1).

**Steve's answer:** Agree

### Q4. May I run `npm install` in my scratchpad clone to test the candidate fixes end to end?

I would install each candidate `jsdom` version in a throwaway clone, run `npm audit`, and run `content.js` against a real page (no API key needed). `search.js` needs `BRAVE_API_KEY`, which is set in your shell.

**Claude's recommended answer:** Yes to installs in the scratchpad clone, and yes to two or three `search.js` runs using your existing key.

**Why:** A recommendation that has been run is worth more than one that only looks right, and the report should be able to say "content.js and search.js produce the same output before and after" for whichever option we propose. The scratchpad clone is isolated from your projects. `search.js --content` is the only path that combines the Brave API call with the jsdom parsing, so a couple of runs with your key cover the full code path. Brave's free tier allows this comfortably and I will keep it to a handful of calls.

**Steve's answer:** Yes

### Q5. Severity wording on issue #49

The issue says "Medium". npm reports "high" because of the second advisory.

**Claude's recommended answer:** Edit the issue body's Severity line yourself to say "High according to npm audit (GHSA-96hv-2xvq-fx4p is high, GHSA-58qx-3vcg-4xpx is medium)" and, in the same edit or in the Q2 comment, add one sentence saying the vulnerable code is not reachable from the skill because page scripts never run.

**Why:** Maintainers triage on severity, and pi's `SECURITY.md` explicitly asks dependency reports to say whether the shipped dependency is reachable. Saying "high, but not reachable, and the fix is a one-package lockfile bump" is accurate, honest and easy to act on. Leaving "Medium" invites a "low priority" dismissal or a correction from the maintainer, and overstating it would look like noise. Editing the body keeps the issue accurate for anyone who reads it later; a comment alone leaves the wrong number at the top.

**Steve's answer:** Agree

### Q6. Your local clone now has an uncommitted lockfile change

Reproducing the bug in `~/dev/pi/test-pi-paint-app/.pi/skills/pi-skills/brave-search` left `package-lock.json` modified (`ws` 8.21.3) and `node_modules` fixed.

**Claude's recommended answer:** Keep it as it is.

**Why:** The modified lockfile is exactly the fix, so your installed skill is protected now, and any later `npm install` in that folder will keep `ws` 8.21.3 rather than reinstalling 8.18.3. The PR branch will be made from a separate clone, so this copy does not need to stay clean. The only cost is a trivial merge conflict if you `git pull` upstream after they fix it, which `git checkout brave-search/package-lock.json` resolves in one command at that point.

**Steve's answer:** Agree


Steve final note before you write your document: I am realising that this is fixable by a tiny version bump, and that neither security issue applies to Pi Skills because the vulnerable part isn't used makes me think I should:
(1) Completely reword my bug report so it becomes clear it's probably not a security issue, but just scary for users who see it appear.  I'll remove all the AI generated stuff and hand-create it (please don't create the wording) and I'll incorporate the suggested wording about the lgtm stuff above in that.
(2) Let you write your report - but ask you now not to go into too much detail as it's probably not a security issue and very quick and easy to fix (I'll need evidence the version bump won't break anything and understand why (myself!) using your report)

