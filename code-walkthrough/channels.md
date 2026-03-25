# OpenClaw Channels and Channel Plugins Walkthrough

*2026-03-25T05:44:01Z by Showboat 0.6.1*
<!-- showboat-id: d117bd64-3f27-4988-a1f8-b8949fe6d64f -->

## 1) The plugin contract is the boundary

Channels are implemented through a `ChannelPlugin` contract rather than hardcoded providers.

- **What**: typed adapter surface for channel capabilities.
- **Why**: lets OpenClaw add/replace channels without changing core routing.
- **Upstream**: gateway/channel startup.
- **Downstream**: concrete plugin implementations.
- **State**: plugin config + channel account context.

```bash
sed -n '45,140p' src/channels/plugins/types.plugin.ts
```

```output
};

/** JSON-schema-like config description published by a channel plugin. */
export type ChannelConfigSchema = {
  schema: Record<string, unknown>;
  uiHints?: Record<string, ChannelConfigUiHint>;
};

/** Full capability contract for a native channel plugin. */
// oxlint-disable-next-line typescript/no-explicit-any
export type ChannelPlugin<ResolvedAccount = any, Probe = unknown, Audit = unknown> = {
  id: ChannelId;
  meta: ChannelMeta;
  capabilities: ChannelCapabilities;
  defaults?: {
    queue?: {
      debounceMs?: number;
    };
  };
  reload?: { configPrefixes: string[]; noopPrefixes?: string[] };
  setupWizard?: ChannelSetupWizard;
  config: ChannelConfigAdapter<ResolvedAccount>;
  configSchema?: ChannelConfigSchema;
  setup?: ChannelSetupAdapter;
  pairing?: ChannelPairingAdapter;
  security?: ChannelSecurityAdapter<ResolvedAccount>;
  groups?: ChannelGroupAdapter;
  mentions?: ChannelMentionAdapter;
  outbound?: ChannelOutboundAdapter;
  status?: ChannelStatusAdapter<ResolvedAccount, Probe, Audit>;
  gatewayMethods?: string[];
  gateway?: ChannelGatewayAdapter<ResolvedAccount>;
  auth?: ChannelAuthAdapter;
  elevated?: ChannelElevatedAdapter;
  commands?: ChannelCommandAdapter;
  lifecycle?: ChannelLifecycleAdapter;
  execApprovals?: ChannelExecApprovalAdapter;
  allowlist?: ChannelAllowlistAdapter;
  bindings?: ChannelConfiguredBindingProvider;
  streaming?: ChannelStreamingAdapter;
  threading?: ChannelThreadingAdapter;
  messaging?: ChannelMessagingAdapter;
  agentPrompt?: ChannelAgentPromptAdapter;
  directory?: ChannelDirectoryAdapter;
  resolver?: ChannelResolverAdapter;
  actions?: ChannelMessageActionAdapter;
  heartbeat?: ChannelHeartbeatAdapter;
  // Channel-owned agent tools (login flows, etc.).
  agentTools?: ChannelAgentToolFactory | ChannelAgentTool[];
};
```

## 2) Built-ins are registered as plugins

Core-supported channels are assembled in one bundled registry. This keeps startup wiring explicit and discoverable.

- **What**: bundled plugin list + loading helpers.
- **Why**: one place to control what channel surfaces are enabled by default.
- **Upstream**: channel startup and plugin discovery.
- **Downstream**: registry/cache + runtime channel start.
- **State**: plugin catalog and version-scoped cache.

```bash
sed -n '1,90p' src/channels/plugins/bundled.ts
```

