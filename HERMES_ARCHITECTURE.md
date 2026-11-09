# Hermes Agent: Source Architecture Audit

Audit date: **2026-11-09**. Repository: [NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent). Pinned main revision: **4ad61cee29316e21e03c4585bb4ececfa52385fe**, dated 2026-11-09, “chore(catalog): review disclosure for sealed”.

Method: static source/call-site/configuration/test-assertion inspection in an acquired shallow/sparse checkout. Scope: Python runtime/integrations/skills; UI/website/exhaustive external code excluded. **No Hermes/tests execution, dependency installation or source modification.** No Pixel evaluation.

Status: **V** implementation plus inspected test assertions; **I** implementation without inspected test verification; **O** optional/experimental; **D** documentation/config surface, runtime consumer not located; **U** unresolved/external guarantee. V/I never mean passing execution. Pinned `path:line` references and expanded tests link through [source index](HERMES_SOURCE_INDEX.md).

## 1. Executive summary

Hermes combines a synchronous Python engine with profile-aware gateway ownership/delivery. Classic CLI/ACP use gateway clients/viewers; direct embedding remains.

| State | Mechanism |
|---|---|
| Working context | Pinned prompt/tools, messages and compaction. |
| Curated memory | Bounded MEMORY.md/USER.md, model mutations, frozen prompt snapshot. |
| Historical recall | SQLite transcripts/FTS, including compacted originals. |
| Learning | Skill artifacts, detached review, usage/curator metadata; no weight training/success proof. |

Core discipline: persist before effects/projection, preserve prefix except controlled rebuilds, extend through explicit seams. No shared transaction with services or automatic plugin isolation. Complexity lies in profile scopes, transport/runtime identity, leases/generations, reused clients and async delivery; facade modules still share mutable state.

## 2. System architecture diagram

~~~mermaid
flowchart TB
  CLI[CLI / one-shot] --> GW
  UI[TUI / web / desktop / ACP] --> GW
  CHAT[Messaging adapters] --> GW
  GW[Gateway: identity / authorization / delivery] --> SA
  CRON[Cron / scheduler provider] --> SA
  EVENT[Process completion / heartbeat] --> SA
  SA[Profile SessionAuthority: durable admissions] --> W[TurnRunner / workers]
  W --> A[AIAgent: prepare / request / tools / finalize]
  EMBED[Direct Python / legacy entry] --> A
  A --> P[Provider resolution / credentials / transport]
  P --> LLM[Model endpoint]
  A --> T[Inline tools / model_tools / ToolRegistry]
  T --> X[Terminal / files / browser / Python]
  T --> MCP[MCP / connectors / API plugins]
  T --> CHILD[Delegated AIAgent]
  A <--> DB[(Profile state.db)]
  CHILD --> DB
  A <--> MEM[Markdown / optional provider memory]
  A --> R[Detached review]
  R --> MEM
  R --> S[Skills / support files]
  CUR[Curator / archive / rollback] --> S
  S --> A
  CRON <--> J[Jobs / execution ledger / admission journals]
~~~

Platform events may invoke hooks without turns: hermes_cli/main.py:1601,2904; acp_adapter/entry.py:206; gateway/run_runtime.py:38; gateway/run_adapters.py:1660; agent/turn_facade.py:23.

## 3. Component inventory

| Area | Runtime role / principal seam |
|---|---|
| hermes; hermes_cli/; hermes_bootstrap.py; pm/ | Startup/profile dispatch/recovery/dependency generations. |
| run_agent.py; agent/turn_*.py | Synchronous AIAgent facade/phase machine. |
| agent/system_prompt.py; prompt_builder.py; context_compressor*.py | Prompt/context/compaction; optional engine overrides. |
| hermes_state.py; hermes_state_*.py | SessionDB/transcripts/search/generations/leases/repair. |
| gateway/; tui_gateway/; acp_adapter/ | Daemon authority/adapters/delivery and JSON-RPC/viewer surfaces. |
| tools/; model_tools.py; toolsets.py | Schemas/grants/guarded dispatch/backends; service-gated integrations. |
| providers/; plugins/model-providers/; agent/transports/ | Route metadata/provider conversion/clients. |
| agent/memory_*.py; background_review.py; curator*.py; tools/skill_*.py | Curated/external memory, artifact learning/maintenance. |
| cron/; hermes_cli/kanban_*.py | Scheduled execution and separate optional board/workers. |
| hermes_cli/plugins*.py; plugins/; optional-mcps/ | Activation/ownership/extensions; bundled or opt-in. |
| tests/; tests-js/; scripts/; evals/; website/ | Development/test/build/evaluation/intended behavior; supporting evidence. |

Entries: hermes→hermes_cli.main:main, hermes-agent→agent.legacy_cli:main, hermes-acp→acp_adapter.entry:main (pyproject.toml:592). Python metadata >=3.11,<3.15 supports updater; primary dependency markers >=3.14. OpenAI/httpx, dotenv/YAML, Rich/prompt_toolkit, croniter/browser-harness base; provider/platform SDKs also optional (pyproject.toml:11,32).

Bootstrap normalizes I/O/recovers update/activates PM, may relaunch Python; main handles trust store/update/dependency readiness/fast-full dispatch/plugins/hooks. PM leases generations and changes imports/PATH (hermes_bootstrap.py:576; hermes_cli/main.py:3336; pm/environments.py:464).

## 4. Data and storage architecture

Paths relative to **active profile home**, context override→HERMES_HOME→platform default (POSIX ~/.hermes, Windows LOCALAPPDATA/hermes); launch/routed home differ (hermes_constants.py:57,194).

| Store | Representation / lifecycle |
|---|---|
| config.yaml / .env / auth.json | Config/scoped secrets/credential state; separate precedence (§8). |
| state.db | SQLite WAL schema **33**; sessions/messages/prompts/admissions/receipts/generations/leases/delegation/usage. |
| memories/MEMORY.md / USER.md | Delimited locked/atomic curated entries; frozen snapshot, no vectors/TTL. |
| skills/ | SKILL.md/support files; hash-sync, view/edit. |
| skills/.usage.json / .curator_state / .archive/ | Ownership/counters/pin/lock/cadence and moved directories. |
| skills/.curator_backups/ | Tree tar, keep **2**. |
| skills/.curator_ledger.jsonl; home/.curator_backups/blobs | Best-effort mutation JSONL/SHA256 blobs and guarded undo. |
| pending/memory/; pending/skills/ | Unapplied approval proposals; destructive memory pins observed content. |
| cron/jobs.json; cron/output/ | Locked/replaced definitions, output files. |
| cron/executions.db; cron/admissions/ | synchronous FULL firing ledger/receipt journals. |
| cache/delegation/; cache/tool_discovery_cache.json | Full child results; path/mtime/size scan metadata. |
| logs/; logs/curator/; optional trajectories | Operational reports/JSONL, not canonical history. |
| root/kanban.db; board workspaces | Independent SQLite tasks/claims/files, board overrides. |

Sessions hold provenance/model/config/title/cwd/usage/revision and deduplicated prompt hashes. Messages distinguish clean text/api_content/reasoning/provider sidecars, stable IDs, active/compacted state (hermes_state_common.py:295,467,537,570). One writer/multiple readers; patience routine/transcript/activity **20/60/0.5s**, compression **5s**, checkpoint every **50** writes. DB/WAL replacement refuses continuation, recovery may JSONL (hermes_state.py:464,481,1282; agent/session_persistence.py:420). No cross-store/external-effect transaction. Normal IDs timestamp+six hex; special lanes differ; embedded persistence optional (hermes_state_ids.py:21; agent/agent_init.py:1098).

## 5. Detailed subsystem descriptions

### A. Agent core — V

AIAgent is a lifecycle/turn/persistence/compression/interrupt/provider facade. init_agent resolves identity/policy/budget → route/client/fallback/tools → DB → config/memory/context engine/accounting. User content/history/runtime state yields answer/messages/calls, completion/failure/interruption/exit reason, session/model/provider and costs (run_agent.py:214; agent/agent_init.py:2056).

run_conversation preempts review, leases ownership/scopes; build_turn_context restores prompt/history, compacts, invokes hooks/recall and persists inbound content. Shared phase state drives:

~~~text
reserve → prepare/build/preflight → call/retry → normalize
 tools: validate → persist calls → guarded dispatch → persist results → compact → repeat
 text: empty/truncation/continuation/stop gates → transform/persist/finalize
~~~

Calls persist before effects; canonical result-persistence failure stops inference. Argument/handler errors normally become paired tool results. Safe parallel segments alternate with barriers: **8 workers**, image cap **4**, batch timeout **420s** (agent/conversation_loop.py:1646; agent/turn_tool_round.py:139; agent/tool_executor.py:136).

| Control | Actual default/behavior |
|---|---|
| Iterations | Constructor **sys.maxsize**, direct run_agent.main **10**; sole execute_code can refund; one tool-less grace call. |
| Wall clock | Optional run_budget_seconds, warning **80%**. |
| Ownership/liveness | Lease TTL/wait/renew **300/1800/60s**; watchdog idle/poll **600/15s**, nonpositive disables. |
| API | **3 attempts**, fallback, then transient pre-delivery ladder **5** cycles, **15/30/60/60/60s**, 20% jitter; Retry-After cap **120s**. |
| Empty/errors | Empty retries **3**, costly **1** at **$0.25**; proven repeated empties skip ahead; escaping errors cap **8**, deterministic local bugs stop. |
| Stop | Verification/plugin/Kanban can continue; late LSP max **3**. |

Evidence: run_agent.py:241,1492; agent/turn_iteration_prep.py:352; agent/turn_facade_lease.py:32; agent/turn_liveness.py:23; agent/agent_init.py:1291; agent/turn_recovery_autorecover.py:44; agent/empty_response_guard.py:63; agent/conversation_loop.py:239; agent/turn_stop_gates.py:103,149. Auth/policy/billing/format failures bypass transient ladder; fallback rebuilds/preflights for new capacity. Independently guarded finalizer cleanup preserves answer and reports cleanup_errors (agent/turn_finalizer.py:619,736).

### B–E. Memory, learning, skills and curator — V / O

| System | Inputs → outputs / state | Components/interactions / failure boundary |
|---|---|---|
| B. Memory | Facts → curated entries; query/turn → recall | MemoryStore/inline tool + optional MemoryManager; budget/match/scan/approval; prompt/disk freshness differ. |
| C. Learning | Eligible conversation/correction → artifact mutation | Counter-gated background-review AIAgent or /learn; model judgment, whitelist/budget/cancel; partial writes survive. |
| D. Skills | Name/index/source → instruction/support files | Search/catalog, skill_view/manage/sync/hub; trust/readiness/ambiguity/dependencies and batch rollback. |
| E. Curator | Usage/cadence → lifecycle or optional merge | Metadata/archive/reports/backups/ledger/LLM; ownership/pin/lock/cron protection, best-effort capture. |

Algorithms, numeric defaults, mutation/configuration rules and evidence are consolidated in §7. Load frequency is not success; these stores do not share one transaction.

### F. Sessions and context — V; engine overrides O

Prompt tiers: stable identity/guidance/auto-load → caller/project/workspace → “volatile” skill/memory/plugin/runtime. All freeze normally. Resume restores exact prompt/pinned tools; compaction, explicit refresh, stale runtime/model or Bot Chat epoch permits rebuild; api_content retains sent injections apart from display text (agent/system_prompt.py:751; agent/conversation_loop.py:802).

Instruction precedence: nearest .hermes.md/HERMES.md → merged root-to-CWD AGENTS → CWD CLAUDE.md → Cursor; SOUL separate. File cap **6%** context, **20,000–500,000 chars**, explicit positive override; read **5s** (agent/prompt_builder.py:42,1033,1513,1720).

