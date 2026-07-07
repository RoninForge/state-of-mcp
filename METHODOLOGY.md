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
