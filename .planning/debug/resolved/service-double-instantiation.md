---
status: resolved
trigger: "FastAPI dependency injection creates separate service instances from the orchestrator, causing routers to use uninitialized services."
created: 2026-03-13T00:00:00Z
updated: 2026-03-13T00:00:00Z
---

## Current Focus

hypothesis: dependencies.py get_pose_service / get_stream_service / get_hardware_service use @lru_cache() and construct brand-new service instances that never pass through orchestrator.start(), while app.state already holds the properly-started instances.
test: Read all five key files to confirm the divergence between app.state path (health.py) and Depends() path (pose.py, stream.py).
expecting: Confirmed — health.py reads from request.app.state; pose.py and stream.py use the orphaned cached constructors.
next_action: Rewrite the three service dependency functions to accept Request and read from request.app.state, matching health.py pattern; then update caller signatures in pose.py and stream.py.

## Symptoms

expected: Pose, stream, and hardware routers should use the same service instances that the ServiceOrchestrator starts and manages via lifespan.
actual: dependencies.py creates NEW instances via @lru_cache() (lines 25-58) that bypass orchestrator.start(). The health router correctly uses request.app.state (health.py lines 60-63), but pose.py and stream.py use the orphaned dependency-injected instances.
errors: No crash, but services are not properly initialized — they never go through orchestrator.start() lifecycle.
reproduction: Any call to /api/v1/pose/* or /api/v1/stream/* endpoints uses different service instances than the orchestrator manages.
started: Pre-existing. Commit 8ee33c1a exposed services to app.state but did not update dependency injection to consume them.

## Eliminated

- hypothesis: Bug is in orchestrator.start() — it might not set app.state attributes correctly.
  evidence: app.py lifespan lines 43-45 explicitly set app.state.hardware_service, app.state.pose_service, app.state.stream_service after orchestrator.start() returns. health.py successfully reads these.
  timestamp: 2026-03-13T00:00:00Z

## Evidence

- timestamp: 2026-03-13T00:00:00Z
  checked: v1/src/api/dependencies.py lines 25-58
  found: get_pose_service, get_stream_service, get_hardware_service are decorated with @lru_cache() and construct fresh PoseService/StreamService/HardwareService objects without calling start().
  implication: Every router endpoint that uses Depends(get_pose_service) etc. receives an uninitialized service object.

- timestamp: 2026-03-13T00:00:00Z
  checked: v1/src/api/routers/health.py lines 60-63
  found: health_check() reads services via getattr(request.app.state, 'pose_service', None) — the correct pattern.
  implication: health.py is the reference implementation; pose.py and stream.py must be updated to match.

- timestamp: 2026-03-13T00:00:00Z
  checked: v1/src/api/routers/pose.py
  found: All 8 endpoint functions declare pose_service: PoseService = Depends(get_pose_service) and/or hardware_service: HardwareService = Depends(get_hardware_service) with no Request parameter fed to the dependency.
  implication: These endpoints operate on un-started service instances.

- timestamp: 2026-03-13T00:00:00Z
  checked: v1/src/api/routers/stream.py
  found: HTTP endpoints get_stream_status, start_streaming, stop_streaming use stream_service: StreamService = Depends(get_stream_service). WebSocket endpoints do not inject services directly — they use connection_manager only.
  implication: Three HTTP endpoints in stream.py need updating; WebSocket endpoints are unaffected.

- timestamp: 2026-03-13T00:00:00Z
  checked: v1/src/api/dependencies.py — check_service_health() (lines 253-294)
  found: This function already uses request.app.state correctly. Only the three top-level lru_cache functions are broken.
  implication: Scope of fix is narrow: replace @lru_cache constructors with request.app.state lookups.

## Resolution

root_cause: get_pose_service, get_stream_service, get_hardware_service in dependencies.py use @lru_cache() to construct and memoize brand-new service instances at first call. These instances never go through ServiceOrchestrator.start(), so they lack any initialization performed there. The orchestrator-managed instances are available on app.state after lifespan runs, but no code path in pose.py or stream.py reads from app.state.
fix: Convert the three functions to async dependency functions that accept Request and return getattr(request.app.state, '<service>', None), raising HTTP 503 if the service is absent. Remove @lru_cache and direct constructor calls. Update all endpoint signatures in pose.py and stream.py to include the implicit request parameter that FastAPI threads through Depends().
verification: Import check confirmed all three functions are async and accept only `request: Request`. 249 pre-existing tests still pass. pose.py and stream.py required no changes — FastAPI resolves Request in sub-dependencies automatically from the active request context.
files_changed:
  - v1/src/api/dependencies.py