Compactor prunes tools, chooses safe head/middle/tail, makes one structured summary request/attempt and reconstructs identifiers/tool pairing. Base threshold **0.50**, >=**0.75** below **512,000 tokens**; input reserves output, bounded **64,000-token** floor. Default lean retains first **3**, minimum last **20** messages/>=**1** real user; tail **2.5%**, normally **10,000–25,000 tokens**, <=half trigger (agent/context_compressor.py:1002,2751,2799; agent/context_compressor_tail.py:26; agent/model_metadata.py:277).

**in_place=true** archives summarized rows and replaces active context; watermarks/covered IDs plus lease/CAS preserve concurrent arrivals. Rotation supported; no middle → unchanged; required checkpoint can block. Attempts **3**, range **1–10**; anti-thrash includes **300s**; proactive prune/microcompaction off (agent/conversation_compression.py:2859,4028; hermes_state_messages.py:908; agent/agent_init.py:1372).

Active-tip model history differs from display/compacted originals; parent ID alone is not compression lineage. session_search supports discovery/anchor-scroll/read/browse: **300** FTS hits, **2000 chars/item**, default ±**5**, lineage dedup/current-live exclusion but archived-current access. Named-profile reads explicit/read-only; BM25/boolean/prefix/trigram/LIKE fallback and source/cron ranking apply; absent FTS can yield no results (hermes_state_messages.py:1468,1672; tools/session_search_tool.py:23,603; hermes_state_search.py:1154). Prune/archive off; enabled **90d retention/3d idle/24h maintenance**, pin/open/writer guards; resume/export **20,000 messages** (hermes_cli/config_defaults.py:2255,2293).

### G. Multi-agent — V; alternatives O

delegate_task resolves batch/routes and focused children: explicit goal/context, not parent history; ordinary context/memory skipped. Fresh DB/lineage/budget, propagated scoped grants/credentials. Operator defaults **10 concurrent/depth1/250 iterations/no timeout**; configured timeout inactivity >=**30s**; one-shot **2**; worktrees/autoapproval off (tools/delegate_tool.py:203,566; tools/delegate_tool_config.py:19,23,57,87,168,187).

Grants inherit minus delegate/clarify/memory/send_message/cronjob_manage/kanban; eligible orchestrators regain delegation. Terminal/container/files usually shared; optional worktrees local Git only. Heartbeat **30s**, stale **450s idle/1200s tool**. One schema correction; invalid raw result preserved. Summary <=**24,000 chars**, parent-context bounded, full spill (tools/delegate_tool_toolsets.py:13; tools/terminal_tool.py:373; tools/delegate_tool_child_run.py:547; tools/delegate_tool_results.py:163).

Eligible top-level tasks detach, nested/unsupported/one-shot join. Tracked steer/stop/list; durable dispatch/finished-child/final rows; delivery TTL **300s**, **8 attempts**, age **48h**. Restart recovers outputs/abandonment, not stacks. HMAC parent-bound SubagentLifecycleService handles cannot reconnect after restart. Parent stop need not cancel detached work; explicit stop/reset does (tools/async_delegation.py:48,115,244,586; agent/subagent_lifecycle.py:269,389).

Optional /moa gathers parallel reference advice per iteration; ordinary agent retains tools/termination, advisor context trims per model. Separate from delegate execution (agent/moa_loop.py:1,414,500,619).

### H. Tools and integrations — V / O

AST discovery imports sorted tools/*.py and tools/*/tool.py with top-level registration; MCP separate. Failures log/skip; cache path/mtime/size. ToolEntry holds schema/handler/availability/async/dynamic schemas/timeouts/result limits; profile/session overlays. Reachability checks cache **30s**; enabled/disabled bundles/grants determine exposure. Inline agent tools and registry/connectors/catalog dispatch through coercion/aliases/middleware/guards/hooks/transforms; bridges unwrap before policy, exceptions become structured errors (tools/registry.py:94,121,178,216,249,735,972; model_tools.py:227,897).

| Lane | Execution/state | Defaults / limits / boundary |
|---|---|---|
| Terminal | Reused local or Docker/Modal/Daytona/Vercel/SSH/plugin; owned task/background registry | **180s**, foreground **600s**, longer promotes background, timeout **124**; local host authority. |
| Files | Same backend via ShellFileOperations; atomic optimization | **100,000-char** reads/offsets; path/lock/stale-baseline guards; line offset cannot recover huge single-line remainder. |
| Python | Persistent kernel/remote; RPC whitelist/budget | **300s/50 calls**; stdout/stderr **50KB/10KB**, spill **5MB**; not a jail. |
| Browser | browser_exec → harness; CDP/forced-local/cloud/local route | **300s**, **5–1800s** clamp; explicit local fallback; fresh signed-out profile if consented unavailable. |
| Web/API | Plugin backend, connector grants | Search **5**, **1–100** clamp; unavailable explicit provider fails. |

Sources: tools/terminal_tool.py:78,517,854,1244,1472; tools/terminal_tool_backends.py:133; tools/file_tools.py:51,312,605,863; tools/code_execution_tool.py:42,707; tools/browser_use_cli.py:774; tools/browser_session.py:279; tools/web_tools.py:302. Output result/turn **100,000/200,000 chars**, preview **1500**, MCP **50,000**; scaled **15%/30%**, floors **8000/16,000**; read_file avoids recursive spill, retains own cap (tools/budget_config.py:10).

MCP: optional SDK/lazy schemas; stdio/HTTP/SSE, OAuth/certs/headers. Connect **60s**, reconnect **5**, tool **300s**, breaker **3/60s**. Trust full default; invalid→untrusted; untrusted writes approved via read-only annotations. Auth may retry once; ambiguous write expiry/stdio death returns uncertainty, not replay (tools/mcp_tool.py:143,237,516; tools/mcp_tool_registration.py:40; tools/mcp_tool_handlers.py:124,316,349,396).

Native manifest/register(ctx) plugins track ownership teardown; portable packages supply MCP/skills. Bundled/user/opt-in project/entry-point sources; category activation, disabled wins; backends auto-load/platforms defer/others enable. Builtin override requires allow_tool_override. Loader **10s**, max **600s**, **8** abandoned daemon loaders stops attempts; native code not sandboxed (hermes_cli/plugins_discovery.py:190,269; hermes_cli/plugins_context.py:270; hermes_cli/plugins_loader.py:40).

### I. Gateway and automation — V; scheduler provider experimental O

Startup gates restore ingress, warms, connects primary/secondary adapters, wires delivery, reopens work. Optional failures can park/degrade; replay isolates messages. Routing freezes transport versus runtime profile/home: shared bot can execute another profile, receiving adapter replies; invalid route refuses. Keys profile/platform/chat/thread/optional user; group-per-user true/thread-per-user false. Per-profile SessionAuthority precedes legacy busy fallback (gateway/run.py:3322; gateway/run_startup.py:1702; gateway/session_identity.py:43,141; gateway/session.py:699; gateway/run_runtime.py:38; gateway/run_inbound.py:1305).

Gateway threads unbounded, idle **60s**; max_concurrent_sessions None/default or admission cap. Empty Context/explicit profile binding required on reuse; delivery reconciliation separate from model success (gateway/turn_executor.py:13,25; gateway/config.py:617).

Cron supports interval/duration/cron/ISO one-shot; zone/provider/model/skills/context/workdir/script/delivery pins. Job zone wins; grace half cadence **120s–2h**, ticker tolerance even zero grace, on-time **300s**, tick **60s**. Experimental provider decides timing only (cron/jobs.py:787,947,1191,1809; cron/jobs_time.py:114; cron/scheduler_provider.py:1,136).

Recurring next_run advances before dispatch. Execution states claimed/running/completed/failed/unknown/cancelled/missed; canonical admission/frozen policy for agents. Headless missing gateway leaves pending; normal run can connect/start. Lost acknowledgment journals/pauses, not resubmits. Agent inactivity **600s**, script **3600s**; **skip_memory=false/skip_background_review=true**. Python bounded default pool differs from gateway (cron/scheduler.py:1186,1200,2481,2561; cron/scheduler_tick.py:92; cron/scheduler_authority.py:55).

Process completion rebinds/admit follow-up: refusal **8** retries/backoff <=**60s**. Heartbeat admissions and authorized platform hooks differ. Separate Kanban DB claims/tasks spawn assigned-profile CLI workers, scrub routing/secrets, require complete/block/review protocol: **2** failures, systemic exit **78**, dispatch **60s** (gateway/run_notifications.py:1828,1922; gateway/run_goals.py:138; hermes_cli/kanban_db_dispatch.py:2355,2754).

### J. Security and configuration — V / O

Profiles are logical state/secret islands, not OS principals; unbound multiplex/foreign-home secret fallback refuses. Native Python/local shell remains trusted authority (agent/secret_scope.py:230,407).

Command policy assesses backend: contained Docker/no host mounts or selected cloud skips most checks, keeps deny. Host catastrophic/self-delete/sudo-password/deny floors precede YOLO/off, then patterns/grants/human/smart. Default smart, approval **300s**, unattended/cron/query deny, denial breaker **3**. Flagged-only auxiliary assessment temperature **0**, **16 tokens**; ambiguity/error escalates; operation-only smart grant, attended denial override once (tools/approval.py:1161,1189,1276; tools/approval_smart.py:103,123; hermes_cli/config_defaults.py:1660).

File guards cover secrets/system paths, refuse active config.yaml, require specific approval for project instructions; profile trees use store guards. Not universal shell interception. Context regex scanning blocks flagged project content; user-authored SOUL warns/loads. Remote private URLs default denied, metadata always blocked; browser routes differ. Redirects strip credentials; logs redact/route, not tamper-proof (agent/file_safety.py:178,214,375; tools/file_tools_write_guards.py:147,214,235,263; agent/prompt_builder.py:84; tools/url_safety.py:316; hermes_cli/urllib_security.py:38; hermes_logging.py:357,646).

### K. Model providers — V / O; adaptive routing D/U

ProviderProfile declares aliases/modes/endpoints/auth/catalog/headers/quirks, not clients/retry. Discovery entry points→bundled→legacy plus home overlays; transports convert OpenAI-shaped history, with app-server/native/external alternatives (providers/base.py:142; providers/__init__.py:140,218,889; agent/transports/base.py:12).

Runtime resolution handles disabled/special/custom/local, explicit keys/auth, pool/OAuth/external/Anthropic/Bedrock/registered-key and fallback. Explicit OAuth fails, auto can continue; API mode depends on model/runtime. Optional openai_runtime=codex_app_server uses its own login (hermes_cli/runtime_provider.py:539,1044,1111).

Anthropic signed thinking/tool order; transport fallback output **16,384**. Chat strips persistence metadata/maps reasoning/output/aliases. Cache hashes prompt/tools/lineage; explicit key <=**64 chars**. Deferred client retirement avoids cross-thread socket corruption (agent/transports/anthropic.py:27,65; agent/transports/chat_completions.py:37,77; agent/client_lifecycle.py:209).

Pool auth.json persists credential/status/priority; soft leases are in-memory. Counts persist on later writes (round-robin immediately); **fill-first**, random/least-used/round-robin alternatives, provider/endpoint fences and refresh locks. Aux normally same provider/model; explicit task overrides, title-only prefer_fast_model (agent/credential_pool.py:559,981,2152,2154,2335,2375; agent/auxiliary_client.py:838,4498).

