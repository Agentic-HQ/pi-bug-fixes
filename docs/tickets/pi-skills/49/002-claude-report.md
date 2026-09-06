# Ticket 49 report: the `ws` audit warning in the brave-search skill

Written by Claude on 2026-09-06 in reply to [001-steve-prompt.md](001-steve-prompt.md). Upstream issue: [badlogic/pi-skills#49](https://github.com/badlogic/pi-skills/issues/49).

## TL;DR

- Installing `brave-search` does not fail. `npm install` succeeds and prints one warning line: `1 high severity vulnerability`. That warning is the whole user-facing problem.
- The warning comes from `ws` 8.18.3, pulled in by `jsdom` 27.0.1, which the skill uses only to parse HTML. The vulnerable code in `ws` runs only when a WebSocket connection is opened, and nothing in the skill ever opens one. This is not a reachable security bug in the skill.
- The fix is three changed lines in `brave-search/package-lock.json`, bumping `ws` from 8.18.3 to 8.21.3. No code change, no `package.json` change, no `overrides` block. It is exactly what `npm audit fix` produces.
- Evidence it is safe: the bump stays inside the range `jsdom` already declares, every `ws` change in that range is a fix or an addition, and running both skill scripts before and after the bump gives byte-identical output.
- Same-maintainer precedent: the `pi` repo treats lockfile bumps as normal reviewed changes, runs a daily `npm audit`, and has fixed a nested vulnerable dependency the same way. `pi-skills` has none of that tooling, which is why this sat unfixed.

## Contents

- [1. What users see](#1-what-users-see)
- [2. The two advisories and why npm says high](#2-the-two-advisories-and-why-npm-says-high)
- [3. Is the skill actually affected](#3-is-the-skill-actually-affected)
- [4. Why users get the old version](#4-why-users-get-the-old-version)
- [5. The fix and the evidence it is safe](#5-the-fix-and-the-evidence-it-is-safe)
- [6. Alternatives considered](#6-alternatives-considered)
- [7. How pi handles this and what pi-skills lacks](#7-how-pi-handles-this-and-what-pi-skills-lacks)
- [8. Recommended next steps](#8-recommended-next-steps)
- [Appendix A: exact reproduction output](#appendix-a-exact-reproduction-output)
- [Appendix B: references](#appendix-b-references)

## 1. What users see

The README tells users to run `npm install` inside `brave-search`. On Node 24.15 / npm 11.12 that prints:

```
added 50 packages, and audited 51 packages in 2s
1 high severity vulnerability
To address all issues, run:
  npm audit fix
```

The install completes and the skill works. The text is alarming, especially "high", and every new user sees it. That is the problem worth fixing.

## 2. The two advisories and why npm says high

`npm audit` matches the installed `ws` 8.18.3 against the GitHub Advisory Database and prints the worst match. Two advisories match:

| Advisory | What it is | GitHub severity | Fixed in |
|---|---|---|---|
| [GHSA-58qx-3vcg-4xpx](https://github.com/advisories/GHSA-58qx-3vcg-4xpx) | `websocket.close()` leaks uninitialised memory to the peer if a TypedArray is passed as the reason | Medium, CVSS 4.4 | 8.20.1 |
| [GHSA-96hv-2xvq-fx4p](https://github.com/advisories/GHSA-96hv-2xvq-fx4p) | A peer sends many tiny fragments and the process runs out of memory | High, CVSS 7.5 | 8.21.0 |

So "high" is not npm's own opinion. It is the second advisory, which was published in June, after the first. The issue as filed says "Medium" because it was written from the first advisory only.

Both advisories describe attacks on a running WebSocket connection: a peer at the other end of a socket sends something nasty. There is no way to trigger either without a socket being opened.

## 3. Is the skill actually affected

The skill has two scripts, [search.js](https://github.com/badlogic/pi-skills/blob/main/brave-search/search.js) and [content.js](https://github.com/badlogic/pi-skills/blob/main/brave-search/content.js). Both do the same thing with a page: fetch the HTML with Node's built-in `fetch`, build a `JSDOM` document from it, hand that document to Mozilla's Readability to find the article, and convert the result to Markdown with Turndown. Neither script mentions WebSockets.

Where `ws` enters: `jsdom` implements the browser `WebSocket` API for pages that use it, and it does so on top of the `ws` package. Two facts from the installed code decide reachability:

- `jsdom` loads its WebSocket implementation, and therefore the `ws` module, when a `Window` is created. See [WebSocket-impl.js line 7](https://github.com/jsdom/jsdom/blob/v27.0.1/lib/jsdom/living/websockets/WebSocket-impl.js#L7). So the module is in memory. Loading it is the only thing the skill ever asks of `ws`.
- A real socket is created only inside the `WebSocket` constructor ([line 129](https://github.com/jsdom/jsdom/blob/v27.0.1/lib/jsdom/living/websockets/WebSocket-impl.js#L129)), which runs only if something calls `new WebSocket(...)` on the jsdom window. The skill's own code never does. Scripts inside the fetched page cannot either, because the skill never sets jsdom's `runScripts` option, so page JavaScript is never executed.

Conclusion: the vulnerable functions cannot be reached from this skill. pi's own [SECURITY.md](https://github.com/earendil-works/pi/blob/main/SECURITY.md) asks exactly this question of dependency reports: "include evidence that the shipped dependency is affected and that the issue is reachable through Pi." Here it is not reachable. The honest framing is "an audit warning that scares users", not "a security hole".

The same warning appears in `browser-tools`, which uses the same `jsdom` version and also gets `ws` through `puppeteer-core`. There the WebSocket path is real, because puppeteer talks to Chrome over a WebSocket, so that skill deserves its own look later.

## 4. Why users get the old version

Two files control what `npm install` installs. This is the part worth understanding properly, because it is why the fix is so small.

**`package.json` declares ranges.** It says `"jsdom": "^27.0.1"`. The caret means "27.0.1 or any newer 27.x". `jsdom` 27.0.1 in turn declares `"ws": "^8.18.3"`, meaning 8.18.3 or any newer 8.x. `ws` is a transitive dependency: the skill never asks for it, `jsdom` does.

**`package-lock.json` records exact choices.** When the lockfile was generated in late 2025, npm picked the newest versions allowed at that time and wrote them into the lockfile: `jsdom` 27.0.1 and `ws` 8.18.3. The lockfile is committed, and `npm install` always obeys a lockfile that is present. That is the point of a lockfile: everyone gets the same tree. It is also why every user today still gets `ws` 8.18.3 even though the range allows 8.21.3 and 8.21.3 has been out since August.

**`npm audit fix` edits the lockfile only.** It looks for a newer version that both fixes the advisory and still satisfies every declared range. `ws` 8.21.3 satisfies `^8.18.3`, so npm simply rewrites the three lockfile lines for `ws` and nothing else. Had the patched version been outside the allowed range, the fix would have needed an `overrides` entry in `package.json` or an upgrade of `jsdom` itself. Neither is needed here.

## 5. The fix and the evidence it is safe

The complete change:

```diff
 		"node_modules/ws": {
-			"version": "8.18.3",
-			"resolved": "https://registry.npmjs.org/ws/-/ws-8.18.3.tgz",
-			"integrity": "sha512-PEIGCY5tSlUt50cqyMXfCzX+oOPqN0vuGqWzbcJ2xvnkzkq46oOpz7dQaTDBdfICb4N14+GARUDw2XV2N4tvzg==",
+			"version": "8.21.3",
+			"resolved": "https://registry.npmjs.org/ws/-/ws-8.21.3.tgz",
+			"integrity": "sha512-201TZ/kPWxoPr/OKWjquZR1SWKXcvxdH+e1xrx89b3YbmzLMFCLfnaG1HFIgWzJOEWZ7MvpK++odZufgYR50Rw==",
```

Three lines of evidence that this cannot break the skill.

**1. It stays inside the declared range.** `jsdom` asks for `^8.18.3`. Under semantic versioning a minor or patch bump inside a major version must not change existing behaviour, and 8.21.3 is a minor bump. `ws` still supports Node 10 and up, so no engine constraint changes.

**2. Every change between 8.18.3 and 8.21.3 is a fix or an addition.** From the [ws release notes](https://github.com/websockets/ws/releases):

| Version | Date | Change |
|---|---|---|
| 8.19.0 | 2026-01-05 | Added a `closeTimeout` option; handled a forthcoming Node.js core change |
| 8.20.0 | 2026-03-21 | Exported some internal classes and header utilities |
| 8.20.1 | 2026-05-12 | Fixed the memory disclosure in `close()` |
| 8.21.0 | 2026-05-22 | Fixed the memory-exhaustion DoS; added `maxBufferedChunks` and `maxFragments` limits |
| 8.21.1 | 2026-07-14 | Counted empty fragments toward the limit; lowered the default limits |
| 8.21.2 | 2026-08-03 | Fixed a test |
| 8.21.3 | 2026-08-06 | Server rejects a bad `permessage-deflate` offer correctly |

The only behavioural changes are new limits on incoming fragments, which apply to open sockets. The skill opens none.

**3. The skill produces identical output before and after.** In a throwaway clone of upstream `main`, I installed from the committed lockfile, ran both scripts, applied the bump, and ran them again.

| Step | Before, ws 8.18.3 | After, ws 8.21.3 |
|---|---|---|
| `npm install` | 1 high severity vulnerability | found 0 vulnerabilities |
| `content.js` on a Wikipedia article | exit 0, 63,114 bytes of Markdown | exit 0, 63,114 bytes, byte-for-byte identical |
| `search.js "npm package-lock.json explained" -n 2 --content` | exit 0, two results | exit 0, same two results |
| `npm audit` | ws 8.0.0 - 8.20.1, high | found 0 vulnerabilities |

`content.js` exercises the full path that matters: fetch, jsdom parse, Readability, Turndown. Since `ws` is loaded but never called, the only thing the bump could break is module loading, and both runs prove it loads.

## 6. Alternatives considered

| Option | Removes `ws`? | Cost | Verdict |
|---|---|---|---|
| Lockfile bump of `ws` to 8.21.3 (this report) | No, but patches it | 3 lines, no behaviour change | Recommended |
| Bump `jsdom` to 27.4.0 within `^27` | No, still `ws` 8.21.3 | Bigger lockfile diff, no extra security benefit | Not needed |
| Upgrade `jsdom` to 29.1.1 | Yes, replaced by `undici` from [28.0.0](https://github.com/jsdom/jsdom/releases/tag/28.0.0) | Major version of the HTML parser; brings in `undici`, a large package with its own advisory history; needs retesting | Good follow-up, not a first PR |
| Upgrade `jsdom` to 30.0.1 | Yes | Raises the Node floor to 22.22.2 ([v30.0.0](https://github.com/jsdom/jsdom/releases/tag/v30.0.0)), which is above pi's own floor of 22.19 | Not now |
| Add `ws` to `overrides` or as a direct dependency | No | Config to maintain for no gain, since the range already allows the fix | Not needed |

The issue as filed suggested adding `ws` as an explicit dependency and pinning. Neither is necessary once you see that the lockfile is the thing that pinned the old version.

## 7. How pi handles this and what pi-skills lacks

The maintainer's main repo, [earendil-works/pi](https://github.com/earendil-works/pi), has a full dependency policy. Worth knowing before submitting, because it shows what "normal" looks like to them:

- Exact versions only. [`.npmrc`](https://github.com/earendil-works/pi/blob/main/.npmrc) sets `save-exact=true`, and a check script ([check-pinned-deps.mjs](https://github.com/earendil-works/pi/blob/main/scripts/check-pinned-deps.mjs)) fails the build if any direct dependency uses a range.
- A two-day cooldown. The same `.npmrc` sets `min-release-age=2`, an npm 11.10+ feature that refuses to install any version published less than two days ago. This blunts supply-chain attacks where a compromised package is yanked within hours ([npm config docs](https://docs.npmjs.com/cli/v11/using-npm/config#min-release-age), [Brandon Pugh's note](https://www.brandonpugh.com/til/node/package-version-cooldown/)).
- Lockfiles are reviewed code. [AGENTS.md](https://github.com/earendil-works/pi/blob/main/AGENTS.md) says "Treat npm dep and lockfile changes as reviewed code", and a pre-commit hook blocks lockfile commits unless explicitly allowed.
- Daily audit. [npm-audit.yml](https://github.com/earendil-works/pi/blob/main/.github/workflows/npm-audit.yml) runs `npm audit --omit=dev --audit-level=moderate` every morning.
- Precedent for nested vulnerable packages. Issue [#7005](https://github.com/earendil-works/pi/issues/7005) reported a vulnerable `protobufjs` nested under another package. The fix, commit [ec1a87e8](https://github.com/earendil-works/pi/commit/ec1a87e8), added an `overrides` entry and regenerated the lockfiles. That is the `overrides` route this skill does not need. A broader "fix: update vulnerable dependencies" commit ([ea65a51a](https://github.com/earendil-works/pi/commit/ea65a51a)) bumped versions and lockfiles in one go.

`pi-skills` has none of this: no `.github` folder, no CI, no audit workflow, no `CONTRIBUTING.md`. Two skills commit a lockfile (`brave-search`, `browser-tools`) and one does not (`youtube-transcript`). Nothing tells the maintainer when a lockfile goes stale.

How other projects handle the same class of problem:

- GitHub's [Dependabot security updates](https://docs.github.com/en/code-security/dependabot/dependabot-security-updates/about-dependabot-security-updates) open exactly this kind of lockfile-bump PR automatically, and for npm they can bump a parent package when needed. A repo with several `package.json` files in subfolders can list them with the `directories` key ([GitHub changelog](https://github.blog/changelog/2024-06-25-simplified-dependabot-yml-configuration-with-multi-directory-key-directories-and-wildcard-glob-support/)).
- When the patched version is outside the allowed range, the standard tool is `overrides` in `package.json` ([npm docs](https://docs.npmjs.com/cli/v11/configuring-npm/package-json#overrides), [HeroDevs guide](https://www.herodevs.com/blog-posts/a-guide-to-npm-overrides-take-control-of-your-dependencies), [Mend guide](https://docs.mend.io/wsk/how-to-resolve-vulnerable-npm-transitive-dependenc)).
- Running `npm audit` in CI, as pi does, is the usual way to stop this recurring.

## 8. Recommended next steps

1. **You rewrite issue #49** in your own words, as you planned. The facts above that matter for triage: the install warning is the user-facing problem, neither advisory is reachable from the skill, and the fix is a three-line lockfile bump. Say you would like to submit the PR and ask for `lgtm`, per pi's [CONTRIBUTING.md](https://github.com/earendil-works/pi/blob/main/CONTRIBUTING.md).
2. **Wait about a week.** pi-skills has no auto-close bot, but the maintainer has not commented there since June. If nothing comes back, open the PR anyway with one line saying you asked first.
3. **The PR** is the lockfile diff in section 5 and nothing else. Its description should state the advisories, that they are unreachable from the skill, the semver argument, and the before/after run. I will draft that description for your review when you are ready.
4. **Optional, for the maintainer's benefit:** the PR description could mention that `browser-tools` has the same stale `ws`, and that a `dependabot.yml` using `directories` would have caught this. Whether to include suggestions in a first PR is your call.
5. **Follow-up PR** for `browser-tools`, where the WebSocket path is real and the bump matters more.

## Appendix A: exact reproduction output

Captured on 2026-09-06 in your clone at `~/dev/pi/test-pi-paint-app/.pi/skills/pi-skills/brave-search`, Node v24.15.0, npm 11.12.1, after deleting `node_modules`.

```
$ npm install

added 50 packages, and audited 51 packages in 2s

8 packages are looking for funding
  run `npm fund` for details

1 high severity vulnerability

To address all issues, run:
  npm audit fix

Run `npm audit` for details.

$ npm ls ws
brave-search@1.0.0
└─┬ jsdom@27.0.1
  └── ws@8.18.3

$ npm audit
# npm audit report

ws  8.0.0 - 8.20.1
Severity: high
ws: Uninitialized memory disclosure - https://github.com/advisories/GHSA-58qx-3vcg-4xpx
ws: Memory exhaustion DoS from tiny fragments and data chunks - https://github.com/advisories/GHSA-96hv-2xvq-fx4p
fix available via `npm audit fix`
node_modules/ws

1 high severity vulnerability

$ npm audit fix

changed 1 package, and audited 51 packages in 1s

found 0 vulnerabilities

$ npm ls ws
brave-search@1.0.0
└─┬ jsdom@27.0.1
  └── ws@8.21.3

$ git status --short
 M package-lock.json
```

## Appendix B: references

Upstream files and issues:

- [pi-skills issue #49](https://github.com/badlogic/pi-skills/issues/49)
- [brave-search/package.json](https://github.com/badlogic/pi-skills/blob/main/brave-search/package.json) and [package-lock.json](https://github.com/badlogic/pi-skills/blob/main/brave-search/package-lock.json)
- [pi CONTRIBUTING.md](https://github.com/earendil-works/pi/blob/main/CONTRIBUTING.md), [SECURITY.md](https://github.com/earendil-works/pi/blob/main/SECURITY.md), [AGENTS.md](https://github.com/earendil-works/pi/blob/main/AGENTS.md), [.npmrc](https://github.com/earendil-works/pi/blob/main/.npmrc), [npm-audit.yml](https://github.com/earendil-works/pi/blob/main/.github/workflows/npm-audit.yml)
- [pi issue #7005](https://github.com/earendil-works/pi/issues/7005) and fix commit [ec1a87e8](https://github.com/earendil-works/pi/commit/ec1a87e8)

Advisories and changelogs:

- [GHSA-58qx-3vcg-4xpx](https://github.com/advisories/GHSA-58qx-3vcg-4xpx), [GHSA-96hv-2xvq-fx4p](https://github.com/advisories/GHSA-96hv-2xvq-fx4p)
- [ws releases](https://github.com/websockets/ws/releases)
- [jsdom 28.0.0](https://github.com/jsdom/jsdom/releases/tag/28.0.0) (dropped `ws`), [jsdom v30.0.0](https://github.com/jsdom/jsdom/releases/tag/v30.0.0) (Node floor raised)
- [jsdom 27.0.1 WebSocket-impl.js](https://github.com/jsdom/jsdom/blob/v27.0.1/lib/jsdom/living/websockets/WebSocket-impl.js)

General practice:

- [npm: overrides](https://docs.npmjs.com/cli/v11/configuring-npm/package-json#overrides), [npm: config, min-release-age](https://docs.npmjs.com/cli/v11/using-npm/config#min-release-age), [npm audit](https://docs.npmjs.com/cli/v11/commands/npm-audit)
- [GitHub: about Dependabot security updates](https://docs.github.com/en/code-security/dependabot/dependabot-security-updates/about-dependabot-security-updates), [Dependabot `directories` key](https://github.blog/changelog/2024-06-25-simplified-dependabot-yml-configuration-with-multi-directory-key-directories-and-wildcard-glob-support/)
- [HeroDevs: a guide to npm overrides](https://www.herodevs.com/blog-posts/a-guide-to-npm-overrides-take-control-of-your-dependencies), [Mend: resolving a vulnerable npm transitive dependency](https://docs.mend.io/wsk/how-to-resolve-vulnerable-npm-transitive-dependenc)
- [Brandon Pugh: npm 11.10 adds min-release-age](https://www.brandonpugh.com/til/node/package-version-cooldown/), [Matteo Collina: configuring minimum release age](https://gist.github.com/mcollina/b294a6c39ee700d24073c0e5a4e93104)
