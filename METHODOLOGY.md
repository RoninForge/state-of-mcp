# Methodology

How the census is drawn, what each probe does, and exactly what the numbers do and do not mean.
The whole value of this dataset is that the method is honest and visible, so this document is
part of the product.

## Editions

Every census is a dated snapshot and keeps its own directory under `data/censuses/`. The method
below applies to all of them; the figures quoted in this document are always the **latest**
edition, labelled as such. Earlier editions' figures live in their own `summary.json` and are not
rewritten.

| Edition | Drained | Registered servers |
|---|---|---|
| `2026-07` | 2026-07-02 | 14,559 |
| `2026-07-22` | 2026-07-22 | 18,032 |
| `2026-08-03` | 2026-08-03 | 19,804 |
| `2026-09-09` | 2026-09-09 | 29,522 |
| **`2026-09-13`** (current) | 2026-09-13 | **31,538** |

## Denominator

- **31,538 servers**, drained from the official registry (`registry.modelcontextprotocol.io`) at
  their latest version, on **2026-09-13**. The registry is growing fast: 13,886 on 2026-06-26,
  14,559 on 2026-07-02, 18,032 on 2026-07-22, 19,804 on 2026-08-03, 29,522 on 2026-09-09, so this
  edition is +6.8% in four days. The largest jump recorded remains 2026-09-09's +49.1% in
  thirty-seven days.
- Growth is concentrated, and a reader should know that before reading any rate as an ecosystem
  trend. Two namespaces account for 19.4% of the net growth on their own: `io.github.mcp-dir` 97 to
  1,113 (+1,016) and `io.github.sadri-dridi` 0 to 868. Excluding `io.github.mcp-dir` alone, the
  registry grew 44.2% rather than 49.1%.
- Every rate in this document and in `summary.json` is computed over all 31,538 servers unless
  stated otherwise. 276 of those servers declared only entrypoints that cannot be probed without a
  key (`unknown`, see Definitions) and are excluded from no rate; we report the all-31,538 figure
  because it is the more conservative, complete denominator, not the narrower "probeable-only" one.

## What each probe does

