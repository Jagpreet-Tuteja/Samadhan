# Test verification notes (2026-09-13 foundation slice)

`AbstractIntegrationTest` uses Testcontainers, which is the correct,
portable way to run these tests on a real dev machine or in CI (GitHub
Actions per `docs/deployment.md`) — it needs nothing but Docker installed,
no manually-managed local Postgres.

The sandboxed session that first built this slice could drive the `docker`
CLI (confirmed: `docker ps`, `docker run hello-world`, `docker compose up`
all worked normally) but every Testcontainers call failed identically:

```
NpipeSocketClientProviderStrategy: failed with exception BadRequestException
(Status 400: {"ID":"","Containers":0, ...all-empty/zeroed fields...})
```

This happened regardless of `DOCKER_HOST` pointing at the active Docker
Desktop context's actual engine pipe (`dockerDesktopLinuxEngine`) vs. the
OS-default pipe (`docker_engine`) — both returned the identical stub body,
which strongly suggests the sandbox intercepts direct Docker Engine
API/socket access from arbitrary processes while still allowing the
vetted `docker` CLI executable itself through. This is an environment
restriction of that one sandboxed session, not a defect in the approach.

To get a real pass/fail signal in that session anyway, `AbstractIntegrationTest`
was temporarily changed to point at a `docker compose up -d`-started
Postgres on `localhost:5432` instead of a Testcontainers-managed
container (same schema, same restricted `nexora_app` role, same
migration — only the container lifecycle mechanism differed), the real
test classes were run via `./mvnw test`, and both passed:

```
Tests run: 4, Failures: 0, Errors: 0 -- RoleSeedDataTest
Tests run: 1, Failures: 0, Errors: 0 -- TenantIsolationIT
```

`AbstractIntegrationTest` was then reverted to the Testcontainers version
that's actually checked in. The RLS policy itself was additionally
verified directly with `psql` as the restricted `nexora_app` role
(same-org read visible, cross-org read empty, no-context read empty, and
a cross-org `INSERT` rejected by `WITH CHECK`) — see
`docs/progress.md`'s 2026-09-13 entry for what that surfaced (the
text-vs-uuid-cast fix to the RLS policy in the migration).

If a future session hits the same Testcontainers failure, this file
documents that it's a known sandbox quirk, not a regression — check
`docker ps`/`docker run hello-world` first; if those work but
Testcontainers still can't find a Docker environment, this is it.

## 2026-09-13, auth + Module 1 slice — two real bugs the same verification method caught

Same method as above (temporarily pointing `AbstractIntegrationTest` at a
`docker compose` Postgres, confirming green, then reverting) surfaced two
genuine bugs — worth recording so a future session doesn't waste time
rediscovering them:

1. **`JwtAuthenticationFilter` must never be a `@Component`/`@Bean`.**
   Spring Boot auto-registers every `Filter`-typed bean as an independent
   servlet-container filter. A custom Spring Security filter also wired
   in via `HttpSecurity.addFilterBefore(...)` ends up invoked twice, from
   two different places in the overall filter pipeline, with confusing
   results. Fix: never register it as a bean at all — construct it
   directly inside `SecurityConfig.securityFilterChain(...)`
   (`new JwtAuthenticationFilter(jwtService, permissionService)`), so it
   exists only where `addFilterBefore` puts it.

2. **Setting authentication via `SecurityContextHolder.getContext().setAuthentication(...)` doesn't reliably stick** in Spring Security 6.
   `SecurityContextHolderFilter`'s deferred-supplier mechanism means a
   later `getContext()` call (e.g. from `AuthorizationFilter`) can hand
   back a freshly-created empty context instead of the one a filter
   earlier in the chain mutated — manifesting as every authenticated
   request being rejected with "Full authentication is required," even
   though the filter's own logging showed authentication being set
   correctly moments earlier. Fix (the pattern Spring's own docs now
   recommend): build a whole context and install it explicitly —
   ```java
   SecurityContext context = SecurityContextHolder.createEmptyContext();
   context.setAuthentication(authentication);
   SecurityContextHolder.setContext(context);
   ```
   — not `SecurityContextHolder.getContext().setAuthentication(...)`.

Diagnosing #2 took adding `@EnableWebSecurity(debug = true)` temporarily
to print the actual filter chain, and testing a real protected endpoint
in isolation (a temporary `/api/v1/_debug/whoami` controller) to confirm
authentication reached the controller layer correctly even while a
different, unmapped path (`/api/v1/customers`, not built yet at the
time) kept returning 401 — which turned out to be an unrelated,
expected 404-via-error-dispatch artifact of testing an endpoint that
didn't exist yet, not a bug.

A third, unrelated issue: `TestRestTemplate`'s default JDK
`HttpURLConnection`-backed client throws `HttpRetryException: cannot
retry due to server authentication, in streaming mode` on any POST that
gets a 401/403 response back. Fix: add `org.apache.httpcomponents.client5:httpclient5`
as a test dependency — Spring Boot auto-prefers it once present.

## 2026-11-03 — Testcontainers now works directly; the 2026-09-13 diagnosis was likely wrong

On a normal Windows machine with Docker Desktop (Engine 29), the same
`NpipeSocketClientProviderStrategy ... Status 400 {"ID":"",...all-empty...}`
failure described at the top of this file appeared, outside any sandbox.
The cause is a version mismatch: Docker Engine 29 dropped support for
the old Docker API version Testcontainers 1.20.x requests by default, and
answers with that empty stub. Upgrading to Testcontainers 1.21.4
(`pom.xml`) fixed it. The docker-compose workaround above is no longer
needed.

A second issue surfaced once the containers really ran: with a per-class
`@Container` field, each test class got a fresh container on a new port,
but Spring's cached application context (shared by classes with
identical configuration) still pointed at the previous, stopped one —
every query then timed out after 30s. `AbstractIntegrationTest` now
starts one shared container for the whole run. Full suite:

```
./mvnw test '-Dtest=*Test,*IT'    # 33 tests, 0 failures
```
