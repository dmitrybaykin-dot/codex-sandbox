# Hermes source index

Companion to [HERMES_ARCHITECTURE.md](HERMES_ARCHITECTURE.md). Repository: NousResearch/hermes-agent. Revision **4ad61cee29316e21e03c4585bb4ececfa52385fe**, main acquired 2026-11-09. All links pin that commit; path:line anchors are not references to a moving branch.

Method: static source/call-site/configuration inspection and selected test-assertion reading. **No Hermes execution, dependency installation or test run.** Tests below indicate supporting coverage that was inspected, not passing results. Tables index architectural seams rather than every sibling file. V = implementation plus inspected test assertions; I = implementation only for the cited mechanism; O = optional/experimental; D = setup/config-only with no runtime consumer found; U = unresolved/external semantics. Configuration defaults are call-site-specific.

## Entry points, composition and dependencies

| Source / key symbols | Purpose and important dependencies | Evidence / related tests |
|---|---|---|
| [pyproject.toml:592][entries]; hermes:9; hermes_cli/main.py:3336,1601 | Console commands/main; bootstrap, PM activation, profile/argument dispatch; current classic chat launches gateway client. | I. Python metadata and base/extra dependencies: pyproject.toml:11,32,224. |
| [hermes_bootstrap.py:576][boot]; pm/environments.py:464 activate_dependencies | Update recovery/relaunch, selected dependency generation leasing and import/PATH activation; platform/bootstrap side effects precede application imports. | I; PM is runtime lifecycle, not just development setup. |
| [acp_adapter/entry.py:163][acp]; HermesACPAgent in acp_adapter/server.py:273 | ACP stdio client/viewer; scoped env, logging, protocol loop, daemon-owned execution/MCP. Depends on optional ACP SDK and gateway. | I/O; true-boundary teardown comments at acp_adapter/entry.py:224. |
| [tui_gateway/entry.py:321][tui]; tui_gateway/server.py:485,567 | JSON-RPC host/startup, profile-scoped requests and transport events; shared core/control-plane seams. | I; web hosting/auth entry: hermes_cli/web_server.py:1415. |
| [hermes_constants.py:194][homes]; get_hermes_home, hermes_home_key, get_process_hermes_home | Context/home resolution and profile identity; platform defaults at :57; constants consumed throughout state/config/tools. | I for home resolution; tests/agent/test_secret_scope.py:36,101 covers related secret-scope behavior. |

## A. Core, prompts and recovery

| Source / key symbols | Purpose and important dependencies | Relevant tests inspected |
|---|---|---|
| [run_agent.py:214][core]; AIAgent; agent/agent_init.py:2056 init_agent | Facade/mixin composition and ordered client/tool/DB/memory/context initialization. Constructor max_iterations=sys.maxsize at run_agent.py:241; direct main max_turns10 at :1492. | V: tests/agent/test_max_iteration_handoff.py:34; not evidence of a universal10/500 cap. |
| [agent/turn_facade.py:23][turn]; run_conversation; agent/turn_facade_lease.py:32; agent/turn_liveness.py:23 | Review preemption, durable lease and profile/accounting scopes; separate progress watchdog. Lease300s, renew60s, wait1800s; idle600s. | V: tests/agent/test_turn_facade_lease.py:81,106,181. |
| [agent/conversation_loop.py:1646][loop]; _run_conversation_turn, _run_api_retry_loop at :1577 | Phase verdict/state machine; prepare/request/recovery/tool/text/finalize; exception bound at :239. Depends on turn siblings, IterationBudget and transports. | V: tests/agent/test_outer_error_retry_bound.py:89,120,158. |
| [agent/turn_context.py:1189][context]; build_turn_context; build_api_messages at :1418 | User row/history/prompt/preflight, hook veto and external-memory prefetch; api_content sidecar at :1090. | V: tests/agent/test_prompt_cache_prefix_contract.py:81,95. |
| [agent/system_prompt.py:751][prompt]; build_system_prompt_parts, invalidate_system_prompt at :829; agent/prompt_builder.py:1720 | Three frozen prompt tiers; project-context precedence, SOUL, memory, skill/plugin blocks. Context cap at agent/prompt_builder.py:1033. | V: same prefix-contract tests. Restore/rebuild exceptions: agent/conversation_loop.py:802. |
| [agent/turn_tool_round.py:53][toolround]; run_tool_round; agent/turn_tool_validation.py:72 | Validate, persist assistant calls before side effects, execute, commit results and compact. Uses inline executors and registry dispatch. | V: tests/agent/test_text_turn_incremental_persistence.py:108,155. |
| [agent/tool_executor.py:136][executor]; run_agent.py:1315 _execute_tool_calls | Ordered parallel-safe segments and sequential barriers; 8 workers/420s; dispatch at :1714, inline table agent/inline_tool_executors.py:291. | I for batching details; persistence contracts tested in preceding row. |
| [agent/turn_api_error.py:56][errors]; handle_api_error; agent/turn_recovery_autorecover.py:70 | Typed repair/classification/fallback/retry; transient-only post-exhaustion recovery before visible answer. Depends on credentials, request rebuilding and preflight. | V for outer bound; tests/agent/test_empty_response_guard.py:65,182 supports empty-response branch. |
| [agent/turn_finalizer.py:619][final]; finalize_turn; agent/turn_stop_gates.py:149 | Verification gates, output transform/canonical save, independent cleanup, accounting and review scheduling; cleanup errors retained. | V: tests/agent/test_turn_finalizer_cleanup_guard.py:148. |

## B/F. Memory, sessions and context

