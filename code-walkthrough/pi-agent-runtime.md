# OpenClaw Pi Agent Runtime Walkthrough

*2026-03-25T05:47:58Z by Showboat 0.6.1*
<!-- showboat-id: b4ffc8b3-0c88-4911-aea3-e9cd9d2da8fe -->

## 1) Runtime entry surface

The embedded Pi runtime is exposed as a focused API (`runEmbeddedPiAgent`) used by higher-level command/channel paths.

- **What**: primary callable runtime entrypoint.
- **Why**: centralize model/tool/session orchestration in one execution surface.
- **Upstream**: channels, CLI, and gateway flows invoking agent work.
- **Downstream**: lane resolution, prompt assembly, model stream execution.
- **State**: session identity, run IDs, model/tool context.

```bash
sed -n '1,120p' src/agents/pi-embedded.ts
```

```output
export type {
  EmbeddedPiAgentMeta,
  EmbeddedPiCompactResult,
  EmbeddedPiRunMeta,
  EmbeddedPiRunResult,
} from "./pi-embedded-runner.js";
export {
  abortEmbeddedPiRun,
  compactEmbeddedPiSession,
  isEmbeddedPiRunActive,
  isEmbeddedPiRunStreaming,
  queueEmbeddedPiMessage,
  resolveEmbeddedSessionLane,
  runEmbeddedPiAgent,
  waitForEmbeddedPiRunEnd,
} from "./pi-embedded-runner.js";
```

## 2) Lane resolution enforces concurrency policy

Before execution, the runtime resolves command lanes to serialize per-session work and apply broader limits.

- **What**: derive session/global lane keys.
- **Why**: avoid race conditions and overlapping writes in one session.
- **Upstream**: `runEmbeddedPiAgent` call site.
- **Downstream**: queued execution and run lifecycle tracking.
- **State**: lane occupancy and active-run metadata.

```bash
sed -n '1,130p' src/agents/pi-embedded-runner/lanes.ts
```

```output
import { CommandLane } from "../../process/lanes.js";

export function resolveSessionLane(key: string) {
  const cleaned = key.trim() || CommandLane.Main;
  return cleaned.startsWith("session:") ? cleaned : `session:${cleaned}`;
}

export function resolveGlobalLane(lane?: string) {
  const cleaned = lane?.trim();
  // Cron jobs hold the cron lane slot; inner operations must use nested to avoid deadlock.
  if (cleaned === CommandLane.Cron) {
    return CommandLane.Nested;
  }
  return cleaned ? cleaned : CommandLane.Main;
}

export function resolveEmbeddedSessionLane(key: string) {
  return resolveSessionLane(key);
}
```

```bash
sed -n '1,110p' src/process/lanes.ts
```

```output
export const enum CommandLane {
  Main = "main",
  Cron = "cron",
  Subagent = "subagent",
  Nested = "nested",
}
```

## 3) Run loop orchestrates model, tools, and session state

`runEmbeddedPiAgent` is where runtime decisions happen: model resolution, tool exposure, plugin hooks, and stream lifecycle wiring.

- **What**: main orchestration loop.
- **Why**: one place to coordinate behavior and safety across providers/tools.
- **Upstream**: lane-gated run invocation.
- **Downstream**: streaming subscription, result finalization, lifecycle events.
- **State**: run metadata, errors, usage, and completion status.

```bash
sed -n '268,380p' src/agents/pi-embedded-runner/run.ts
```

