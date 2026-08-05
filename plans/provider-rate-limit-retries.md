# Subscription Rate-Limit Recovery

Status: research complete; implementation not started  
Research date: 2026-08-05

## Goal

Automatically resume failed Codex and Claude Code turns when a user's subscription usage window resets. The primary cases are limits such as Codex subscription quota exhaustion and Claude Code's five-hour or seven-day limits—not ordinary burst throttling.

Version one is optimized for the common five-hour window. It preserves the provider's structured account/window state, coordinates every affected thread on the same subscription while the server is running, and automatically continues only turns that stopped before producing output or side effects. Its timers are intentionally in memory: a plugin or server restart cancels the pending automatic continuation and leaves the failed turn available for manual retry.

## Recommendation

Build an auto-installed, default-enabled `plugins/provider-retry` built-in plugin, supported by two small core capabilities:

1. A normalized provider subscription-limit state that retains window type, reset time, exhaustion reason, overage state, and an opaque subscription/account scope.
2. A narrow server-owned continuation operation that starts a new system-authored turn on the same provider thread after a safe rate-limit failure, without adding a visible user message or resending the original prompt.

The ownership split should be:

- The host daemon reports raw structured provider observations. It does not decide whether or when product-level recovery should happen.
- The server merges sparse observations into current subscription state and exposes typed SDK/CLI queries and change notifications.
- The built-in plugin owns in-memory waiting turns, reset timers, account-level coordination, user controls, and retry policy.
- Provider runtimes retain ownership of any internal retry loop they report as active, whether the delay is seconds or hours. A `willRetry: true` error never creates a plugin job.

Do not implement this by matching freeform error strings in the plugin or by calling `bb.sdk.threads.send`. `send` records a user-authored turn and can duplicate the prompt. Subscription metadata should be normalized at the provider boundary, and the hidden system continuation should be server-owned.

## Current bb Behavior and the Missing Data

### Codex

Codex turn errors can contain `codexErrorInfo: "usageLimitExceeded"`. The [Codex translator](../packages/agent-runtime/src/codex/event-translation.ts) maps it to:

```ts
{
  category: "rate-limit",
  providerCode: "usageLimitExceeded",
  httpStatusCode: null,
}
```

This correctly recognizes the failed turn, but it loses the kind of limit and its reset time.

Codex separately emits `account/rateLimits/updated` with substantially richer data:

- Primary and secondary windows with `usedPercent`, `windowDurationMins`, and `resetsAt`.
- Credits state and balance.
- Individual spend-control state.
- Plan type.
- `rateLimitReachedType`, including ordinary rate-limit reached, owner/member credits depleted, and owner/member usage-limit reached.

bb currently classifies that notification as noise in [`visibility.ts`](../packages/agent-runtime/src/codex/visibility.ts) and deliberately translates it to no thread event. This is the main Codex gap. The subscription state required for correct recovery is already sent by Codex but discarded.

Codex documents rolling rate-limit notifications as sparse: nullable account metadata may be absent in an update without clearing the previously observed value. The server therefore needs a mergeable projection rather than treating every notification as a full replacement.

### Claude Code

Claude Code exposes subscription limits through a structured `rate_limit_event`. The [Claude schema](../packages/agent-runtime/src/claude-code/schemas.ts) currently parses:

- `status`: allowed, allowed warning, or rejected.
- A closed subset of `rateLimitType` values: five-hour, seven-day, seven-day Opus, seven-day Sonnet, and overage.
- `resetsAt`.
- `overageStatus`.
- Detailed reasons such as out of credits, member/group zero-credit limits, disabled overage, or no limits configured.

The Opus and Sonnet keys are real provider values, not invented product categories: the pinned `@anthropic-ai/claude-agent-sdk` 0.3.197 type declares `seven_day_opus` and `seven_day_sonnet`. However, they should not be copied into a shared bb enum. The same installed SDK also declares `seven_day_overage_included`, which bb's hand-written enum already omits, and it has no Fable-specific subscription key even though Fable is a supported model. A new provider key such as a future Fable window would currently fail the entire Zod parse. This is evidence that model-family names must remain provider-issued data, not closed shared-domain variants.