| Source / key symbols | Purpose and important dependencies | Relevant tests inspected |
|---|---|---|
| [tools/memory_tool.py:250][memorytool]; _memory_tool; agent/inline_tool_executors.py:144 _memory | add/patch/remove and target contract; replace rejected at :261,280; destructive review staging at :179. Uses live MemoryStore and write approval. | V: tests/tools/test_memory_tool.py:493,532; tests/tools/test_memory_target_contract.py:86. |
| [tools/memory_tool_store.py:350][memorystore]; MemoryStore.load_from_disk at :223, apply_batch at :562, format_for_system_prompt at :649 | Locked fresh-read atomic Markdown mutation, exact/folded matching, final capacity validation and frozen prompt snapshot. Lock hardening at :259. | V: tests/tools/test_memory_tool.py:334,450,626,754,806. |
| [agent/memory_provider.py:103][memoryprovider]; MemoryProvider; agent/memory_manager.py:373,459,659 | External hook/prefetch/FIFO-sync/end-switch contract; manager timeout/hang suppression, redaction and bounded shutdown. Init at agent/agent_init_memory_provider.py:91. | V/O: tests/agent/test_memory_async_sync.py:71,93,135; tests/agent/test_memory_session_switch.py:70,89. |
| [plugins/memory/__init__.py:52][memoryplugins]; discover; plugins/memory/retaindb/__init__.py:184,357; plugins/memory/byterover/__init__.py:212,223 | Category activation and optional in-tree backend examples: remote HTTP durable queue or CLI query/curate. Depends on external services/CLI. | O/U: framework inspected; backend semantics are not universal memory guarantees. |
| [hermes_state.py:201][state]; SessionDB; hermes_state_common.py:295,467,537 | Profile state.db, schema33, sessions/messages/prompt hashes, runtime/admission/usage tables; WAL at hermes_state.py:464. | V for session behaviors below; I for full schema/lock implementation. |
| [agent/session_persistence.py:509][persist]; _persist_session, _db_flush_failed at :420 | Incremental durable markers, token deltas, failure diversion and optional trajectory output; depends on SessionDB and compaction-tip handling. | V: tests/agent/test_text_turn_incremental_persistence.py:108,155. |
| [hermes_state_messages.py:908][messages]; archive_and_compact; resolve_resume_session_id at :1468 | Atomic in-place history replacement, archived originals/concurrent arrivals, lineage-aware model/display history at :1672 and resume safety at :1739. | V: tests/agent/test_in_place_compaction.py:61. |
| [agent/context_compressor.py:2799][compress]; ContextCompressor; compress at :5422; agent/context_compressor_tail.py:26 | Trigger/head/tail planning, single summary request at :1002, anchors/tool-pair reconstruction; effective threshold at :2751. | V for in-place commit contract; I for particular tail/threshold defaults. |
| [agent/conversation_compression.py:4028][compression]; compression admission/commit, prompt refresh at :3052 | Default in_place=true; extraction checkpoint at :2859; lease/CAS/race-aware orchestration, rotation compatibility. | V: tests/agent/test_in_place_compaction.py:61. |
| [tools/session_search_tool.py:603][recall]; session_search; hermes_state_search.py:1154 | Discovery/read/scroll/browse of real messages; FTS/trigram/LIKE, lineage dedup, archived current recall, explicit read-only cross-profile access. | V: tests/tools/test_session_search.py:428,573,705,727. |
| [hermes_state_maintenance.py:490][retention]; maybe_auto_maintenance; hermes_cli/config_defaults.py:2255 | Ended-session retention/automation close ordering, archive/prune protections; defaults off; config resume/export cap at :2293. | I; distinguishes archival/retention from compaction. |

## C/D/E. Learning, skills and curator

| Source / key symbols | Purpose and important dependencies | Relevant tests inspected |
|---|---|---|
| [agent/background_review.py:979][review]; build_cache_parity_fork, _run_review_fork at :1189 | Detached artifact learning, shared prefix with dispatch whitelist at :1116, budget/routing/cancellation; trigger via agent/turn_finalizer.py:591,862 and run_agent.py:759. | V: tests/agent/test_background_review_cache_parity.py:126; tests/agent/test_background_review_cost_controls.py:44. |
| [agent/learn_prompt.py:136][learn]; build_learn_prompt; hermes_cli/cli_commands_mixin.py:1313 | Explicit foreground /learn authoring prompt; normal tools write artifacts; separate from /refine review and automatic counters. | I; model judgment is not an objective learning validator. |
| [agent/skill_utils.py:413][skillroots]; get_skill_search_roots; project iterator at :635 | Trusted-project/profile/create/external precedence, ambiguity checks, platform/readiness metadata and quarantine. Depends on scanning/trusted_project_dirs. | I; runtime source policy distinct from hub install policy. |
| [tools/skills_tool.py:613][skillview]; skill_view, _skill_catalog at :229 | Skill/supporting-file load, read marks, dependency pm.ensure at :671, use/view at :758; prompt index in agent/prompt_builder.py:1233. | I; skill use is observed load, not successful application. |
| [tools/skill_manager_tool.py:783][skillwrite]; skill_manage, _guarded_write at :407, _patch_skill at :523 | Shape/path/size/frontmatter/fresh-read checks, fuzzy patch and atomic writes; guards in tools/skill_manager_guards.py:143,165,224; optional staging in tools/write_approval.py:201. | V: tests/tools/test_skill_linter.py:45,93,114,131,196; batch tests below. |
| [tools/skill_manager_batch.py:219][skillbatch]; _skill_manage_batch | Max20 operations, stable per-skill locks, touched-tree snapshots and filesystem rollback; delete sole operation. | V: tests/tools/test_skill_manage_batch.py:46,69,81,124,279. |
| [tools/skills_sync.py:113][skillsync]; bundled hash sync; tools/skills_hub_install.py:143 install | Preserve modified/suppressed bundled copies; hub source/scan/quarantine/install locks; source ABC tools/skills_hub_models.py:127, HubLockFile tools/skills_hub.py:322. | I/O; optional skills/hub packages not automatically active. |
| [tools/skill_usage.py:348][usage]; activity/management policy at :123,290; archive/restore at :642,672 | Atomic usage JSON, ownership, pins/locks and lifecycle; learn-owned exclusion at :522. | V via tests/agent/test_curator.py:277,379; I for each counter field. |
| [agent/curator.py:931][curator]; run_curator_review, transitions at :209, maybe_run_curator at :1220 | Weekly deterministic stale/archive plus optional LLM consolidation; first-run defer, protected references, reports/hooks. Idle caller caveats gateway/run.py:4331 and hermes_cli/cli_tui_runtime_mixin.py:412. | V: tests/agent/test_curator.py:77,98,277,379. Consolidation O, disabled default. |
| [agent/curator_backup.py:406][backup]; rollback; snapshot/retention at :30,113,153 | Tar backup safety capture, staged extraction and rollback recovery; excluded archive/ledger caches preserved. | V: tests/agent/test_curator_backup.py:103,138,165,395,495. |
| [tools/skill_ledger.py:569][ledger]; rollback; blob/JSONL store at :84 | Best-effort deltas, content-addressed recovery and maintenance cap/GC; undo requires complete safe capture. | V: tests/tools/test_skill_ledger.py:59,144,297,371. |
| [agent/learning_graph.py:192][graph] | Derived lexical visualization; top4 overlaps/name weight6; not memory retrieval or a trained graph. | V: tests/agent/test_learning_graph.py:38,96. |

## G/H. Delegation, tools and integrations