```output
export async function runEmbeddedPiAgent(
  params: RunEmbeddedPiAgentParams,
): Promise<EmbeddedPiRunResult> {
  const sessionLane = resolveSessionLane(params.sessionKey?.trim() || params.sessionId);
  const globalLane = resolveGlobalLane(params.lane);
  const enqueueGlobal =
    params.enqueue ?? ((task, opts) => enqueueCommandInLane(globalLane, task, opts));
  const enqueueSession =
    params.enqueue ?? ((task, opts) => enqueueCommandInLane(sessionLane, task, opts));
  const channelHint = params.messageChannel ?? params.messageProvider;
  const resolvedToolResultFormat =
    params.toolResultFormat ??
    (channelHint
      ? isMarkdownCapableMessageChannel(channelHint)
        ? "markdown"
        : "plain"
      : "markdown");
  const isProbeSession = params.sessionId?.startsWith("probe-") ?? false;

  return enqueueSession(() =>
    enqueueGlobal(async () => {
      const started = Date.now();
      const workspaceResolution = resolveRunWorkspaceDir({
        workspaceDir: params.workspaceDir,
        sessionKey: params.sessionKey,
        agentId: params.agentId,
        config: params.config,
      });
      const resolvedWorkspace = workspaceResolution.workspaceDir;
      const redactedSessionId = redactRunIdentifier(params.sessionId);
      const redactedSessionKey = redactRunIdentifier(params.sessionKey);
      const redactedWorkspace = redactRunIdentifier(resolvedWorkspace);
      if (workspaceResolution.usedFallback) {
        log.warn(
          `[workspace-fallback] caller=runEmbeddedPiAgent reason=${workspaceResolution.fallbackReason} run=${params.runId} session=${redactedSessionId} sessionKey=${redactedSessionKey} agent=${workspaceResolution.agentId} workspace=${redactedWorkspace}`,
        );
      }
      ensureRuntimePluginsLoaded({
        config: params.config,
        workspaceDir: resolvedWorkspace,
        allowGatewaySubagentBinding: params.allowGatewaySubagentBinding,
      });

      let provider = (params.provider ?? DEFAULT_PROVIDER).trim() || DEFAULT_PROVIDER;
      let modelId = (params.model ?? DEFAULT_MODEL).trim() || DEFAULT_MODEL;
      const agentDir = params.agentDir ?? resolveOpenClawAgentDir();
      const fallbackConfigured = hasConfiguredModelFallbacks({
        cfg: params.config,
        agentId: params.agentId,
        sessionKey: params.sessionKey,
      });
      await ensureOpenClawModelsJson(params.config, agentDir);

      // Run before_model_resolve hooks early so plugins can override the
      // provider/model before resolveModel().
      //
      // Legacy compatibility: before_agent_start is also checked for override
      // fields if present. New hook takes precedence when both are set.
      let modelResolveOverride: { providerOverride?: string; modelOverride?: string } | undefined;
      let legacyBeforeAgentStartResult: PluginHookBeforeAgentStartResult | undefined;
      const hookRunner = getGlobalHookRunner();
      const hookCtx = {
        agentId: workspaceResolution.agentId,
        sessionKey: params.sessionKey,
        sessionId: params.sessionId,
        workspaceDir: resolvedWorkspace,
        messageProvider: params.messageProvider ?? undefined,
        trigger: params.trigger,
        channelId: params.messageChannel ?? params.messageProvider ?? undefined,
      };
      if (hookRunner?.hasHooks("before_model_resolve")) {
        try {
          modelResolveOverride = await hookRunner.runBeforeModelResolve(
            { prompt: params.prompt },
            hookCtx,
          );
        } catch (hookErr) {
          log.warn(`before_model_resolve hook failed: ${String(hookErr)}`);
        }
      }
      if (hookRunner?.hasHooks("before_agent_start")) {
        try {
          legacyBeforeAgentStartResult = await hookRunner.runBeforeAgentStart(
            { prompt: params.prompt },
            hookCtx,
          );
          modelResolveOverride = {
            providerOverride:
              modelResolveOverride?.providerOverride ??
              legacyBeforeAgentStartResult?.providerOverride,
            modelOverride:
              modelResolveOverride?.modelOverride ?? legacyBeforeAgentStartResult?.modelOverride,
          };
        } catch (hookErr) {
          log.warn(
            `before_agent_start hook (legacy model resolve path) failed: ${String(hookErr)}`,
          );
        }
      }
      if (modelResolveOverride?.providerOverride) {
        provider = modelResolveOverride.providerOverride;
        log.info(`[hooks] provider overridden to ${provider}`);
      }
      if (modelResolveOverride?.modelOverride) {
        modelId = modelResolveOverride.modelOverride;
        log.info(`[hooks] model overridden to ${modelId}`);
      }

      const { model, error, authStorage, modelRegistry } = await resolveModelAsync(
        provider,
        modelId,
        agentDir,
        params.config,
```