The [Claude translator](../packages/agent-runtime/src/claude-code/translate-message.ts) correctly treats a rejection as blocking only when usable overage is not allowed. However, it flattens the window, reset, and overage information into the human-readable `detail` field and retains only this generic error info:

```ts
{
  category: "rate-limit",
  providerCode: "rate_limit_event",
  httpStatusCode: null,
}
```

Claude Code also emits `system/api_retry` with attempt, maximum attempts, retry delay, error status, and error code. bb preserves it as `willRetry: true`; this is a provider-owned short retry and must remain separate from subscription recovery.

A later failed Claude `result` may contain HTTP 429, but that alone is less informative than the preceding `rate_limit_event`. Recovery should correlate the failure with the latest structured subscription observation instead of reducing both to a generic 429.

### Shared limitations

[`ProviderErrorInfo`](../packages/domain/src/provider-event.ts) currently contains only category, provider code, and HTTP status. Plugins receive `thread.failed` with an error string and can query events, but there is no subscription-state API and no server-owned safe-continuation operation.

As a result, bb can display that a turn was rate limited but cannot reliably answer:

- Is this a burst throttle or a subscription window?
- Is it the five-hour, weekly, model-specific, credits, or spend-control limit?
- When does it reset?
- Which other threads share the same exhausted subscription?
- Has overage made the request usable despite the primary limit?

## External Research

The first review drew the boundary incorrectly. Automatic rate-limit retry is common; what varies is how subscription exhaustion is recognized, how long the wait may be, and whether the wait survives a process restart. The closest implementations are:

| Project | Subscription/rate-limit behavior | Lifetime and scope | Lesson for bb |
| --- | --- | --- | --- |
| [OpenCode](https://github.com/anomalyco/opencode/tree/4a57013cf8cb163f58638273fd9da8538cd33cb7) | Explicitly recognizes its account subscription `GoUsageLimitError`, reads `Retry-After`, displays the named usage limit and reset duration, and retries the same `llm.stream(...)` through an Effect schedule with no attempt ceiling. Header-provided waits may be almost 25 days; only the no-header fallback is capped at 30 seconds. See [classification and delay calculation](https://github.com/anomalyco/opencode/blob/4a57013cf8cb163f58638273fd9da8538cd33cb7/packages/opencode/src/session/retry.ts#L26-L122) and [same-stream retry](https://github.com/anomalyco/opencode/blob/4a57013cf8cb163f58638273fd9da8538cd33cb7/packages/opencode/src/session/processor.ts#L627-L675). | The V1 retry runner is session-local and in memory; its runner map is cancelled during process/layer shutdown. See [run state](https://github.com/anomalyco/opencode/blob/4a57013cf8cb163f58638273fd9da8538cd33cb7/packages/opencode/src/session/run-state.ts#L35-L68). | This is direct prior art for subscription-limit auto-retry, exact reset UX, and an interruptible in-memory wait. bb v1 deliberately adopts the same process-lifetime tradeoff while adding shared-account coordination. |
| [Roo Code](https://github.com/RooCodeInc/Roo-Code/tree/b867ec9145750d0ae1ff7f02d35406e9bf2a0b16) | With auto-approval enabled, first-chunk failures are retried recursively without an overall attempt limit. It shows a per-second countdown, preserves Google `RetryInfo` on 429s, and otherwise exponentially backs off to a 10-minute cap. See [the retry path](https://github.com/RooCodeInc/Roo-Code/blob/b867ec9145750d0ae1ff7f02d35406e9bf2a0b16/src/core/task/Task.ts#L4217-L4253) and [delay selection](https://github.com/RooCodeInc/Roo-Code/blob/b867ec9145750d0ae1ff7f02d35406e9bf2a0b16/src/core/task/Task.ts#L4267-L4325). | The countdown and retry attempt live on the active task; it has no subscription-window projection or restart-safe timer. A provider reset longer than 10 minutes is polled at the cap unless supplied as supported `RetryInfo`. | Infinite/unattended retry is already an expected coding-agent behavior. bb should retain cancellation/countdown UX but schedule known multi-hour resets once instead of polling. |
| [t3code](https://github.com/pingdotgg/t3code/tree/de592a00e89776d2e0614f3be6e666012d90cd51) | Keeps provider-owned Codex retries nonterminal and forwards both Codex and Claude account rate-limit observations onto its runtime event bus. See its [Codex session state](https://github.com/pingdotgg/t3code/blob/de592a00e89776d2e0614f3be6e666012d90cd51/apps/server/src/provider/Layers/CodexSessionRuntime.ts#L950-L965), [Codex adapter](https://github.com/pingdotgg/t3code/blob/de592a00e89776d2e0614f3be6e666012d90cd51/apps/server/src/provider/Layers/CodexAdapter.ts#L1127-L1139), and [Claude adapter](https://github.com/pingdotgg/t3code/blob/de592a00e89776d2e0614f3be6e666012d90cd51/apps/server/src/provider/Layers/ClaudeAdapter.ts#L2908-L2915). | No downstream t3code consumer currently turns `account.rate-limits.updated` into a durable replay job; it relies on the provider runtime while `willRetry` is true. | Preserve native retry ownership. The event normalization is useful precedent, but bb still needs a terminal-failure recovery layer. |
| [OpenAI Codex](https://github.com/openai/codex/tree/9d00bb01c0a712fb7c2f5b002bdf33bcc0fc352c) | Ordinary subscription exhaustion is terminal (`will_retry: false`). Its newer durable Goals feature persists a goal as `UsageLimited` instead of losing it, but the UI instructs the user to run `/goal resume`; it does not schedule automatic resume at the account reset. See [usage-limit transition](https://github.com/openai/codex/blob/9d00bb01c0a712fb7c2f5b002bdf33bcc0fc352c/codex-rs/ext/goal/src/runtime.rs#L240-L275), [manual resume UI](https://github.com/openai/codex/blob/9d00bb01c0a712fb7c2f5b002bdf33bcc0fc352c/codex-rs/tui/src/bottom_pane/footer.rs#L540-L548), and [terminal error fixture](https://github.com/openai/codex/blob/9d00bb01c0a712fb7c2f5b002bdf33bcc0fc352c/codex-rs/tui/src/app/tests/rate_limits.rs#L88-L98). | Goal intent is durable, but resumption is manual and goal-level rather than an exact automatic replay of the failed turn. | Durable intent across usage exhaustion is already an upstream concept. bb can automate the safe subset once reset metadata says the account is available. |
| [Claude Code / Agent SDK](https://github.com/anthropics/claude-agent-sdk-typescript/blob/main/CHANGELOG.md) | Claude Code emits `system/api_retry` with attempt, maximum attempts, delay, and error status for native automatic retries, separately from `rate_limit_event` account telemetry. | The public SDK contract proves native retry and subscription telemetry, but not a restart-safe scheduler for a rejected five-hour or seven-day window; the bundled Claude Code implementation is not public enough to make a stronger claim. | Treat `willRetry` as authoritative and do not double-retry. Build bb recovery only after Claude reports a terminal result. |
| [Cline v3.3.0](https://github.com/cline/cline/blob/02e45fed534ae14cc35755692b3bf13741305d4d/src/api/retry.ts#L1-L61) | This historical release used three bounded 429 retries and honored retry/reset headers. It only retried before the first streamed chunk. | Short-lived and process-local. This old release is not evidence of the current market norm. | Its [stream guard](https://github.com/cline/cline/blob/02e45fed534ae14cc35755692b3bf13741305d4d/src/core/Cline.ts#L1310-L1336) remains useful precedent for bb's no-output/no-side-effect continuation boundary. |
| [Aider](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/coders/base_coder.py#L1449-L1488) | Retries within a 60-second window. | Short-lived and process-local. | Useful only for transient throttles. |

The corrected conclusion is: subscription-limit auto-retry is a common and expected behavior, especially while the coding-agent process remains alive. Version one should match that expectation with an in-memory wait, honor provider reset metadata, coordinate all threads sharing the exhausted subscription, and avoid an outer continuation when the provider is already retrying. Restart durability is a possible later enhancement, not a v1 requirement.

## Canonical Subscription-Limit Model

Introduce a provider observation on the daemon/server wire and a server-owned merged projection. A subscription can have several simultaneous windows, so the shared contract should preserve provider-issued window keys without enumerating current model families. The exact names can be settled during implementation, but the internal state should retain data shaped like:

```ts
type ProviderRateLimitWindow = {
  // Provider-issued identifier such as "primary", "five_hour", or
  // "seven_day_sonnet". New identifiers remain valid without a bb release.
  providerKey: string | null;
  label: string | null;
  status: "allowed" | "warning" | "blocked" | "unknown";
  usedPercent: number | null;
  resetsAtMs: number | null;
  // Populated only when the provider explicitly identifies the model group.
  modelIds: string[];
};

type ProviderRateLimitState = {
  scopeKey: string;
  providerId: string;
  status: "allowed" | "warning" | "blocked" | "unknown";
  kind:
    | "request-throttle"
    | "subscription-window"
    | "credits"
    | "spend-control"
    | "unknown";
  windows: ProviderRateLimitWindow[];
  reachedReason: string | null;
  overageStatus: "allowed" | "warning" | "rejected" | "unavailable" | null;
  overageReason: string | null;
  observedAtMs: number;
  source: "codex-account" | "claude-rate-limit" | "http";
};
```

`scopeKey` must be an opaque, stable identity for the provider subscription/account on a host; it must not contain credentials. All threads sharing the subscription must resolve to the same key. If providers do not expose a safe stable account identifier, use a conservative host/provider installation key initially and document that separate accounts on one host may be grouped.

At the external boundary, validate `rateLimitType` as a non-empty provider string rather than a closed enum. Known values may receive better labels, but retry correctness must depend on structured status and reset time, not recognition of the name. Thus a future `seven_day_fable` observation would be preserved and scheduled immediately; it would merely use a fallback label until bb learns a nicer display name.

The existing host-daemon [`provider.usage`](../apps/host-daemon/src/provider-usage.ts) path already reads subscription usage for Settings, but it currently normalizes only Claude's overall five-hour and seven-day windows and intentionally ignores model-specific sub-limits. Extend and reuse that contract where practical instead of building a parallel account-usage reader. Live provider events are still needed to associate a blocked turn with the relevant state.

Because this changes data sent from the host daemon to the server, increment `HOST_DAEMON_PROTOCOL_VERSION` and update its version tests.

## Provider Ingestion

### Codex ingestion

1. Stop dropping `account/rateLimits/updated`.
2. Emit a typed raw observation containing both windows, credits, spend-control, plan type, and reached type.
3. Merge sparse rolling updates into the latest server projection for the subscription scope. A missing nullable account field does not automatically clear a prior value.
4. Correlate terminal `usageLimitExceeded` errors with the current projection. If multiple windows are exhausted, choose the latest relevant reset for automatic release and expose all exhausted windows in status details.
5. Add a server/daemon refresh operation backed by Codex `account/rateLimits/read`, so startup reconciliation and user-triggered refresh do not depend on another turn failing.

### Claude Code ingestion

1. Preserve every `rate_limit_event` as structured subscription state, including allowed and warning observations. Allowed observations are not failures, but they are important because they can release waiting turns early.
2. Mark the state blocked only when the primary status is rejected and overage is neither allowed nor allowed-with-warning.
3. Preserve `rateLimitType` verbatim as `providerKey`, plus `resetsAt`, utilization, overage reset/status, disabled reason, and any explicitly reported model-group metadata. Do not derive a closed model enum from the key.
4. Correlate the structured observation with the failed turn/result. Do not create two retry jobs when a hard `rate_limit_event` is followed by a terminal failed `result`.
5. Keep `system/api_retry` provider-owned through `willRetry: true`.
6. If Claude Code has no explicit account-state read operation, represent refresh support as unavailable rather than fabricating a result. Waiting jobs can still be released by a new allowed observation, their reset timestamp, or a user-requested safe retry.

## Product Policy

Version one should distinguish three cases:

### Timed subscription window

- A blocked five-hour, seven-day, primary, secondary, or model-specific window with `resetsAtMs` creates an in-memory waiting turn.
- Schedule one continuation at the provider reset plus a small buffer and jitter. Multi-day reset times are valid even though a server/plugin restart intentionally cancels the v1 timer.
- If the provider publishes a changed reset time, reschedule every in-memory waiting turn on that `scopeKey`.
- If the provider reports the subscription allowed before the timestamp, release waiting turns early.
- After the reset, pace safe continuations for that account one at a time so dozens of threads do not stampede it.
- If the host is disconnected when the timer fires, retain the waiting turn in memory and continue it after the host reconnects. This does not survive a server/plugin restart.

### Credits or spend-control exhaustion

- Treat out-of-credits, zero-credit, and spend-control exhaustion as handled blocked states, not ignored errors.
- Do not run blind exponential retries when there is no meaningful reset time.
- Show the precise reason and provide Refresh and Retry now controls, but do not create an indefinite automatic waiting job in v1 when there is no reset timestamp.
- If a later observation supplies a reset time while the server remains running, it may enter the normal timed-window flow.

### Ordinary request throttle

- Provider-native retries remain authoritative while `willRetry: true`.
- Generic terminal 429 recovery is secondary and out of scope for the first subscription-focused release unless it is correlated with structured subscription state.
- Do not let a generic short-backoff rule mask or overwrite a known subscription reset.

Across all cases:

- Automatically continue only turns whose original input received `turn/input/accepted` and which then failed before assistant output, tool activity, approval flow, file changes, or any other possible side effect.
- A newer user turn, manual retry, archive/delete, or provider/account switch supersedes the waiting turn.
- Do not consume an attempt merely because the same blocked window is observed again. The state machine waits for a state transition or reset boundary.
- Never schedule an outer retry while the provider reports `willRetry: true`.
- Plugin disable/reload or server restart clears all pending v1 timers. The original failed turn remains visible and manually retryable.

## Safe Continuation After Reset

After a provider has declared a turn terminal, neither the Codex nor Claude Code session protocol gives bb a primitive to retry that exact provider turn ID. Version one should therefore start a fresh turn on the same provider thread rather than claim to replay the old request.

Add a narrow server operation exposed consistently as:

- SDK: `bb.sdk.threads.continueAfterRateLimit({ threadId, failedRequestId })`
- CLI: `bb thread retry <thread-id> [--request-id <id>] [--json]`
- Internal route/command used by the plugin

The server should transactionally:

1. Locate the original `client/turn/requested` and its terminal failure.
2. Verify that the same failed turn is still latest and the current provider subscription scope matches the waiting job.
3. Require `turn/input/accepted`; if the provider never accepted the original input, leave the turn for manual retry rather than guessing whether its transcript contains the prompt.
4. Reject automatic continuation after output or any possible side effect.
5. Claim the failed request once so a timer, Retry now action, and newer user input cannot race into duplicate continuations.
6. Start a new `initiator: "system"` turn on the same provider thread using the original execution and permission settings. `system` describes who initiated it in bb; the Codex/Claude adapter may still encode the new turn input as a normal provider-side user message because those protocols do not expose a mid-conversation system-role continuation. Its only prompt input is agent-only text such as:

   > Please continue.

7. Render a system operation such as “Automatically resumed after the Claude five-hour limit reset,” not a user message bubble.
8. Record lineage from the continuation request to the original failed request and reset observation.

Return typed outcomes such as `started`, `already-started`, `superseded`, `unsafe`, `provider-changed`, `not-rate-limited`, and `not-found` rather than using expected races as exceptions.

The route accepts no replacement input. It is a guarded system continuation, not another form of public `send` and not a byte-for-byte replay of the original prompt. Provider-specific integration tests must verify that an accepted, pre-output failed prompt remains in the resumed Codex and Claude Code conversation context; if either provider does not guarantee that, disable automatic continuation for that provider until its adapter has a safe provider-specific strategy.

## Built-In Plugin State (In Memory for v1)

Register `plugins/provider-retry` in the [built-in registry](../apps/server/src/services/plugins/builtin-registry.ts) with `autoInstall: true` and `defaultEnabled: true`.

Keep the minimal live state in maps owned by the plugin service:

```text
scopes: Map<scopeKey, {
  providerId, currentObservation, timer, waitingTurns[]
}>

waitingTurns: Map<failedRequestId, {
  threadId, scopeKey, dueAtMs,
  state: "waiting" | "releasing" | "cancelled"
}>
```

The plugin should:

- Use `thread.failed` to find safe failed-turn candidates.
- Use SDK realtime subscription-state changes to update or release account-level jobs; plugin lifecycle events alone are insufficient.
- Use one timer per subscription scope and serialize continuation starts within that scope.
- Cancel/supersede jobs on newer turns, manual retry, archive/delete, or provider/account changes.
- Clear all timers through `onDispose`; plugin disable/reload and server restart intentionally lose the waiting state in v1.
- Never copy prompt content into plugin memory or logs. The server continuation operation derives everything from the event log.

## UX and Agent Surfaces

Use a plugin-provided composer banner with provider-specific information:

> Claude five-hour usage limit reached. This thread will continue at 3:12 PM while this bb server remains running. **Retry now** · **Cancel**

> Codex weekly usage limit reached. This thread will continue Friday at 9:04 AM while this bb server remains running. **Refresh** · **Retry now** · **Cancel**

> Claude Code credits are exhausted. There is no automatic reset time. **Refresh** · **Retry now**

For a timed window, clarify that automatic continuation requires the current bb server/plugin process to remain running. Required v1 states include waiting-for-reset, waiting-for-host, releasing, continued, superseded, cancelled, unsafe, and refresh-unavailable. Show every affected thread's state, while the underlying subscription status is shared.

Expose equivalent non-UI surfaces:

- SDK: provider subscription-state get/refresh/subscribe and guarded continuation after a rate-limit failure.
- Core CLI: a discoverable provider-limit status/refresh command plus `bb thread retry`.
- Plugin CLI: `bb provider-retry status|now|cancel <thread-id>` for job controls.

Update the CLI guide, command help, and plugin-command skill surfaces required by [`docs/cli-guide-and-skill.md`](../docs/cli-guide-and-skill.md). Any new public plugin API outside existing SDK areas must use the `experimental_` prefix and be recorded in `docs/api_to_audit.md`.

## Delivery Plan

### Phase 1: Preserve subscription state end to end

- Define raw Codex and Claude provider observations and the unified server projection.
- Stop dropping Codex account rate-limit updates.
- Preserve Claude window, reset, overage, and reason fields structurally.
- Replace the Claude `rateLimitType` closed boundary enum with a validated provider string; preserve unknown/future keys and add coverage for the installed SDK's `seven_day_overage_included` value.
- Extend the existing `provider.usage` contract rather than duplicating its account usage fetch where it can supply current state.
- Establish the opaque subscription scope key and server-side sparse merge rules.
- Add get/refresh/subscribe SDK and CLI contracts, with explicit refresh-unavailable behavior.
- Increment `HOST_DAEMON_PROTOCOL_VERSION` and update contract/version tests.

### Phase 2: Add safe continuation

- Implement failed-request correlation and the safe-continuation predicate: accepted input, no output, and no side effects.
- Add the guarded system-continuation route, SDK method, and `bb thread retry` command.
- Add continuation lineage and a system operation to the event log without another user message.
- Verify on real Codex and Claude Code sessions that an accepted pre-output failed prompt remains in provider conversation history. Fail closed per provider if this is not guaranteed.
- Verify original execution and permission settings, warm/cold sessions, and provider/account changes.

### Phase 3: Implement in-memory subscription recovery

- Scaffold and register the built-in plugin.
- Implement lifecycle ingestion, account-state subscriptions, one timer per scope, reset rescheduling, single-flight release, host-reconnect waiting, and per-scope pacing.
- Clear timers on plugin disposal. Do not add plugin tables, migrations, restart reconciliation, leases, or durable jobs in v1.
- Inject clock and random sources for deterministic reset-buffer and jitter tests.
- Deduplicate a Claude hard rate-limit event and its later failed result into one waiting turn.

### Phase 4: Add UI, CLI, and documentation

- Implement provider-specific banners and Refresh/Retry now/Cancel controls.
- Add plugin CLI/RPC/realtime surfaces.
- Update CLI, guide, and skill discovery surfaces.
- Make the process-lifetime limitation and non-timed credit blocks explicit rather than presenting them as a generic failed turn.

### Phase 5: Roll out and tune

- Ship timed subscription windows plus explicitly handled credit/spend-control states.
- Measure blocked subscriptions, waiting turns, reset changes, early releases, post-reset continuation success, unsafe skips, cancellations, and refresh availability. Never record prompt content.
- Only after this is reliable, consider server-restart durability, generic terminal 429 backoff, or other transient failures.

## Test Matrix

### Codex

- Primary/secondary reset timestamps, duration, percent used, plan type, credits, and reached type survive the daemon/server boundary.
- Sparse account updates merge without clearing previously known account metadata.
- Owner/member usage-limit and credits-depleted reasons remain distinguishable.
- `usageLimitExceeded` correlates with the latest matching account projection.
- `account/rateLimits/read` refresh updates the projection and releases eligible jobs.

### Claude Code

- Five-hour, weekly, model-specific, and overage observations remain structured.
- Current Opus/Sonnet keys, the installed SDK's `seven_day_overage_included`, and an arbitrary future model-window key all parse and survive unchanged.
- Scheduling a future Fable-specific key requires no domain/schema release; only its optional display label may initially be generic.
- Allowed and allowed-warning observations update state without creating a failed-turn job.
- Primary rejection with allowed overage does not block.
- Hard rejection preserves `resetsAt` and reason and creates at most one job even when followed by a failed result.
- `system/api_retry` remains nonterminal and never creates a plugin job.

### Account state and scheduling

- Every thread on one scope sees the same blocked subscription state.
- A changed reset time reschedules all waiting turns.
- Allowed state releases jobs early.
- Plugin disposal clears timers and does not auto-reconstruct them after reload/server restart.
- Credits without a reset remain visible but create no automatic waiting job.
- A disconnected host retains its job only while the server/plugin remains alive and continues after reconnection.
- Post-reset continuation turns are serialized and paced per scope.

### Continuation safety

- The original request must have `turn/input/accepted`, no provider output, and no side-effecting activity.
- The continuation is a system-authored, agent-only prompt on the same provider thread; it creates no duplicate user message and does not resend the original input.
- Original execution and permission settings are retained.
- Concurrent timer, Retry now, and user-send races start at most one continuation.
- Newer input, manual retry, archive/delete, account switch, or provider switch supersedes the job.
- Any assistant output, tool activity, approval, file operation, missing input acceptance, or ambiguous event rejects automatic continuation as unsafe.

### UI and integration

- Codex usage-limit failure plus account snapshot waits until its real reset and then continues with a hidden system turn.
- Claude five-hour rejection waits until `resetsAt` and then continues with a hidden system turn.
- Claude overage-allowed flow continues without a waiting job.
- Credits-exhausted flow remains visible with Refresh and Retry now controls but creates no automatic waiting job without a reset timestamp.
- Banner, SDK, and CLI report the same scope and waiting-turn state.

Use Turbo for package build, typecheck, and test orchestration as required by the repository guidelines.

## Decisions to Confirm During Implementation

1. **Subscription identity:** determine the safest stable Codex and Claude account identity available on each host. A conservative host/provider fallback is acceptable for v1 but may group separate logins.
2. **Codex multiple windows:** when primary and secondary windows are simultaneously exhausted, automatic release should wait for the last blocking reset; the UI should show both.
3. **Reset buffer:** begin with a small 15–45 second jittered buffer after the provider timestamp and tune from observed post-reset failures.
4. **Refresh support:** Codex has an explicit rate-limit read operation. Confirm whether the installed Claude Code protocol exposes an equivalent; model absence explicitly.
5. **Safety threshold:** start with no provider output or side effects. Do not loosen this as part of the subscription recovery release.
6. **Provider window labels:** keep the provider key authoritative. Known keys can have friendly labels, but unknown model/window names must remain operable and visible instead of being rejected.
7. **Future durability:** keep the waiting-turn interface separable from timer storage so a later release can persist it, but do not add persistence abstractions, tables, or recovery logic until that release is actually planned.

## Acceptance Criteria

- Codex and Claude subscription-window metadata is preserved structurally instead of discarded or flattened into text.
- A five-hour, weekly, primary, secondary, or model-specific blocked turn can wait for its real reset while the server/plugin remains running and continue on the same provider thread.
- Credits and spend-control exhaustion remain visible handled states, but v1 does not invent an automatic wait when no reset time exists.
- All affected threads coordinate through one in-memory subscription scope and continue with pacing rather than stampeding the provider.
- Provider-native retries are never duplicated.
- Concurrent callbacks, later user actions, provider/account switches, and unsafe partial turns cannot produce duplicate or invalid continuations. Server/plugin restarts intentionally cancel pending automatic continuation.
- Users and agents can inspect subscription state, refresh it when supported, retry immediately, cancel, or disable the plugin through equivalent UI, SDK, and CLI surfaces.