| Source / key symbols | Purpose and important dependencies | Relevant tests inspected |
|---|---|---|
| [tools/delegate_tool.py:566][delegate]; delegate_task, _build_child_agent at :203, _run_single_child at :377 | Batch tasks, trusted routes, focused children, schema/result/cleanup; default config tools/delegate_tool_config.py:19,23,187; inherited grants tools/delegate_tool_toolsets.py:13. | V: tests/tools/test_delegate_timeout_cleanup.py:62; route/backends optional. |
| [tools/async_delegation.py:115][asyncdelegate]; dispatch/completion at :175,231; record_unit_child at :244; claim at :586 | Durable result delivery with bounded leases/replay, abandoned-owner recovery at :295; no worker-stack continuation. | V: tests/tools/test_async_delegation.py:499,1063. |
| [agent/subagent_lifecycle.py:269][lifecycle]; SubagentLifecycleService; parent-bound HMAC at :389 | Optional plugin child API status/wait/steer/cancel, process-lifetime handles; distinct from durable async rows. | O/I; not a restart-reconnect API. |
| [tools/registry.py:121][registry]; discover_builtin_tools, ToolRegistry.register at :735, dispatch at :972 | AST discovery/cache, schema/handler metadata, scope overlays, availability probes and exception normalization. | V: tests/tools/test_registry.py:32,43,65,83,287,382. |
| [model_tools.py:897][dispatch]; handle_function_call; get_tool_definitions at :227 | Toolset selection, aliases/bridges, middleware/guards/hooks/result transforms; inline core tool dispatch separate. Depends on toolsets.py and registry. | V for registry contracts; I for every bridge path. |
| [tools/terminal_tool.py:1472][terminal]; terminal_tool, task/container resolution at :517 | Backend/cwd/timeout/approval/lifecycle plan and foreground/background dispatch; backend constructors tools/terminal_tool_backends.py:133; child alias at :373. | I/O; supports local and optional remote/container backends. |
| [tools/file_tools.py:605][files]; read_file_tool, write_file_tool at :863; _get_file_ops at :312 | Backend-neutral reads, line/cap behavior, path locks/stale-write protection; tools/file_operations.py:207 and native atomic_write at tools/file_operations_native.py:53. | I; security seams in section J. |
| [tools/code_execution_tool.py:707][code]; execute_code | Persistent Python/RPC orchestration, approval, scrub/whitelist/budget/output limits; explicit non-jail boundary at :713. | I; local project environment is not an OS sandbox. |
| [tools/browser_use_cli.py:774][browser]; browser_exec; tools/browser_session.py:279 | Browser-harness execution, URL routing, CDP/cloud/local sessions and signed-out/cloud fallback metadata. | V/O: tests/tools/test_browser_cloud_fallback.py:31,49,62. |
| [tools/web_tools.py:302][web]; web_search_tool; tools/connectors/dispatch.py:11 | Web provider selection/search cap and connector operations; explicit backend failures are not silently rerouted. | I/O; API/service/plugin availability required. |
| [tools/mcp_tool.py:143][mcp]; MCPServerTask; tools/mcp_tool_discovery.py:604; tools/mcp_tool_transport.py:154 | Optional SDK, discovery/schema cache/cooldowns, stdio/HTTP/SSE, OAuth/certs; breaker at tools/mcp_tool.py:516. | V/O: tests/tools/test_mcp_circuit_breaker.py:105. |
| [tools/mcp_tool_handlers.py:124][mcptrust]; tools/mcp_tool_registration.py:40 | Trust/annotation/write/cost approval, auth recovery and ambiguous-write no-replay at :316,349,396. | I/O; external annotations remain trusted inputs. |
| [hermes_cli/plugins.py:264][plugins]; PluginManager; hermes_cli/plugins_discovery.py:190,269; hermes_cli/plugins_context.py:270 | Manifest/category activation, profile-owned registrations/unload, override grants, portable MCP and skills; load deadline hermes_cli/plugins_loader.py:40. | V/I: tests/hermes_cli/test_plugins_disable_reaches_gateway.py:49,60,105; arbitrary native plugin code is not contained. |

## I. Gateway and automation

| Source / key symbols | Purpose and important dependencies | Relevant tests inspected |
|---|---|---|
| [gateway/run.py:3322][gateway]; GatewayRunner; gateway/run_startup.py:1702 | Mixin composition, restore gate, warmup/platform startup, degraded service, profile adapters/watchers. | I; facade imports alone do not describe routing/ownership. |
| [gateway/session_identity.py:43][identity]; RoutingIdentity, canonical_identity at :141; gateway/session.py:699 build_session_key | Separate runtime/transport profile ownership, refusal of unresolved routes and stable conversation keys. | V: tests/gateway/test_multiplex_transport_matrix.py:86. |
| [gateway/run_runtime.py:38][authority]; initialize per-profile SessionAuthority; gateway/run_inbound.py:1305 | Canonical state.db authority admission before legacy busy fallback; authorization/startup admission at gateway/run_inbound.py:149. | V/I; gateway/config.py:617 optional admission cap; gateway/turn_executor.py:25 unbounded workers. |
| [gateway/run_turn.py:2132][gatewayturn]; _run_agent_inner; gateway/run_turn_runner.py:1001 | TurnContext/TurnRunner, off-loop synchronous core, approvals/watchers/final stream reconciliation and finally cleanup. | V: tests/gateway/test_stream_final_contract.py:63. |
| [cron/jobs.py:1809][jobs]; create_job, parse_schedule at :787, compute_next_run at :1191 | Locked/replaced jobs.json, schedule/timezone/grace policy; tool create path tools/cronjob_tools.py:676. | V for occurrence behavior below; I for all parser branches. |
| [cron/executions.py:50][executions]; execution schema/state | Separate synchronous-FULL ledger, occurrence/process identity, terminal/unknown/delivery outcomes; independent from state.db. | V via canonical/scheduled-occurrence tests. |
| [cron/scheduler_tick.py:33][tick]; tick/_tick_admitted; cron/scheduler.py:3273 _run_one_job_body | Lifecycle/tick locks, advance-before-dispatch, dedup/claims, emergency stop and owner requirement. Agent construction cron/scheduler.py:2481; pool :1200. | V: tests/cron/test_scheduled_occurrence.py:56; tests/cron/test_canonical_cron.py:8. |
| [cron/scheduler_authority.py:55][cronauthority]; run_canonical_job; reconcile_pending at :149 | Stable admission journals/RPC or direct owner request; unknown outcome observation and pause, no blind resubmit. | V: tests/cron/test_canonical_cron.py:26,59. |
| [gateway/session_cron.py:104][cronowner]; operation/execute at :276; cron/scheduler_bookkeeping.py:52 finish_completed_run | Frozen stored-job policy, exact owner execution, result receipt and firing-side persistence/delivery cleanup. | V via canonical cron tests; execution/delivery distinct. |
| [cron/scheduler_provider.py:136][scheduler]; CronScheduler, ticker/profile enumeration at :74 | Experimental timing provider seam, shared execution/delivery, default60s, failure fallback/backoff. | O, explicitly experimental at :1. |
| [gateway/run_notifications.py:1828][events]; arm_process_watcher; gateway/run_goals.py:138 | Scoped process completion synthetic turns, admission receipts/retries, heartbeat and event lanes. | I; plugin platform event authorization at gateway/run_adapters.py:1660. |
| [hermes_cli/kanban_db_dispatch.py:2355][kanban]; dispatch_once/_default_spawn at :2754 | Separate durable board claims, budgets/review reserve, scrubbed profile worker processes and explicit completion protocol; hermes_cli/kanban_db.py:563 path. | V: tests/hermes_cli/test_kanban_dispatch_lock.py:45,79. |