## 4) Stream subscription translates model events to runtime outputs

The runtime consumes the model stream through a dedicated subscriber that normalizes chunks, tool events, and completion/error transitions.

```bash
sed -n '1,140p' src/agents/pi-embedded-subscribe.ts
```

```output
import type { AgentMessage } from "@mariozechner/pi-agent-core";
import { parseReplyDirectives } from "../auto-reply/reply/reply-directives.js";
import { createStreamingDirectiveAccumulator } from "../auto-reply/reply/streaming-directives.js";
import { formatToolAggregate } from "../auto-reply/tool-meta.js";
import { emitAgentEvent } from "../infra/agent-events.js";
import { createSubsystemLogger } from "../logging/subsystem.js";
import type { InlineCodeState } from "../markdown/code-spans.js";
import { buildCodeSpanIndex, createInlineCodeState } from "../markdown/code-spans.js";
import { EmbeddedBlockChunker } from "./pi-embedded-block-chunker.js";
import {
  isMessagingToolDuplicateNormalized,
  normalizeTextForComparison,
} from "./pi-embedded-helpers.js";
import type { BlockReplyPayload } from "./pi-embedded-payloads.js";
import { createEmbeddedPiSessionEventHandler } from "./pi-embedded-subscribe.handlers.js";
import { consumePendingToolMediaIntoReply } from "./pi-embedded-subscribe.handlers.messages.js";
import type {
  EmbeddedPiSubscribeContext,
  EmbeddedPiSubscribeState,
} from "./pi-embedded-subscribe.handlers.types.js";
import { filterToolResultMediaUrls } from "./pi-embedded-subscribe.tools.js";
import type { SubscribeEmbeddedPiSessionParams } from "./pi-embedded-subscribe.types.js";
import { formatReasoningMessage, stripDowngradedToolCallText } from "./pi-embedded-utils.js";
import { hasNonzeroUsage, normalizeUsage, type UsageLike } from "./usage.js";

const THINKING_TAG_SCAN_RE = /<\s*(\/?)\s*(?:think(?:ing)?|thought|antthinking)\s*>/gi;
const FINAL_TAG_SCAN_RE = /<\s*(\/?)\s*final\s*>/gi;
const log = createSubsystemLogger("agent/embedded");

export type {
  BlockReplyChunking,
  SubscribeEmbeddedPiSessionParams,
  ToolResultFormat,
} from "./pi-embedded-subscribe.types.js";

export function subscribeEmbeddedPiSession(params: SubscribeEmbeddedPiSessionParams) {
  const reasoningMode = params.reasoningMode ?? "off";
  const toolResultFormat = params.toolResultFormat ?? "markdown";
  const useMarkdown = toolResultFormat === "markdown";
  const state: EmbeddedPiSubscribeState = {
    assistantTexts: [],
    toolMetas: [],
    toolMetaById: new Map(),
    toolSummaryById: new Set(),
    lastToolError: undefined,
    blockReplyBreak: params.blockReplyBreak ?? "text_end",
    reasoningMode,
    includeReasoning: reasoningMode === "on",
    shouldEmitPartialReplies: !(reasoningMode === "on" && !params.onBlockReply),
    streamReasoning: reasoningMode === "stream" && typeof params.onReasoningStream === "function",
    deltaBuffer: "",
    blockBuffer: "",
    // Track if a streamed chunk opened a <think> block (stateful across chunks).
    blockState: { thinking: false, final: false, inlineCode: createInlineCodeState() },
    partialBlockState: { thinking: false, final: false, inlineCode: createInlineCodeState() },
    lastStreamedAssistant: undefined,
    lastStreamedAssistantCleaned: undefined,
    emittedAssistantUpdate: false,
    lastStreamedReasoning: undefined,
    lastBlockReplyText: undefined,
    reasoningStreamOpen: false,
    assistantMessageIndex: 0,
    lastAssistantTextMessageIndex: -1,
    lastAssistantTextNormalized: undefined,
    lastAssistantTextTrimmed: undefined,
    assistantTextBaseline: 0,
    suppressBlockChunks: false, // Avoid late chunk inserts after final text merge.
    lastReasoningSent: undefined,
    compactionInFlight: false,
    pendingCompactionRetry: 0,
    compactionRetryResolve: undefined,
    compactionRetryReject: undefined,
    compactionRetryPromise: null,
    unsubscribed: false,
    messagingToolSentTexts: [],
    messagingToolSentTextsNormalized: [],
    messagingToolSentTargets: [],
    messagingToolSentMediaUrls: [],
    pendingMessagingTexts: new Map(),
    pendingMessagingTargets: new Map(),
    successfulCronAdds: 0,
    pendingMessagingMediaUrls: new Map(),
    pendingToolMediaUrls: [],
    pendingToolAudioAsVoice: false,
    deterministicApprovalPromptSent: false,
  };
  const usageTotals = {
    input: 0,
    output: 0,
    cacheRead: 0,
    cacheWrite: 0,
    total: 0,
  };
  let compactionCount = 0;

  const assistantTexts = state.assistantTexts;
  const toolMetas = state.toolMetas;
  const toolMetaById = state.toolMetaById;
  const toolSummaryById = state.toolSummaryById;
  const messagingToolSentTexts = state.messagingToolSentTexts;
  const messagingToolSentTextsNormalized = state.messagingToolSentTextsNormalized;
  const messagingToolSentTargets = state.messagingToolSentTargets;
  const messagingToolSentMediaUrls = state.messagingToolSentMediaUrls;
  const pendingMessagingTexts = state.pendingMessagingTexts;
  const pendingMessagingTargets = state.pendingMessagingTargets;
  const replyDirectiveAccumulator = createStreamingDirectiveAccumulator();
  const partialReplyDirectiveAccumulator = createStreamingDirectiveAccumulator();
  const emitBlockReplySafely = (
    payload: Parameters<NonNullable<SubscribeEmbeddedPiSessionParams["onBlockReply"]>>[0],
  ) => {
    if (!params.onBlockReply) {
      return;
    }
    void Promise.resolve()
      .then(() => params.onBlockReply?.(payload))
      .catch((err) => {
        log.warn(`block reply callback failed: ${String(err)}`);
      });
  };
  const emitBlockReply = (payload: BlockReplyPayload) => {
    emitBlockReplySafely(consumePendingToolMediaIntoReply(state, payload));
  };

  const resetAssistantMessageState = (nextAssistantTextBaseline: number) => {
    state.deltaBuffer = "";
    state.blockBuffer = "";
    blockChunker?.reset();
    replyDirectiveAccumulator.reset();
    partialReplyDirectiveAccumulator.reset();
    state.blockState.thinking = false;
    state.blockState.final = false;
    state.blockState.inlineCode = createInlineCodeState();
    state.partialBlockState.thinking = false;
    state.partialBlockState.final = false;
    state.partialBlockState.inlineCode = createInlineCodeState();
    state.lastStreamedAssistant = undefined;
    state.lastStreamedAssistantCleaned = undefined;
    state.emittedAssistantUpdate = false;
    state.lastBlockReplyText = undefined;
    state.lastStreamedReasoning = undefined;
```

