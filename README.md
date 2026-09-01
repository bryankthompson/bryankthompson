# Bryan Thompson

**MCP infrastructure engineering.** Most recently for Anthropic — the build, audit and publish
automation that carries third-party plugins and MCP servers from submission to installable listing.
Before that, a career in reliability engineering on Fortune 500 payment systems at PayPal and
Fiserv, which is where the instinct for staged rollouts and blast-radius containment comes from.

I write automation for catalogs that have to stay correct while everything they point at keeps
moving. Thousands of listings, each pinned to an upstream repository with its own opinions about
when to change. No person does that by hand, so the interesting problem is never the listing — it
is the machinery that keeps the listing honest, and what that machinery does when it fails.

**A representative piece is public and you can read the diff.** In a public marketplace repository I
built the scheduled job that re-pins every catalog entry to its upstream's current commit, and the
recovery job that reverts only the entries failing validation, so one broken upstream cannot hold
back the rest. A crashed validation run and a clean pass both produce zero failures — so the run
emits a separate did-this-run signal, and the job holding write access treats a crash as do-nothing
rather than as all-clear:

**[Automated SHA-freshness: in-repo bump + validate-gated auto-revert](https://github.com/anthropics/claude-plugins-community/pull/70)**

---

## The public record

**Four diffs, each a different failure mode.** Read the code rather than the counts.

- **[A CI install that reported success and left no usable binary](https://github.com/anthropics/claude-plugins-community/pull/233)** — the package fetches its runtime in a postinstall, and a reinstall npm treats as already satisfied skips it. The fix forces the platform-native dependency, checks the binary actually landed, clears the package directory between attempts so a retry is a real reinstall, and bounds every network step so a stalled fetch fails fast. Extends a shared action I did not write.

- **[A contribution scope guard, widened to exempt the repository's own automation](https://github.com/anthropics/claude-plugins-official/pull/3402)** — the guard is mine; this change carries the argument for why a fork cannot impersonate that author written into the file, and fixes a permission lookup that used to throw for non-collaborators, which is the exact population the guard exists to evaluate. It now falls through to the scope check instead of erroring the job.

- **[A bump that would have re-pinned a plugin whose content directory had vanished upstream](https://github.com/anthropics/claude-plugins-community/pull/267)** — skipped rather than pinned against a fabricated placeholder manifest, with the tests confirmed by disabling the fix and checking that they fail. Extends a shared action I did not write; the test harnesses are mine.

- **[An injection path in a repository I do not own](https://github.com/modelcontextprotocol/mcpb/pull/230)** — a reference example declares `tab_id` a number, but that server's request path did not enforce it, and the value was interpolated into an AppleScript template where a quote character could close the literal and append arbitrary script. Someone else reviewed and merged the fix.

### Counts

Every figure is a **floor**, not an estimate — these only grow, so a floor stays true while an
approximation is wrong the moment the number moves. Each links to a live GitHub search that
recomputes the current number when you open it.

| Repository | Stars | Merged PRs authored | Pull requests merged |
|---|---:|---:|---:|
| [anthropics/claude-plugins-official](https://github.com/anthropics/claude-plugins-official) | 33k+ | [210+](https://github.com/anthropics/claude-plugins-official/pulls?q=is%3Apr+is%3Amerged+author%3Abryan-anthropic) | [2,500+](https://github.com/anthropics/claude-plugins-official/pulls?q=is%3Apr+is%3Amerged+reviewed-by%3Abryan-anthropic) |
| [anthropics/knowledge-work-plugins](https://github.com/anthropics/knowledge-work-plugins) | 23k+ | [50+](https://github.com/anthropics/knowledge-work-plugins/pulls?q=is%3Apr+is%3Amerged+author%3Abryan-anthropic) | [490+](https://github.com/anthropics/knowledge-work-plugins/pulls?q=is%3Apr+is%3Amerged+reviewed-by%3Abryan-anthropic) |
| [anthropics/claude-plugins-community](https://github.com/anthropics/claude-plugins-community) | 3k+ | [75+](https://github.com/anthropics/claude-plugins-community/pulls?q=is%3Apr+is%3Amerged+author%3Abryan-anthropic) | [2,100+](https://github.com/anthropics/claude-plugins-community/pulls?q=is%3Apr+is%3Amerged+reviewed-by%3Abryan-anthropic) |
| [modelcontextprotocol/mcpb](https://github.com/modelcontextprotocol/mcpb) | 2k+ | [10+](https://github.com/modelcontextprotocol/mcpb/pulls?q=is%3Apr+is%3Amerged+author%3Abryan-anthropic) | [85+ threads](https://github.com/modelcontextprotocol/mcpb/issues?q=commenter%3Abryan-anthropic) † |
| | | **[350+ merged](https://github.com/search?q=is%3Apr+is%3Amerged+author%3Abryan-anthropic+repo%3Aanthropics%2Fclaude-plugins-official+repo%3Aanthropics%2Fknowledge-work-plugins+repo%3Aanthropics%2Fclaude-plugins-community+repo%3Amodelcontextprotocol%2Fmcpb&type=pullrequests)** | **[5,000+](https://github.com/search?q=is%3Apr+is%3Amerged+reviewed-by%3Abryan-anthropic+repo%3Aanthropics%2Fclaude-plugins-official+repo%3Aanthropics%2Fknowledge-work-plugins+repo%3Aanthropics%2Fclaude-plugins-community&type=pullrequests)** |

† Open-source maintenance — issue and pull request threads participated in — rather than throughput,
and excluded from the total.

**Read that last column as a fingerprint of automation, not of typing.** Those are directory
submissions and rolling updates arriving through automated intake, scanned and validated before they
reach a merge decision, so the merge is a final gate on an already-vetted change. Getting that class
of update to merge safely and automatically is what the work actually was. The volume is what it
produced.

---

## MCP tool annotations

A deliberately public campaign, carried out in other people's repositories, where every claim can be
checked by a stranger.

When a model is handed a set of tools, it gets a name, a description and a schema for each. Nothing
in that tells a client whether calling one reads a file or deletes a repository. MCP has a small
answer: four boolean **annotations** describing a tool's behaviour rather than its interface.

**The defaults are the point.** `destructiveHint` defaults to `true`. `openWorldHint` defaults to
`true`. A server shipping no annotations is not making a neutral statement about its tools — it is
asking every client to assume each one is destructive and reaches into the open world. Annotating a
read-only tool does not add a warning; it *removes* one the specification had already applied on the
tool's behalf.

**And the ceiling, in the spec's own terms.** Every one of these is a *hint*, with no guarantee it
faithfully describes what the tool does, and the schema instructs clients never to make tool-use
decisions on annotations from servers they do not trust. An annotation is a claim a server makes
about itself, and a claim about yourself is not evidence. What it buys is narrower and more useful
than trust: where trust already exists, annotations let a client be *less* pessimistic than the
defaults require. Where it does not, they are inert — correctly.

So I read MCP servers maintained by other people, worked out which of their tools read and which
write, and sent pull requests adding the annotations. Not my servers — theirs. Each one had to
persuade a maintainer who had no reason to accept it.

| | |
|---|---|
| [Pull requests opened, all outcomes](https://github.com/search?q=is%3Apr+author%3Abryankthompson+-user%3Abryankthompson+-user%3Abryan-anthropic&type=pullrequests) | **165+** |
| [Merged into external repositories](https://github.com/search?q=is%3Apr+is%3Amerged+author%3Abryankthompson+-user%3Abryankthompson+-user%3Abryan-anthropic&type=pullrequests) | **85+** |
| [Closed without being merged](https://github.com/search?q=is%3Apr+is%3Aclosed+is%3Aunmerged+author%3Abryankthompson+-user%3Abryankthompson+-user%3Abryan-anthropic&type=pullrequests) | **35+** |
| [Distinct external repositories that accepted them](https://github.com/search?q=is%3Apr+is%3Amerged+author%3Abryankthompson+-user%3Abryankthompson+-user%3Abryan-anthropic&type=pullrequests) | **75+** |
| [Combined stars of those repositories](https://github.com/search?q=is%3Apr+is%3Amerged+author%3Abryankthompson+-user%3Abryankthompson+-user%3Abryan-anthropic&type=pullrequests) | **390,000+** |

A merged-only figure is a flattering one, so the denominator is published beside it: everything
opened, what was accepted, what was turned down. The gap is the batch still awaiting a decision. The
declines were mostly reasonable and a few were interesting — maintainers who had deliberately chosen
a different convention, or who wanted a hint set differently from the way I had read their code.
Being told that a tool I marked additive is in fact destructive is a better outcome than silence.

None of these are in repositories I own under either account. The search excludes both, which is
what makes the record independently verifiable rather than self-reported — drop either exclusion and
it sweeps in my own repositories, where merging my own pull request proves nothing.

**Where they landed** —
[github/github-mcp-server](https://github.com/github/github-mcp-server),
[googleapis/mcp-toolbox](https://github.com/googleapis/mcp-toolbox),
[Azure/aks-mcp](https://github.com/Azure/aks-mcp),
[IBM/mcp-context-forge](https://github.com/IBM/mcp-context-forge),
[makenotion/notion-mcp-server](https://github.com/makenotion/notion-mcp-server),
[grafana/mcp-grafana](https://github.com/grafana/mcp-grafana),
[prometheus/prometheus-mcp](https://github.com/prometheus/prometheus-mcp),
[elevenlabs/elevenlabs-mcp](https://github.com/elevenlabs/elevenlabs-mcp),
[line/line-bot-mcp-server](https://github.com/line/line-bot-mcp-server),
[razorpay/razorpay-mcp-server](https://github.com/razorpay/razorpay-mcp-server),
[neo4j-contrib/mcp-neo4j](https://github.com/neo4j-contrib/mcp-neo4j),
[upstash/context7](https://github.com/upstash/context7),
[PrefectHQ/fastmcp](https://github.com/PrefectHQ/fastmcp),
[apify/apify-mcp-server](https://github.com/apify/apify-mcp-server),
[brightdata/brightdata-mcp](https://github.com/brightdata/brightdata-mcp) and
[getsentry/XcodeBuildMCP](https://github.com/getsentry/XcodeBuildMCP), among others. Every pull
request in every one of them is public.

The full write-up, including how the campaign was received, is at
[triepod.ai/annotations](https://triepod.ai/annotations).

---

## How I build

Principles, not a system tour. Each is a technique I use, and most of them show up somewhere in the
diffs above.

**Treat model output as untrusted input, and enforce anything that matters mechanically.** Run a
deterministic check suite and a model review separately, then combine them through a cascade of hard
caps. A model that can only ever make an outcome stricter cannot be prompt-injected into shipping
something, because there is no path from anything it says to a more permissive result. The worst a
successful injection achieves is a false refusal, which a human sees. The absence of a review is
itself a cap: a missing review is a refusal, not a pass.

**Pick the fail direction before you need it.** Version pins move forward only, and if the
comparison itself fails, the pin holds. Drift detection reports and never gates, which is what keeps
a detector bug from becoming a publishing bug. A redactor returns a sentinel on any error rather
than ever returning its input unmodified — skipping a row is safe, persisting a secret is not — and
its pattern set is asserted at import, so an empty set is a hard startup failure rather than silent
pass-through.

**Nothing ships armed.** Every new gate goes out advisory and is armed on its measured
false-positive rate rather than on how confident it felt to write, because an unmeasured gate is a
future outage. Changes stage one canary before the batch.

**No message decides anything.** In a multi-agent setup, a message can say where to look, never what
is true. Verification goes back to the primary record, classification is by positive proof rather
than absence of evidence, and a differential test pins the property down by checking that an
instruction-shaped summary produces a byte-identical decision to a bland one.

**Running the same prompt twice measures confidence; asking different questions measures coverage.**
My review harness runs tiered reviewers over a diff — one model, then two given an identical prompt
with disagreements surfaced per-model, then reviewers with disjoint scopes. For every finding marked
critical, and every entry in a reviewer's list of things it could not verify — which is the half
that usually evaporates — an orchestrator runs one empirical check and relabels the finding
confirmed, refuted or inconclusive, with the command and its output inline.

**An audit trail should survive the thing it records.** Local-first SQLite under the operational
surfaces, phased cutovers behind flip gates and dual-writer patterns, and events that spool outside
the working tree and fold back idempotently, so a reset cannot erase the record of the session that
ran it.

---

## Public code

The curated index with inclusion criteria is at [triepod.ai/repos](https://triepod.ai/repos). A few
that earn a line here:

| Project | What it is |
|---|---|
| [viewer-parity](https://github.com/bryan-anthropic/viewer-parity) | A GitHub Action that checks published figures and links from the reader's seat. GitHub search returns what the *caller* can see, so `is:pr is:merged author:you` silently counts pull requests in private repositories you have access to — publish that and it is larger than the number a stranger gets from the link beside it. The publisher is structurally the last to find out, because the publisher is always signed in. It re-derives each claim unauthenticated, fails when a figure falls below its published floor, fails when a query's answer could depend on who is asking, fails when the link beside a figure does not resolve signed out, and fails when a pull request lowers a floor. Zero dependencies, no build step. |
| [inspector-assessment](https://github.com/bryankthompson/inspector-assessment) | An expanded build of the [MCP Inspector](https://github.com/modelcontextprotocol/inspector) that adds an automated assessment pass over a server: it exercises every declared tool, checks the declarations against observed behaviour, and reports what it found rather than only giving you a console to poke at. **Archived — it was the research phase, and the experiment returned a negative result.** Still published to npm as `@bryan-thompson/inspector-assessment` and still runnable against your own server. [What it found, and why it stopped](https://triepod.ai/inspector-assessment). |
| [mcp-dashboard](https://github.com/bryankthompson/mcp-dashboard) | A management and monitoring interface for running several MCP servers at once: connection state, tool inventories, and request inspection across servers in one view. Built on the same Inspector foundation. |
| [memory-system-mcp](https://github.com/bryankthompson/memory-system-mcp) | A knowledge-graph memory server — entities, relations and observations that persist across sessions instead of living only in a context window. Derived from the upstream memory MCP server. |
| [mcp_vulnerable_testbed](https://github.com/bryankthompson/mcp_vulnerable_testbed) | A deliberately vulnerable MCP server, built as a target for security testing. |

Run the same check over your own published numbers, in a workflow:

```yaml
- uses: bryan-anthropic/viewer-parity@v1
  with:
    claims: .github/published-claims.json
```

Run the assessment tool against your own server without installing it:

```bash
npx -p @bryan-thompson/inspector-assessment mcp-inspector-assess --help
```

---

## Accounts

I publish from two GitHub accounts and both are mine. Two earlier handles were renamed and belong to
the same person.

| Handle | Status | What lives there |
|---|---|---|
| [@bryankthompson](https://github.com/bryankthompson) | Active — this account | The MCP tool-annotation campaign, and open-source MCP tooling and servers |
| [@bryan-anthropic](https://github.com/bryan-anthropic) | Archived | Directory automation work. Kept in place so existing links and citations continue to resolve |
| `@bthompson-sys` | Renamed — now @bryan-anthropic | Placeholder; nothing current |
| `@triepod-ai` | Renamed — now @bryankthompson | Placeholder; nothing current |
| `@bryan-thompson` (npm) | Active | The npm scope my packages publish under |

GitHub redirects renamed *repository* URLs but does not redirect renamed *profile* URLs, so
citations pointing at either old profile address would not reach an active account on their own.
Both are held as placeholders that point a reader to the right place. A repository URL under an old
owner name generally still redirects, provided that repository existed at the time of the rename and
the handle's current holder has not created one of the same name.

---

## Check any of this

These run live GitHub searches. GitHub generates them from third-party repository data rather than
from this page, so nothing here can be overstated:

- [Merged PRs authored across the four directory repositories](https://github.com/search?q=is%3Apr+is%3Amerged+author%3Abryan-anthropic+repo%3Aanthropics%2Fclaude-plugins-official+repo%3Aanthropics%2Fknowledge-work-plugins+repo%3Aanthropics%2Fclaude-plugins-community+repo%3Amodelcontextprotocol%2Fmcpb&type=pullrequests)
- [Pull requests merged through those repositories](https://github.com/search?q=is%3Apr+is%3Amerged+reviewed-by%3Abryan-anthropic+repo%3Aanthropics%2Fclaude-plugins-official+repo%3Aanthropics%2Fknowledge-work-plugins+repo%3Aanthropics%2Fclaude-plugins-community&type=pullrequests)
- [Issue and pull request threads on modelcontextprotocol/mcpb](https://github.com/modelcontextprotocol/mcpb/issues?q=commenter%3Abryan-anthropic)
- [The annotation campaign — merged, authored as @bryankthompson, in repositories neither account owns](https://github.com/search?q=is%3Apr+is%3Amerged+author%3Abryankthompson+-user%3Abryankthompson+-user%3Abryan-anthropic&type=pullrequests)

---

## Read further

| | |
|---|---|
| [The annotations campaign](https://triepod.ai/annotations) | Why an unannotated tool is assumed destructive, and how the campaign was received |
| [The record](https://triepod.ai/record) | Every published number beside the query that proves it |
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

*Personal account. Views are my own.*