`akashi` runs the same fixed, keyless check set against every server. A check either passes,
fails, warns, or is skipped (not applicable to that server's declared entrypoints).

- **Registry status** - passes if the registry has not marked the entry deprecated or deleted.
- **`server.json` valid** - the server's manifest is validated against the JSON Schema it itself
  declares. A failure downgrades the verdict.
- **Repository reachable** - if a GitHub repository is declared, akashi checks it is not 404 and
  not archived, via the public GitHub API.
- **Repository freshness** - the repository's last push: under 90 days passes, under a year warns,
  over a year fails.
- **Package `<npm|pypi|oci>`** - if a package entrypoint is declared, akashi checks it resolves
  (is published and installable) against the public npm registry, PyPI, or Docker Hub's anonymous
  manifest API.
- **Remote reachable** - if a hosted remote endpoint is declared, akashi first attempts a
  capability-only MCP `initialize` handshake (which also produces the conformance signal below),
  and falls back to a plain GET for transports the handshake cannot exercise, so a transport
  mismatch is never scored as dead.
- **MCP conformance** - the `initialize` response negotiates a protocol version and echoes the
  request id, ruling out an HTML page or other impostor answering HTTP 200. A 401/403 to
  `initialize` is scored `auth_gated`: alive, but not verifiable without credentials, which akashi
  never supplies.
- **`tools/list`** - for remotes that pass conformance and are not auth-gated, akashi opens a full
  MCP session with the official
  [modelcontextprotocol/go-sdk](https://github.com/modelcontextprotocol/go-sdk) client and lists
  the server's tools, then closes. It runs no tool. This is informational only: it never downgrades
  a verdict, because a valid MCP server may legitimately advertise no tools. It is, however, the
  strongest keyless proof that an endpoint is a real, working MCP server rather than something that
  merely answers one handshake.
- **License present** - a license is declared on the repository.
- **At least one live entrypoint** - true if any of the server's declared entrypoints (package,
  repository, or remote) is reachable. A server with zero declared, probeable entrypoints cannot
  pass this check.

## Definitions

- **healthy** - at least one live entrypoint and nothing broken. The only verdict that earns a
  green badge.
- **degraded** - usable, but something is broken: a 404 repo link while the package still installs,
  a stale-over-a-year or archived repo, a deprecated registry entry, a remote that answers HTTP but
  is not a conformant MCP server, or an invalid `server.json`.
- **dead** - registry-deleted, or every probed entrypoint is broken.
- **unknown** - only entrypoints we cannot probe without a key were declared (for example an
  OCI-only image on a private registry). Reported as its own count, never folded into "broken."
- **keyless** - every probe touches only public endpoints: the registry, the public GitHub API,
  npm, PyPI, anonymous Docker Hub, and a capability-only MCP `initialize`/`tools/list`. akashi
  authenticates to no probed server and runs none of its tools. A GitHub token, if present, only
  raises the public-API rate limit.
- **reproducible** - same registry, same probe set, same documented criteria, `akashi v0.3.0`. Any
  row in `records.jsonl` is re-checkable with `akashi check <server>`.

## Headline (2026-09-13 edition)

| Verdict | Count | Share | 2026-09-09 |
|---|---|---|---|
| healthy | 25,411 | 80.6% | 79.7% |
| degraded | 4,667 | 14.8% | 15.8% |
| dead | 1,184 | 3.8% | 3.9% |
| unknown | 276 | 0.9% | 0.5% |

Dead-or-broken (degraded + dead): 5,851, about 1 in 5. Fully dead: 1,184, about 1 in 27.

The registry grew **6.8% in four days**, 29,522 to 31,538. Health rates have been stable across all
five editions while it grew 117% from the first (healthy 79.6%, 79.7%, 80.7%, 79.7%, 80.6%). The
ecosystem is adding servers faster than it is rotting them, but the dead-or-broken share has not
moved: roughly one registered server in five does not fully work, edition after edition. That is
now five editions across ten weeks and a registry that more than doubled, without the ratio
shifting, which makes it the most durable finding in this dataset.

**The scan took 8h57m against the previous edition's 16h59m**, for 2,016 more servers: 1.02 seconds
per server against 2.07. The composition that explained the earlier blowout did not reverse
(remote-bearing servers rose 17,355 to 19,150), and nothing in the scanner changed that would
account for it, so this is recorded as an observation about upstream latency, not claimed as a fix.
It may revert.

## Per-layer findings (2026-09-13 edition)

Failure is not evenly spread across a server's declared entrypoints. Package and remote counts
below are ENTRYPOINTS, not servers; repository counts are servers. Never mix the two in one rate.

- **Packages.** Of 14,772 declared npm/PyPI/OCI package entrypoints, 121 are broken, 0.8% (was
  1.3%). A further 2,084 are OCI images on private registries that cannot be probed keylessly
  (part of the `unknown` cohort, not counted as broken).
- **Repositories.** Of 24,383 declared repos, 3,370 (13.8%) return 404/gone and 217 are archived,
  so 3,587 (14.7%) are rotted.
  - **Ambiguity band, and it is much WIDER this edition.** A further 2,582 repo probes returned an
    ambiguous "error" response, 10.6% of the bearing population against 933 (4.1%) last edition.
    Those are excluded from the 14.7% figure entirely rather than counted as either working or
    broken, so the true value lies in **[14.7%, 25.3%]**, against [16.0%, 20.1%] last edition.
    **The apparent fall from 16.0% to 14.7% is not evidence that repositories rotted less.** The
    new band contains the old one entirely: the point estimate moved down and the uncertainty
    roughly tripled. Quote the band or quote nothing. What made 1,649 more repo probes ambiguous
    is not known, and is the open question this layer leaves for the next edition.
- **Hosted remotes.** Of 19,661 declared remote endpoints, 1,924 (9.8%) are down (unreachable,
  5xx, or 404), improved from 10.8%. Remote-bearing servers remain the sickest cohort in the
  registry: **22.5% dead-or-broken and 5.4% fully dead**, against 3.8% dead overall. 4,422
  endpoints are auth-gated: alive, but not keyless-verifiable, and never treated as broken on that
  basis alone.
  - **366 endpoints answer HTTP 200 but never complete an `initialize` handshake**, against 363
    last edition. Flat. The previous edition called this "the fastest-moving negative signal"
    after it went 63 to 363; one edition later that jump stayed a one-off, and impostors should
    not be described as a growing problem on this evidence.
- **Conformance.** 12,003 remote endpoints complete an MCP `initialize` handshake, spread over
  11,929 servers. 10,641 endpoints successfully resolved `tools/list`, the strongest keyless proof
  that a server actually works end to end. (Both are endpoint counts. Earlier editions of this
  document described the `tools/list` figure as servers; the number was right, the label was not.)
- **`server.json` validity.** 387 servers (1.2%) publish a manifest that fails validation against
  its own declared JSON Schema, improved from 1.3%.

## Claims we make, and claims we refuse

**We make:** the four verdict rates above, dated 2026-07-02, over the full 14,559-server
denominator, with the per-layer breakdown and the keyless, reproducible framing that backs them.

**We refuse:**

- Any unreproduced figure from an earlier draft of this project, including a "52%" or a
  "domain-verified remotes at 36%" claim. We could not reproduce either number from this census
  (our segmentation is "has a declared remote endpoint," not a narrower "domain-verified" subset),
  so we do not use them. The measured, honest number for the hosted-remote segment is 25.9%
  dead-or-broken and 7.5% fully dead.
- Any security or malware claim. akashi measures reachability, freshness, and protocol conformance.
  It does not scan for vulnerabilities, malicious code, or supply-chain risk, and this dataset says
  nothing about whether a server is safe to run.
- Any trend claim. This is census #1. There is no prior measurement under this methodology to
  compare against, so no "getting better" or "getting worse" claim is made anywhere in this
  dataset or its documentation.
- Any per-vendor ranking or naming-and-shaming. This is a measurement of the registry as a whole,
  not a leaderboard of individual maintainers or organizations.

## Limitations

- A verdict reflects the registry's declared entrypoints at the moment probed. A server can change
  state at any time; each census is a snapshot dated by its directory name, not a live status.
- Keyless probing cannot verify anything behind authentication. Auth-gated remotes are scored
  alive-but-unverified, never broken, so the true health of that cohort could be better or worse
  than what a keyless probe can see.
- `tools/list` completion is informational and does not affect a verdict, since a valid MCP server
  may advertise no tools.
- **A census is not a snapshot.** This edition's scan ran 8h57m, against 16h59m for 2026-09-09 and
  4h47m for 2026-08-03. Over a window of
  hours a single operator's outage lands in the dataset as if it were a property of its servers.
  It did in 2026-09-09, which is disclosed below, and it did again here, in both directions: see
  the second pass. Rates that depend on a runtime handshake should be read with the window in mind.
  Since 2026-09-13 the scan re-probes drifting namespaces itself rather than relying on a reader to
  suspect one, but that is a detector, not a fix: a small operator's outage still lands in the data
  below the threshold that triggers it.

## Spec-readiness pass (2026-09-13 edition)

Censuses from 2026-07-22 onward carry a readiness dimension for the 2026-07-28 specification
revision, measured with akashi's readiness pass. Each edition carries the ruleset stamp that
actually computed it: `2026-07-28-rc` for 2026-07-22 (akashi `d54c52d`) and 2026-08-03 (akashi
`b1c497b`), `2026-07-28` for 2026-09-09 (akashi `v0.3.0-7-g15983e0`) and 2026-09-13 (akashi
`v0.3.0-11-g69de215`). The rules are identical across the two stamps; see Ruleset status below. It runs
only against a server's first remote that answered a conformant `initialize`, and adds a small
set of read-only, unauthenticated calls: a handshake-free `tools/list` carrying the protocol
version in `_meta`, `server/discover`, a `tools/list` whose `Mcp-Method` routing header
deliberately mismatches, a `subscriptions/listen` existence check (a genuine result only; error
codes are ambiguous across SDKs), one GET (to detect the legacy HTTP+SSE `endpoint` event), a
`resources/read` of a sentinel URI that cannot exist (only when the server declares the
resources capability), and a fetch of the public RFC 9728 `/.well-known/oauth-protected-resource`
metadata. No call authenticates and no call executes a tool.

Verdicts, first match wins:

1. **No verdict (unknown)**: the server has no keylessly reachable conformant MCP endpoint
   (package-only, auth-gated, or down). Absence of evidence is recorded as absence.
2. **at-risk**: reachable AND on a removed-or-deprecated surface: declares the deprecated
   HTTP+SSE transport, was built on the removed experimental Tasks API, or hard-requires the
   removed `Mcp-Session-Id` session mechanism.
3. **ready**: reachable AND all four required-conformance signals pass: `server/discover`
   implemented, handshake-free calls accepted, no session minted or required, routing-header
   mismatch rejected.
4. **needs-migration**: everything else reachable.

Advisory observations (`resultType` and cache metadata on results, the resource-not-found error
code, a declared logging capability) never affect the verdict; they are recorded as warnings.

**Flag counts and reason counts are not the same number, and must not be quoted as if they were.**
Because at-risk resolves on first match, a server that hard-requires `Mcp-Session-Id` *and*
declares the deprecated HTTP+SSE transport is recorded with the transport reason, so it carries
the `sessionRequired` observable without carrying the session reason. In this edition 1,709
servers carry the flag and 1,688 cite it as their reason (2026-09-09: 1,694 and 1,676; 2026-08-03:
546 and 529; 2026-07-22: 544 and 528). Quote the
observable (`sessionRequired`) when describing what servers do, and the reason count only when
describing why a verdict was assigned. The same caution applies to every other signal that also
appears as a reason string.

Denominator rules: readiness percentages are computed ONLY over servers with a readiness verdict
(11,929 in this edition), never over the registered total. A registered-total denominator may only
carry declared metadata (for example the server.json schema-version distribution), never runtime
readiness.

### Findings (2026-09-13 edition, 11,929 servers with a verdict)

| Verdict | Count | Share | 2026-09-09 |
|---|---|---|---|
| ready | **31** | 0.26% | 29 (0.3%) |
| needs-migration | 10,000 | 83.8% | 8,426 (81.5%) |
| at-risk | 1,898 | 15.9% | 1,881 (18.2%) |

**The ready cohort went 29 to 31 while the conformant population grew 15.4%.** The previous edition
noted migration briefly outpacing its population; four days later that has not held. Thirty-one
servers out of 11,929 is 0.26%, and ten weeks of data now say the same thing: migration has started
and has not progressed.

**At-risk fell as a share while rising in absolute terms**, 1,881 to 1,898 servers but 18.2% to
15.9% of a denominator that grew 15.4%. The at-risk population is close to static while the
conformant population grows around it. That is the opposite reading from 2026-09-09, where at-risk
nearly tripled, and it is a reminder that a share and a count can move in different directions in
the same dataset. Its dominant reason is unchanged: `requires the removed Mcp-Session-Id session
mechanism`, 1,676 to 1,688.

The leading indicators kept moving but far more slowly than last edition: `server/discover`
implemented 393 to 420, final-code routing-header enforcement 242 to 267.

**The error-code split holds.** The 2026-07-28 renumbering moved `HeaderMismatch` from `-32001` to
`-32020`. Across four editions the old-code cohort has been 39, 37, 38, **40**, while the final-code
cohort went 0, 45, 242, **267**. Servers are still not migrating off `-32001`; new servers are
simply being built on `-32020`. A probe accepting only one of the two numbers would still miss this.

**Stateless tolerance recovered to 63.3%** (7,555 of 11,929) from 61.0%, which is what a partly
contaminated prior reading predicts. Against 2026-08-03's 80.9% the fall is real but smaller than
the raw series suggests; the control is in the disclosure below.

**A year-old protocol version is the fastest-growing negotiation result.** `2025-03-26` doubled,
1,214 to 2,467, while `2026-07-28` moved 120 to 123. This document does not assert a cause: a large
publisher changing a default and an ecosystem regression look identical from here, and separating
them needs a namespace breakdown that has not been done.

#### The second pass: this edition checked itself

2026-09-13 is the first census scanned with `--compare`, so any namespace whose aggregate signals
moved more than 10 points against the previous edition was re-probed during the run rather than
investigated by hand afterwards. Three tripped the threshold, all re-probed within 13 minutes of the
main pass finishing.

| Namespace | Signal | Census | Re-probed | Agreed | Changed |
|---|---|---|---|---|---|
| `io.github.wyre-technology` | healthy | 100% to 0% | 57 | 0 | **57** |
| `io.tooloracle` | healthy | 23% to 95.1% | 61 | 13 | **48** |
| `io.github.pipeworx-io` | statelessAccepted | 21% to 0% | 1,337 | **1,294** | 0 |

It found contamination in **both directions**: 57 servers this census recorded as unhealthy answered
healthy minutes later, and 48 it recorded as healthy did not. **Neither is corrected.** The census is
published exactly as observed and the second readings sit beside it in `reprobe.jsonl`, which is
what the disclosure practice below looks like once it is automatic.

pipeworx-io is the case where the second reading confirmed the first: its 0.0% stateless is real.

#### Disclosure: an operator outage sits inside the 2026-09-09 stateless rate

This section documents the PREVIOUS edition and is kept because that edition is published and
citable. Stateless tolerance read **61.0%** there (6,305 of 10,336), against 80.9% in 2026-08-03.
**That fall was roughly half real and half an artefact, and the artefact had one cause.**

**Amended 2026-09-12, corrected 2026-09-14.** The 09-12 amendment added a second namespace,
`io.github.cyanheads`, and called it contaminated. **That was wrong, and the 2026-09-13 census
disproves it.** cyanheads read 0.0% stateless again, four days later, in an independent scan: a
reading that persists across two censuses is a change, not an outage. The claim was inferred from a
pattern (verdicts steady while one signal collapses) without the second reading that alone can tell
the two apart, which is exactly the mistake the drift detector was built to stop. It is corrected
below rather than quietly removed. The records have never been edited.

`io.github.pipeworx-io` registers 1,320 servers, 4.5% of the registry. Between the two censuses its
health verdicts barely moved (1,286 healthy to 1,295) and every one of its servers carries the same
readiness verdict, `needs-migration`, in both. But its `statelessAccepted` readings flipped
wholesale:

| | 2026-08-03 | 2026-09-09 |
|---|---|---|
| statelessAccepted true | 1,277 | 270 |
| statelessAccepted false | 34 | 1,017 |

Those 1,014 servers are 48.3% of this edition's entire "not stateless, no session required, header
enforcement unknown" cohort. On a re-probe the following day, 501 of the 1,014 answered
`statelessAccepted: true` again and 118 did not answer at all; the equivalent re-probe of the other
1,086 servers in that cohort flipped only 12. The operator's endpoints were unstable during the
scan window, and a 17-hour window recorded that instability as a property of the servers.

`io.github.cyanheads` was named here on 2026-09-12 as a second contaminated namespace. It is not
one. Its stateless readings went 93.2% (2026-08-03) to 0.0% (2026-09-09) and **0.0% again on
2026-09-13**, an independent scan four days later that took 9 hours rather than 17. A transient
outage does not reproduce exactly, twice, across separate scan windows. cyanheads turned something
off between the first two censuses and has left it off.

The signature that prompted the error was real but not sufficient: its 125 servers keep identical
verdicts across all three editions (118 healthy, 6 degraded, 1 dead) while one signal goes to zero.
**A configuration change produces that signature just as cleanly as an outage does.** Only a second
reading separates them, and cyanheads never had one. That is the whole reason the drift detector
re-probes rather than just flags.

What pipeworx-io did is now clearer too. Its stateless rate ran 97.4%, then 21.0%, then **0.0% on
2026-09-13, where a same-day re-probe agreed on 1,294 of the 1,294 servers that answered.** So the
direction was real and the operator has withdrawn stateless support; what the 17-hour window added
was noise on top of a genuine change, caught mid-transition. The 2026-09-10 re-probe finding 501 of
1,014 reading true again is what marks the 21.0% specifically as unstable.

**The record is published exactly as observed. Nothing has been corrected after the fact.** What
follows is the control a reader needs, computed by removing the one namespace whose reading that
edition is known to be unstable:

| Stateless tolerance | 2026-08-03 | 2026-09-09 |
|---|---|---|
| all verdict-bearing servers | 80.9% | 61.0% |
| excluding `io.github.pipeworx-io` | 75.8% | 66.7% |

Read the second row. Stateless tolerance did fall, by about 9 points rather than 20. **Do not quote
61.0% as an ecosystem trend**; quote it as what this census measured, with the operator named. Do
not exclude cyanheads: it really did change, and removing it would understate a real fall.

The detector also flagged five further namespaces on the health signal rather than the stateless
one, the largest being `io.github.Evozim` (375 servers, healthy 99.0% to 26.4%). They are not in
the control above: their health verdicts moved with their signals, which is what a population that
really changed looks like. They are recorded so a later reader knows they were examined rather than
missed, and on the evidence now available the same is true of cyanheads.

Ruleset status: the readiness rules were compiled from the 2026-07-28 draft changelog while the
specification was still a release candidate, which is what the `2026-07-28-rc` ruleset version
records. **The rules were re-diffed against the final specification text on 2026-08-03, and every
verdict-bearing rule is unchanged.** This edition is the first scanned with the ruleset stamped
final: every record carries `2026-07-28`, not `2026-07-28-rc`. The rules themselves did not
change, so verdicts remain comparable across the two stamps. All eighteen breaking changes and deprecations the ruleset
encodes appear in the final changelog with the same SEP or PR number and the same substance; the
error-code renumbering landed exactly as anticipated (HeaderMismatch `-32001` to `-32020`,
MissingRequiredClientCapability `-32003` to `-32021`, UnsupportedProtocolVersion `-32004` to
`-32022`), and the probe already accepts both numbers, so which one a server returns dates its
build rather than changing its verdict. The final text removes one thing the ruleset does not
list, `notifications/elicitation/complete` and the URL-mode `elicitationId` field; like the rest
of elicitation it only occurs during tool execution, which this probe never performs, so it is
not keylessly observable and bears on no verdict.

No published verdict therefore needs recomputation. Censuses stamped `2026-07-28-rc` keep that
stamp, because it names the ruleset revision that actually computed them; rewriting a published
record to look tidier is precisely the edit this dataset must never make. The suffix is a
statement about when the rules were compiled, not a caveat about whether they are right. Scans
from akashi `b1c497b` onward stamp `2026-07-28`; this edition was scanned with akashi
`v0.3.0-11-g69de215`.

Should a future revision ever change a rule, the raw observables are persisted per server, so
verdicts recompute without a re-scan.
