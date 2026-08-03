# Methodology

How the census is drawn, what each probe does, and exactly what the numbers do and do not mean.
The whole value of this dataset is that the method is honest and visible, so this document is
part of the product.

## Denominator

- **14,559 servers**, drained from the official registry (`registry.modelcontextprotocol.io`) at
  their latest version, on **2026-07-02**. The registry is growing: it held 13,886 servers on
  2026-06-26.
- Every rate in this document and in `summary.json` is computed over all 14,559 servers unless
  stated otherwise. 169 of those servers declared only entrypoints that cannot be probed without a
  key (`unknown`, see Definitions) and are excluded from no rate; we report the all-14,559 figure
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

## Headline

| Verdict | Count | Share |
|---|---|---|
| healthy | 11,582 | 79.6% |
| degraded | 2,212 | 15.2% |
| dead | 596 | 4.1% |
| unknown | 169 | 1.2% |

Dead-or-broken (degraded + dead): 2,808, about 1 in 5. Fully dead: 596, about 1 in 25.

## Per-layer findings

Failure is not evenly spread across a server's declared entrypoints.

- **Packages.** Of about 9,029 probeable npm/PyPI/OCI package entrypoints, about 86 are broken,
  roughly 1%. A further 971 declared package entrypoints are OCI images on private registries that
  cannot be probed keylessly (these are part of the `unknown` cohort, not counted as broken).
- **Repositories.** Of 12,308 declared GitHub repos, 1,621 (13.2%) return 404/gone, and 81 are
  archived.
  - **Honesty caveat:** a further 1,675 repo probes returned an ambiguous "error" response
    (possibly transient, possibly rate-limit related). Those are excluded from the 13.2% rotted
    figure entirely, rather than counted as either working or broken, so the figure is not
    inflated by noise akashi cannot yet disambiguate.
- **Hosted remotes.** Of 7,158 declared remote endpoints, 975 (13.6%) are down (unreachable, 5xx,
  or 404). Remote-bearing servers are the sickest cohort in the registry: **25.9% dead-or-broken
  and 7.5% fully dead**, against 4.1% dead overall. 1,762 endpoints are auth-gated: alive, but not
  keyless-verifiable, and never treated as broken on that basis alone. 50 endpoints answer HTTP 200
  but are not MCP servers (an impostor guard: something is listening, but it never completes an
  `initialize` handshake).
- **Conformance.** 3,914 remotes complete an MCP `initialize` handshake. Of the servers on which a
  full session was opened, 3,719 successfully resolved `tools/list`, the strongest keyless proof
  that a server actually works end to end.
- **`server.json` validity.** 305 servers (2.1%) publish a manifest that fails validation against
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
  state at any time; the census is a snapshot dated 2026-07-02, not a live status.
- Keyless probing cannot verify anything behind authentication. Auth-gated remotes are scored
  alive-but-unverified, never broken, so the true health of that cohort could be better or worse
  than what a keyless probe can see.
- `tools/list` completion is informational and does not affect a verdict, since a valid MCP server
  may advertise no tools.

## Spec-readiness pass (2026-07-22 edition)

The 2026-07-22 census adds a readiness dimension for the 2026-07-28 specification revision,
measured with akashi's readiness pass (akashi commit d54c52d, ruleset `2026-07-28-rc`). It runs
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
the `sessionRequired` observable without carrying the session reason. In this edition 544 servers
carry the flag and 528 cite it as their reason. Quote the observable (`sessionRequired`) when
describing what servers do, and the reason count only when describing why a verdict was assigned.
The same caution applies to every other signal that also appears as a reason string.

Denominator rules: readiness percentages are computed ONLY over servers with a readiness verdict
(4,937 in this edition), never over the registered total. A registered-total denominator may only
carry declared metadata (for example the server.json schema-version distribution), never runtime
readiness.

Release-candidate caveat: the 2026-07-28 specification is a release candidate until its
publication day. The probe accepts both the RC and the renumbered final error codes for the
routing-header check, and every verdict records the ruleset version that produced it. The rules
will be re-diffed against the final specification text when it publishes; because the raw
observables are persisted per server, a rule change recomputes verdicts without a re-scan.