Usage separates input/output/cache/reasoning: Anthropic input excludes cache buckets, chat/Responses includes them, requiring normalization. Decimal cost prefers provider-reported cost, then included-subscription or rate estimates; missing usage/rates stay **unknown**. Session/DB/log accounting; provider-executed tools commit without redispatch (agent/usage_pricing.py:589,640,700; agent/turn_usage.py:73; agent/provider_projection.py:25). smart_model_routing setup/config exists, runtime consumer not located (hermes_cli/setup_quick.py:251; hermes_cli/config.py:955,2967). Inspected tests are mapped in the source index.

## 6. Key execution workflows

### 6.1 User message → agent → tools → response

~~~mermaid
sequenceDiagram
  participant U as User / adapter
  participant G as Gateway / authority
  participant A as TurnRunner / AIAgent
  participant D as SessionDB
  participant M as Model
  participant T as Tools
  U->>G: Event / identity / authorization
  G->>A: Durable owned admission off event loop
  A->>D: Inbound row and prompt
  A->>M: Frozen prefix + history
  M-->>A: Tool calls
  A->>D: Persist calls before dispatch
  A->>T: Guarded inline / registry execution
  T-->>A: Results or paired errors
  A->>D: Persist results
  A->>M: Continue
  M-->>A: Candidate answer
  A->>A: Stop gates and transform
  A->>D: Authoritative final and usage
  A-->>G: Result
  G-->>U: Stream/final reconciliation
~~~

Event → BasePlatformAdapter.handle_message → _hm_admit_event/authority → TurnRunner.run_sync → run_conversation (gateway/platforms/base.py:3915; gateway/run_inbound.py:1305; gateway/run_turn_runner.py:1001). Busy handling is alternate. State: admission/transcript/usage/delivery. Bad route/auth or hook veto refuses; persistence loss halts; model recovery §5A; delivery cannot undo effects. Config: session/admission/grants/approval/provider/budgets.

### 6.2 New information → persistent memory → future retrieval

~~~mermaid
sequenceDiagram
  participant A as Foreground / review
  participant T as memory tool
  participant S as MemoryStore / Markdown
  participant N as New agent / compaction
  A->>T: add / patch / remove
  T->>T: Validate / approval policy
  alt Staged
    T-->>A: Pending proposal, no write
  else Admitted
    T->>S: Locked reread / batch validation
    S->>S: Match / final budget / atomic replace
    S-->>A: Current entries and usage
  end
  N->>S: load_from_disk
  N->>N: Freeze prompt snapshot
~~~

Model chooses save → _memory/memory_tool → _mutate/apply_batch; external notification after success (tools/memory_tool.py:250; tools/memory_tool_store.py:350,562). State: atomic entries, not current frozen prefix; lifecycle/compaction reloads. Match/cap/scan/stale proposal refuses; memory.write_approval/review staging apply (§7.1).

### 6.3 Successful task → experience review → reusable skill

~~~mermaid
sequenceDiagram
  participant P as Parent / finalizer
  participant R as Review fork
  participant S as Skill tools / files
  P->>P: Eligible nonempty response / counters
  P->>R: Snapshot and schedule
  R->>R: Cache parity / dispatch and input limits
  R->>S: Search / fresh read
  alt Reusable lesson selected
    R->>S: Patch umbrella/support file or create
    S-->>R: Validation / mutation result
  else No useful lesson
    R-->>P: Nothing to save
  end
~~~

Counter/final-response eligibility, not proven success → _mirror_turn_to_memory/_spawn_background_review → build_cache_parity_fork/_run_review_fork → skill_manage (agent/turn_finalizer.py:591,862; agent/background_review.py:979,1189). State: artifacts/provenance/parent cost, no user-history writes. Config/safe/finite/cron skips, foreground cancels; earlier writes survive (§7.2).

### 6.4 Existing skill → modification → validation or rollback

~~~mermaid
sequenceDiagram
  participant A as Agent / reviewer
  participant V as skill_view
  participant B as skill_manage batch
  participant F as Files / recovery
  A->>V: Read exact target
  V-->>A: Content / freshness mark
  A->>B: operations array
  B->>B: Shape / operation guards / lock / optional staging
  B->>F: Ordered locks and touched-tree snapshots
  B->>F: Patch / validate / atomic write
  alt Operation failure
    B->>F: Restore snapshots
    B-->>A: Error / retained incomplete-recovery evidence
  else Success
    B->>F: Best-effort ledger and usage
    B-->>A: Result / advisory lint
  end
~~~

Correction/refine/review → skill_view → skill_manage/_skill_manage_batch → _patch_skill/_guarded_write (tools/skill_manager_tool.py:407,523,783; tools/skill_manager_batch.py:219). Operation-specific validation/staging (§7.3); snapshot rollback covers files, not all metadata. Later undo requires safety capture/valid paths/blobs (§7.4).

### 6.5 Task → specialist agent → result returned

~~~mermaid
sequenceDiagram
  participant P as Parent
  participant D as delegate_task
  participant C as Child
  participant S as DB / result spill
  P->>D: Goal / explicit context
  D->>D: Depth / grants / route / policy
  D->>S: Dispatch and lineage
  D->>C: Fresh prompt / session / budget
  C->>C: run_conversation
  C-->>D: Result / failure
  D->>S: Persist each finished child
  D->>D: Schema correction / trim / costs / cleanup
  alt Joined lane
    D-->>P: Structured result
  else Detached lane
    D->>S: Claim completion delivery
    S-->>P: Admitted follow-up
  end
~~~

Task → delegate_task/_build_children → _run_single_child/await_child → run_conversation → finalize (tools/delegate_tool.py:203,377,566,917). State: fresh child history, shared files, durable results. Per-task route failure; timed-out cleanup waits for unwind; policy limits/restart semantics §5G.

### 6.6 Scheduled task → execution → persistence

~~~mermaid
sequenceDiagram
  participant T as Tick
  participant J as Jobs / execution DB / output
  participant C as Admission journal
  participant G as Gateway authority
  participant A as Owner scheduler / agent
  participant D as Delivery
  T->>J: Reconcile / due jobs / advance next_run / claim
  T->>C: Stable request ID
  T->>G: Submit stored job ID / request identity
  G->>G: Freeze job policy under profile authority
  G->>A: Execute admitted request
  A->>A: Prompt / skills / core
  A-->>G: Result or classified failure
  G-->>T: Receipt / result
  T->>J: Output and execution bookkeeping
  T->>D: Compose / deliver
  T->>J: Separate delivery outcome
  T->>C: Remove completed journal
  Note over T,G: Lost acknowledgement: observe receipt; pause unknown outcome
~~~

cronjob_manage/create_job → tick/_tick_admitted → run_canonical_job → session_cron.operation/execute → _construct_cron_agent/run_conversation → finish_completed_run (cron/scheduler_tick.py:33; cron/scheduler_authority.py:55; gateway/session_cron.py:104,276; cron/scheduler_bookkeeping.py:52). State: jobs/ledger/profile DB, journal/output; usage via core/audit, authority usage receipt empty. Script bypasses model. Headless owner absent→pending; normal run may start owner; lost acknowledgment pauses/reconciles, delivery separate. Schedule/zone/route/skills/timeouts §5I.

## 7. Learning, memory and skills mechanisms in greater depth

### 7.1 Curated versus historical/provider memory

MemoryStore exact-deduplicates MEMORY.md/USER.md, both enabled; budgets **2200/1375 chars** including separators. Oversized legacy loads warn. Strict scan substitutes BLOCKED in prompt snapshot, preserving removable raw entries (agent/agent_init.py:1155; tools/memory_tool_store.py:188,223,649).

Model actions **add/patch/remove**, single/batch; replace rejected despite legacy internal method. Match exact→substring→typography/whitespace-folded; ambiguous distinct entries, repeated patch spans/no-op refuse. _mutate uses owner-only/symlink-protected lock, rereads disk and atomically replaces. Batch validates final budget/all operations before writing, cannot empty nonempty store; single remove can (tools/memory_tool.py:250,261,280; tools/memory_tool_store.py:98,147,259,350,562,615).

Unreadable disk refuses mutation; drift/oversized destructive rewrite may require backup/refusal. Failed consolidation <=**3/turn**, fourth save-skipped. No semantic conflict resolver/decay/TTL. memory.write_approval false, yet **unattended review patch/remove stages**; failed staging denies, proposals pin full observed entry/refuse intervening change. Success may notify external backend (tools/memory_tool.py:64,179,353; agent/inline_tool_executors.py:155).

MemoryProvider hooks: initialize/static prompt/prefetch/queue/sync/tools/shutdown plus turn/session/delegation/write/precompression. One configured external provider joins built-in; manager fanout does not activate all installed. Empty/builtin/default/none disables external. RetainDB/ByteRover implemented in-tree; catalog listing alone proves nothing (agent/memory_provider.py:42,103; agent/agent_init.py:1204).