```output
import { bluebubblesPlugin } from "../../../extensions/bluebubbles/index.js";
import { discordPlugin, setDiscordRuntime } from "../../../extensions/discord/index.js";
import { discordSetupPlugin } from "../../../extensions/discord/setup-entry.js";
import { feishuPlugin } from "../../../extensions/feishu/index.js";
import { imessagePlugin } from "../../../extensions/imessage/index.js";
import { imessageSetupPlugin } from "../../../extensions/imessage/setup-entry.js";
import { ircPlugin } from "../../../extensions/irc/index.js";
import { linePlugin, setLineRuntime } from "../../../extensions/line/index.js";
import { lineSetupPlugin } from "../../../extensions/line/setup-entry.js";
import { mattermostPlugin } from "../../../extensions/mattermost/index.js";
import { nextcloudTalkPlugin } from "../../../extensions/nextcloud-talk/index.js";
import { signalPlugin } from "../../../extensions/signal/index.js";
import { signalSetupPlugin } from "../../../extensions/signal/setup-entry.js";
import { slackPlugin } from "../../../extensions/slack/index.js";
import { slackSetupPlugin } from "../../../extensions/slack/setup-entry.js";
import { synologyChatPlugin } from "../../../extensions/synology-chat/index.js";
import { telegramPlugin, setTelegramRuntime } from "../../../extensions/telegram/index.js";
import { telegramSetupPlugin } from "../../../extensions/telegram/setup-entry.js";
import { zaloPlugin } from "../../../extensions/zalo/index.js";
import type { ChannelId, ChannelPlugin } from "./types.js";

export const bundledChannelPlugins = [
  bluebubblesPlugin,
  discordPlugin,
  feishuPlugin,
  imessagePlugin,
  ircPlugin,
  linePlugin,
  mattermostPlugin,
  nextcloudTalkPlugin,
  signalPlugin,
  slackPlugin,
  synologyChatPlugin,
  telegramPlugin,
  zaloPlugin,
] as ChannelPlugin[];

export const bundledChannelSetupPlugins = [
  telegramSetupPlugin,
  discordSetupPlugin,
  ircPlugin,
  slackSetupPlugin,
  signalSetupPlugin,
  imessageSetupPlugin,
  lineSetupPlugin,
] as ChannelPlugin[];

function buildBundledChannelPluginsById(plugins: readonly ChannelPlugin[]) {
  const byId = new Map<ChannelId, ChannelPlugin>();
  for (const plugin of plugins) {
    if (byId.has(plugin.id)) {
      throw new Error(`duplicate bundled channel plugin id: ${plugin.id}`);
    }
    byId.set(plugin.id, plugin);
  }
  return byId;
}

const bundledChannelPluginsById = buildBundledChannelPluginsById(bundledChannelPlugins);

export function getBundledChannelPlugin(id: ChannelId): ChannelPlugin | undefined {
  return bundledChannelPluginsById.get(id);
}

export function requireBundledChannelPlugin(id: ChannelId): ChannelPlugin {
  const plugin = getBundledChannelPlugin(id);
  if (!plugin) {
    throw new Error(`missing bundled channel plugin: ${id}`);
  }
  return plugin;
}

export const bundledChannelRuntimeSetters = {
  setDiscordRuntime,
  setLineRuntime,
  setTelegramRuntime,
};
```

```bash
sed -n '1,110p' src/channels/plugins/registry.ts
```

