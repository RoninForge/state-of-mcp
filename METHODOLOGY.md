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
| **`2026-09-09`** (current) | 2026-09-09 | **29,522** |

## Denominator

- **29,522 servers**, drained from the official registry (`registry.modelcontextprotocol.io`) at
  their latest version, on **2026-09-09**. The registry is growing fast: 13,886 on 2026-06-26,
  14,559 on 2026-07-02, 18,032 on 2026-07-22, 19,804 on 2026-08-03, so this edition is +49.1% in
  thirty-seven days, the largest jump we have recorded.
- Growth is concentrated, and a reader should know that before reading any rate as an ecosystem
  trend. Two namespaces account for 19.4% of the net growth on their own: `io.github.mcp-dir` 97 to
  1,113 (+1,016) and `io.github.sadri-dridi` 0 to 868. Excluding `io.github.mcp-dir` alone, the
  registry grew 44.2% rather than 49.1%.
- Every rate in this document and in `summary.json` is computed over all 29,522 servers unless
  stated otherwise. 158 of those servers declared only entrypoints that cannot be probed without a
  key (`unknown`, see Definitions) and are excluded from no rate; we report the all-29,522 figure
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

## Headline (2026-09-09 edition)

| Verdict | Count | Share | 2026-08-03 |
|---|---|---|---|
| healthy | 23,538 | 79.7% | 80.7% |
| degraded | 4,664 | 15.8% | 14.1% |
| dead | 1,162 | 3.9% | 4.3% |
| unknown | 158 | 0.5% | 0.9% |

Dead-or-broken (degraded + dead): 5,826, about 1 in 5. Fully dead: 1,162, about 1 in 25.

Health rates have been stable across all four editions while the registry grew 103% from the first
(healthy 79.6%, 79.7%, 80.7%, 79.7%). The ecosystem is adding servers faster than it is rotting
them, but the dead-or-broken share has not moved: roughly one registered server in five does not
fully work, edition after edition. That is now four editions and a doubled registry without the
ratio shifting, which makes it the most durable finding in this dataset.

## Per-layer findings (2026-09-09 edition)

Failure is not evenly spread across a server's declared entrypoints. Package and remote counts
below are ENTRYPOINTS, not servers; repository and conformance-server counts are servers. Never
mix the two in one rate.

- **Packages.** Of 14,481 declared npm/PyPI/OCI package entrypoints, 186 are broken, 1.3% (was
  0.8%). A further 2,001 are OCI images on private registries that cannot be probed keylessly
  (part of the `unknown` cohort, not counted as broken).
- **Repositories.** Of 22,642 declared repos, 3,388 (15.0%) return 404/gone and 224 are archived,
  so 3,612 (16.0%) are rotted.
  - **Ambiguity band, and it is much narrower this edition.** A further 933 repo probes returned
    an ambiguous "error" response. Those are excluded from the 16.0% figure entirely rather than
    counted as either working or broken, so the true value lies in **[16.0%, 20.1%]**. The
    equivalent band last edition was [14.0%, 24.2%], because 1,630 probes (10.2% of the bearing
    population) came back ambiguous against 933 (4.1%) here. **The apparent rise from 14.0% to
    16.0% is not evidence that repositories rotted faster.** Both editions' bands overlap heavily;
    what actually improved is the precision of the estimate, not the health of the repos.
- **Hosted remotes.** Of 17,860 declared remote endpoints, 1,923 (10.8%) are down (unreachable,
  5xx, or 404), improved from 13.5%. Remote-bearing servers remain the sickest cohort in the
  registry: **24.5% dead-or-broken and 5.8% fully dead**, against 3.9% dead overall. 4,211
  endpoints are auth-gated: alive, but not keyless-verifiable, and never treated as broken on that
  basis alone.
  - **363 endpoints answer HTTP 200 but never complete an `initialize` handshake**, against 63 last
    edition. That is 5.8x growth in something listening at an MCP address that is not an MCP
    server, while remote endpoints themselves grew 1.7x. It is the fastest-moving negative signal
    in this edition.
- **Conformance.** 10,412 remote endpoints complete an MCP `initialize` handshake, spread over
  10,336 servers. 9,034 servers successfully resolved `tools/list`, the strongest keyless proof
  that a server actually works end to end.