Prefetch once before loop, trivial-query skip, secret/scaffold stripping, current-turn API injection. Timeout **8s** leaves daemon call alive; suppress after **4** stuck, <=**16** inflight, probe/**60s**. Completed noninterrupted turns FIFO-sync then prefetch; true end precedes switch, bounded shutdown may abandon. RetainDB HTTP/local queue; ByteRover CLI/substantive curation; backend retention/conflicts external (agent/memory_manager.py:42,52,145,360,459,659).

### 7.2 Experience capture and review

Memory trigger **10 user turns**, skill trigger **10 prepared iterations** (before inference, not successful tool count). Eligible nonempty/noninterrupted final can review; no independent success/reward/weight-learning mechanism established.

Review default on even config-read failure; safe/finite-gateway/explicit/cron skips. Same-route fork copies full history/exact prompt/tool advertisement/cache identity, with separate restricted dispatch (skills/read, memory only if triggered). Different explicit aux route digests **24 recent messages** plus older summary; route failure falls back parent (agent/background_review.py:173,204,228,332,979,1116).

Budget **16 iterations**, aggregate input **min(600,000,75% context)**, fallback **120,000**; <=0 disables cap. Shared session ID/live MemoryStore, no transcript DB/persistence/external manager; parent cost rollup. Unattended dangerous deny, foreground cancellation handshake **2s**; prior writes survive. Loaded skill→umbrella→support→new/nothing priority is prompt policy. /learn is foreground build_learn_prompt; /refine focused review. Plugin on_session_end is per turn, provider hook true boundary (agent/learn_prompt.py:136; hermes_cli/cli_commands_mixin.py:1313; agent/turn_finalizer.py:882).

### 7.3 Skill discovery, loading and mutation

Format: YAML name/description + nonempty SKILL.md; support files. Precedence **trusted project→profile→create_dir→external_dirs**; project roots require trusted_project_dirs/scan/quarantine, ancestor <=**64**. Same-tier divergent duplicates ambiguous, identical may select shallowest (agent/skill_utils.py:402,413,511,635).

Index **60-char** descriptions; model/slash/preload/auto_load selection, no learned ranker. skill_view checks disabled/platform/readiness, resolves content, read-marks and increments use/view; unchanged repeat may stub, review fresh-read required. Dependencies may **pm.ensure on load**; secrets may prompt setup. Templates on/inline shell off, timeout **10s**, community exclusion; auto_load once/lifecycle (tools/skills_tool.py:613,671,758; agent/skill_preprocessing.py:142; agent/system_prompt.py:323). Bundled hash sync preserves edits/suppression; optional install/hub adds adapters/scan/locks/audit (tools/skills_sync.py:113,356; tools/skills_hub_install.py:143).

Writer legacy flat or <=**20 operations**, delete sole op. Name <=**64** safe chars; SKILL.md <=**100,000 chars**, support <=**1MiB**; legacy parsed description <=**1024**, authored description <=**60**. Fuzzy patch unique unless replace_all. Lock blocks mutation; pin protects the **delete action**, not patch. Unlocked user/bundled/hub files can be patched after fresh read; ownership mainly constrains autonomous delete (tools/skill_manager_tool.py:108,146,173,407,523; tools/skill_manager_guards.py:143,165,224).

**Operation-specific validation gap (I):** root SKILL.md is a valid file_path; _write_file omits frontmatter validation, explicit-root patch skips it, and remove_file can unlink root without delete-action pin/ownership/absorbed_into guards. Content-lock preflight still applies. Thus format/delete guarantees cannot be generalized across all operations; bypass tests were not established (tools/skill_manager_tool.py:376,560,619,644,660).

Batch ordered locks/snapshots restore touched files, not all metadata/events. Authoring lint advisory; guard_agent_created false; optional scan error permits. write_approval false/enabled stages whole call; some config errors fail open. Ledger/usage/cache updates best effort; prompt freshness waits (§8).

### 7.4 Curator lifecycle, consolidation and recovery

Defaults enabled/**168h**/**2h supplied idle**/**14d stale**/**30d archive**/**consolidate=false**. First check seeds cadence/defer. CLI/gateway idle=infinity, web measures (agent/curator.py:31,122,160,1220; gateway/run.py:4331; hermes_cli/cli_tui_runtime_mixin.py:412).

Usage records ownership/use/view/patch/reuse/state/pin/lock; latest activity wins. Review-created “agent” auto-managed; /learn “learn” not auto-enrolled; adopt enrolls local. Hub/external/protected builtins excluded; prune_builtins allows deterministic archive only (tools/skill_usage.py:123,290,348,522).

Transitions skip pin/lock/cron refs (paused included); failed cron read→empty protection set. Use reactivates stale; archive collision-handles, flat restore refuses collision. Optional skills-only LLM consolidation **9999 iterations**, no terminal/recursion. Autonomous **delete action** needs ownership/existing absorbed_into umbrella, archives; bare pruning refused, cron rewrite best effort. Cadence before LLM; reports/post_curator_run. Dry-run skips transitions/cadence, writes reports, model report-only is prompt policy (agent/curator.py:184,209,677,931,1103; tools/skill_manager_guards.py:266).

| Recovery | Capture/default | Undo boundary |
|---|---|---|
| Tree tar | Best effort before consolidation; keep **2**; excludes archive/ledger/hub/locks | Safety capture, staging/path checks; restore stage on extract failure/preserve incomplete evidence. |
| Ledger/blobs | Enabled, **5MB** maintenance, **1h** orphan GC grace | Missing blob/unsafe path/failed safety capture refuses; writes can succeed without ledger. |

Evidence: agent/curator_backup.py:30,113,406; tools/skill_ledger.py:84,113,569. Tests include batch rollback/safety-capture/extraction failures (companion index). Learning graph: lexical top **4** skill links/name weight **6**, not retrieval/causal knowledge (agent/learning_graph.py:192).

## 8. Configuration and lifecycle rules

| Concern | Precedence / lifecycle | Source |
|---|---|---|
| Home | Context override → HERMES_HOME → platform default | hermes_constants.py:194 |
| Full config | Defaults + deep user merge → canonicalization/env expansion → managed last | hermes_cli/config.py:1373,1979,2170 |
| Effective/CLI | Effective omits defaults; CLI has separate default loader; managed last in both | hermes_cli/config_effective.py:52; hermes_cli/cli_config_load.py:184 |
| Backend | HERMES_CONFIG_BACKEND, default file; unavailable raises SystemExit subtype | hermes_cli/config_backend.py:63,145,329 |
| Dotenv/secrets | Profile replaces stale shell; project fills missing when profile file exists, otherwise may override; external then managed; terminal bridges reapplied. Routed lookup cannot borrow foreign-profile values | hermes_cli/env_loader.py:652,716,767; agent/secret_scope.py:230,407 |
| Provider | Explicit → model.provider → scoped/env inference provider → auto | hermes_cli/runtime_provider.py:539 |
| Reasoning/output | Runtime/model capability, per-model config, explicit request override | agent/transports/chat_completions.py:37 |
| Approvals | Launch YOLO + scoped/persisted session toggle; once/session/permanent grants, mandatory floors | tools/approval_yolo.py:32 |
| Freshness | Store writes immediate; prompt waits for admitted rebuild/new lifecycle | agent/system_prompt.py:829 |
| Cron pool | HERMES_CRON_MAX_PARALLEL → cron.max_parallel_jobs → Python default | cron/scheduler.py:4194 |

Broken YAML may recover last-good; fail_closed refuses. Noninteractive startup normally requires parseable config; low-level fallback differs. Managed writes strip/refuse pinned keys. External-secret transient failures can retain prior values (hermes_cli/config_effective.py:89; hermes_cli/main.py:2857). Finalization, true session end, switch, compaction, refresh, profile reload and restart differ; ACP disconnect need not end daemon-owned sessions (acp_adapter/entry.py:224).

## 9. Architectural tradeoffs and limitations

| Assessment | Operational implication / evidence |
|---|---|
| Stable-prefix discipline | Lower replay cost, deferred memory/skill freshness and nuanced rebuilds (agent/conversation_loop.py:802). |
| Durable ownership, incomplete atomicity | Admission/call persistence limits replay; external effects remain outside transactions (agent/turn_tool_round.py:139; cron/scheduler_authority.py:149). |
| Modular but coupled phases | Facades/siblings improve navigation; shared attributes and late imports require call tracing (agent/conversation_loop.py:1751). |
| Shared process/workspace | Profile scope must follow threads; children generally share files, logical isolation is not containment (agent/secret_scope.py:230; tools/terminal_tool.py:373). |
| Conservative recovery | Unknown cron work pauses; finished child outputs recover, crashed stacks do not (tools/async_delegation.py:295). |
| Uneven concurrency | Gateway unbounded unless admission-capped; cron bounded, children10/tool batches8 (§5). |
| Model-directed learning | Loading ≠ success; semantic conflicts/dedup/utility unproven; writer invariants are operation-specific (§7). |
| Recovery/trust gaps | Writes can outlive failed capture; native Python/local shell broad, MCP annotations trusted (§5H/J, §7.4). |

Live availability/reliability/containment were not tested. Unknown cost and lexical graph are not learning-performance measurements (agent/usage_pricing.py:700; agent/learning_graph.py:192).

## 10. Consolidated technical findings

| Mechanism | Status | Finding |
|---|---|---|
| Core/replay | V | Phased loop, durable calls, bounded errors, controlled prefix rebuilds. |
| Memory/history | V | Bounded locked Markdown separate from transcripts; compaction retains originals. |
| External memory | O/U | Prefetch/FIFO implemented; backend guarantees external. |
| Background learning | V/U | Counter-gated artifact review; semantic success unproven. |
| Skill lifecycle/recovery | V/I | Batch undo/maintenance exist; root-file mutation bypasses lack established tests (§7.3). |
| LLM consolidation | O | Disabled default, model consolidation/patch, best-effort capture. |
| Delegation | V | Focused histories/shared files; finished results durable, stacks unrecoverable. |
| MCP/plugins | O | Optional SDK/services/trust; uncertainty-aware write handling. |
| Gateway/cron | V | Profile admission, frozen policy, receipt reconciliation, separate delivery. |
| Scheduler provider | O | Experimental timing contract. |
| Security/config | V/I | Secret/approval tests inspected; other controls static. |
| Provider/accounting | V | Route/auth/wire policies; unknown usage/rates remain unknown. |
| smart_model_routing | D/U | Setup key located, runtime consumer not found. |

Source corrections: AIAgent **sys.maxsize**, legacy direct entry **10**, not stale500; delegation **10/depth1/250**, timeout absent or inactivity; completed detached results durable; cron keeps memory/skips review; compaction **in-place/lean**, one summary request/attempt; review “agent” versus /learn “learn” ownership differs; pin ≠ lock and root-file operations have bypasses; curator **2h idle** depends on supplied value (CLI/gateway infinity).

**Static validation only; no passing tests or live-agent behavior claimed.**


## 12. Evaluation & Quality Assurance

**Append-only follow-up scope.** All findings below use commit **4ad61cee29316e21e03c4585bb4ececfa52385fe**. The user authorized preserving the existing workspace documents and appending §§12–19/C18–C25. The available report ends at §10 and its index has no C01–C17; no missing §11 or earlier contracts have been reconstructed. V/I/O/D/U retain their original meanings. Every test below was **inspected, never executed**.

### 12.1 Ownership and execution lanes

| Lane | Owner, inputs → outputs and consumer | Status / boundary |
|---|---|---|
| Regression testing | scripts/run_tests.sh selects the test environment, scrubs environment and delegates per-file pytest subprocesses; CI selects slices/platform lanes. | V/I: engineering checks, not production task grading. |
| Compaction evaluation | runner.py loads transcript, generates/caches questions, applies policies, answers from compacted context and judges against gold; JSON feeds report.py/scorecard.md. | I/O: explicitly run model-assisted experiment, not a post-task runtime service. |
| Live canaries | Dedicated workflow injects provider keys into live tests, records usage and JUnit output. | I/O: tag/manual triggers, secret-gated cases; workflow existence does not establish mandatory branch-protection gating. |
| Runtime verification | Edit/check-evidence ledger feeds bounded finish-time nudges and plugin pre_verify. | V/O: policy asking the model to verify; does not execute a test suite or prove correctness. |

Evidence: [scripts/run_tests.sh:1](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/scripts/run_tests.sh#L1), [scripts/run_tests_parallel.py:556](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/scripts/run_tests_parallel.py#L556), [.github/workflows/tests.yml:108](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/.github/workflows/tests.yml#L108), [.github/workflows/live-providers.yml:14](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/.github/workflows/live-providers.yml#L14), [evals/compaction/runner.py:330](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/evals/compaction/runner.py#L330), [evals/compaction/report.py:35](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/evals/compaction/report.py#L35), [agent/verification_stop.py:1](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/verification_stop.py#L1).

Runner defaults: CPU-count workers or HERMES_TEST_WORKERS, **300s/file**, **one fresh-process retry** for eligible failures; duration history can increase timeout. Exit 5 can mean entirely deselected rather than failed; fail-on-skip is a separate stronger requirement. Thus successful runner exit does not mean every discovered test executed. The live-canary workflow intentionally invokes pytest directly to preserve its explicitly supplied keys. [scripts/run_tests_parallel.py:146](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/scripts/run_tests_parallel.py#L146), [scripts/run_tests_parallel.py:502](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/scripts/run_tests_parallel.py#L502), [scripts/run_tests_parallel.py:556](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/scripts/run_tests_parallel.py#L556), [scripts/run_tests_parallel.py:1105](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/scripts/run_tests_parallel.py#L1105), [scripts/run_tests_parallel.py:1250](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/scripts/run_tests_parallel.py#L1250), [.github/workflows/live-providers.yml:90](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/.github/workflows/live-providers.yml#L90).

### 12.2 Compaction experiment algorithm, state and limitations

Defaults: **500,000 transcript tokens**, **15 questions**, **current+recovery** policy. The question cache hashes transcript path and cap, not complete transcript content, so editing a file need not invalidate an existing cache. Policies compress copied messages. The recovery arm generates a search query and simulates archived retrieval with in-memory FTS5/BM25, **4 hits / 2500-character excerpts**, with term-frequency fallback. It does not execute the whole production session-search/gateway path. The LLM judge scores **0/1/2**; recall is sum(scores)/(2 × question count). Judge parse failure becomes 0, conflating infrastructure failure with incorrect recall. Unknown policies skip; failed compaction can emit unknown after_tokens/recall plus an error. [evals/compaction/runner.py:91](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/evals/compaction/runner.py#L91), [evals/compaction/runner.py:296](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/evals/compaction/runner.py#L296), [evals/compaction/runner.py:388](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/evals/compaction/runner.py#L388), [evals/compaction/runner.py:411](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/evals/compaction/runner.py#L411).

Per-policy JSON stores answers/gold/scores, before/after tokens, compaction timing/error and metered usage. report.py renders a Markdown scorecard. _AuxMeter wraps the compressor call binding; OpenRouter list prices provide a comparison yardstick, not necessarily actual-route billing. Generated questions and model grades are evaluation methods, not independent proof of factual correctness. The Jev alternative is an evaluation arm, not evidence that its engine is installed/selected in production. [evals/compaction/runner.py:232](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/evals/compaction/runner.py#L232), [evals/compaction/runner.py:252](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/evals/compaction/runner.py#L252), [evals/compaction/runner.py:394](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/evals/compaction/runner.py#L394), [evals/compaction/report.py:35](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/evals/compaction/report.py#L35), [tests/evals/test_compaction_jev_arm.py:19](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/tests/evals/test_compaction_jev_arm.py#L19).

### 12.3 Verification and relationship to learning

verify_on_stop defaults **off**; environment override precedes config, with legacy surface-sensitive auto mode. Enabled evidence lives in profile verification_evidence.db; disabled policy must not create the ledger. Bounds include **30 days**, **100 events/session-root**, **10,000 unreferenced events**. Pure documentation edits are exempt. pre_verify has a separate default **3 nudges**; consumer errors may log/fall through. This is attributable-check evidence and continuation policy, not a semantic task-success oracle. [agent/verification_stop.py:27](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/verification_stop.py#L27), [agent/verification_stop.py:62](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/verification_stop.py#L62), [agent/verification_evidence.py:21](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/verification_evidence.py#L21), [agent/verification_evidence.py:123](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/verification_evidence.py#L123), [agent/verify_hooks.py:16](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/verify_hooks.py#L16), [agent/turn_stop_gates.py:36](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/turn_stop_gates.py#L36).

**Evaluation→learning: no automatic connection verified.** The inspected runner writes experiment artifacts; review/curator entry points consume conversation, counters, skills and usage rather than benchmark scorecards. No scorecard consumer promoting skills, changing rewards, tuning thresholds or learning model routes was located in these inspected paths. Explicitly reading a report as ordinary content is possible but is not that feedback architecture. [evals/compaction/runner.py:406](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/evals/compaction/runner.py#L406), [agent/turn_finalizer.py:591](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/turn_finalizer.py#L591), [agent/background_review.py:979](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/background_review.py#L979), [agent/curator.py:931](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/curator.py#L931).

Tests inspected: tests/evals/test_compaction_jev_arm.py:19,35 checks paired tool calls/results and preserved text using a fake asker; tests/agent/test_verification_stop.py:138 checks the continuation bound. No benchmark or suite was run.

## 13. Observer Hooks, Middleware & Event Lifecycle

### 13.1 Component ownership, ordering and interfaces

hermes_cli.lifecycle invokes first-party observation before compatibility plugin hooks; safe-worker policy suppresses ordinary lifecycle work. PluginManager owns profile callback collections and registrations; call sites decide which return values affect execution. Native narrow signatures receive only accepted kwargs; **kwargs receives additive fields. Observer envelopes use **hermes.observer.v1**, middleware adds **hermes.middleware.v1**. gateway_platform_event has event-local rather than bus-wide versioning. Hook names are not themselves execution/permission guarantees. [hermes_cli/lifecycle.py:11](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/lifecycle.py#L11), [hermes_cli/lifecycle.py:30](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/lifecycle.py#L30), [hermes_cli/middleware.py:16](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/middleware.py#L16), [hermes_cli/plugins_dispatch.py:275](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/plugins_dispatch.py#L275).

| Seam | Inputs → outputs / consuming component | State/lifecycle boundary |
|---|---|---|
| pre_llm_call | Current turn → context/block, consumed by turn preparation | Can veto before inference; does not replace all persisted history. |
| pre_tool_call | Effective tool/args/context → recognized directive | Policy callback errors/timeouts fail closed. |
| tool_request / llm_request | Dict → replacement args/request, trace entries | Sequential middleware rewriting, not a passive observer. |
| tool_execution / llm_execution | Effective payload + next_call → result | Wrapping/short-circuiting actual execution. |
| Transform hooks | Tool/terminal/final output → transformed content | Stream holding and final persistence depend on consumer. |
| on_session_end / on_session_finalize | Turn completion / true conversation boundary | Plugin end is per-turn; finalize_session closes true core conversation. Provider hooks have separate semantics. |

Evidence: [hermes_cli/plugins_dispatch.py:275](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/plugins_dispatch.py#L275), [hermes_cli/middleware.py:88](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/middleware.py#L88), [hermes_cli/middleware.py:102](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/middleware.py#L102), [hermes_cli/middleware.py:162](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/middleware.py#L162), [agent/turn_context.py:903](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/turn_context.py#L903), [agent/turn_finalizer.py:882](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/turn_finalizer.py#L882), [hermes_cli/lifecycle.py:86](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/lifecycle.py#L86).

Relay tool intercepts can precede plugin request middleware; effective args then reach guards/approvals. LLM middleware receives assembled provider kwargs. _safe_copy normally deep-copies but falls back to a shallow dict when copying fails, so arbitrary nested objects have no unconditional isolation guarantee. Execution middleware permits **one next_call/frame**; downstream errors propagate, wrapper failure after downstream success returns the already-completed result, and failure before calling downstream can skip the wrapper. That chain has no general per-callback wall-clock timeout: callback/outer execution must provide bounds. [hermes_cli/middleware.py:50](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/middleware.py#L50), [hermes_cli/middleware.py:102](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/middleware.py#L102), [hermes_cli/middleware.py:162](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/middleware.py#L162), [hermes_cli/middleware.py:202](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/middleware.py#L202), [agent/turn_api_request.py:110](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/turn_api_request.py#L110).

### 13.2 Callback failure, concurrency and events

Selected hot hooks and pre_tool_call default to **30s** timeout, cap **600s**; zero can disable. Timed-out Python workers are abandoned, not killed. Dispatch suppresses the callback for **60s**, tracks up to **3 abandoned workers**, and fail-closes pre_tool_call; other observers generally skip failure. subagent_stop remains caller-thread serialized, and hooks outside the bounded set can run without this cap. Async await/wait_for cancellation differs from abandoning synchronous threads. [hermes_cli/plugins_dispatch.py:42](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/plugins_dispatch.py#L42), [hermes_cli/plugins_dispatch.py:183](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/plugins_dispatch.py#L183), [hermes_cli/plugins_dispatch.py:295](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/plugins_dispatch.py#L295), [hermes_cli/plugins_dispatch.py:351](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/plugins_dispatch.py#L351), [hermes_cli/plugins_dispatch.py:621](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/plugins_dispatch.py#L621).

The plugin event bus is separate from hooks and gateway durable admissions: owner subscriptions, one daemon worker, captured ContextVars, generation-aware teardown, **64 pending events**, **depth 8** recursion. Full queue/budget drops/logs the emit; accepted subscriber count is not durable delivery. Process death loses queued events. Unload/replacement must avoid stale subscribers affecting the next generation. No persistence, transaction or replay guarantee is provided by this bus. [hermes_cli/plugins_dispatch.py:130](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/plugins_dispatch.py#L130), [hermes_cli/plugins_dispatch.py:438](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/plugins_dispatch.py#L438), [hermes_cli/plugins_dispatch.py:470](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/plugins_dispatch.py#L470), [hermes_cli/plugins_dispatch.py:529](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/plugins_dispatch.py#L529).

### 13.3 Shell hooks and outbound webhooks

Shell hooks use event/command allowlisting or explicit acceptance; hooks_auto_accept defaults false. Records contain script mtime, but runtime consent checks the event/command pair, not a cryptographic content identity. JSON stdin feeds scripts; timeout **60s**, max **300s**, process-tree teardown on timeout. Blocking pre_tool_call interprets exit **2**; spawn/timeout/invalid output normally fail open unless fail_closed is configured on a blocking event. This is different from native pre_tool_call callback-error policy. [agent/shell_hooks.py:37](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/shell_hooks.py#L37), [agent/shell_hooks.py:319](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/shell_hooks.py#L319), [agent/shell_hooks.py:484](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/shell_hooks.py#L484), [agent/shell_hooks.py:560](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/shell_hooks.py#L560), [agent/shell_hooks.py:787](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/shell_hooks.py#L787), [hermes_cli/config_defaults.py:1724](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/config_defaults.py#L1724).

Outbound webhooks are notify-only async exports, not approval gates. Defaults: **256 in-memory pending deliveries**, timeout **10s**, cap **60s**, **2 attempts**, **1s backoff**. Queue full drops. Connection/5xx retry; 3xx/4xx terminate, redirects are refused. Neither retries nor a delivery ID establishes exactly-once or restart replay. [agent/outbound_webhooks.py:39](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/outbound_webhooks.py#L39), [agent/outbound_webhooks.py:224](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/outbound_webhooks.py#L224), [agent/outbound_webhooks.py:272](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/outbound_webhooks.py#L272), [agent/outbound_webhooks.py:315](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/outbound_webhooks.py#L315).

Tests inspected: tests/hermes_cli/test_plugins_middleware_chain.py:50,60 checks ordered rewrites/copy protection for normal payloads; tests/hermes_cli/test_plugin_api_compat.py:14 checks additive fields. These do not make arbitrary plugin side effects transactional.

## 14. Observability, Correlation & Execution Tracing

### 14.1 Separate products and persistent state

| Product / owner | Inputs → outputs/storage | Actual limit |
|---|---|---|
| SessionDB/routed logging | Transcript/accounting plus redacted operational logs | Not a complete span tree or tamper-proof audit log. |
| Core NeMo Relay | Profile/session/turn/logical-call scopes and managed LLM/tool events | Requires supported runtime/active consumers; not automatic external export. |
| Shared metrics | Validated lifecycle/Relay projection → profile SQLite aggregates | Not raw transcript tracing or semantic quality scoring. |
| Trace upload | Explicit transcript → viewer JSONL → optional authenticated upload | Conversion representation, not original call timings/span identities. |

Evidence: [hermes_logging.py:357](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_logging.py#L357), [agent/relay_runtime.py:399](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/relay_runtime.py#L399), [agent/relay_llm.py:681](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/relay_llm.py#L681), [agent/relay_tools.py:16](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/relay_tools.py#L16), [hermes_cli/observability/shared_metrics_subscriber.py:68](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/observability/shared_metrics_subscriber.py#L68), [agent/trace_upload.py:145](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/trace_upload.py#L145).

### 14.2 Correlation across components

Preparation selects task ID and creates turn ID. Hooks carry session/task/turn/API/tool IDs when available; auxiliary hooks inherit current-turn identity or emit empty parent fields for turn-less work. Profile-keyed RelayRuntime owns an instance UUID and in-memory session/subagent-parent maps. Children can parent under spawning turn/session handles. Logical LLM scopes group physical attempts by api_request_id; logical versus physical call accounting must remain distinct. [agent/turn_context.py:640](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/turn_context.py#L640), [agent/api_request_hooks.py:163](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/api_request_hooks.py#L163), [agent/auxiliary_hooks.py:25](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/auxiliary_hooks.py#L25), [agent/relay_runtime.py:399](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/relay_runtime.py#L399), [agent/relay_runtime.py:461](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/relay_runtime.py#L461), [agent/relay_runtime.py:863](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/relay_runtime.py#L863), [agent/relay_llm.py:681](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/relay_llm.py#L681).

Propagation is explicit. ContextVars/copied contexts support in-process paths; hosted-plugin RPC tracks origin while a request is in flight. Cron has durable occurrence/request/receipt identity, not a demonstrated single span tree shared by every transport. Relay can forward traceparent through supported SDK header paths; unsupported native transports strip it. Universal W3C trace propagation across MCP servers, external CLI processes, cron and remote backends was **not established**. External collectors may add coverage; that guarantee remains U. [hermes_cli/plugin_host_wire.py:326](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/plugin_host_wire.py#L326), [gateway/session_cron.py:104](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/gateway/session_cron.py#L104), [agent/relay_llm.py:785](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/relay_llm.py#L785), [tools/mcp_tool_transport.py:154](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/tools/mcp_tool_transport.py#L154).

### 14.3 Lifecycle, failures and export

Managed execution is retained by active consumers, including Relay plugin behavior. Unsupported platforms can receive explicit NoopRelayRuntime. Scope operations are bounded **10s**, using an **8-worker daemon pool**; timeout can lose a span instead of holding the caller indefinitely. A non-top concurrent turn scope can defer closure to session drain rather than pop another live scope. Observation best-effort behavior must not be generalized to request-interceptor permission policy. [agent/relay_runtime.py:37](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/relay_runtime.py#L37), [agent/relay_runtime.py:104](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/relay_runtime.py#L104), [agent/relay_runtime.py:118](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/relay_runtime.py#L118), [agent/relay_runtime.py:444](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/relay_runtime.py#L444), [agent/relay_runtime.py:842](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/relay_runtime.py#L842).

Core owns Relay lifecycle; plugin-configuration activation is process-shared even while hosts are profile-keyed. Therefore not every native-library setting is tenant-local. First-party observer failure logs before plugin dispatch; finalize_session must close the true boundary, not merely a turn. [agent/relay_runtime.py:154](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/relay_runtime.py#L154), [agent/relay_runtime.py:291](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/relay_runtime.py#L291), [hermes_cli/lifecycle.py:11](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/lifecycle.py#L11), [hermes_cli/lifecycle.py:99](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/lifecycle.py#L99).

API diagnostics mask sensitive keys and bound depth/sequences/strings; payload budget defaults **50,000 characters**, minimum override **1000**. This is not proof every prose value is secret-free. Trace conversion defaults redaction on and fails closed on redactor failure. It omits system messages/images, generates UUID/parentUuid and conversion-time timestamps: this is a transcript-viewer export, not the original execution timeline. Upload requires an explicit operation/credentials; none was performed. [agent/api_request_hooks.py:51](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/api_request_hooks.py#L51), [agent/api_request_hooks.py:67](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/api_request_hooks.py#L67), [agent/trace_upload.py:49](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/trace_upload.py#L49), [agent/trace_upload.py:145](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/trace_upload.py#L145), [agent/trace_upload.py:170](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/trace_upload.py#L170).

Tests inspected: tests/agent/test_relay_compaction.py:85,103,114 checks correct parent scope; tests/agent/test_trace_upload.py:43,62,68,79 checks conversion/redaction. No live trace/export coverage was measured.

## 15. Replaceable Context Engine & Strategy Boundaries

### 15.1 Selection and interface ownership

ContextEngine defines update_from_response/should_compress/compress plus optional selection/pruning/lifecycle/tools/status/model seams. Core also directly reads mutable token/threshold/context/compression/protection attributes: it is a behavioral-and-state contract, not one interchangeable compress callback. Base threshold **0.75**, protected head/tail **3/6** are interface defaults, not the built-in compressor's complete effective policy. [agent/context_engine.py:48](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/context_engine.py#L48), [agent/context_engine.py:56](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/context_engine.py#L56), [agent/context_engine.py:98](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/context_engine.py#L98), [agent/context_engine.py:121](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/context_engine.py#L121).

context.engine defaults **compressor**; safe workers force built-in behavior. Non-default selection searches category loaders then matching general-plugin registration; failure warns/falls back. Explicit compressor does not activate a plugin automatically. Category engines can be selected without general plugins.enabled; bundled names win. Registered singleton engines clone_for_agent (default deepcopy), avoiding parent/child budget leakage; uncopyable-state failures fall back. [agent/agent_init.py:1752](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/agent_init.py#L1752), [plugins/context_engine/__init__.py:32](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/plugins/context_engine/__init__.py#L32), [plugins/context_engine/__init__.py:60](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/plugins/context_engine/__init__.py#L60), [agent/context_engine.py:226](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/context_engine.py#L226).

Host resolves model capacity/per-model thresholds and calls update_model; plugin engines own compaction policy rather than every built-in adjustment. Tools remain subject to host grants/name checks. Turn admission, transcript commits, provider sanitization, checkpoints and native-compaction coordination remain host-owned; retrieval/index storage belongs to the selected implementation. [agent/agent_init.py:1822](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/agent_init.py#L1822), [agent/agent_init.py:1965](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/agent_init.py#L1965), [agent/context_engine.py:204](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/context_engine.py#L204).

### 15.2 Request selection versus durable compression

~~~text
history → assembled request → optional select_context → sanitizers/cache/provider
history → compress → host lease/CAS/commit → replacement durable history
finalization → best-effort on_turn_complete(cloned history, usage, metadata)
~~~

select_context can replace the provider context at request assembly independently of should_compress. It receives cloned reference history/incoming content and budget, but the request list itself is mutable. None retains that assembled list; nonempty list-of-dicts replaces it; persistence fields are stripped and downstream role/tool/provider sanitizers still apply. Selection does not commit history and can change cache prefix intentionally. Invalid return/exception falls back to the **same request object**, so prior in-place request mutations need not be undone. Fail-open is not transactional restoration of original request bytes. [agent/context_engine.py:121](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/context_engine.py#L121), [agent/conversation_loop.py:1357](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/conversation_loop.py#L1357), [agent/turn_request_assembly.py:145](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/turn_request_assembly.py#L145).

**Call-site qualification (I):** the ABC describes selection on every provider request, including retries, but actual selection is called by assemble_api_request outside the inner API retry loop. That loop rebuilds provider kwargs with build_api_request; it does not itself re-invoke select_context. Selection repeats when request assembly runs again, not necessarily for every physical retry. This is an interface-description/implementation distinction, not an executed behavior claim. [agent/conversation_loop.py:1757](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/conversation_loop.py#L1757), [agent/conversation_loop.py:1577](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/conversation_loop.py#L1577), [agent/turn_api_request.py:177](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/turn_api_request.py#L177).

on_turn_complete observes cloned history, ignores return and logs failure; abnormal early exits can skip it, so it is not durable-ingestion acknowledgment. on_session_end is a real boundary; reset/model-switch update calibration. No generic state-transfer protocol was located for arbitrary index/retrieval state when switching engine implementations. [agent/context_engine.py:143](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/context_engine.py#L143), [agent/context_engine.py:181](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/context_engine.py#L181), [agent/context_engine.py:226](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/context_engine.py#L226), [agent/conversation_loop.py:1407](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/conversation_loop.py#L1407).

### 15.3 Strategy limits and optionality

Prepared-ahead compression is built-in ContextCompressor-specific. Required pre-compress checkpoints suppress microcompaction; incompatible app-server paths refuse rather than pretend to guarantee extraction. Default engine update_model imports shared compressor threshold logic. Thus engines are replaceable within the host contract, not interchangeable authorities over persistence/caching/native compaction/safety. The category directory inspected supplies discovery, not evidence that LCM/vector retrieval is bundled or active; alternatives require their actual implementation/configuration. [agent/agent_init.py:1871](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/agent_init.py#L1871), [agent/context_engine.py:233](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/context_engine.py#L233), [agent/conversation_compression_checkpoint.py:63](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/conversation_compression_checkpoint.py#L63), [agent/conversation_compression_checkpoint.py:97](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/conversation_compression_checkpoint.py#L97), [plugins/context_engine/__init__.py:1](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/plugins/context_engine/__init__.py#L1).

Tests inspected: tests/agent/test_context_engine_select_context.py:114,137,178 (fallback/history protection/sanitizers); tests/agent/test_plugin_context_engine_clone.py:49,56 (clone/model isolation). The request-mutate-then-fail caveat is I; no executed regression is claimed.


## 16. SQLite & Configuration Migrations

### 16.1 Independent version domains and ownership

| Domain | Owner and persistent state | Upgrade semantics |
|---|---|---|
| Session DB | SessionSchemaMixin/SessionDB; schema_version **33** | Declarative shape reconciliation plus version-gated data transforms and idempotent repairs. |
| FTS storage | Separate layout/stale/rebuild markers | Legacy layout can remain until explicit optimize-storage while main schema advances. |
| Config | migrate_config/run_migrations; _config_version **53** | Ordered ladder; automatic-ladder support floor **12**. |
| Other stores | Verification, metrics, cron and board initializers | Separate schema domains, not all covered by SessionDB version33. |

Evidence: [hermes_state_common.py:295](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_state_common.py#L295), [hermes_state_schema.py:899](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_state_schema.py#L899), [hermes_state_schema.py:1100](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_state_schema.py#L1100), [hermes_cli/config_defaults.py:2703](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/config_defaults.py#L2703), [hermes_cli/config_migrations.py:23](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/config_migrations.py#L23), [agent/verification_evidence.py:21](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/verification_evidence.py#L21), [hermes_cli/observability/shared_metrics.py:36](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/observability/shared_metrics.py#L36), [cron/executions.py:50](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/cron/executions.py#L50).

### 16.2 Session database algorithm, concurrency and recovery

Writable initialization creates missing tables, reconciles columns from SCHEMA_SQL, heals special primary-key shapes, adds deferred indexes, repairs certain NULL active flags, probes FTS and runs data backfills/stamping. _reconcile_columns compares PRAGMA information with declared shape: duplicate-column races are accepted; lock errors propagate for retry; other DDL failures warn. A current schema stamp therefore does not prove all idempotent shape repairs succeeded. Read-only opens do not perform initialization. [hermes_state_schema.py:731](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_state_schema.py#L731), [hermes_state_schema.py:899](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_state_schema.py#L899), [agent/insights.py:108](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/insights.py#L108).

There is no universal all-or-nothing transaction for the complete migration process: startup executes DDL, table-shape rebuilds have their own handling, and message-UID backfill uses autocommit slices. Backfill uses **2000-rowid chunks**, approximately **1s budget after the first chunk**, and a durable cursor for later opens; legacy rows can temporarily lack UID. Startup reports a **600s progress lease** so large I/O-bound upgrades are not mistaken for a startup deadlock; that lease is not a completion deadline. [hermes_state_schema.py:53](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_state_schema.py#L53), [hermes_state_schema.py:778](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_state_schema.py#L778), [hermes_state_schema.py:899](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_state_schema.py#L899), [hermes_state_schema.py:1127](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_state_schema.py#L1127).

FTS capability, corruption and layout are separate from canonical messages. Stale recovery may detach triggers and defer/retry rebuilding; absent FTS can prevent claiming the gated chain complete. Heavy legacy→external-content storage optimization is explicitly opt-in with independent markers, rather than an automatic full rewrite on every open. No universal pre-migration snapshot or downgrade converter was established in the schema initializer. Update snapshots are separate and best-effort (§17). [hermes_state_schema.py:558](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_state_schema.py#L558), [hermes_state_schema.py:962](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_state_schema.py#L962), [hermes_state_schema.py:1100](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_state_schema.py#L1100).

### 16.3 Configuration migration lifecycle and failure guarantees

migrate_config validates parsing before normal dotenv sanitation, captures starting version, invokes the ladder, checks suspicious MCP/platform settings and stamps latest when eligible. Missing defaults remain read-time defaults instead of materializing DEFAULT_CONFIG. Explicit stamps below **12** skip the ladder/stamp with a warning; surrounding sanitation/security operations are outside that gate, so “below floor” does not mean absolutely no other file/config operation. Unversioned configuration receives selected legacy-key transforms, preserving current choices instead of applying all historical v0 defaults. [hermes_cli/config.py:1175](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/config.py#L1175), [hermes_cli/config.py:1213](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/config.py#L1213), [hermes_cli/config.py:1236](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/config.py#L1236), [hermes_cli/config_migrations.py:893](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/config_migrations.py#L893).

Every step compares with the same original version. Stamp-independent steps are **46/50**; config_only suppresses ancillary local-file changes for backend transformations. **A raising step logs/skips while later steps continue, and the caller can still stamp latest.** A latest stamp is not proof every transformation succeeded; a skipped one-time transform may need operator recovery. There is no rollback transaction spanning YAML, .env, SOUL, plugin conversion and all profiles. Managed overlays/backend write policy remain separate (§8). [hermes_cli/config_migrations.py:30](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/config_migrations.py#L30), [hermes_cli/config_migrations.py:893](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/config_migrations.py#L893), [hermes_cli/config_migrations.py:906](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/config_migrations.py#L906), [hermes_cli/config.py:1240](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/config.py#L1240).

Tests inspected: tests/hermes_cli/test_config_unversioned_migration.py:45 asserts user-choice/SOUL preservation plus legacy conversion; tests/hermes_cli/test_config_migration_nested_guards.py:65 asserts a failed step warning alongside latest stamp. No schema downgrade or crash-recovery experiment was executed.

## 17. Updates, Deployment & Runtime Lifecycle

### 17.1 Ownership, admission and state transitions

evaluate_update_admission checks explicit installation contract, steward/package authority and filesystem heuristics. Malformed contract data can refuse rather than infer that any Git checkout is self-updatable. Source, externally managed package and container lifecycles are separate. [hermes_cli/update_contract.py:82](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/update_contract.py#L82), [tests/hermes_cli/test_update_contract.py:90](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/tests/hermes_cli/test_update_contract.py#L90).

~~~text
admit/lock/receipt → target + checkout inspection → best-effort snapshot
→ preserve local work / syntax validation / durable obligations
→ source-tree movement commit point
→ new-tree dependency/build/config/maintenance completion
→ runtime resume/restart/verification, or explicitly owed follow-ups
~~~

_cmd_update_impl rejects active Git operations, handles proven dead-owner debris, resolves target/channel, records backup outcome and coordinates platform-specific pauses. Before tree movement, update_cmd_commit arms interrupted-pull recovery, completion and fleet obligations. Disarm/rollback is ownership-qualified. After movement, HEAD must match selected expected SHA before completion. Critical startup files are syntax-checked; syntax rollback is a specific branch, not a promise to reverse all later state changes. [hermes_cli/update_cmd.py:1771](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/update_cmd.py#L1771), [hermes_cli/update_cmd_commit.py:1](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/update_cmd_commit.py#L1), [hermes_cli/update_cmd.py:1731](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/update_cmd.py#L1731), [hermes_cli/update_cmd.py:891](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/update_cmd.py#L891).

### 17.2 Snapshots, dependency generations and failure

Defaults: pre_update_backup **quick**, full backup retention **5**, noninteractive local changes **stash**, clean parked branch auto-switch true, strategy **switch**. Quick snapshots skip files over **1GiB**; full adds a home archive. A failed snapshot logs/warns and **does not block update**; configured backup policy is not proof a recovery point exists. Stash conflicts/custom branches/install stewardship have their own handling rather than blanket reset/discard semantics. [hermes_cli/config_defaults.py:2331](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/config_defaults.py#L2331), [hermes_cli/update_cmd_maint.py:756](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/update_cmd_maint.py#L756), [hermes_cli/update_cmd.py:822](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/update_cmd.py#L822).

Completion runs from the new tree and syncs selected dependencies. Core-update PM sync may disable/evict incompatible plugins, whereas ordinary plugin-set admission rejects incompatible proposed changes. Active processes lease dependency generations rather than universally modifying their imported environment in place. Code movement, dependency convergence, build success and fleet restart are distinct outcomes; receipts/markers retain owed work across interruption. [hermes_cli/update_completion.py:322](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/update_completion.py#L322), [hermes_cli/plugins_admission.py:50](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/plugins_admission.py#L50), [pm/environments.py:464](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/pm/environments.py#L464), [hermes_cli/update_cmd.py:746](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/update_cmd.py#L746).

Early launch recovery checks interrupted-pull state before importing a torn application tree, can restore the prior source closure and relaunch. Owed-tail retry backs off after **2** failed attempts for **15 minutes**, stops automatic retries at **6** for matching inputs, and re-arms for changed inputs. Unreadable retry bookkeeping is permissive instead of bricking launch. A committed update can leave desktop build or fleet restart owed; this is repair policy, not process-stack checkpointing or arbitrary data rollback. [hermes_cli/_early_recovery.py:249](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/_early_recovery.py#L249), [hermes_cli/main.py:26](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/main.py#L26), [hermes_cli/venv_sync.py:191](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/venv_sync.py#L191), [hermes_cli/venv_sync.py:251](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/venv_sync.py#L251), [hermes_cli/update_completion.py:252](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/update_completion.py#L252).

### 17.3 Deployment and publication boundaries

Docker checks actual PID1 ownership: PID1 execs s6-overlay /init; under another init the entrypoint performs stage2/main-wrapper bootstrap and warns that supervised services are unavailable. Therefore not every container launch is supervised. Native sources/packaged launchers/gateway viewers also retain separate process ownership, and restart cannot undo external tool effects. [docker/entrypoint-dispatch.sh:17](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/docker/entrypoint-dispatch.sh#L17), [hermes_bootstrap.py:576](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_bootstrap.py#L576), [acp_adapter/entry.py:224](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/acp_adapter/entry.py#L224).

Plugin update publication is a related but separate PM transaction: staged digest validation, durable publication metadata, old-tree backup/replacement and matching dependency selection. Inspected tests assert code/metadata/config/environment preservation across manifest/version/dependency/publication failures and matching generation on success. Those tests are not evidence that core-update/config/DB mutations share the same rollback guarantee. [pm/publication.py:163](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/pm/publication.py#L163), [tests/hermes_cli/test_plugin_update_transaction.py:171](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/tests/hermes_cli/test_plugin_update_transaction.py#L171), [tests/hermes_cli/test_plugin_update_transaction.py:207](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/tests/hermes_cli/test_plugin_update_transaction.py#L207).

Remaining U: live service availability/restarts across platforms, complete installed-artifact authenticity, machine-specific deployment success and downgrade compatibility after data transformations. No update/deployment/sync/restart was performed.

## 18. Insights, Efficiency Analytics & Resource Governance

### 18.1 Analytics inputs, storage and output

InsightsEngine receives a SessionDB connection, defaults to **30 days**, optionally filters source, flushes queued token accounting, then aggregates overview/models/platforms/tools/skills/activity/top sessions from persisted sessions/messages/session_model_usage. The query cutoff uses **session started_at**, not every individual call timestamp; old sessions spanning the reporting window may be omitted. Source aliases and missing partial-index shapes have compatibility handling. Output is a report/rendering, not a model/skill update instruction. [agent/insights.py:93](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/insights.py#L93), [agent/insights.py:164](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/insights.py#L164), [agent/insights.py:196](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/insights.py#L196), [hermes_cli/subcommands/insights.py:8](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/subcommands/insights.py#L8).

Per-model route rows keep estimated/actual cost distinct; unknown rates/incomplete usage do not prove zero spend. Tool/skill counts describe observed calls, not success/value. No InsightsEngine result consumer was located in the inspected review/curator paths that automatically optimizes skills, models or policies; a custom plugin could add one. [agent/insights.py:150](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/insights.py#L150), [agent/insights.py:241](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/insights.py#L241), [agent/insights.py:263](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/insights.py#L263), [tests/agent/test_insights.py:264](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/tests/agent/test_insights.py#L264), [agent/background_review.py:979](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/background_review.py#L979), [agent/curator.py:931](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/curator.py#L931).

### 18.2 Optional telemetry and efficiency metrics

Shared metrics is separate from InsightsEngine. Collection **enabled=false** and sending **send=false** by default. A Relay subscriber validates bounded counter/dimension projections into profile telemetry/shared_metrics/metrics.sqlite3, not complete transcript/events. Package schema **hermes.shared_metrics.v4**, store schema **2**; write patience **250ms**, schema **5000ms**, in-memory deferred writes cap **1024**, local retention **30d**. Deferred/overflow paths mean it is not a lossless audit log. [hermes_cli/config_defaults.py:2311](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/config_defaults.py#L2311), [hermes_cli/observability/shared_metrics_subscriber.py:68](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/observability/shared_metrics_subscriber.py#L68), [hermes_cli/observability/shared_metrics.py:36](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/observability/shared_metrics.py#L36), [hermes_cli/observability/relay_shared_metrics.py:1393](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/observability/relay_shared_metrics.py#L1393).

Efficiency tracks eligible user-turn tokens, tool-schema overhead/use, truncation, cache breaks, undo/retry/interrupt waste. Internal review forks are distinguished from user turns; repeated discard of an already interrupted turn avoids duplicate waste attribution. Missing usage stays unknown. Cache classification uses a **300s TTL heuristic**, not proof of provider-internal cause; cancellation/token buckets are not semantic inefficiency or task-success metrics. [hermes_cli/observability/shared_metrics_efficiency.py:97](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/observability/shared_metrics_efficiency.py#L97), [hermes_cli/observability/shared_metrics_efficiency.py:173](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/observability/shared_metrics_efficiency.py#L173), [hermes_cli/observability/shared_metrics_efficiency.py:233](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/observability/shared_metrics_efficiency.py#L233), [tests/hermes_cli/test_shared_metrics_efficiency.py:69](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/tests/hermes_cli/test_shared_metrics_efficiency.py#L69).

Export requires consent-window eligibility independently of collection; active send passes recheck consent. Sender bounds: **30s request timeout**, **3 immediate attempts**, **20 packages/pass**, **25 cumulative attempts**. Default endpoint uses HTTPS and is not environment-overridable. This optional exporter is not a global quota coordinator. [hermes_cli/observability/shared_metrics_sender.py:31](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/observability/shared_metrics_sender.py#L31), [hermes_cli/observability/shared_metrics_sender.py:206](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/observability/shared_metrics_sender.py#L206), [hermes_cli/observability/shared_metrics_consent.py:57](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/observability/shared_metrics_consent.py#L57), [hermes_cli/observability/relay_shared_metrics.py:1306](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/observability/relay_shared_metrics.py#L1306), [hermes_cli/config_defaults.py:2314](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/config_defaults.py#L2314).

### 18.3 Governance scope and guarantees

| Limit | Actual ownership/enforcement | Architectural consequence |
|---|---|---|
| Iterations | Thread-safe consume/refund; normal parent/children have separate counters; constructor sys.maxsize, child250 | Parent cap is not a tree-wide allowance; grace/refunds/child calls matter. |
| Context/output | Per-model compaction/capacity and tool/turn output thresholds | Not a host RAM/disk limit; aux/children incur separate consumption. |
| run_budget_seconds | Per-turn, default None; 80% nudge plus implicit stale timeout min(existing,max(60s,half remaining)); explicit deadlines can win | Not a hard kill at expiry or global spend ceiling. |
| Concurrency | Optional gateway admission/unbounded workers; delegate10/tool batch8; independent cron pool; plugin RPC serving threads | Local caps do not compose into one host-wide maximum. |
| Tool timeouts | Backend/tool-owned termination; Python/hook worker may be abandoned | Timeout does not imply all effects/descendants stopped. |
| Credits/rate limits | Provider-specific response/account state, route/error policy | External quota is not a single atomic local balance across every provider/profile/tool. |
| Storage/retention | Store-specific prune/archive/snapshot/metric limits | No unified disk/RSS/CPU allowance established across profiles, browsers, kernels and plugins. |

Evidence: [agent/iteration_budget.py:25](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/iteration_budget.py#L25), [tools/delegate_tool.py:203](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/tools/delegate_tool.py#L203), [agent/agent_init.py:603](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/agent_init.py#L603), [agent/chat_completion_helpers.py:678](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/chat_completion_helpers.py#L678), [agent/conversation_loop.py:126](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/conversation_loop.py#L126), [gateway/turn_executor.py:25](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/gateway/turn_executor.py#L25), [tools/delegate_tool_config.py:19](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/tools/delegate_tool_config.py#L19), [agent/tool_executor.py:136](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/tool_executor.py#L136), [hermes_cli/plugin_host_wire.py:356](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/plugin_host_wire.py#L356), [agent/credits_tracker.py:583](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/credits_tracker.py#L583), [agent/rate_limit_tracker.py:89](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/rate_limit_tracker.py#L89).

Reporting cost rollup is not a reservation before fanout or atomic child stop. Stronger plugin/provider/backend policies remain possible but are not implied by these counters. Tests inspected: tests/agent/test_insights.py:264; tests/hermes_cli/test_shared_metrics_efficiency.py:69; tests/agent/test_run_budget.py:103,118,143. These inspect reporting/timeout contracts, not a global governor.

## 19. Plugin Lifecycle, Compatibility & Knowledge Provenance

### 19.1 Activation, ownership and publication

PluginManager loads manifests/register(ctx), retaining host-owned reversible handles. Tools/hooks/middleware/commands/providers have distinct registries/category activation. PluginRegistration.dispose is idempotent; replacement leases are identity-qualified so stale unload cannot clear a new owner. Successful load publishes staged registrations; failure disposes that load's handles. Some process-global registrations deliberately survive unload-all and reconcile on rediscovery. Cleanup is not rollback of arbitrary plugin side effects. [hermes_cli/plugins_ledger.py:30](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/plugins_ledger.py#L30), [hermes_cli/plugins_ledger.py:88](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/plugins_ledger.py#L88), [hermes_cli/plugins_ledger.py:118](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/plugins_ledger.py#L118), [hermes_cli/plugins_ledger.py:132](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/plugins_ledger.py#L132), [registration_lifecycle.py:40](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/registration_lifecycle.py#L40).

Install/enable/update passes PM dependency admission/publication, separate from advisory requires_plugins. Proposed enabled sets resolve under install ownership; staged digest and metadata journal bind publication. Live handler/command activation does not force replacement of every conversation's pinned prompt/tool array. Python importer timeout can leave abandoned code running. [hermes_cli/plugins_admission.py:50](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/plugins_admission.py#L50), [pm/publication.py:163](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/pm/publication.py#L163), [hermes_cli/plugins_activation.py:31](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/plugins_activation.py#L31), [hermes_cli/plugins_loader.py:106](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/plugins_loader.py#L106).

### 19.2 Compatibility: enforced versus advisory

Native parser supports manifest **2**, tolerates newer/unknown fields with warnings and carries api_version metadata; no mandatory monolithic native API-generation match was located in the inspected loader. requires_hermes is load-blocking, but unparseable versions are permissive and suffixes simplify to numeric release tuples. Config schema mismatch warns. requires_plugins orders topologically, but missing dependencies warn/load and cycles fall back alphabetically. python_dependencies metadata is surfaced rather than automatically installed by its parser; PM admission is the separate enforcement path. [hermes_cli/plugins_manifest.py:162](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/plugins_manifest.py#L162), [hermes_cli/plugins_manifest.py:206](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/plugins_manifest.py#L206), [hermes_cli/plugins_manifest.py:238](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/plugins_manifest.py#L238), [hermes_cli/plugins_manifest.py:471](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/plugins_manifest.py#L471), [hermes_cli/plugins_manifest.py:502](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/plugins_manifest.py#L502), [hermes_cli/plugins_loader.py:256](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/plugins_loader.py#L256).

Compatibility comprises local behavioral contracts—callback kwargs, ABC defaults, manifest fields, wire/persistence schemas—not preservation of arbitrary internal import paths. Frozen legacy-plugin tests load through discovery and accept additive fields/narrow callbacks. Documentation's deprecation policy is not by itself an executable multi-release guarantee. [hermes_cli/plugins_dispatch.py:275](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/plugins_dispatch.py#L275), [agent/context_engine.py:143](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/agent/context_engine.py#L143), [hermes_cli/plugin_host_wire.py:39](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/plugin_host_wire.py#L39), [tests/hermes_cli/test_plugin_api_compat.py:14](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/tests/hermes_cli/test_plugin_api_compat.py#L14).

### 19.3 Optional hosted native plugins

plugins.isolation defaults **in_process**; **host** runs supported non-bundled code in one host process/profile. Bundled code remains in-process; managed overlay participates, unknown mode warns/falls back to in_process. Configurable launcher prefix plus profile child environment launches bidirectional JSON RPC with generation-qualified callable/object proxies and ABC type restoration. This is a real process boundary, not automatically filesystem/network/CPU containment: default launcher prefix is empty. [hermes_cli/plugin_isolation.py:32](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/plugin_isolation.py#L32), [hermes_cli/plugin_isolation.py:100](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/plugin_isolation.py#L100), [hermes_cli/plugin_isolation.py:113](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/plugin_isolation.py#L113), [hermes_cli/plugin_host.py:127](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/plugin_host.py#L127), [hermes_cli/plugin_host_wire.py:174](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/plugin_host_wire.py#L174).

Live-object capabilities such as platform/approval/auth/channel handles refuse across the boundary; CLI registration skips, and pre_gateway_dispatch degrades live-handle information. Memory/context/provider objects can proxy. Host hello timeout **30s**, shutdown grace **3s**, restart budget **3 deaths/600s** before further restarts refuse. Pending calls fail on crash; reload excludes the plugin that killed its host while loading. Generation checks avoid reused-ID confusion; category objects may re-resolve after restart, but in-flight effects are not transactionally resumed/safely replayed. [hermes_cli/plugin_isolation.py:39](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/plugin_isolation.py#L39), [hermes_cli/plugin_host.py:46](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/plugin_host.py#L46), [hermes_cli/plugin_host.py:179](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/plugin_host.py#L179), [hermes_cli/plugin_host.py:354](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/plugin_host.py#L354), [hermes_cli/plugin_host.py:571](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/plugin_host.py#L571).

RPC retains caller context while pending and creates daemon serving threads; most calls have no universal RPC timeout. Timing out local pending state does not kill already-running code. Process hosting improves crash separation without supplying durable calls, bounded resource use or complete rollback. Tests assert separate PID, nested ctx calls, provider proxies and restart after crash, not comprehensive live security. [hermes_cli/plugin_host_wire.py:326](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/plugin_host_wire.py#L326), [hermes_cli/plugin_host_wire.py:356](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/plugin_host_wire.py#L356), [tests/hermes_cli/test_plugin_host.py:66](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/tests/hermes_cli/test_plugin_host.py#L66).

### 19.4 Artifact provenance versus knowledge truth

| Record | Contents / consumer | Limit |
|---|---|---|
| plugins/.install-metadata.json | Installer source/revision/pin/catalog/update metadata, locked atomic updates | Application provenance authority, not immutable/authenticated local ledger. |
| Plugin provenance class | Row+.git managed Git; neither manual; .git-only self-cloned; row-only drift | Installation/update ownership, not harmlessness. |
| skills/.hub/lock.json | Source/ID/trust/scan verdict/hash/path/files/scan provenance/times | Installation/scan provenance, not semantic instruction validity. |
| .bundled_manifest / .usage.json | Bundled hashes; origin/management/use/patch state | Management policy and usage, not utility/truth. |
| Curator ledger/blobs | Best-effort deltas/actor/recovery content | Not a universal fact→original-evidence passage chain. |

Evidence: [hermes_cli/plugins_cmd_git.py:33](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/plugins_cmd_git.py#L33), [hermes_cli/plugins_cmd_git.py:61](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/plugins_cmd_git.py#L61), [hermes_cli/plugins_provenance.py:107](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/plugins_provenance.py#L107), [tools/skills_hub.py:322](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/tools/skills_hub.py#L322), [tools/skills_sync.py:113](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/tools/skills_sync.py#L113), [tools/skill_provenance.py:8](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/tools/skill_provenance.py#L8), [tools/skill_ledger.py:84](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/tools/skill_ledger.py#L84).

Installer review can bind an exact **40-hex SHA** and reject source movement; catalog provenance is outside the plugin-supplied tree. This is stronger than trusting a repository's self-description but not proof of safe code. Skill origin is ContextVar state; attendedness is orthogonal to background-review origin and affects destructive memory gates. No signed knowledge graph, fact confidence calculus or eval-certified skill-promotion path was located in these mechanisms. [hermes_cli/plugins_cmd_git.py:103](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/plugins_cmd_git.py#L103), [hermes_cli/plugins_cmd_install.py:360](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/plugins_cmd_install.py#L360), [hermes_cli/plugins_cmd_install.py:392](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/hermes_cli/plugins_cmd_install.py#L392), [tools/skill_provenance.py:29](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/tools/skill_provenance.py#L29), [tools/memory_tool.py:179](https://github.com/NousResearch/hermes-agent/blob/4ad61cee29316e21e03c4585bb4ececfa52385fe/tools/memory_tool.py#L179).

Tests inspected: tests/hermes_cli/test_plugin_api_compat.py:14; tests/hermes_cli/test_plugin_host.py:66; tests/hermes_cli/test_plugin_update_transaction.py:171,207. Identity/publication/rollback checks are not content-quality certification.

### Follow-up correction proposals — original text intentionally unchanged

1. **Native plugin boundaries (§§5H/5J/9):** default native code remains broad authority, but optional per-profile hosted execution exists. Host mode is not a general OS sandbox (§19.3).
2. **Frozen prefix (§§1/5F):** default behavior preserves prefix; optional select_context can replace request context/prefix per request without rewriting stored history (§15.2).
3. **Run budget (§5A):** remaining budget also caps implicit stale-call patience with a 60s floor/explicit-setting exceptions; it is still not a hard global deadline/spend cap (§18.3).
4. **Evidence strength:** V means implementation plus selected inspected assertions, not executed passing tests for every branch; interfaces/artifacts/version stamps do not establish all optional consumers or guarantees (§§12–19).

### Follow-up synthesis and uncertainty

Verified additions: executable evaluation/CI lanes; separate hook/middleware/event contracts; Relay/correlation and optional telemetry/export; actual context selection/clone boundaries; independent DB/config migration ladders; durable update commit/completion obligations; scoped analytics/resource limits; hosted plugins and artifact provenance.

Not established: automatic benchmark→learning promotion, universal distributed tracing, generic context-engine state migration, all-or-nothing config migration, atomic code/dependency/service convergence, one global resource governor, or provenance proving knowledge truth. Live service/provider/backend guarantees remain U without execution.

Original bytes and the C18–C25 evidence map are preserved/recorded in the source index. No preceding section, link definition, contract or Hermes source was edited.


