---
status: proposed
contact: jpalvarezl
date: 2026-09-09
deciders: [eavanvalkenburg, moonbox3]
---

# Separate Foundry trace attribution from exporter configuration

## Context and Problem Statement

[Issue #7492](https://github.com/microsoft/agent-framework/issues/7492) reports
Foundry client spans reaching Application Insights but missing from the Foundry
agent trace view. Exporting telemetry and identifying the project/agent an
operation belongs to are separate concerns, yet the initial fix resolved project
identity only through `FoundryAgent.configure_azure_monitor()`. That leaves
applications configuring their own exporters without an attribution path. How
should a `FoundryAgent` obtain its project identity independently of how
telemetry is exported?

## Decision Drivers

- Support application-managed exporters without reconfiguring global providers.
- Keep identity scoped to the agent/project rather than a process-wide setting.
- Avoid implicit network work in the invocation path.
- Do not generalize one successful span arrangement into a universal requirement
  that every application root carry project attributes.

## Considered Options

- **Helper-only discovery.** Resolve identity solely inside
  `configure_azure_monitor()`. Simple, but couples identity to one exporter
  helper and leaves application-managed configurations unsupported.
- **Automatic discovery on each run.** Convenient, but introduces implicit
  network work, latency, and failure handling into the invocation path.
- **Explicit per-agent identity plus cached helper discovery.** Accept the
  project ARM ID on the agent; the helper discovers and caches it only when none
  was supplied.

## Decision Outcome

Chosen option: **explicit per-agent identity plus cached helper discovery**,
because it supports both setup styles without run-time discovery.

- Add keyword-only `project_arm_id` to `RawFoundryAgent` and `FoundryAgent`,
  validated as a full project ARM ID at construction; it is not inferred from the
  data-plane endpoint or read implicitly from an environment variable.
- The Azure Monitor helper discovers and caches identity only when no ID was
  supplied. Expected discovery/metadata errors log a warning and preserve export;
  unexpected programming errors and cancellation still propagate.
- Identity is emitted on the agent's `invoke_agent` span, which may be nested
  beneath an application span, keeping attribution separate from exporter
  configuration.

The public SDK does not yet expose project identity directly
([Azure/azure-sdk-for-python#48825](https://github.com/Azure/azure-sdk-for-python/issues/48825)).
Connection-ID parsing remains a bounded discovery workaround, not a requirement
for applications that already know their project ARM ID. Supported examples use
full project ARM IDs; alternate formats and service-side legacy-key behavior are
not new guarantees of this API.