- **`server.json` validity.** 386 servers (1.3%) publish a manifest that fails validation against
  its own declared JSON Schema, improved from 1.9%.

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
- **A census is no longer a snapshot, and this edition proves it.** The 2026-09-09 scan ran
  16h59m against 4h47m for 2026-08-03: 1.49x the servers but 2.4x the time per server, because
  remote-bearing endpoints grew 9,957 to 17,355 and remote probes are the slow ones. Over a window
  that long a single operator's outage lands in the dataset as if it were a property of its
  servers. It did, in this edition; see the readiness findings below. Rates that depend on a
  runtime handshake should be read with that window in mind, and a large single-namespace cohort
  should be checked before any such rate is read as an ecosystem trend.

## Spec-readiness pass (2026-09-09 edition)

Censuses from 2026-07-22 onward carry a readiness dimension for the 2026-07-28 specification
revision, measured with akashi's readiness pass. Each edition carries the ruleset stamp that
actually computed it: `2026-07-28-rc` for 2026-07-22 (akashi `d54c52d`) and 2026-08-03 (akashi
`b1c497b`), `2026-07-28` for 2026-09-09 (akashi `v0.3.0-7-g15983e0`). The rules are identical
across the two stamps; see Ruleset status below. It runs
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
the `sessionRequired` observable without carrying the session reason. In this edition 1,694
servers carry the flag and 1,676 cite it as their reason (2026-08-03: 546 and 529; 2026-07-22: 544
and 528). Quote the
observable (`sessionRequired`) when describing what servers do, and the reason count only when
describing why a verdict was assigned. The same caution applies to every other signal that also
appears as a reason string.

Denominator rules: readiness percentages are computed ONLY over servers with a readiness verdict
(10,336 in this edition), never over the registered total. A registered-total denominator may only
carry declared metadata (for example the server.json schema-version distribution), never runtime
readiness.

### Findings (2026-09-09 edition, 10,336 servers with a verdict)

| Verdict | Count | Share | 2026-08-03 |
|---|---|---|---|
| ready | **29** | 0.3% | 6 (0.1%) |
| needs-migration | 8,426 | 81.5% | 4,798 (87.0%) |
| at-risk | 1,881 | 18.2% | 711 (12.9%) |

**The ready cohort grew from 6 to 29 while the conformant population grew 87%.** Migration is
moving faster than the population it moves through, which is the first edition where that is true.
It is still 0.3% of conformant servers, so the honest reading remains that migration has started.

The leading indicators moved harder than the verdicts. `server/discover` implemented went 75 to
393, a 5.2x rise. Final-code routing-header enforcement went 45 to 242, 5.4x. Neither is affected
by the outage described below.

**The error-code split is now decisive.** The 2026-07-28 renumbering moved `HeaderMismatch` from
`-32001` to `-32020`. In the 2026-07-22 census all 39 enforcing servers returned the RC code; in
2026-08-03 it was 45 on `-32020` against 37 still on `-32001`; here it is **242 against 38**. The
old-code cohort is frozen in absolute terms (37, then 38) while the final-code cohort quintupled.
Servers are not migrating off `-32001`; new servers are simply being built on `-32020`. A probe
accepting only one of the two numbers would have missed this entirely.

**At-risk nearly tripled, 711 to 1,881, and its share rose 12.9% to 18.2%.** The driver is a single
reason: `requires the removed Mcp-Session-Id session mechanism`, 529 to 1,676. That is a server-side
observable, not a probe failure, and it is the one health-adjacent number in this edition moving
clearly in the wrong direction.

#### Disclosure: a single-operator outage sits inside this edition's stateless rate

Stateless tolerance reads **61.0%** (6,305 of 10,336), against 80.9% in 2026-08-03. **That fall is
roughly half real and half an artefact, and the artefact has one cause.**

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

**The record is published exactly as observed. Nothing has been corrected after the fact.** What
follows is the control a reader needs, computed by removing that one namespace from both editions:

| Stateless tolerance | 2026-08-03 | 2026-09-09 |
|---|---|---|
| all verdict-bearing servers | 80.9% | 61.0% |
| excluding `io.github.pipeworx-io` | 75.8% | 66.7% |

Read the second row. Stateless tolerance did fall, by about 9 points rather than 20. **Do not quote
61.0% as an ecosystem trend**; quote it as what this census measured, with the operator named.

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
`v0.3.0-7-g15983e0`.

Should a future revision ever change a rule, the raw observables are persisted per server, so
verdicts recompute without a re-scan.
