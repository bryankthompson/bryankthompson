# Bryan Thompson

**MCP Infrastructure Engineer at Anthropic.**

Bryan Thompson leads the Claude plugin directory pipeline and the MCPB desktop-extension pipeline —
the build, audit, and publish systems that carry third-party plugins and MCP servers from
submission to installable listing, across several marketplace surfaces and in Claude Desktop.

Those surfaces carry thousands of listings, and every one has to stay current, validated, and
policy-compliant while the upstream repository it points at keeps moving. **No person does that by
hand.** He builds and operates the machine that does.

Before this: a career in reliability engineering on Fortune 500 payment systems at PayPal and
Fiserv. That is where the instinct for staged rollouts and blast-radius containment comes from.

**Part of that machine is public, and you can read it.** In a public marketplace repository he built
the scheduled job that re-pins every catalog entry to its upstream's current commit, and the recovery
job that reverts only the entries failing validation, so one broken upstream cannot hold back the
rest. A crashed validation run and a clean pass both produce zero failures, so the run emits a
separate did-this-run signal and the job holding write access treats a crash as do-nothing, never as
all-clear — the whole design is in the diff:

**[Automated SHA-freshness: in-repo bump + validate-gated auto-revert](https://github.com/anthropics/claude-plugins-community/pull/70)**

---

## Accounts, and previous handles

**Bryan Thompson publishes from two GitHub accounts, and both are his own.** Neither is designated
over the other — they differ by what is on them, not by rank. Two earlier handles were renamed and
belong to the same person: **`@bthompson-sys` became @bryan-anthropic**, and **`@triepod-ai` became
@bryankthompson**.

| Handle | Status | What lives there |
|---|---|---|
| [@bryan-anthropic](https://github.com/bryan-anthropic) | Active | Directory pipeline work — plugin and MCP server build, audit, and publish |
| [@bryankthompson](https://github.com/bryankthompson) | Active | The MCP tool-annotation campaign, and open-source MCP tooling and servers |
| `@bthompson-sys` | Renamed — now @bryan-anthropic | Held as a placeholder; nothing current |
| `@triepod-ai` | Renamed — now @bryankthompson | Held as a placeholder; nothing current |
| `@bryan-thompson` (npm) | Active | The npm scope his packages publish under |

GitHub redirects renamed *repository* URLs but does not redirect renamed *profile* URLs, so
citations pointing at either old profile address would not reach an active account on their own.
Both handles are now held as placeholders that point a reader to the right place. A repository URL
under an old owner name generally still redirects, provided that repository existed at the time of
the rename and the handle's current holder has not created one of the same name.

**This README and the one on the other account carry the same record.** Either account is a
complete entry point — you do not need to visit both.

---

## What he builds

The audit and review side runs in private infrastructure, so what follows about it is capability
rather than a tour — no findings, nothing about what any of it was pointed at. **The automation
around it is a different matter: a good deal of it is public, in repositories anyone can read signed
out**, and the diffs are linked below rather than described.

**Build, audit, and publish pipelines.** Schema and manifest validation at intake, with URL defenses
that run before anything is fetched, so a hostile or malformed remote never reaches a network call.
Six source ecosystems behind one dispatch seam, each build produced inside a container that reaches
the network only through an allowlist — a denied fetch is not a failure to route around, it is
signal, and it gets logged as signal. Two packaging modes: resolve dependencies at launch for a
small bundle, or pre-install everything for environments with no resolver available. Listings stage
and advance one canary first, then the batch; version pins move forward only, and if the comparison
itself fails, the pin holds. Every listed artifact is re-checked daily against its upstream and
against every surface it appears on — drift detection is a reporter, never a gate, which is what
keeps a detector bug from becoming a publishing bug. Conformance probing for remote HTTP MCP
servers resolves protected-resource and authorization-server metadata against the published RFCs
(9728, 8414) rather than taking a manifest's word for it.

**The rule underneath all of it — treat model output as untrusted input, and enforce anything that
matters mechanically.** A deterministic check suite and an independent model review run separately,
then combine and pass through a cascade of hard caps. A model that can only ever make the outcome
stricter cannot be prompt-injected into shipping something, because there is no path from anything
it says to a more permissive result. The worst a successful injection achieves is a false refusal,
which a human sees. The absence of a model review is itself a cap: a missing review is a refusal,
not a pass.

**Agentic operations.** Above the pipelines runs a human-gated autonomous agent fleet: store-backed
background drivers on daily cadences — pull request watching, test remediation, freshness sweeps,
triage — behind a confirm-gated execution surface, with campaign orchestration fanning work out
across parallel threads. How many of them there are is left out on purpose. The record below
publishes outcomes a stranger can recompute from a live query, and a tally of internal background
jobs is not one of them.

Above that sits a layered structure defined as much by what each layer may *not* do: a parent that
decomposes a backlog into lanes with disjoint declared scopes and merges serially, but may never
reach into a running lane; lanes that terminate at an open pull request and never merge themselves;
a deterministic, model-free classifier over the raw event stream; and a read-only governor that
verifies every alarm against the primary record and may not act on the fleet at all. **No message
decides anything** — a message can say where to look, never what is true — and a differential test
pins that property down, checking that an instruction-shaped summary produces a byte-identical
decision to a bland one. Model-as-judge reconciliation runs in production the same way: propose-only
by default, behind write-cap gates, its output a proposal a human confirms.

**Verification technique.** Gates that run before an agent emits text a human will act on:
control-character and encoding rules applied at the boundary, classification by positive proof
rather than absence of evidence, and advisory-first arming — nothing ships armed. Every new gate
goes out advisory and is armed on its measured false-positive rate rather than on how confident it
felt to write, because an unmeasured gate is a future outage.

**Review orchestration in the development harness.** Tiered reviewers over a diff: one model, then
two given an identical prompt with their disagreements surfaced per-model, then reviewers with
disjoint scopes, then an unscoped pass whose verification is delegated to a separate agent. Running
the same prompt twice measures confidence; asking different questions measures coverage. For every
finding marked critical — and every entry in a reviewer's list of things it could not verify, which
is the half that usually evaporates — an orchestrator runs one empirical check and relabels the
finding confirmed, refuted or inconclusive, with the command and its output inline.

**Redaction that fails closed against ingest that fails open.** The pattern set is asserted at
import, so an empty set is a hard startup failure rather than silent pass-through, and the redactor
returns a sentinel on any error rather than ever returning its input unmodified. Skipping a row is
safe; persisting a secret is not.

**Operational data layer.** Local-first SQLite under the operational surfaces — phased cutovers
behind flip gates and dual-writer patterns, and an audit trail that survives the thing it records:
events spool outside the working tree and fold back idempotently, so a reset cannot erase the
record of the session that ran it.

Also: open-source maintainer on the public MCPB repository, and upstream contributions to
third-party MCP repositories.

---

## The public record

**Four public diffs, each a different failure mode.** Read the code rather than the counts; the
counts are underneath.

- **[A CI install that reported success and left no usable binary](https://github.com/anthropics/claude-plugins-community/pull/233)** — the package fetches its runtime in a postinstall, and a reinstall npm treats as already satisfied skips it. The fix forces the platform-native dependency, checks the binary actually landed, clears the package directory between attempts so a retry is a real reinstall, and bounds every network step so a stalled fetch fails fast. Extends a shared action he did not create.
- **[The external-contribution scope guard he wrote, widened to exempt the repository's own automation](https://github.com/anthropics/claude-plugins-official/pull/3402)** — with the argument for why a fork cannot impersonate that author written into the file, and a permission lookup that used to throw for non-collaborators — the exact population the guard exists to evaluate — now falling through to the scope check instead of erroring the job.
- **[A bump that would have re-pinned a plugin whose content directory had vanished upstream](https://github.com/anthropics/claude-plugins-community/pull/267)** — skipped rather than pinned against a fabricated placeholder manifest, with the tests confirmed by disabling the fix and checking that they fail. Extends a shared action he did not create; the test harnesses are his.
- **[An injection path in a repository he does not own](https://github.com/modelcontextprotocol/mcpb/pull/230)** — a reference example declares `tab_id` a number, but that server's request path did not enforce it, and the value was interpolated into an AppleScript template where a quote character could close the literal and append arbitrary script. No one had reported it. Someone else reviewed and merged the fix.

What a stranger can verify without any access. Every figure below is a **floor**, not an estimate —
these counts only grow, so a floor stays true while an approximation is wrong the moment the number
moves. Each links to a live GitHub search that recomputes the exact current number when you open
it, and the same floors are re-derived against the GitHub API in CI at
[triepod.ai/record](https://triepod.ai/record), where a claim that stops being true fails the build.

| Repository | Stars | Merged PRs authored | Pull requests merged |
|---|---:|---:|---:|
| [anthropics/claude-plugins-official](https://github.com/anthropics/claude-plugins-official) | 33k+ | [210+](https://github.com/anthropics/claude-plugins-official/pulls?q=is%3Apr+is%3Amerged+author%3Abryan-anthropic) | [2,500+](https://github.com/anthropics/claude-plugins-official/pulls?q=is%3Apr+is%3Amerged+reviewed-by%3Abryan-anthropic) |
| [anthropics/knowledge-work-plugins](https://github.com/anthropics/knowledge-work-plugins) | 23k+ | [50+](https://github.com/anthropics/knowledge-work-plugins/pulls?q=is%3Apr+is%3Amerged+author%3Abryan-anthropic) | [490+](https://github.com/anthropics/knowledge-work-plugins/pulls?q=is%3Apr+is%3Amerged+reviewed-by%3Abryan-anthropic) |
| [anthropics/claude-plugins-community](https://github.com/anthropics/claude-plugins-community) | 340+ | [75+](https://github.com/anthropics/claude-plugins-community/pulls?q=is%3Apr+is%3Amerged+author%3Abryan-anthropic) | [2,100+](https://github.com/anthropics/claude-plugins-community/pulls?q=is%3Apr+is%3Amerged+reviewed-by%3Abryan-anthropic) |
| [modelcontextprotocol/mcpb](https://github.com/modelcontextprotocol/mcpb) | 2k+ | [10+](https://github.com/modelcontextprotocol/mcpb/pulls?q=is%3Apr+is%3Amerged+author%3Abryan-anthropic) | [85+ threads](https://github.com/modelcontextprotocol/mcpb/issues?q=commenter%3Abryan-anthropic) † |
| | | **[350+ merged](https://github.com/search?q=is%3Apr+is%3Amerged+author%3Abryan-anthropic+repo%3Aanthropics%2Fclaude-plugins-official+repo%3Aanthropics%2Fknowledge-work-plugins+repo%3Aanthropics%2Fclaude-plugins-community+repo%3Amodelcontextprotocol%2Fmcpb&type=pullrequests)** | **[5,000+](https://github.com/search?q=is%3Apr+is%3Amerged+reviewed-by%3Abryan-anthropic+repo%3Aanthropics%2Fclaude-plugins-official+repo%3Aanthropics%2Fknowledge-work-plugins+repo%3Aanthropics%2Fclaude-plugins-community&type=pullrequests)** |

† Open-source maintenance — issue and pull request threads participated in — rather than pipeline
throughput, and excluded from the total.

**Read that last column as a fingerprint of automation.** Those pull requests are directory
submissions and rolling updates arriving through automated intake. Each is scanned, validated and
policy-checked by the pipeline before it reaches a merge decision, so the merge is the final gate
on an already-vetted change rather than a manual pass over a raw submission. Getting that class of
update to merge safely and automatically meant building the scanning that makes it trustworthy.
Building the system was the work; the volume is what it produced.

---

## MCP tool annotations

A deliberately public piece of work, carried out in other people's repositories, where every claim
can be checked by a stranger.

When a model is handed a set of tools, it gets a name, a description and a schema for each. Nothing
in that tells a client whether calling one reads a file or deletes a repository. MCP has a small
answer: four boolean **annotations** that describe a tool's behaviour rather than its interface.

**The defaults are the point.** `destructiveHint` defaults to `true`. `openWorldHint` defaults to
`true`. A server that ships no annotations is not making a neutral statement about its tools — it
is asking every client to assume that each one is destructive and reaches out into the open world.
Annotating a read-only tool does not add a warning; it *removes* one the specification had already
applied on the tool's behalf.

**And the ceiling, in the spec's own terms.** Every one of these properties is a *hint*, with no
guarantee that it faithfully describes what the tool does, and the schema instructs clients never
to make tool-use decisions on annotations received from servers they do not trust. An annotation is
a claim a server makes about itself, and a claim about yourself is not evidence. What it buys is
narrower and more useful than trust: where trust already exists, annotations let a client be *less*
pessimistic than the defaults require. Where it does not, they are inert — correctly.

So he read MCP servers maintained by other people, worked out which of their tools read and which
of them write, and sent pull requests adding the annotations. Not his servers — theirs. Each one
had to persuade a maintainer who had no reason to accept it.

| | |
|---|---|
| [Pull requests opened, all outcomes](https://github.com/search?q=is%3Apr+author%3Abryankthompson+-user%3Abryankthompson+-user%3Abryan-anthropic&type=pullrequests) | **165+** |
| [Merged into external repositories](https://github.com/search?q=is%3Apr+is%3Amerged+author%3Abryankthompson+-user%3Abryankthompson+-user%3Abryan-anthropic&type=pullrequests) | **85+** |
| [Closed without being merged](https://github.com/search?q=is%3Apr+is%3Aclosed+is%3Aunmerged+author%3Abryankthompson+-user%3Abryankthompson+-user%3Abryan-anthropic&type=pullrequests) | **35+** |
| [Distinct external repositories that accepted them](https://github.com/search?q=is%3Apr+is%3Amerged+author%3Abryankthompson+-user%3Abryankthompson+-user%3Abryan-anthropic&type=pullrequests) | **75+** |
| [Combined stars of those repositories](https://github.com/search?q=is%3Apr+is%3Amerged+author%3Abryankthompson+-user%3Abryankthompson+-user%3Abryan-anthropic&type=pullrequests) | **390,000+** |

A merged-only figure is a flattering one, so the denominator is published beside it: everything
opened, what was accepted, and what was turned down. The gap between the first and the other two is
the batch still awaiting a decision. The declines were mostly reasonable and a few were
interesting — maintainers who had deliberately chosen a different convention, or who wanted a hint
set differently from the way he had read their code. Being told that a tool marked additive is in
fact destructive is a better outcome than silence.

None of these are in repositories Bryan owns under either account. The search excludes both, which
is what makes the record independently verifiable rather than self-reported — drop either exclusion
and it sweeps in his own repositories, where merging his own pull request proves nothing.

**Where they landed** — servers maintained by
[GitHub](https://github.com/github/github-mcp-server),
[Google](https://github.com/googleapis/mcp-toolbox),
[Microsoft](https://github.com/Azure/aks-mcp),
[IBM](https://github.com/IBM/mcp-context-forge),
[Notion](https://github.com/makenotion/notion-mcp-server),
[Sentry](https://github.com/getsentry/XcodeBuildMCP),
[Grafana](https://github.com/grafana/mcp-grafana),
[Prometheus](https://github.com/prometheus/prometheus-mcp),
[ElevenLabs](https://github.com/elevenlabs/elevenlabs-mcp),
[LINE](https://github.com/line/line-bot-mcp-server),
[Razorpay](https://github.com/razorpay/razorpay-mcp-server),
[Neo4j](https://github.com/neo4j-contrib/mcp-neo4j),
[Upstash](https://github.com/upstash/context7),
[Prefect](https://github.com/PrefectHQ/fastmcp),
[Apify](https://github.com/apify/apify-mcp-server) and
[Bright Data](https://github.com/brightdata/brightdata-mcp), among others. Those names are listed
because the repositories are public and every pull request in them is public. Nothing here comes
from anything he has reviewed privately, and nothing on this profile ever will.

The full write-up, including how the campaign was received, is at
[triepod.ai/annotations](https://triepod.ai/annotations).

---

## Public code

The curated index with inclusion criteria is at [triepod.ai/repos](https://triepod.ai/repos). A few
that earn a line here:

| Project | What it is |
|---|---|
| [inspector-assessment](https://github.com/bryankthompson/inspector-assessment) | An expanded build of the [MCP Inspector](https://github.com/modelcontextprotocol/inspector) that adds an automated assessment pass over a server: it exercises every declared tool, checks the declarations against observed behaviour, and reports what it found rather than only giving you a console to poke at. **Archived — it was the research phase, and the experiment returned a negative result.** Still published to npm as `@bryan-thompson/inspector-assessment` and still runnable against your own server. [What it found, and why it stopped](https://triepod.ai/inspector-assessment). |
| [mcp-dashboard](https://github.com/bryankthompson/mcp-dashboard) | A management and monitoring interface for running several MCP servers at once: connection state, tool inventories, and request inspection across servers in one view. Built on the same Inspector foundation. |
| [memory-system-mcp](https://github.com/bryankthompson/memory-system-mcp) | A knowledge-graph memory server — entities, relations and observations that persist across sessions instead of living only in a context window. Derived from the upstream memory MCP server. |
| [mcp_vulnerable_testbed](https://github.com/bryankthompson/mcp_vulnerable_testbed) | A deliberately vulnerable MCP server, built as a target for security testing. |

Run the assessment tool against your own server without installing it:

```bash
npx -p @bryan-thompson/inspector-assessment mcp-inspector-assess --help
```

---

## Check any of this

These links run live GitHub searches. GitHub generates them from third-party repository data rather
than from this page, so nothing here can be overstated:

- [Merged PRs authored across the four directory repositories](https://github.com/search?q=is%3Apr+is%3Amerged+author%3Abryan-anthropic+repo%3Aanthropics%2Fclaude-plugins-official+repo%3Aanthropics%2Fknowledge-work-plugins+repo%3Aanthropics%2Fclaude-plugins-community+repo%3Amodelcontextprotocol%2Fmcpb&type=pullrequests)
- [Pull requests merged through those repositories](https://github.com/search?q=is%3Apr+is%3Amerged+reviewed-by%3Abryan-anthropic+repo%3Aanthropics%2Fclaude-plugins-official+repo%3Aanthropics%2Fknowledge-work-plugins+repo%3Aanthropics%2Fclaude-plugins-community&type=pullrequests)
- [Issue and pull request threads on modelcontextprotocol/mcpb](https://github.com/modelcontextprotocol/mcpb/issues?q=commenter%3Abryan-anthropic)
- [The annotation campaign — merged, authored as @bryankthompson, in repositories neither account owns](https://github.com/search?q=is%3Apr+is%3Amerged+author%3Abryankthompson+-user%3Abryankthompson+-user%3Abryan-anthropic&type=pullrequests)

A verification page that goes stale is worse than no verification page at all, so
[triepod.ai/record](https://triepod.ai/record) is wired to break loudly: every figure lives in one
module beside the exact query that produces it, and a check re-derives all of them against the
GitHub API.

---

## Read further

| | |
|---|---|
| [The annotations campaign](https://triepod.ai/annotations) | Why an unannotated tool is assumed destructive, and how the campaign was received |
| [The record](https://triepod.ai/record) | Every published number beside the query that proves it |
| [How the pipelines work](https://triepod.ai/pipelines) | The build, audit and publish architecture |
| [How I verify agent work](https://triepod.ai/verification) | Gates, fail directions, and reproductions you can run in one line |
| [Agents that do not take each other's word](https://triepod.ai/fleet) | The trust boundary inside a multi-agent system |
| [Measuring what an agent actually does](https://triepod.ai/skill-eval-rig) | Why a negative result is a claim about your instrument first |
| [Public code](https://triepod.ai/repos) | The curated index, with its inclusion tests stated |

---

## Contact

- Website: [triepod.ai](https://triepod.ai)
- LinkedIn: [bryan-thompson-it](https://linkedin.com/in/bryan-thompson-it)
- Email: bryan@triepod.ai

Focus areas: the Model Context Protocol, tool annotations and agent safety, review and audit
automation, and testing infrastructure for AI tooling.

---

## Submitting an MCP server to the Anthropic directory

If you have built an MCP server and want it listed in the Anthropic MCP Directory for one-click
installation in Claude Desktop:

- [Submission guidelines](https://claude.com/docs/connectors/building/submission)
- [Submission form](https://clau.de/desktop-extention-submission) (requires a Google account)

---

*This is the profile README for **@bryankthompson**. Bryan Thompson also publishes as
[@bryan-anthropic](https://github.com/bryan-anthropic) — both accounts are his, and both carry this
same record, so either is a complete entry point.*

*Personal account. Views expressed are his own; nothing here is an official Anthropic
communication.*
