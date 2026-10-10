<!-- three.ws:badges -->
[![GitHub stars](https://img.shields.io/github/stars/nirholas/agent-runtime?style=flat&logo=github)](https://github.com/nirholas/agent-runtime/stargazers) [![License](https://img.shields.io/github/license/nirholas/agent-runtime?style=flat)](https://github.com/nirholas/agent-runtime/blob/HEAD/LICENSE) [![Last commit](https://img.shields.io/github/last-commit/nirholas/agent-runtime?style=flat)](https://github.com/nirholas/agent-runtime/commits) [![PRs welcome](https://img.shields.io/badge/PRs-welcome-brightgreen?style=flat)](https://github.com/nirholas/agent-runtime/pulls) [![AI agent friendly](https://img.shields.io/badge/AI%20agents-AGENTS.md%20%2B%20llms.txt-6d5dfc?style=flat)](https://github.com/nirholas/agent-runtime/blob/HEAD/AGENTS.md)
<!-- /three.ws:badges -->

<p align="center">
  <a href="https://three.ws"><img src="https://three.ws/three-ws-mcp-icon.svg" width="72" height="72" alt="three.ws" /></a>
</p>

<h1 align="center">@three-ws/agent-runtime</h1>

<p align="center"><strong>The three.ws agent engine - a plan/execute decision loop with a seven-layer guard chain, a tamper-evident action ledger, and a multi-step transaction pipeline.</strong></p>

<p align="center">
  <a href="#quick-start">Quick start</a> ·
  <a href="#the-guardchain">GuardChain</a> ·
  <a href="#the-decision-loop">Decision loop</a> ·
  <a href="#action-ledger">Action ledger</a> ·
  <a href="https://three.ws">three.ws</a>
</p>

---

> The runtime that sits between a model and anything irreversible. The
> **decision loop** turns an LLM into an agent (plan, call tools, pause for
> humans, finish); the **GuardChain** composes every enforcement layer into one
> deterministic verdict with a per-layer trace; the **action ledger** makes the
> resulting history tamper-evident. It is pure engine: no chain SDKs, no
> database, no HTTP - every external fact is injected, which is why the same
> code path serves the live `/api/agent/guard` preflight, simulators, and
> tests. Powers the transaction-guard preflight in [three.ws/chat](https://three.ws/chat).

## Why

An agent that can call `solana_transfer` is one bad tool call away from an
empty wallet. Most stacks scatter their defenses: an approval modal here, a
spend cap there, a blacklist somewhere else - each invoked from a different
place, with different inputs, and *nothing that can answer which layer would
stop a given call, or which layers would silently not evaluate it at all*.
A guard that never runs looks exactly like a guard that passed.

This package makes enforcement composable and observable:

| Piece | What it does |
|---|---|
| `AgentRuntime` | Plan → execute loop: `call_llm`, `call_tool`, batches, `finish`, and three human-in-the-loop instructions (`approve` / `prompt` / `select`), with cost limits, interrupts, and resume |
| `GeneralChatAgent` | The default "brain": routes phases to instructions, splits tool calls into execute-now vs. needs-approval, compresses context when it outgrows the window |
| `GuardChain` | Runs all seven layers over one call, returns decision + per-layer trace + **blind spots** (enforcement that should have applied but didn't) + a coverage score |
| `TradeGuard` | Default domain guard: tier per-tx caps, rolling 24h/7d windows, auto-execute ceiling, MEV slippage clamp, protocol audit lookup |
| `SpendGuard` | Per-agent spend envelope: per-tx / rolling / daily caps, reserve floor, token + destination firewall, custody-breach latch |
| `InterventionChecker` | Policy engine for human approval (`never` / `required` / `always`, argument-level rules, security blacklist) |
| `ActionLedger` | Hash-chained (sha256) append-only ledger primitives: `computeEntryHash`, `verifyChain`, drift audits |
| `TransactionPipeline` | Multi-step plan execution: dependency order, parallel independent steps, rollback on failure, approval gates |
| `DecisionJournal` | Fire-and-forget reasoning journal with an injected sink |
| Reasoning utilities | `SelfReflection` (structured retry prompts), `ExitDecisionEngine`, `ResponseQualityEvaluator`, `ToolRelevanceScorer`, token counting + context-compression checks |

## Install

```bash
npm install @three-ws/agent-runtime
```

ESM, Node 18+. One runtime dependency (`tokenx`, for token estimation).

## Quick start

### Preflight a tool call through every guard layer

```js
import { GuardChain, SpendGuard, TradeGuard, createX402Hook } from '@three-ws/agent-runtime';

const chain = new GuardChain({
  defiGuard: new TradeGuard(),                       // tier caps + slippage clamp
  spendGuard: new SpendGuard({ perTxMaxUsd: 5_000 }), // hard envelope
  x402Hook: createX402Hook(5),                        // $5/hour autonomy budget
});

const verdict = await chain.evaluate({
  identifier: 'solana_swap',
  apiName: 'solana_swap',
  arguments: { inputMint: 'So11111111111111111111111111111111111111112', amount: 120, slippageBps: 300 },
  valueUsd: 18_400,
  userTier: 'pro',
});

verdict.decision;      // 'block'        ($18,400 is over the $5,000 per-tx envelope)
verdict.blockedBy;     // 'spend_guard'  the layer that decided it
verdict.modifiedArguments; // { ..., slippageBps: 100 }  (MEV clamp, still applied)
verdict.coverageScore; // 74   two layers were left unwired below
verdict.blindSpots;    // [CAPABILITY_UNWIRED, PERMISSION_UNWIRED]
```

The capability and permission layers report themselves as blind spots because
this chain was built without a `checkCapability` / `checkPermission` resolver.
Inject both (each an `async (request) => ({ allowed })` backed by your
capability-token and permission records) and the same call scores `100` with no
blind spots. Raise `perTxMaxUsd` above the notional and the decision becomes
`require_approval` instead of `block`, driven by the trade guard's
auto-execute ceiling.

The layer order is `security_blacklist → intervention → capability →
permission → defi_guard → spend_guard → x402`. The chain never
short-circuits: a call blocked by the spend envelope still reports the MEV
exposure the trade guard saw, and every layer that could not evaluate the
call says so instead of reading as "green".

### Run the decision loop

```js
import { AgentRuntime, GeneralChatAgent } from '@three-ws/agent-runtime';

const agent = new GeneralChatAgent({
  modelRuntimeConfig: { model: 'llama-3.3-70b-versatile', provider: 'groq' },
});
agent.modelRuntime = myStreamingLlm; // async generator: yields { content?, tool_calls? }
agent.tools = { get_price: async (args) => fetchPrice(args) };

let state = AgentRuntime.createInitialState({ operationId: 'op-1' });
state.messages.push({ role: 'user', content: 'What is SOL at?' });

const runtime = new AgentRuntime(agent);
let context;
while (state.status !== 'done' && state.status !== 'error') {
  const step = await runtime.step(state, context);
  state = step.newState;
  context = step.nextContext;
  if (state.status === 'waiting_for_human') break; // surface state.pendingToolsCalling to the user
}
```

When the user approves a pending call, continue with
`runtime.approveToolCall(state, approvedCall)`. `runtime.interrupt()` /
`runtime.resume()` cover cancellation.

### Verify a ledger

```js
import { computeEntryHash, verifyChain } from '@three-ws/agent-runtime';

// Each row is a ledger entry plus the two chain columns: `entryHash`
// (computeEntryHash(entry, prevHash)) and `prevHash`, the first seeded with
// LEDGER_GENESIS_HASH. verifyChain sorts by `seq`, so store order is fine.
const rows = await loadAgentLedger(agentId);
const result = verifyChain(rows);
result.valid;         // false if any historical row was edited or deleted
result.brokenAtIndex; // index of the first broken link, -1 when the chain is intact
result.brokenAtSeq;   // that row's `seq`, or null when intact
result.reason;        // why it broke (null when intact)
```

The hashed fields are fixed and ordered (`LEDGER_CANONICAL_FIELD_ORDER`:
`seq, ts, userId, agentId, event, target, amountWei, valueUsd, txHash, reason,
balanceBeforeWei, balanceAfterWei, network, detail`), so editing any one of
them after the fact breaks that row's link and every link after it.

## Registering your tools

The guard layers key off two module-level registries seeded with the three.ws
tool surface (`solana_transfer`, `solana_swap`, `pumpfunBuy`, ...). A host
adding a new fund-moving tool registers it at boot:

```js
import { registerFundMovingTool, registerMutatingApi } from '@three-ws/agent-runtime';

registerFundMovingTool('my_bridge_tool');
registerMutatingApi('executeBridgeV2');
```

An unregistered fund-moving tool is not "unguarded by design" - the GuardChain
reports it as a critical `TOOL_UNREGISTERED` blind spot.

## Where it runs on three.ws

- **`POST /api/agent/guard`** preflights tool calls for the chat client: the
  chat wallet tools (`solana_transfer`, `solana_swap`, `evm_*`, pump.fun
  trades) are evaluated through this exact chain before their tool body runs,
  a `block` verdict never reaches the wallet, and the transaction-approval
  modal renders the verdict (decision, warnings, unchecked blind spots).
- The server resolves SOL notionals (amount × live SOL price) before
  evaluating, so dollar caps compare against real numbers.

## Relation to `@three-ws/agent-guards`

[`@three-ws/agent-guards`](../agent-guards) is the *client* for the platform's
custodial wallet policy (it wraps `/api/agents/:id/trade` limits). This
package is the *engine*: it evaluates arbitrary tool calls before execution
anywhere - client wallets, custodial paths, simulators - and is what
`/api/agent/guard` runs. The custodial trade path keeps its own enforcement;
both speak the same language of per-tx/daily caps and deny-lists.

## Provenance

Core of the engine (decision loop, guard composition, spend envelope,
pipeline) was battle-tested in a sibling project of the same author and
ported here de-branded; the action ledger's hash-chain construction
originated in the three.ws economy master ledger, so this is that design
coming home. All chain-specific analysis was left behind: this package is
chain-agnostic by construction, and three.ws wires Solana-first specifics
(notional resolution, tool registries) at the edges.

## License

See [LICENSE](./LICENSE).

<!-- three.ws:growth -->
## Support the project

If agent-runtime saves you time, **[star it on GitHub](https://github.com/nirholas/agent-runtime)**. Stars are how other developers and AI agents find the repositories worth trusting, and they cost you one click.

Know someone who would use it? [Post on X](https://twitter.com/intent/tweet?text=agent-runtime%3A%20The%20three.ws%20agent%20engine%3A%20a%20plan%2Fexecute%20decision%20loop%20with%20human-approval%20gates%2C%20a%20seven-layer&url=https%3A%2F%2Fgithub.com%2Fnirholas%2Fagent-runtime) · [Share on Bluesky](https://bsky.app/intent/compose?text=agent-runtime%3A%20The%20three.ws%20agent%20engine%3A%20a%20plan%2Fexecute%20decision%20loop%20with%20human-approval%20gates%2C%20a%20seven-layer%20https%3A%2F%2Fgithub.com%2Fnirholas%2Fagent-runtime) · [Share on LinkedIn](https://www.linkedin.com/sharing/share-offsite/?url=https%3A%2F%2Fgithub.com%2Fnirholas%2Fagent-runtime) · [Submit to Hacker News](https://news.ycombinator.com/submitlink?u=https%3A%2F%2Fgithub.com%2Fnirholas%2Fagent-runtime&t=agent-runtime%3A%20The%20three.ws%20agent%20engine%3A%20a%20plan%2Fexecute%20decision%20loop%20with%20human-approval%20gates%2C%20a%20seven-layer) · [Share on Reddit](https://www.reddit.com/submit?url=https%3A%2F%2Fgithub.com%2Fnirholas%2Fagent-runtime&title=agent-runtime%3A%20The%20three.ws%20agent%20engine%3A%20a%20plan%2Fexecute%20decision%20loop%20with%20human-approval%20gates%2C%20a%20seven-layer)

## Built for AI agents too

Coding agents and LLM tooling can read this repo directly: [AGENTS.md](./AGENTS.md), [llms.txt](./llms.txt), [llms-full.txt](./llms-full.txt). Point an agent at `https://github.com/nirholas/agent-runtime` and it has the context it needs.

## More from the same author

- [All repositories by nirholas](https://github.com/nirholas/nirholas#readme): the full catalog, grouped by topic
- [three.ws](https://three.ws): the platform for 3D AI agents with Solana wallets, a skill marketplace and x402 payments
- Questions or ideas: [open an issue](https://github.com/nirholas/agent-runtime/issues) or [start a discussion](https://github.com/nirholas/agent-runtime/discussions)

## Contributors

[![Contributors](https://contrib.rocks/image?repo=nirholas/agent-runtime)](https://github.com/nirholas/agent-runtime/graphs/contributors)

## Star history

[![Star History Chart](https://api.star-history.com/svg?repos=nirholas/agent-runtime&type=Date)](https://www.star-history.com/#nirholas/agent-runtime&Date)

<!-- /three.ws:growth -->

<!-- three.ws:ecosystem -->
## Part of three.ws

[three.ws](https://three.ws) is a platform for 3D AI agents with Solana wallets: avatars, a skill marketplace, x402 payments and more than seventy MCP servers. `@three-ws/agent-runtime` is one package from it.

- Documentation site: https://nirholas.github.io/agent-runtime/
- npm: https://www.npmjs.com/package/@three-ws/agent-runtime
- Canonical source: https://github.com/nirholas/three.ws (this repository is a generated mirror of `packages/agent-runtime`)
- Agent-readable summary: [llms.txt](./llms.txt) and [AGENTS.md](./AGENTS.md)
- Issues and ideas: https://github.com/nirholas/agent-runtime/issues
<!-- /three.ws:ecosystem -->