## J/K. Security, configuration and model providers

| Source / key symbols | Purpose and important dependencies | Relevant tests inspected |
|---|---|---|
| [hermes_cli/config.py:1979][config]; load_config; hermes_cli/config_effective.py:52 load_user_config_effective | Full/default versus no-default effective view; deep merge/env expansion/managed-last, versioned cache and last-good/fail-closed modes. Backend protocol hermes_cli/config_backend.py:63. | V: tests/hermes_cli/test_config_effective.py:57,82. |
| [hermes_cli/env_loader.py:652][env]; load_hermes_dotenv; agent/secret_scope.py:407 build_profile_secret_scope | Dotenv/external/managed precedence, multiplex-safe bound mappings; get_secret at agent/secret_scope.py:230 rejects unscoped/foreign leaks. | V: tests/agent/test_secret_scope.py:36,101,169. |
| [tools/approval.py:1276][approval]; check_all_command_guards; floors at :1189; tools/approval_smart.py:123 | Isolation-aware guard order, deny/YOLO/grants/human/smart; hermes_cli/config_defaults.py:1660; persisted toggle tools/approval_yolo.py:32. | V: tests/tools/test_smart_approval_injection.py:103,133,164; tests/tools/test_approval_yolo.py:22. |
| [agent/file_safety.py:178][safety]; tools/file_tools_write_guards.py:147,214,263 | Sensitive path/read/write policy; config hard denial and operation-specific project-instruction approval; home exemptions at :235. | I for indexed path policy; not universal shell enforcement. |
| [tools/url_safety.py:316][urls]; is_always_blocked_url; hermes_cli/urllib_security.py:38,128 | Metadata/private URL policy and credential stripping on cross-origin redirects; context scanning agent/prompt_builder.py:84; logs hermes_logging.py:357,646. | I; regex/redaction do not establish universal injection prevention. |
| [providers/base.py:142][providerprofile]; ProviderProfile; providers/__init__.py:140,218,889 | Declarative profiles/aliases/home overlays and discovery; plugins/model-providers holds bundled implementations. | I; transport tests below cover selected behavior. |
| [hermes_cli/runtime_provider.py:1044][runtime]; resolve_runtime_provider; resolve_requested_provider at :539 | Explicit/config/env/auto selection, special/custom/auth/pool/OAuth/provider ladder at :1111, app-server overlay. | V: tests/hermes_cli/test_provider_precedence.py:38,46. |
| [agent/transports/base.py:12][transport]; ProviderTransport; agent/transports/__init__.py:23; agent/transports/anthropic.py:33; agent/transports/chat_completions.py:37 | Provider wire conversion/normalization, thinking/tool replay and request quirks; cache key agent/transports/chat_completions.py:77. Dedicated clients remain separate. | V: tests/providers/test_transport_parity.py:31,135. |
| [agent/credential_pool.py:966][pool]; CredentialPool; strategy at :559; provider match at :638; rotate at :2255 | auth.json credential/status/priority persistence; soft leases in memory at :981,2335,2375. Counts update in memory and persist on later writes, immediately for round-robin at :2152,2154. Client lifecycle agent/client_lifecycle.py:209. | V: tests/agent/test_credential_pool_provider_boundary.py:13,32. |
| [agent/auxiliary_client.py:4498][aux]; _resolve_auto_route; call_llm at :7903 | Main provider/model preference, explicit per-task override/fallback, separate completion hooks; title fast-model opt-in at :838. | I; review route behavior tests at tests/agent/test_background_review_cost_controls.py:44. |
| [agent/moa_loop.py:619][moa]; _run_references_parallel; _run_reference at :414; _trim_messages_for_reference at :500 | Optional parallel reference-model advice before ordinary tool-owning agent iterations; MoAClient at :1486. Separate from delegate_task child execution. | O/I; no tests inspected for this additional lane. |
| [agent/usage_pricing.py:640][pricing]; normalize_usage, estimate_usage_cost at :700; agent/turn_usage.py:73 | Provider-specific usage buckets, pricing precedence/Decimal estimates, unavailable usage/rates, session accounting; agent/provider_projection.py:25 avoids double tool execution. | V: tests/agent/test_usage_pricing.py:144,163,655,671. |
| [hermes_cli/setup_quick.py:251][smartkey]; hermes_cli/config.py:955,2967 smart_model_routing | Setup/config key registered; no runtime task-complexity/model-routing consumer located in inspected Python search. | D/U, not an implemented adaptive routing claim. |

## Reading and validation notes

- Facades frequently late-import sibling implementations. For example, AIAgent/run_agent, SessionDB/hermes_state, GatewayRunner/gateway.run, MCPServerTask/tools.mcp_tool and PluginManager/hermes_cli.plugins are navigation/compatibility seams, not self-contained implementations.
- Skill validation and delete guards are operation-specific: explicit root SKILL.md file writes/patches can skip full frontmatter validation, and remove_file can unlink it without delete-specific pin/ownership/absorption guards. Lock preflight still applies (tools/skill_manager_tool.py:376,560,619,644). Static finding, not executed.
- A test listed for a compound row supports the described contract, not every branch or numeric default in that row. Untested defaults were read at implementation/config call sites.
- Repository AGENTS.md files, comments and sample configuration contain drift. Use actual defaults and call sites: AIAgent unlimited constructor; child10/depth1; durable async results; cron memory on/review off; in-place lean compaction; caller-dependent curator idle.
- The main report's six traces connect these source seams. External provider/model/platform service success and optional backend semantic guarantees remain unverified without execution.

