# State of MCP

[![Data: CC BY 4.0](https://img.shields.io/badge/data-CC%20BY%204.0-informational)](DATA-LICENSE.md)
[![Cite this](https://img.shields.io/badge/cite-CITATION.cff-informational)](CITATION.cff)

**A dated, keyless health and conformance census of the official Model Context Protocol
registry.** Every server in `registry.modelcontextprotocol.io` is checked with the same public,
unauthenticated probes and given a verdict: healthy, degraded, dead, or unknown. The result is a
snapshot, not a leaderboard: what fraction of the registry actually works, on a given date, and
where it breaks.

## Headline (2026-07-22): the 2026-07-28 spec-readiness edition

Twenty days after the first census the registry has grown 24%, from 14,559 to **18,032 servers**,
and the health rates barely moved: 79.7% healthy, 15.8% degraded, 4.0% dead, 0.5% unknown. The
growth is real servers, not rot.

This edition adds a **2026-07-28 spec-readiness pass**, run six days before the specification's
publication date against its release candidate. Of the **4,937 servers reachable through a
conformant keyless `initialize`** (the only population whose runtime behavior can be measured):

| Readiness verdict | Count | Share of reachable |
|---|---|---|
| ready | 0 | 0.0% |
| needs-migration | 4,225 | 85.6% |
| at-risk (on a removed or deprecated surface) | 712 | 14.4% |

- **Zero servers are fully ready**: none passes all four required-conformance signals
  (`server/discover`, handshake-free stateless calls, session independence, routing-header
  enforcement). 37 implement `server/discover`, 39 enforce routing headers, and 40 servers are
  exactly one condition away.
- **The counterweight: 81.2% (4,010) already tolerate handshake-free stateless calls**, the
  heart of the new revision. The ecosystem is closer to the stateless core than the zero
  suggests.
- 805 servers (16.3%) still mint the removed `Mcp-Session-Id`; 528 hard-require it and cannot
  serve new-spec clients without change.
- Negotiated protocol versions: 2,799 on 2025-06-18, 1,536 on 2025-03-26, 528 still on
  2024-11-05, 61 on 2025-11-25, and exactly one server already answering 2026-07-28.

Readiness percentages use ONLY the reachable-remote denominator (4,937), never the 18,032
registered total; a package-only server has no runtime endpoint to measure. Verdicts were
computed under ruleset `2026-07-28-rc` (the release candidate); they will be re-verified against
the final specification text on its 2026-07-28 publication day. Full rules in
[METHODOLOGY.md](METHODOLOGY.md).

## Headline (2026-07-02)

We checked all 14,559 servers in the official registry. Most are not dead: about 80% are
healthy, roughly 1 in 5 is dead or broken, and only about 1 in 25 is fully dead.

| Verdict | Count | Share |
|---|---|---|
| healthy | 11,582 | 79.6% |
| degraded | 2,212 | 15.2% |
| dead | 596 | 4.1% |
| unknown | 169 | 1.2% |

Dead-or-broken (degraded + dead): 2,808, about 19%. Fully dead: 596, about 4%.

## Failure concentrates differently per layer

The overall rate hides where things actually break. Checked separately by entrypoint:

- **Packages are the most reliable layer.** Of about 9,029 probeable npm/PyPI/OCI package
  entrypoints, about 86 are broken, roughly 1%.
- **Repository links rot.** Of 12,308 declared GitHub repos, 1,621 (13.2%) return 404/gone and 81
  are archived. A further 1,675 repo probes returned an ambiguous error (possibly transient or
  rate-limit related) and are excluded from that rotted figure rather than counted as broken.
- **Hosted remotes are the fragile layer.** Of 7,158 declared remote endpoints, 975 (13.6%) are
  down (unreachable, 5xx, or 404). Remote-bearing servers are the sickest cohort: 25.9%
  dead-or-broken and 7.5% fully dead, against 4.1% overall. 1,762 endpoints are auth-gated (alive,
  but not keyless-verifiable). 50 answer HTTP 200 but are not MCP servers.
- **Conformance, one layer deeper than an uptime ping:** 3,914 remotes complete an MCP
  `initialize` handshake; of the servers we opened a full session on, 3,719 successfully resolved
  `tools/list` over the official go-sdk client, the strongest keyless proof a server actually
  works. 305 servers (2.1%) publish a `server.json` that fails validation against its own declared
  JSON Schema.

Full definitions, denominators, and the honesty caveats behind these numbers are in
[METHODOLOGY.md](METHODOLOGY.md).

## What is in here

```
data/censuses/<date>/records.jsonl   one probe result per server, one JSON line each
data/censuses/<date>/summary.json    verdict counts and rates, segments, run parameters
```

- `records.jsonl` - the raw census. Each line is a single server's result: its registry name,
  verdict, the individual checks that ran (registry status, server.json validity, repo
  reachability and freshness, package publish status, remote reachability, MCP conformance), and
  the underlying signals behind each check. Every line is independently reproducible: it is
  byte-identical to running `akashi check <server> --json` against that one server.
- `summary.json` - the rolled-up counts and rates behind the headline, a remote-bearing segment
  breakdown, name-validation findings, and the run's reproducibility parameters (registry URL,
  akashi version, concurrency, timeout, start and finish time).

There are two censuses so far: `2026-07` (the 2026-07-02 baseline) and `2026-07-22` (the
spec-readiness edition, whose records add a `readiness` object per reachable server). Each
census lands in its own dated `data/censuses/<date>/` directory; the dataset accumulates,
never overwrites.

## How this is produced

Every row is generated by [`akashi`](https://github.com/RoninForge/akashi) (v0.3.0), a small
open-source CLI that verifies an MCP server is alive and conformant using only public,
unauthenticated signals: the official registry, the public GitHub API, npm, PyPI, anonymous
Docker Hub, and a capability-only MCP `initialize`/`tools/list` exchange. It authenticates to no
probed server and runs none of its tools. A GitHub token, if present, is used only against the
public GitHub API to raise the rate limit, exactly as a human running `gh` would.

This census was produced with:

```sh
akashi scan --out ./data/censuses/2026-07
```

Because the check set is keyless and deterministic, any single row in `records.jsonl` is
re-checkable by hand:

```sh
akashi check <server-name>
```

## Read it on the web

A browsable, searchable version of this dataset, with a per-server detail page and an embeddable
"verified on DATE" badge, lives at
**[roninforge.org/data/state-of-mcp/](https://roninforge.org/data/state-of-mcp/)**.

## License

- **Data license:** CC BY 4.0 (see [DATA-LICENSE.md](DATA-LICENSE.md)). Use it anywhere, just
  attribute.
- **Tooling license:** MIT (see [LICENSE](LICENSE)).
- Maintained by [RoninForge](https://roninforge.org).

## How to cite

Machine-readable citation metadata is in [`CITATION.cff`](CITATION.cff) (GitHub renders a "Cite
this repository" button from it).

Plain:

> State of MCP by RoninForge (https://github.com/RoninForge/state-of-mcp), CC BY 4.0. Census
> `2026-07`, accessed `<date>`.

## Scope and honesty

This is a measurement of health and protocol conformance, not a security audit, not a
vendor ranking, and not a trend line. This is the first census; there is no history yet to say
whether the registry is getting better or worse. See [METHODOLOGY.md](METHODOLOGY.md) for exactly
what is and is not claimed.