```output
import {
  getActivePluginRegistryVersion,
  requireActivePluginRegistry,
} from "../../plugins/runtime.js";
import { CHAT_CHANNEL_ORDER, type ChatChannelId, normalizeAnyChannelId } from "../registry.js";
import type { ChannelId, ChannelPlugin } from "./types.js";

function dedupeChannels(channels: ChannelPlugin[]): ChannelPlugin[] {
  const seen = new Set<string>();
  const resolved: ChannelPlugin[] = [];
  for (const plugin of channels) {
    const id = String(plugin.id).trim();
    if (!id || seen.has(id)) {
      continue;
    }
    seen.add(id);
    resolved.push(plugin);
  }
  return resolved;
}

type CachedChannelPlugins = {
  registryVersion: number;
  sorted: ChannelPlugin[];
  byId: Map<string, ChannelPlugin>;
};

const EMPTY_CHANNEL_PLUGIN_CACHE: CachedChannelPlugins = {
  registryVersion: -1,
  sorted: [],
  byId: new Map(),
};

let cachedChannelPlugins = EMPTY_CHANNEL_PLUGIN_CACHE;

function resolveCachedChannelPlugins(): CachedChannelPlugins {
  const registry = requireActivePluginRegistry();
  const registryVersion = getActivePluginRegistryVersion();
  const cached = cachedChannelPlugins;
  if (cached.registryVersion === registryVersion) {
    return cached;
  }

  const sorted = dedupeChannels(registry.channels.map((entry) => entry.plugin)).toSorted((a, b) => {
    const indexA = CHAT_CHANNEL_ORDER.indexOf(a.id as ChatChannelId);
    const indexB = CHAT_CHANNEL_ORDER.indexOf(b.id as ChatChannelId);
    const orderA = a.meta.order ?? (indexA === -1 ? 999 : indexA);
    const orderB = b.meta.order ?? (indexB === -1 ? 999 : indexB);
    if (orderA !== orderB) {
      return orderA - orderB;
    }
    return a.id.localeCompare(b.id);
  });
  const byId = new Map<string, ChannelPlugin>();
  for (const plugin of sorted) {
    byId.set(plugin.id, plugin);
  }

  const next: CachedChannelPlugins = {
    registryVersion,
    sorted,
    byId,
  };
  cachedChannelPlugins = next;
  return next;
}

export function listChannelPlugins(): ChannelPlugin[] {
  return resolveCachedChannelPlugins().sorted.slice();
}

export function getChannelPlugin(id: ChannelId): ChannelPlugin | undefined {
  const resolvedId = String(id).trim();
  if (!resolvedId) {
    return undefined;
  }
  return resolveCachedChannelPlugins().byId.get(resolvedId);
}

export function normalizeChannelId(raw?: string | null): ChannelId | null {
  return normalizeAnyChannelId(raw);
}
```

## 3) Inbound messages bridge into sessioned agent execution

Once a channel plugin receives a message, channel/session helpers decide whether to create/update a session record and preserve route metadata.

- **What**: map channel events to session context.
- **Why**: maintain continuity so agent replies route correctly.
- **Upstream**: plugin inbound adapter.
- **Downstream**: session store + routing/agent run path.
- **State**: session records (`lastRoute`, title/source metadata, timestamps).

```bash
sed -n '1,140p' src/channels/session.ts
```

```output
import type { MsgContext } from "../auto-reply/templating.js";
import {
  recordSessionMetaFromInbound,
  type GroupKeyResolution,
  type SessionEntry,
  updateLastRoute,
} from "../config/sessions.js";

function normalizeSessionStoreKey(sessionKey: string): string {
  return sessionKey.trim().toLowerCase();
}

export type InboundLastRouteUpdate = {
  sessionKey: string;
  channel: SessionEntry["lastChannel"];
  to: string;
  accountId?: string;
  threadId?: string | number;
  mainDmOwnerPin?: {
    ownerRecipient: string;
    senderRecipient: string;
    onSkip?: (params: { ownerRecipient: string; senderRecipient: string }) => void;
  };
};

function shouldSkipPinnedMainDmRouteUpdate(
  pin: InboundLastRouteUpdate["mainDmOwnerPin"] | undefined,
): boolean {
  if (!pin) {
    return false;
  }
  const owner = pin.ownerRecipient.trim().toLowerCase();
  const sender = pin.senderRecipient.trim().toLowerCase();
  if (!owner || !sender || owner === sender) {
    return false;
  }
  pin.onSkip?.({ ownerRecipient: pin.ownerRecipient, senderRecipient: pin.senderRecipient });
  return true;
}

export async function recordInboundSession(params: {
  storePath: string;
  sessionKey: string;
  ctx: MsgContext;
  groupResolution?: GroupKeyResolution | null;
  createIfMissing?: boolean;
  updateLastRoute?: InboundLastRouteUpdate;
  onRecordError: (err: unknown) => void;
}): Promise<void> {
  const { storePath, sessionKey, ctx, groupResolution, createIfMissing } = params;
  const canonicalSessionKey = normalizeSessionStoreKey(sessionKey);
  void recordSessionMetaFromInbound({
    storePath,
    sessionKey: canonicalSessionKey,
    ctx,
    groupResolution,
    createIfMissing,
  }).catch(params.onRecordError);

  const update = params.updateLastRoute;
  if (!update) {
    return;
  }
  if (shouldSkipPinnedMainDmRouteUpdate(update.mainDmOwnerPin)) {
    return;
  }
  const targetSessionKey = normalizeSessionStoreKey(update.sessionKey);
  await updateLastRoute({
    storePath,
    sessionKey: targetSessionKey,
    deliveryContext: {
      channel: update.channel,
      to: update.to,
      accountId: update.accountId,
      threadId: update.threadId,
    },
    // Avoid leaking inbound origin metadata into a different target session.
    ctx: targetSessionKey === canonicalSessionKey ? ctx : undefined,
    groupResolution,
  });
}
```