[entries]: https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/pyproject.toml#L592
[boot]: https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_bootstrap.py#L576
[acp]: https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/acp_adapter/entry.py#L163
[tui]: https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/tui_gateway/entry.py#L321
[homes]: https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_constants.py#L194
[core]: https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/run_agent.py#L214
[turn]: https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/turn_facade.py#L23
[loop]: https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/conversation_loop.py#L1646
[context]: https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/turn_context.py#L1189
[prompt]: https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/system_prompt.py#L751
[toolround]: https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/turn_tool_round.py#L53
[executor]: https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/tool_executor.py#L136
[errors]: https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/turn_api_error.py#L56
[final]: https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/turn_finalizer.py#L619
[memorytool]: https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/tools/memory_tool.py#L250
[memorystore]: https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/tools/memory_tool_store.py#L350
[memoryprovider]: https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/memory_provider.py#L103
[memoryplugins]: https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/plugins/memory/__init__.py#L52
[state]: https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_state.py#L201
[persist]: https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/session_persistence.py#L509
[messages]: https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_state_messages.py#L908
[compress]: https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/context_compressor.py#L2799
[compression]: https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/conversation_compression.py#L4028
[recall]: https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/tools/session_search_tool.py#L603
[retention]: https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_state_maintenance.py#L490
[review]: https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/background_review.py#L979
[learn]: https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/learn_prompt.py#L136
[skillroots]: https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/skill_utils.py#L413
[skillview]: https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/tools/skills_tool.py#L613
[skillwrite]: https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/tools/skill_manager_tool.py#L783
[skillbatch]: https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/tools/skill_manager_batch.py#L219
[skillsync]: https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/tools/skills_sync.py#L113
[usage]: https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/tools/skill_usage.py#L348
[curator]: https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/curator.py#L931
[backup]: https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/curator_backup.py#L406
[ledger]: https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/tools/skill_ledger.py#L569
[graph]: https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/learning_graph.py#L192
[delegate]: https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/tools/delegate_tool.py#L566
[asyncdelegate]: https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/tools/async_delegation.py#L115
[lifecycle]: https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/subagent_lifecycle.py#L269
[registry]: https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/tools/registry.py#L121
[dispatch]: https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/model_tools.py#L897
[terminal]: https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/tools/terminal_tool.py#L1472
[files]: https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/tools/file_tools.py#L605
[code]: https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/tools/code_execution_tool.py#L707
[browser]: https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/tools/browser_use_cli.py#L774
[web]: https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/tools/web_tools.py#L302
[mcp]: https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/tools/mcp_tool.py#L143
[mcptrust]: https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/tools/mcp_tool_handlers.py#L124
[plugins]: https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/plugins.py#L264
[gateway]: https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/gateway/run.py#L3322
[identity]: https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/gateway/session_identity.py#L43
[authority]: https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/gateway/run_runtime.py#L38
[gatewayturn]: https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/gateway/run_turn.py#L2132
[jobs]: https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/cron/jobs.py#L1809
[executions]: https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/cron/executions.py#L50
[tick]: https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/cron/scheduler_tick.py#L33
[cronauthority]: https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/cron/scheduler_authority.py#L55
[cronowner]: https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/gateway/session_cron.py#L104
[scheduler]: https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/cron/scheduler_provider.py#L136
[events]: https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/gateway/run_notifications.py#L1828
[kanban]: https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/kanban_db_dispatch.py#L2355
[config]: https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/config.py#L1979
[env]: https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/env_loader.py#L652
[approval]: https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/tools/approval.py#L1276
[safety]: https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/file_safety.py#L178
[urls]: https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/tools/url_safety.py#L316
[providerprofile]: https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/providers/base.py#L142
[runtime]: https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/runtime_provider.py#L1044
[transport]: https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/transports/base.py#L12
[pool]: https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/credential_pool.py#L966
[aux]: https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/auxiliary_client.py#L4498
[pricing]: https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/usage_pricing.py#L640
[smartkey]: https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/setup_quick.py#L251
[moa]: https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/moa_loop.py#L619


## Follow-up contracts C18–C25: verified evidence appendix

Revision: **4ad61cee29316e21e03c4585bb4ececfa52385fe**, source HEAD confirmed before research. Scope: appended architecture §§12–19. The user authorized using the existing workspace files as immutable prefixes. They contain §§1–10 and an unnumbered index; missing §11/C01–C17 have not been fabricated or renumbered. Every preceding byte/link definition is retained. These contracts replace no unseen provisional wording; they state the source-verified contract now established for each requested topic.

Legend: **V** implementation plus selected inspected test assertions; **I** source inspection only for the particular claim; **O** optional/experimental; **D** documentation/config-only; **U** unestablished/external. **Zero tests were executed.** Test references below identify assertions actually inspected, not a claim of full coverage. Link targets pin the revision.

### C18 — Evaluation and quality assurance (§12)

**Verified contract:** engineering tests, explicit model-assisted experiments and opt-in runtime verification are separate lanes. The inspected compaction scorecard is an experiment artifact; no automatic scorecard-to-review/curator consumer was located. Evaluation is not a proven reward/skill-promotion controller. **V/I/O; automatic learning coupling U.**