## 5) Compaction protects long-running sessions

When context exceeds limits, the runtime compacts session history through a dedicated path instead of overloading the main run loop.

This is a non-obvious but central design choice for long-lived sessions.

```bash
sed -n '610,700p' src/agents/pi-embedded-runner/compact.ts
```

```output
export async function compactEmbeddedPiSessionDirect(
  params: CompactEmbeddedPiSessionParams,
): Promise<EmbeddedPiCompactResult> {
  const startedAt = Date.now();
  const diagId = params.diagId?.trim() || createCompactionDiagId();
  const trigger = params.trigger ?? "manual";
  const attempt = params.attempt ?? 1;
  const maxAttempts = params.maxAttempts ?? 1;
  const runId = params.runId ?? params.sessionId;
  const resolvedWorkspace = resolveUserPath(params.workspaceDir);
  ensureRuntimePluginsLoaded({
    config: params.config,
    workspaceDir: resolvedWorkspace,
    allowGatewaySubagentBinding: params.allowGatewaySubagentBinding,
  });
  // Resolve compaction model: prefer config override, then fall back to caller-supplied model
  const compactionModelOverride = params.config?.agents?.defaults?.compaction?.model?.trim();
  let provider: string;
  let modelId: string;
  // When switching provider via override, drop the primary auth profile to avoid
  // sending the wrong credentials (e.g. OpenAI profile token to OpenRouter).
  let authProfileId: string | undefined = params.authProfileId;
  if (compactionModelOverride) {
    const slashIdx = compactionModelOverride.indexOf("/");
    if (slashIdx > 0) {
      provider = compactionModelOverride.slice(0, slashIdx).trim();
      modelId = compactionModelOverride.slice(slashIdx + 1).trim() || DEFAULT_MODEL;
      // Provider changed — drop primary auth profile so getApiKeyForModel
      // falls back to provider-based key resolution for the override model.
      if (provider !== (params.provider ?? "").trim()) {
        authProfileId = undefined;
      }
    } else {
      provider = (params.provider ?? DEFAULT_PROVIDER).trim() || DEFAULT_PROVIDER;
      modelId = compactionModelOverride;
    }
  } else {
    provider = (params.provider ?? DEFAULT_PROVIDER).trim() || DEFAULT_PROVIDER;
    modelId = (params.model ?? DEFAULT_MODEL).trim() || DEFAULT_MODEL;
  }
  const fail = (reason: string): EmbeddedPiCompactResult => {
    log.warn(
      `[compaction-diag] end runId=${runId} sessionKey=${params.sessionKey ?? params.sessionId} ` +
        `diagId=${diagId} trigger=${trigger} provider=${provider}/${modelId} ` +
        `attempt=${attempt} maxAttempts=${maxAttempts} outcome=failed reason=${classifyCompactionReason(reason)} ` +
        `durationMs=${Date.now() - startedAt}`,
    );
    return {
      ok: false,
      compacted: false,
      reason,
    };
  };
  const agentDir = params.agentDir ?? resolveOpenClawAgentDir();
  await ensureOpenClawModelsJson(params.config, agentDir);
  const { model, error, authStorage, modelRegistry } = await resolveModelAsync(
    provider,
    modelId,
    agentDir,
    params.config,
  );
  if (!model) {
    const reason = error ?? `Unknown model: ${provider}/${modelId}`;
    return fail(reason);
  }
  let runtimeModel = model;
  let apiKeyInfo: Awaited<ReturnType<typeof getApiKeyForModel>> | null = null;
  try {
    apiKeyInfo = await getApiKeyForModel({
      model: runtimeModel,
      cfg: params.config,
      profileId: authProfileId,
      agentDir,
    });

    if (!apiKeyInfo.apiKey) {
      if (apiKeyInfo.mode !== "aws-sdk") {
        throw new Error(
          `No API key resolved for provider "${runtimeModel.provider}" (auth mode: ${apiKeyInfo.mode}).`,
        );
      }
    } else {
      const preparedAuth = await prepareProviderRuntimeAuth({
        provider: runtimeModel.provider,
        config: params.config,
        workspaceDir: resolvedWorkspace,
        env: process.env,
        context: {
          config: params.config,
          agentDir,
          workspaceDir: resolvedWorkspace,
```

## If you remember only this

- Pi runtime is invoked through one core entrypoint (`runEmbeddedPiAgent`).
- Lane resolution serializes session work and avoids deadlocks.
- The run loop centralizes model/tool/plugin-hook orchestration.
- Stream subscription normalizes model events into agent-facing outputs.
- Compaction is a dedicated subsystem for context overflow, not a side effect.
- These boundaries are why long-lived, multi-channel sessions stay manageable.