## 4) Run-state tracking guards channel account concurrency

Channel run-state utilities track active agent runs by account/session to avoid overlapping execution surprises at a channel surface.

```bash
sed -n '1,120p' src/channels/run-state-machine.ts
```

```output
export type RunStateStatusPatch = {
  busy?: boolean;
  activeRuns?: number;
  lastRunActivityAt?: number | null;
};

export type RunStateStatusSink = (patch: RunStateStatusPatch) => void;

type RunStateMachineParams = {
  setStatus?: RunStateStatusSink;
  abortSignal?: AbortSignal;
  heartbeatMs?: number;
  now?: () => number;
};

const DEFAULT_RUN_ACTIVITY_HEARTBEAT_MS = 60_000;

export function createRunStateMachine(params: RunStateMachineParams) {
  const heartbeatMs = params.heartbeatMs ?? DEFAULT_RUN_ACTIVITY_HEARTBEAT_MS;
  const now = params.now ?? Date.now;
  let activeRuns = 0;
  let runActivityHeartbeat: ReturnType<typeof setInterval> | null = null;
  let lifecycleActive = !params.abortSignal?.aborted;

  const publish = () => {
    if (!lifecycleActive) {
      return;
    }
    params.setStatus?.({
      activeRuns,
      busy: activeRuns > 0,
      lastRunActivityAt: now(),
    });
  };

  const clearHeartbeat = () => {
    if (!runActivityHeartbeat) {
      return;
    }
    clearInterval(runActivityHeartbeat);
    runActivityHeartbeat = null;
  };

  const ensureHeartbeat = () => {
    if (runActivityHeartbeat || activeRuns <= 0 || !lifecycleActive) {
      return;
    }
    runActivityHeartbeat = setInterval(() => {
      if (!lifecycleActive || activeRuns <= 0) {
        clearHeartbeat();
        return;
      }
      publish();
    }, heartbeatMs);
    runActivityHeartbeat.unref?.();
  };

  const deactivate = () => {
    lifecycleActive = false;
    clearHeartbeat();
  };

  const onAbort = () => {
    deactivate();
  };

  if (params.abortSignal?.aborted) {
    onAbort();
  } else {
    params.abortSignal?.addEventListener("abort", onAbort, { once: true });
  }

  if (lifecycleActive) {
    // Reset inherited status from previous process lifecycle.
    params.setStatus?.({
      activeRuns: 0,
      busy: false,
    });
  }

  return {
    isActive() {
      return lifecycleActive;
    },
    onRunStart() {
      activeRuns += 1;
      publish();
      ensureHeartbeat();
    },
    onRunEnd() {
      activeRuns = Math.max(0, activeRuns - 1);
      if (activeRuns <= 0) {
        clearHeartbeat();
      }
      publish();
    },
    deactivate,
  };
}
```

## If you remember only this

- Channels are plugin adapters behind a shared `ChannelPlugin` contract.
- Built-in channels are just bundled plugin registrations.
- Registry/cache logic keeps plugin resolution stable and ordered.
- Inbound channel events are normalized into session records for continuity.
- Run-state tracking protects channel-facing behavior from overlapping runs.
- This gives OpenClaw a consistent channel model across many providers.