| Source and key symbols | Purpose, dependencies and state | Relevant inspected tests / limits |
|---|---|---|
| [scripts/run_tests.sh:1](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/scripts/run_tests.sh#L1); [scripts/run_tests_parallel.py:556](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/scripts/run_tests_parallel.py#L556) | Canonical environment-scrubbed, per-file process runner; selects test environment, handles subprocess outcomes/timeouts/retries. CI consumer [.github/workflows/tests.yml:108](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/.github/workflows/tests.yml#L108). | Source-inspected only here; 300s/file, retry1, worker policy at :1250; deselected exit5 is not proof all tests ran. |
| [evals/compaction/runner.py:330](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/evals/compaction/runner.py#L330) run_policy; :411 main | Transcript/cached questions → compressor/policy → model answer/judge → result JSON; optional simulated FTS recovery at :91. report consumer [evals/compaction/report.py:35](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/evals/compaction/report.py#L35). | [tests/evals/test_compaction_jev_arm.py:19](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/tests/evals/test_compaction_jev_arm.py#L19) fake-asker text/pair contract; not a production-engine installation test. |
| [.github/workflows/live-providers.yml:14](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/.github/workflows/live-providers.yml#L14) | Tag/manual live-canary workflow; secrets supplied only to live step, direct pytest rather than scrubbed runner. | Optional configured-secret availability; CI execution/branch-protection requirement not established. |
| [agent/verification_stop.py:62](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/verification_stop.py#L62) verify_on_stop_enabled; [agent/verification_evidence.py:123](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/verification_evidence.py#L123) | Opt-in evidence database/finish-time policy; command attribution and edit freshness, not automatic suite execution. pre_verify bound [agent/verify_hooks.py:16](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/verify_hooks.py#L16). | [tests/agent/test_verification_stop.py:138](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/tests/agent/test_verification_stop.py#L138) bounded continuation; default guard off. |

**Failure/acceptance:** judge parse failure=0; failed policy can record unknown recall; runner retry/skip semantics require interpreting artifacts, not just exit. No universal semantic task-success oracle established (§12).

### C19 — Observer hooks, middleware and event lifecycle (§13)

**Verified contract:** first-party observation precedes compatibility callbacks; consumers, not names, decide policy effects. Request middleware rewrites; execution middleware wraps/short-circuits one downstream call per frame. In-memory event bus is not durable gateway admission. **V/I/O.**

| Source and key symbols | Ownership/data flow | Relevant inspected tests / limits |
|---|---|---|
| [hermes_cli/lifecycle.py:30](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/lifecycle.py#L30) invoke_hook; :99 finalize_session | Safe-worker-aware observer→plugin composition; true finalization separate from per-turn on_session_end. | Compatibility behavior via [tests/hermes_cli/test_plugin_api_compat.py:14](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/tests/hermes_cli/test_plugin_api_compat.py#L14). |
| [hermes_cli/middleware.py:88](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/middleware.py#L88) apply_llm_request_middleware; :102 tool request; :162 execution chain | Snapshot/replacement payload and rewrite trace; single-use next_call, downstream-failure propagation; deepcopy can degrade shallow. | [tests/hermes_cli/test_plugins_middleware_chain.py:50](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/tests/hermes_cli/test_plugins_middleware_chain.py#L50), :60 composed rewrite/normal-copy assertions. No general chain timeout. |
| [hermes_cli/plugins_dispatch.py:275](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/plugins_dispatch.py#L275) invoke_hook; :351 bounded callback; :529 emit | Manager-owned callbacks/subscriptions; native pre_tool error fails closed, selected observers skip; queue/depth caps. | Source bounds30s/max600s, suppression60s, abandoned3, pending64/depth8; no durable replay asserted. |
| [agent/shell_hooks.py:319](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/shell_hooks.py#L319) consent; :484 spawn; :560 result | Event/command consent, JSON stdin, subprocess timeout/tree kill, exit2 block; configured fail_closed differs from native default. | Runtime inspected; no new shell-hook tests executed. Timeout60s/max300s; mtime is not content authentication. |
| [agent/outbound_webhooks.py:224](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/outbound_webhooks.py#L224) callback; :272 enqueue; :315 deliver | Notify-only export, in-memory queue and bounded HTTP retry; no redirects. | Queue256, timeout10s/max60s, attempts2; failure/drop is not exactly-once delivery. |

**Failure/acceptance:** a timed-out native thread can still mutate state; not all hooks are timeout-bounded; middleware is executable policy and may short-circuit intentionally. §13 qualifies fail-open/fail-closed per lane.

### C20 — Observability, correlation and execution tracing (§14)

**Verified contract:** logs/transcript, Relay scopes, aggregate telemetry and transcript export are different products. Supported in-process paths have explicit identity propagation, not proven universal distributed trace continuity. **V/I/O; full cross-process/service propagation U.**

| Source and key symbols | Purpose/ownership/state | Relevant inspected tests / boundary |
|---|---|---|
| [agent/turn_context.py:640](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/turn_context.py#L640); [agent/api_request_hooks.py:163](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/api_request_hooks.py#L163); [agent/auxiliary_hooks.py:25](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/auxiliary_hooks.py#L25) | Task/turn/request identity and bounded hook error payloads; auxiliary parent IDs depend on current turn. | Not every turn-less caller has a parent trace identity. |
| [agent/relay_runtime.py:399](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/relay_runtime.py#L399) RelayRuntime; :863 RelayHostRegistry; :842 NoopRelayRuntime | Profile host/session/child maps, runtime UUID, managed-consumer leases; scope operations10s/8 workers. Native plugin activation has process-shared state at :154. | [tests/agent/test_relay_compaction.py:85](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/tests/agent/test_relay_compaction.py#L85), :103, :114 checks own-turn/session parent selection. |
| [agent/relay_llm.py:681](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/relay_llm.py#L681) logical scope; :785 provider header bridge; [agent/relay_tools.py:16](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/relay_tools.py#L16) | Physical attempts/logical requests/tool execution; supported traceparent transport, unsupported native paths strip it. | No blanket remote MCP/CLI/cron trace guarantee. |
| [agent/trace_upload.py:49](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/trace_upload.py#L49) redaction; :145 build_trace_jsonl | Explicit viewer export; omits system/images, synthesizes UUIDs and conversion timestamps; upload credential path :170. | [tests/agent/test_trace_upload.py:43](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/tests/agent/test_trace_upload.py#L43), :62, :68, :79 checks format/redaction refusal; no upload performed. |

**Failure/acceptance:** bounded/best-effort observations can be missing; durable receipts are not persisted span stacks; optional native runtimes/exporters require their real availability/configuration.

### C21 — Replaceable context engine and strategy boundaries (§15)

**Verified contract:** per-agent engine contract plus host-owned persistence/sanitization/safety; request selection and committed compression are distinct. No generic arbitrary-backend state migration contract found. **V/I/O; unrestricted interchangeability U.**

| Source and key symbols | Responsibilities/dependencies | Relevant inspected tests / caveat |
|---|---|---|
| [agent/context_engine.py:48](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/context_engine.py#L48) ContextEngine; :121 select_context; :226 clone_for_agent | ABC methods plus directly read state attributes; model/lifecycle/status/tool seams; deepcopy default. | Contract is more than implementing compress. |
| [agent/agent_init.py:1752](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/agent_init.py#L1752) selection; :1822 construction; :1965 injected tools | Explicit compressor/default, category→registered clone, warning fallback; host model capacity/tool grants. | [tests/agent/test_plugin_context_engine_clone.py:49](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/tests/agent/test_plugin_context_engine_clone.py#L49), :56 verifies usable clone and model isolation. |
| [plugins/context_engine/__init__.py:32](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/plugins/context_engine/__init__.py#L32) discovery; :60 load | Bundled-first category lookup, user/hosted loaders; separate from general enablement. | Optional engine implementation availability not inferred from interface names. |
| [agent/conversation_loop.py:1357](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/conversation_loop.py#L1357) selection; :1407 completion | Reference-history clones, structural replacement validation, stripped persistence fields, best-effort completion. | [tests/agent/test_context_engine_select_context.py:114](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/tests/agent/test_context_engine_select_context.py#L114), :137, :178. Request itself mutable: fallback need not undo in-place mutation (I). |
| [agent/agent_init.py:1871](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/agent_init.py#L1871); [agent/conversation_compression_checkpoint.py:97](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/conversation_compression_checkpoint.py#L97) | Built-in-specific prepared compression, checkpoint/native-compaction compatibility gate. | Cannot assume all strategies have identical caching/lossy-compaction guarantees. |

**Failure/acceptance:** failed engine discovery/clone falls back; invalid selection uses assembled request; abnormal exits can skip completion callback. Host commit authority remains separate from engine-provided context. Selection runs at [agent/turn_request_assembly.py:145](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/turn_request_assembly.py#L145); the inner [API retry loop](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/conversation_loop.py#L1577) rebuilds kwargs, not selection. The ABC's “also retries” description is therefore broader than the directly inspected call path (§15.2, I).

### C22 — SQLite and configuration migrations (§16)

**Verified contract:** independent version domains, declarative shape repair plus gated data transforms; config ladder can skip failed steps and still stamp latest. **V/I; universal downgrade/rollback U.**

| Source and key symbols | Purpose/state/algorithm | Relevant inspected tests / risk |
|---|---|---|
| [hermes_state_schema.py:899](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_state_schema.py#L899) _init_schema; :731 reconciliation; :1020 data chain | Writable initialization, declared columns, PK repairs/index/FTS probes; version33 from [hermes_state_common.py:295](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_state_common.py#L295). | Duplicate-column race tolerated, locks re-raised; other shape failures may warn. Read-only opens skip initialization. |
| [hermes_state_schema.py:1100](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_state_schema.py#L1100); :1127 UID backfill | Independent FTS-layout opt-in and UID cursor/chunks; not one universal transaction. | 2000-rowid chunks, ~1s post-first-chunk budget; current version alone does not prove all repairs/layouts. |
| [hermes_cli/config.py:1175](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/config.py#L1175) migrate_config; :1240 stamp | Parse/env/ladder/security checks/stamp; defaults read-time, latest53 [hermes_cli/config_defaults.py:2703](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/config_defaults.py#L2703). | [tests/hermes_cli/test_config_unversioned_migration.py:45](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/tests/hermes_cli/test_config_unversioned_migration.py#L45) preserves user values/SOUL. |
| [hermes_cli/config_migrations.py:906](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/config_migrations.py#L906) run_migrations; :893 key steps | Starting-version comparison, unversioned legacy-only transforms; independent steps46/50, floor12, config_only. | [tests/hermes_cli/test_config_migration_nested_guards.py:65](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/tests/hermes_cli/test_config_migration_nested_guards.py#L65) explicitly asserts warning plus latest stamp after failure. |

**Failure/acceptance:** no atomic rollback across schema/YAML/.env/artifacts/profiles; updates' optional snapshots are separate. A migration/version marker is not an all-steps-success receipt.

### C23 — Updates, deployment and runtime lifecycle (§17)

**Verified contract:** install ownership gates updates; source movement commits before completion/restart obligations converge. Durable repair is not arbitrary rollback of data/tool effects. **V/I/O; live platform behavior U.**

| Source and key symbols | Responsibilities and persistent state | Relevant inspected tests / limits |
|---|---|---|
| [hermes_cli/update_contract.py:82](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/update_contract.py#L82) admission; [hermes_cli/update_cmd.py:1771](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/update_cmd.py#L1771) command | Install contract/steward checks, target selection, checkout ownership, receipt/backup/pause orchestration. | [tests/hermes_cli/test_update_contract.py:90](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/tests/hermes_cli/test_update_contract.py#L90) managed refusal; malformed contract failure at :105. |
| [hermes_cli/update_cmd_commit.py:1](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/update_cmd_commit.py#L1); [hermes_cli/update_cmd.py:1731](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/update_cmd.py#L1731) | Interrupted-pull marker, completion/fleet obligations before movement; expected HEAD validation. | Scope-specific rollback/disarm, not data migration reversal. |
| [hermes_cli/update_cmd_maint.py:756](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/update_cmd_maint.py#L756) backup; [hermes_cli/config_defaults.py:2331](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/config_defaults.py#L2331) | quick/full/off snapshot policy and result; default quick, full keep5, >1GiB skip. | Failure warns/continues, so configured backup is not a proven recovery point. |
| [hermes_cli/update_completion.py:322](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/update_completion.py#L322); [hermes_cli/venv_sync.py:251](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/venv_sync.py#L251) | New-tree PM generation sync, plugin incompatibility eviction, owed-tail retry/receipts. | Retry after2 backs off15min, max6 for unchanged inputs; unreadable bookkeeping permissive. |
| [pm/publication.py:163](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/pm/publication.py#L163) publication; [pm/environments.py:464](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/pm/environments.py#L464) activation | Staged digest/metadata journal and leased generations; plugin admission separate from core eviction. | [tests/hermes_cli/test_plugin_update_transaction.py:171](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/tests/hermes_cli/test_plugin_update_transaction.py#L171), :207 asserts preservation and matching success generation. |
| [docker/entrypoint-dispatch.sh:17](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/docker/entrypoint-dispatch.sh#L17) | PID1→s6; non-PID1 direct bootstrap without supervised services. | Executable branch inspected; no container/service started. |

**Failure/acceptance:** source code can be committed with dependencies/build/restarts owed; tests for plugin publication do not establish all-or-nothing core update. Early torn-tree recovery precedes application imports ([hermes_cli/main.py:26](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/main.py#L26)).

### C24 — Insights, efficiency and resource governance (§18)

**Verified contract:** reporting/optional telemetry projects measured usage; local limits have explicit scopes and do not constitute one global governor. **V/I/O; automatic optimization/global atomic budget U.**

| Source and key symbols | Inputs → output/state | Relevant inspected tests / limits |
|---|---|---|
| [agent/insights.py:196](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/insights.py#L196) generate; :150 model usage query | SessionDB accounting flush plus report aggregation, default30d, source filter, session-start cutoff. | [tests/agent/test_insights.py:264](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/tests/agent/test_insights.py#L264) separate estimated/actual sums; not success/utility scoring. |
| [hermes_cli/observability/shared_metrics_subscriber.py:68](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/observability/shared_metrics_subscriber.py#L68); [hermes_cli/observability/shared_metrics.py:36](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/observability/shared_metrics.py#L36) | Validated event dimensions/counters→profile aggregate SQLite; schema2/packagev4. | Collection/send off by default [hermes_cli/config_defaults.py:2311](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/config_defaults.py#L2311); deferred1024, retention30d. |
| [hermes_cli/observability/shared_metrics_efficiency.py:173](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/observability/shared_metrics_efficiency.py#L173) SessionEfficiency | Turn cost/discard/cache/tool overhead; TTL300s heuristic and user/internal distinction. | [tests/hermes_cli/test_shared_metrics_efficiency.py:69](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/tests/hermes_cli/test_shared_metrics_efficiency.py#L69) turn buckets and interrupt waste. |
| [hermes_cli/observability/shared_metrics_sender.py:206](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/observability/shared_metrics_sender.py#L206); [hermes_cli/observability/shared_metrics_consent.py:57](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/observability/shared_metrics_consent.py#L57) | Consent-window/rechecked bounded export, not quota enforcement. | Timeout30s, attempts3, packages20/pass, cumulative25; no export executed. |
| [agent/iteration_budget.py:25](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/iteration_budget.py#L25); [agent/chat_completion_helpers.py:678](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/chat_completion_helpers.py#L678); [agent/conversation_loop.py:126](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/conversation_loop.py#L126) | Local consume/refund counter, per-turn nudge/implicit stale timeout cap. | [tests/agent/test_run_budget.py:103](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/tests/agent/test_run_budget.py#L103), :118, :143 checks half-remaining/floor/explicit exceptions. |
| [gateway/turn_executor.py:25](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/gateway/turn_executor.py#L25); [tools/delegate_tool_config.py:19](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/tools/delegate_tool_config.py#L19); [agent/credits_tracker.py:583](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/credits_tracker.py#L583) | Distinct host/session/child/tool concurrency and provider credit state. | Aggregated accounting is not atomic pre-fanout spend reservation or host-wide CPU/RAM control. |

**Failure/acceptance:** unknown usage/pricing remains unknown; deferred metrics/events can be lost; nudge/stale timeout is not hard total wall-clock termination. No report→automatic curator/routing consumer established in inspected paths.

### C25 — Plugin lifecycle, compatibility and knowledge provenance (§19)

**Verified contract:** scoped reversible registrations and PM publication; enforced/advisory compatibility fields differ; optional hosted native plugins; provenance records origin/management rather than fact truth. **V/I/O; universal compatibility/epistemic certification U.**

| Source and key symbols | Ownership/state/algorithm | Relevant inspected tests / boundary |
|---|---|---|
| [hermes_cli/plugins_ledger.py:30](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/plugins_ledger.py#L30) PluginRegistration; :88 leases; :118 staged publish; :132 persistent reconciliation | Identity-qualified cleanup/override restoration; selected infrastructure survives unload-all. | Arbitrary importer/native side effects not undone. |
| [hermes_cli/plugins_manifest.py:162](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/plugins_manifest.py#L162); :238 dependency order; :502 requires_hermes | Manifest2/additive warnings; api_version metadata, advisory deps/config; real version gate permissive for unparseable release. | [tests/hermes_cli/test_plugin_api_compat.py:14](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/tests/hermes_cli/test_plugin_api_compat.py#L14) frozen narrow-signature legacy plugin via real discovery. |
| [hermes_cli/plugin_isolation.py:100](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/plugin_isolation.py#L100); [hermes_cli/plugin_host.py:142](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/plugin_host.py#L142); [hermes_cli/plugin_host_wire.py:326](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/plugin_host_wire.py#L326) | in_process default; per-profile host/proxies/origin contexts when selected; launcher can add external sandbox. | [tests/hermes_cli/test_plugin_host.py:66](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/tests/hermes_cli/test_plugin_host.py#L66) separate PID/ctx/proxy/crash recovery; host alone not OS sandbox, RPC timeout not remote cancellation. |
| [hermes_cli/plugins_cmd_git.py:33](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/plugins_cmd_git.py#L33); [hermes_cli/plugins_provenance.py:107](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/plugins_provenance.py#L107) | Installer metadata/lock and row+.git reconciliation; manual/self-cloned/drift distinguished. | Provenance is mutable local application state, not an authenticated safety/truth proof. |
| [hermes_cli/plugins_cmd_install.py:360](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/plugins_cmd_install.py#L360); :392 | Reviewed exact SHA, source-movement refusal, out-of-tree installer catalog metadata. | Code identity is not harmlessness; PM publication tests in C23 do not certify content. |
| [tools/skills_hub.py:322](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/tools/skills_hub.py#L322); [tools/skill_provenance.py:29](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/tools/skill_provenance.py#L29); [tools/skill_ledger.py:84](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/tools/skill_ledger.py#L84) | Hub source/hash/scan record; write-origin/attendedness; best-effort file history/recovery. | No universal signed fact graph or benchmark-certified promotion path found. |

**Failure/acceptance:** failed load disposes owned registrations; optional host uses30s hello/3s shutdown/restart budget3 per600s, but no universal RPC limit or side-effect rollback. Existing original native-isolation/prefix/budget statements receive appended correction proposals in §19 rather than silent edits.

### Preservation and reproducibility receipt

Baseline files captured before any follow-up edit:

| File | Preserved prefix bytes | SHA256 of that complete original prefix |
|---|---:|---|
| HERMES_ARCHITECTURE.md | 43,435 | f14823e44f211c69a364f4194477b19ff551dd3862af7c5902d320ff4dd39ff9 |
| HERMES_SOURCE_INDEX.md | 35,634 | cb91dbac5b46829c5475cd4672940e958d8de7075d3de4e2e898d40b824d6662 |

Validation method: compare each output's prefix, for the listed byte count, directly with its saved baseline and recompute SHA256. Check that appended main headings are exactly12–19; contracts exactlyC18–C25; substantive mechanisms have pinned source/call-site links and inspected tests are separated from execution. Validate added source paths/line bounds against the pinned Git tree, including sparse-checkout-excluded eval/docker blobs. Confirm unchanged source worktree and HEAD.

No Hermes execution, tests, dependency installation, update, deployment, trace upload, webhook delivery or telemetry export was performed. Source inspection cannot establish live service guarantees. See the main report's final synthesis for remaining U claims.

