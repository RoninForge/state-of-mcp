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
| **`2026-08-03`** (current) | 2026-08-03 | **19,804** |

## Denominator

- **19,804 servers**, drained from the official registry (`registry.modelcontextprotocol.io`) at
  their latest version, on **2026-08-03**. The registry is growing fast: 13,886 on 2026-06-26,
  14,559 on 2026-07-02, 18,032 on 2026-07-22, so this edition is +9.8% in twelve days.
- Every rate in this document and in `summary.json` is computed over all 19,804 servers unless
  stated otherwise. 187 of those servers declared only entrypoints that cannot be probed without a
  key (`unknown`, see Definitions) and are excluded from no rate; we report the all-19,804 figure
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

## Headline (2026-08-03 edition)

| Verdict | Count | Share |
|---|---|---|
| healthy | 15,974 | 80.7% |
| degraded | 2,792 | 14.1% |
| dead | 851 | 4.3% |
| unknown | 187 | 0.9% |

Dead-or-broken (degraded + dead): 3,643, about 1 in 5. Fully dead: 851, about 1 in 23.

Health rates have been stable across all three editions while the registry grew 36% (healthy
79.6%, 79.7%, 80.7%). The ecosystem is adding servers faster than it is rotting them, but the
dead-or-broken share has not moved: roughly one registered server in five does not fully work,
edition after edition.

## Per-layer findings (2026-08-03 edition)

Failure is not evenly spread across a server's declared entrypoints. Package and remote counts
below are ENTRYPOINTS, not servers; repository and conformance-server counts are servers. Never
mix the two in one rate.

- **Packages.** Of 11,498 probeable npm/PyPI/OCI package entrypoints, 97 are broken, 0.8%. A
  further 1,330 declared package entrypoints are OCI images on private registries that cannot be
  probed keylessly (these are part of the `unknown` cohort, not counted as broken).
- **Repositories.** Of 15,997 declared repos, 2,083 (13.0%) return 404/gone, and 152 are archived,
  so 2,235 (14.0%) are rotted.
  - **Honesty caveat, and it is larger this edition than last.** A further 1,630 repo probes
    returned an ambiguous "error" response (possibly transient, possibly rate-limit related).
    Those are excluded from the 14.0% figure entirely, rather than counted as either working or
    broken, so the figure is not inflated by noise akashi cannot yet disambiguate. But 1,630 is
    10.2% of the bearing population, against only 210 (1.4%) in the 2026-07-22 edition, so this
    edition's rotted rate is much less precise: the true value lies somewhere in **[14.0%,
    24.2%]**. **Do not read the drop from 15.8% to 14.0% as repositories getting healthier.** It
    sits well inside the ambiguity band and is more likely an artefact of how many GitHub probes
    came back ambiguous during this run.
- **Hosted remotes.** Of 10,376 declared remote endpoints, 1,401 (13.5%) are down (unreachable,
  5xx, or 404). Remote-bearing servers remain the sickest cohort in the registry: **24.1%
  dead-or-broken and 7.6% fully dead**, against 4.3% dead overall. 2,716 endpoints are auth-gated:
  alive, but not keyless-verifiable, and never treated as broken on that basis alone. 63 endpoints
  answer HTTP 200 but are not MCP servers (an impostor guard: something is listening, but it never
  completes an `initialize` handshake).
- **Conformance.** 5,582 remote endpoints complete an MCP `initialize` handshake, spread over
  5,515 servers. 5,353 endpoints successfully resolved `tools/list`, the strongest keyless proof
  that a server actually works end to end.
- **`server.json` validity.** 367 servers (1.9%) publish a manifest that fails validation against
  its own declared JSON Schema.

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

## Spec-readiness pass (2026-08-03 edition)

Censuses from 2026-07-22 onward carry a readiness dimension for the 2026-07-28 specification
revision, measured with akashi's readiness pass (ruleset `2026-07-28-rc`; akashi `d54c52d` for
the 2026-07-22 edition, `b1c497b` for 2026-08-03). It runs
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
the `sessionRequired` observable without carrying the session reason. In the 2026-08-03 edition
546 servers carry the flag and 529 cite it as their reason (2026-07-22: 544 and 528). Quote the
observable (`sessionRequired`) when describing what servers do, and the reason count only when
describing why a verdict was assigned. The same caution applies to every other signal that also
appears as a reason string.

Denominator rules: readiness percentages are computed ONLY over servers with a readiness verdict
(5,515 in this edition), never over the registered total. A registered-total denominator may only
carry declared metadata (for example the server.json schema-version distribution), never runtime
readiness.

### Findings (2026-08-03 edition, 5,515 servers with a verdict)

| Verdict | Count | Share | 2026-07-22 |
|---|---|---|---|
| ready | **6** | 0.1% | 0 |
| needs-migration | 4,798 | 87.0% | 4,225 (85.6%) |
| at-risk | 711 | 12.9% | 712 (14.4%) |

**The first spec-ready servers exist.** Twelve days earlier not one server in the registry passed
all four required-conformance signals; six now do. That is 0.1% of the conformant population, so
the honest reading is that migration has started, not that it has progressed.

The leading indicators moved in the same direction and by more, which is what makes the six look
like the front of a wave rather than noise: `server/discover` implemented 37 to 75, routing-header
enforcement 39 to 82, three-of-four `nearReady` 40 to 71. Each roughly doubled while the
conformant population grew 12%.

At-risk is flat in absolute terms (712 to 711) while its share fell from 14.4% to 12.9%. Nothing
is being fixed on that surface; new arrivals are simply landing elsewhere. Stateless tolerance,
already the strong point, held at 80.9% (was 81.2%).

**The error-code split is the sharpest migration signal in this edition, and it exists only
because the probe accepts both numbers.** The 2026-07-28 renumbering moved `HeaderMismatch` from
`-32001` to `-32020`. In the 2026-07-22 census all 39 enforcing servers returned `-32001`, the RC
code. In this edition 45 return `-32020` and 37 still return `-32001`. All six ready servers are
in the `-32020` group. A probe that had accepted only one of the two numbers would have reported
either the old cohort or the new one, and missed that a final-spec cohort appeared at all.

Ruleset status: the readiness rules were compiled from the 2026-07-28 draft changelog while the
specification was still a release candidate, which is what the `2026-07-28-rc` ruleset version
records. **The rules were re-diffed against the final specification text on 2026-08-03, and every
verdict-bearing rule is unchanged.** All eighteen breaking changes and deprecations the ruleset
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
from akashi `b1c497b` onward stamp `2026-07-28`.

Should a future revision ever change a rule, the raw observables are persisted per server, so
verdicts recompute without a re-scan.
