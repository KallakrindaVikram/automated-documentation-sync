# Blog Platform Architecture

**Status:** Approved architecture

**Approval:** Explicitly approved by the user on 2026-10-21.

**Requirements baseline:** [Approved Blog Platform requirements](./requirements.md)

**Scope:** Initial demonstration release only; no application code is implemented by this document.

## 1. Architecture goals and constraints

The design satisfies the approved requirements while keeping the application inexpensive, maintainable, testable, and straightforward to run locally. It uses a same-origin web UI and API, one relational database, and server-side sessions. The browser is never an authority for authentication or post ownership.

The repository currently contains documentation and starter-agent files, but no application source, package manifest, or test suite. The environment has Node.js **v22.23.3**, npm **10.9.9**, Python **3.14.7**, and Java **17.0.12**. Recommend Node.js 22 LTS and npm for this project; Python and Java are not needed for the application. In Windows PowerShell, `npm.ps1` is blocked by the current execution policy, while `npm.cmd` is available; use `npm.cmd` or an equivalent approved npm invocation rather than changing machine policy.

## 2. Proposed stack

| Concern | Proposal | Rationale and constraints |
|---|---|---|
| Language/runtime | TypeScript on Node.js 22 LTS, with a pinned Node major and npm lockfile | One language across UI and API; the Node 22 runtime is available locally. Test chosen packages against Windows and the pinned runtime. |
| Frontend | React, Vite, TypeScript, and a small client-side router | Provides a conventional accessible SPA and direct-route support without introducing a larger full-stack framework. Vite serves the UI in development and produces static assets for the API server to serve in a built run. |
| API | Node.js and Express, written in TypeScript | Small, explicit REST boundary and middleware ecosystem. Keep business rules and persistence out of route handlers. |
| Validation/contracts | Shared TypeScript request/response schemas, with Zod as a candidate | Reuse shape and field validation in forms and API, while the API remains authoritative. Audit package maintenance and pin its version before adoption. |
| Persistence | SQLite in a local file, accessed through a mature parameterized-query driver (for example, `better-sqlite3`, subject to Windows/Node compatibility validation) | No paid or separate database service; relational constraints and transactions suit users, posts, and sessions. Native SQLite drivers need an install/build compatibility check. |
| Authentication | `express-session` with a vetted SQLite-backed session-store adapter; Node's built-in asynchronous `crypto.scrypt` for salted password hashing | Meets the explicit server-side-session requirement and avoids plaintext passwords. Do not use Express's in-memory `MemoryStore`. Select the session-store package only after checking maintenance, Node 22 support, and compatibility with the chosen SQLite driver. |
| Tests | Vitest for unit/component tests; Supertest for API integration; Playwright for critical browser journeys where practical | Separates fast logic/API checks from end-to-end navigation and cookie behavior. These are proposed dependencies, not installed or tested. |
| Package management | npm workspaces | Keeps UI, API, and shared contracts in one repository and uses the locally available npm toolchain. |

### Trade-offs

- A single Node application and SQLite file are easy to run and inspect locally, but do not target multi-instance horizontal scaling or concurrent high-volume writes. Such scaling is outside the approved scope.
- Serving the built SPA and API from the same origin avoids production CORS configuration and simplifies cookie/CSRF policy. Development uses a Vite proxy to keep browser requests same-origin.
- SQLite's single-writer behavior is appropriate for the demo. Use transactions and a busy timeout; revisit the database choice only if a future approved workload requires it.
- `scrypt` avoids a native password-hashing dependency and is available in Node's crypto module. Its parameters must be benchmarked on the supported development hardware and recorded; changing to Argon2id later requires a deliberate dependency and migration review.
- A native SQLite driver may require platform-specific prebuilt binaries or build tools. Verify installability on Windows Node 22 before finalizing the dependency; do not silently fall back to in-memory persistence.

## 3. System shape and responsibilities

```text
Browser
  React/Vite UI
    - route and form state
    - loading / empty / error / success views
    - sends same-origin JSON requests with session cookie and CSRF header
          |
          | HTTPS in local development; same-origin in built run
          v
Express API
  middleware: request ID, body-size limit, session, CSRF, safe errors
  routes/controllers: HTTP parsing and response mapping
  services: authentication, post rules, ownership, pagination
  repositories: prepared SQLite statements and transactions
          |
          +---- SQLite application tables: users, posts, migrations
          +---- SQLite-backed server-side session store
```

### Frontend responsibilities

- Render public feed, post detail, registration, login, create/edit forms, and My Blogs routes. Directly loading a valid post URL must load the correct post.
- Perform immediate form feedback using the shared schemas, but never rely on client validation or hidden controls for authorization.
- Obtain a CSRF token before state-changing requests, send it in a dedicated header, and rely on the browser's `HttpOnly` session cookie.
- Render request loading, empty, validation, not-found, authorization, and unexpected-failure states separately. Display success feedback only after an API success response.
- Keep session/user state scoped to the current browser session and clear/refetch it on logout or account change.
- Render titles and body content as plain text. Use normal React text interpolation and CSS whitespace preservation; never use `dangerouslySetInnerHTML` for user content.
- Use `Intl.DateTimeFormat` to display UTC timestamps in the viewer's local timezone. The exact presentation format remains a UI choice.

### API and business responsibilities

- **Middleware** applies request IDs, JSON parsing with a bounded body size, session loading, CSRF/origin checks on mutations, and a final error mapper.
- **Controllers** translate HTTP input into typed service calls and map expected outcomes to status codes and response DTOs. They do not contain SQL or ownership policy.
- **Services** enforce registration/login rules, password hashing, session rotation, post validation, ownership outcomes, and transaction boundaries.
- **Repositories** execute parameterized SQL, transactions, and migrations; never accept SQL fragments assembled from request data.
- The API derives `currentUserId` exclusively from the validated server-side session. Ignore or reject client-supplied `authorId`/`userId`; never use it to grant access.
- Return explicit, safe error responses for expected failures. Log unexpected errors with a request ID but without passwords, password hashes, cookie values, CSRF tokens, session IDs, or request bodies containing credentials.

### Persistence responsibilities

- Keep the SQLite database in a configurable local data directory that is ignored by Git. Create the directory during initialization, not by committing a database file.
- Enable SQLite foreign-key enforcement for every connection. Use prepared statements, transactions for writes, and a bounded busy timeout. WAL mode may be enabled for local read/write coexistence; it does not make SQLite a multi-writer database.
- Keep schema changes in ordered, versioned migrations. A migration either succeeds atomically or reports a startup/command failure; do not silently continue with an incompatible schema.
- Ensure post writes update only the requested record and preserve `author_id` and `created_at` on edit. Deleting one post cannot affect other posts.

## 4. Proposed project layout

```text
apps/
  api/
    src/
      app.ts                 # Express app and middleware wiring
      server.ts              # startup and validated configuration
      routes/                # HTTP boundary only
      services/              # business rules and authorization
      repositories/          # SQLite queries and transactions
      auth/                  # password and session helpers
      db/migrations/          # ordered schema changes
    tests/
  web/
    src/
      app/                    # router, app shell, session state
      pages/                  # feed, detail, auth, create/edit, My Blogs
      components/
      api/                    # typed same-origin API client
    tests/
packages/
  contracts/
    src/                      # shared schemas and public DTO types
data/                         # local SQLite files; ignored by Git
```

This layout is a proposal, not a generated tree. Avoid creating packages until actual shared code requires them. A root npm-workspaces manifest coordinates app scripts.

## 5. Entities and relationships

All identifiers are opaque UUIDs generated by the server. Timestamps are stored as normalized UTC ISO-8601 values with fixed precision. Treat a post's creation time as its publication time because posts publish immediately and drafts are excluded. Apply the agreed limits using one documented Unicode code-point counting rule on both frontend and backend; the backend check is authoritative.

### `users`

| Field | Proposed type / constraint | Notes |
|---|---|---|
| `id` | UUID text primary key | Generated by the server. |
| `display_name` | Text, not null | Displayed publicly with authored posts; no uniqueness requirement. |
| `email_normalized` | Text, not null, unique | Trim and lowercase before lookup/storage so email comparison is case-insensitive. Do not return it in public post DTOs. |
| `password_hash` | Text, not null | Encoded algorithm/parameters, unique random salt, and derived hash; never plaintext. |
| `created_at` | UTC timestamp, not null | Account creation time. |

### `posts`

| Field | Proposed type / constraint | Notes |
|---|---|---|
| `id` | UUID text primary key | Generated by the server; malformed identifiers are rejected before repository lookup. |
| `title` | Text, not null | Trim outer whitespace; reject blank; at most 120 Unicode code points. |
| `content` | Text, not null | Plain text; trim outer whitespace; reject blank; at most 20,000 Unicode code points. |
| `author_id` | UUID text, not null, foreign key to `users.id` | Derived from the authenticated session. Do not accept updates from a client. |
| `created_at` | UTC timestamp, not null | Publication/creation time; immutable after insert. |
| `updated_at` | UTC timestamp, not null | Set on create and changed on every successful edit. |

Relationship: one user authors zero or more posts; each post has exactly one author. Index `posts(created_at DESC, id DESC)` for deterministic newest-first reads and `posts(author_id, created_at DESC, id DESC)` for My Blogs. `ON DELETE RESTRICT` is sufficient because user deletion is not in scope. Do not add drafts, roles, comments, or soft-delete fields without approved scope change.

### Sessions and migrations

The session-store adapter owns its session records and expiry cleanup. Configure it to persist server-side sessions in SQLite, set the idle TTL to 30 minutes, and also enforce the 24-hour absolute lifetime in application session data. Keep schema migrations/version metadata separate from adapter-owned session schema.

## 6. API boundary and response conventions

Prefix JSON endpoints with `/api`. Serve the built frontend from the same origin in a built local run. The Vite development server proxies `/api` to the API server; do not enable permissive cross-origin access.

### Proposed endpoints

| Method and path | Access | Purpose and result |
|---|---|---|
| `GET /api/auth/csrf` | Public | Establish/refresh an anonymous session as needed and return a synchronizer CSRF token with `Cache-Control: no-store`. |
| `POST /api/auth/register` | Public + CSRF | Validate display name/email/password, create account, rotate the session ID, automatically authenticate, and return a safe user DTO plus a newly issued CSRF token. |
| `POST /api/auth/login` | Public + CSRF | Authenticate credentials, rotate the session ID, and return safe session/user data and a new CSRF token. Return the same generic 401 response for unknown email and wrong password. |
| `GET /api/auth/me` | Authenticated | Return the current user's safe session DTO; no password hash, session ID, or unnecessary private account data. |
| `POST /api/auth/logout` | CSRF; authenticated or repeat-safe | Destroy the server session and clear the cookie. Repeating logout is safe. |
| `GET /api/posts` | Public | Return the complete published feed, sorted by `created_at DESC, id DESC`; each item includes title, public author display name, publication timestamp, and a 200-code-point plain-text preview. |
| `GET /api/posts/:id` | Public | Return one post's title, full plain-text content, public author display name, and timestamps; distinguish malformed ID, not found, and retrieval failure. |
| `POST /api/posts` | Authenticated + CSRF | Create and immediately publish a post for the session user. Ignore/reject any client author ID. Return success only after commit. |
| `PUT /api/posts/:id` | Authenticated + CSRF + owner | Validate and update title/content only; preserve author and creation time; update `updated_at`. |
| `DELETE /api/posts/:id` | Authenticated + CSRF + owner | Permanently delete exactly one owned post after the UI's confirmation. |
| `GET /api/me/posts?page=1` | Authenticated | Return only the session user's posts, newest first, 10 items per page, with publication and last-updated timestamps plus pagination metadata. No user ID query parameter is accepted. |

Feed and My Blogs pagination are intentionally different: the public feed is complete and unpaginated, while My Blogs uses 10-item pages. Validate page values as positive bounded integers and define out-of-range behavior consistently (recommended: return an empty item page with valid pagination metadata).

### DTO and error format

Use stable JSON shapes. For example:

```json
{
  "error": {
    "code": "validation_failed",
    "message": "Check the highlighted fields.",
    "fields": { "title": "Title is required." },
    "requestId": "..."
  }
}
```

Never include SQL, stack traces, credentials, hashes, session material, or raw database errors in a response. Expected failures should have stable codes and useful user-facing messages; unexpected failures should return a generic message and request ID.

| HTTP status | Use |
|---|---|
| `400` | Malformed JSON, invalid identifier syntax, or invalid query parameter. |
| `401` | Missing/expired authentication, or generic invalid login credentials. |
| `403 Forbidden` | Valid authenticated user attempting to edit/delete a post owned by another user, as explicitly required by FR-017. |
| `404 Not Found` | Missing post or unknown API route. |
| `409` | Duplicate normalized registration email. |
| `413` | Request body exceeds the configured byte limit. |
| `422` | Well-formed request with field-level validation failures. |
| `429` | Login/registration rate limit reached, if enabled; include a safe retry hint. |
| `500` | Unexpected server or persistence failure, with generic response and request ID. |

The ownership distinction intentionally reveals that an existing post belongs to another account because FR-017 requires 403 for existing foreign posts and 404 for nonexistent posts. Do not “correct” it to a concealed 404 without revising the approved requirement.

## 7. Authentication, authorization, and security controls

### Passwords and registration

- Normalize email by trimming and lowercasing before validation/lookup; enforce a database unique constraint on `email_normalized` to make duplicate prevention race-safe.
- Enforce at least 8 password characters and no mandatory character classes. Enforce agreed length limits on the backend; do not trim or log passwords.
- Use Node's asynchronous `crypto.scrypt` with a unique cryptographically random salt per account and store an encoded algorithm/parameter/salt/hash string. Compare derived values in constant time. Benchmark and record chosen cost parameters on supported hardware.
- Registration and login errors must not leak database or implementation detail. Login uses one generic invalid-credentials response.

### Session lifecycle

- Use `express-session` with a cryptographically random session ID and a maintained SQLite-backed server-side store. The cookie contains only the signed session identifier, not user claims.
- Set cookie attributes explicitly: `Secure=true`, `HttpOnly=true`, `SameSite=Lax`, `Path=/`; omit `Domain` unless a future deployment requires it. Do not relax `Secure` in development.
- Use an idle timeout of 30 minutes and an absolute lifetime of 24 hours. Store session creation time server-side and check it on every authenticated request; set/touch store expiry for the idle limit. The earlier limit wins.
- Regenerate the session ID after successful login and registration to prevent session fixation. Issue a fresh CSRF token after rotation. On logout, destroy the server-side record and clear the cookie using matching attributes; a prior ID must no longer authenticate.
- Never use the Express default in-memory session store. Handle session-store errors explicitly; do not silently treat them as logged-out success or volatile success.

### CSRF, origin, and transport

- Use a synchronizer token bound to the server-side session. Obtain it from `GET /api/auth/csrf`, keep it in UI memory, and send it in a custom header for registration, login, logout, create, edit, and delete. Rotate it with the session.
- Verify the request `Origin` against an explicit configured same-origin allowlist on unsafe methods; reject unexpected origins. Check `Sec-Fetch-Site` where available as defense in depth. Do not configure wildcard CORS.
- `SameSite=Lax` is defense in depth, not the only CSRF control. Do not place CSRF tokens in URLs or logs.
- Use HTTPS for browser traffic in local development as well as any future deployment so the required Secure cookie works. For local development, configure Vite and API with a trusted local certificate (for example, a developer-installed local CA); do not use `Secure=false` or disable certificate checks to make cookies work.
- Set a restrictive Content Security Policy and other appropriate security headers. Because content is plain text and rendered as text nodes, no user-authored HTML rendering or sanitizer pipeline is needed.

### Authorization and data access

- Authentication middleware resolves the user ID from the server session; service functions receive this trusted ID explicitly.
- My Blogs queries always filter on the session user's ID. Do not support `/users/:id/posts` as an alternative ownership boundary.
- For edit/delete, use a transaction that first resolves the resource and owner and then mutates only if the owner matches. Return 404 when absent and 403 when present but owned by another user. The update SQL must modify only `title`, `content`, and `updated_at`; the delete SQL must target exactly one ID.
- Use parameterized SQL for every value. Restrict allowed sort and pagination values in code; never interpolate user input into SQL identifiers or clauses.
- Enforce request-body byte limits that accommodate the maximum 20,000 Unicode code points while bounding memory. A proposed JSON limit is 128 KiB; confirm it against the chosen encoding and parser in implementation.
- Consider a modest configurable rate limit on registration and login to reduce credential-stuffing and signup abuse; it must not reveal account existence. This is a defense proposal, not an additional product requirement.

### Secrets, logs, and errors

- Validate required configuration at process startup and fail clearly if missing or invalid. Keep session signing secrets and local database paths in environment configuration, not source control.
- Generate a high-entropy session secret per local environment; do not use example secrets outside documentation. Ignore local `.env` files, certificates/private keys, SQLite files, WAL/SHM files, and test databases.
- Redact cookies, session IDs, CSRF tokens, passwords, and password hashes from logs. Use structured logs with request ID and safe error category; avoid request-body logging.
- Log unexpected errors server-side without returning internal details. A failed commit returns an error, never a success response.

## 8. Database initialization and seed strategy

1. **Configuration:** Resolve the database path from validated configuration; default to a local ignored `data/blog.sqlite` path. Keep the path overrideable for tests.
2. **Migrations:** `db:migrate` creates the directory and database, enables foreign keys and the chosen journal mode, then applies ordered migrations in transactions. A schema version table records applied versions. The session adapter manages its own table and cleanup independently.
3. **Constraints/indexes:** Add primary keys, not-null constraints, unique normalized email, foreign key from posts to users, and the two post sort indexes. Verify constraints on every connection.
4. **Seed:** `db:seed` is an explicit development-only, idempotent command. Add a small set of clearly fictional published posts and a local demonstration user whose email/password are supplied through local environment variables or a safe interactive prompt. Hash its password through the normal password service. Never hard-code reusable credentials, print passwords, or run seed automatically at server startup. Refuse to seed when `NODE_ENV=production`.
5. **Tests:** Each integration suite uses a new temporary SQLite file, applies the same migrations, seeds only test fixtures, and removes its own temporary database after completion. Test restart persistence by closing and reopening the same file.
6. **Backups/reset:** Document that the database is local persistent state. Provide a deliberate reset command only if implemented with clear confirmation; never automatically erase a user database during startup or migration.

## 9. Test layers and acceptance mapping

All tests below are proposed. The repository currently has no code or tests, and no command in this document has been run.

| Layer | Scope | Required evidence |
|---|---|---|
| Unit | Shared schemas, email normalization, Unicode length checks, preview generation, password hash/verify behavior, session-expiry decision, response mapping, pagination calculations | Boundaries and invalid values; password plaintext never stored; no mandatory class rule; exact 120/20,000 limits; 200-character preview. |
| API integration | Express routes against a temporary migrated SQLite DB and real session middleware/store | Register/login/logout; generic invalid login; duplicate email ignoring case; CSRF rejection/acceptance; idle/absolute expiration; public feed/detail; create/edit/delete persistence; malformed IDs; 400/401/403/404/409/413/422/500 shapes; body failures. |
| Authorization/data integrity | Two registered users and owned/unowned/missing posts | Direct API calls cannot spoof author; My Blogs excludes other users; edit/delete return the required 403/404 distinction; denied operations leave records unchanged; deletion does not affect unrelated posts. |
| Frontend component | Form validation, loading/empty/error/success/not-found states, pagination, session switch/logout, delete confirmation, unsaved-edit warning | API failures never appear as empty results or success; user-provided content is rendered as text, not HTML. |
| End-to-end | Browser flows on HTTPS localhost | Register and auto-login; create and read; edit and preserve metadata; delete after confirmation; public browsing without login; 10-item My Blogs pagination; direct post URL; session logout/expiry and account isolation. |
| Build/dependency | Type-check/build, lockfile install, dependency review and audit | Repeatable `npm ci`; compatible pinned dependency tree; record audit findings and known limitations without claiming an unresolved finding is clear. |

Map these tests to the approved acceptance scenarios [AS-001–AS-014](./requirements.md#8-testable-acceptance-scenarios) and requirements FR-001–FR-022/NFR-001–NFR-009. At minimum, explicitly cover AS-002, AS-004, AS-008, AS-010, AS-011, AS-012, and AS-013 as negative/security-critical scenarios.

## 10. Proposed local setup and run commands

**These commands are proposals for implementation; no package manifests, scripts, lockfile, certificates, or application code exist yet. They have not been run.** Names assume npm workspace names `@blog/api` and `@blog/web`; update them if implementation chooses another convention.

1. Install Node.js 22 LTS and npm; install/configure a trusted local HTTPS certificate for `localhost` for both Vite and the API. Keep certificate private keys outside version control.
2. From the repository root, install dependencies:

   ```powershell
   npm.cmd install
   ```

   Commit the generated lockfile with the implementation and use `npm.cmd ci` for repeatable clean installs thereafter.
3. Set local secrets/configuration in an ignored environment file or process environment; at minimum provide a high-entropy `SESSION_SECRET`, HTTPS certificate/key paths, and an ignored SQLite data path. Never copy a production secret into local development.
4. Initialize and seed local data:

   ```powershell
   npm.cmd run db:migrate --workspace @blog/api
   npm.cmd run db:seed --workspace @blog/api
   ```

   The seed account credentials must be supplied locally, not embedded in the command or repository.
5. Start the API and Vite development server in separate terminals:

   ```powershell
   npm.cmd run dev --workspace @blog/api
   npm.cmd run dev --workspace @blog/web
   ```

   Open the configured HTTPS Vite localhost URL; its `/api` proxy targets the HTTPS API. A built local run should serve the Vite build and API from one origin.
6. Proposed checks:

   ```powershell
   npm.cmd test
   npm.cmd run build
   npm.cmd audit
   ```

`npm audit` reports dependency advisories and is not itself proof of dependency safety. Review each finding and record unresolved issues. Implementers must create the scripts before using these commands; do not report them as working until verified.

## 11. Decisions, assumptions, and risks for review

### Recommended architecture decisions

- **A-001 — Stack:** React/Vite/TypeScript SPA, Express/TypeScript API, SQLite, and npm workspaces on Node 22 LTS.
- **A-002 — Same origin:** Serve built UI and API on one origin; use a Vite `/api` proxy in development; no permissive CORS.
- **A-003 — Data model:** SQLite `users` and `posts` tables with foreign keys, normalized unique email, and immutable post author/creation metadata.
- **A-004 — Password hashing:** Node built-in scrypt with unique salt; benchmark cost parameters before implementation.
- **A-005 — Session/CSRF:** Server-side SQLite sessions, Secure/HttpOnly/SameSite=Lax cookie, 30-minute idle and 24-hour absolute expiry, session rotation, and synchronizer-token plus Origin checks.
- **A-006 — Rendering:** Store/render plain text only; React text nodes; preview first 200 Unicode code points.
- **A-007 — Pagination:** Complete public feed; My Blogs uses page-number pagination of 10 posts, newest first, deterministic `id` tie-break.
- **A-008 — Local data:** Versioned SQLite migrations, explicit development-only idempotent seed, no automatic destructive reset.

These are architecture recommendations aligned to the approved requirements. Human review should confirm the stack and security choices before implementation begins.

### Assumptions and implementation details

- The application is a single-process, local demonstration; no cloud deployment, external database, object storage, email service, or administrator workflow is required.
- One origin is used for browser UI and API; local development uses HTTPS and trusted localhost certificates to preserve Secure-cookie behavior.
- Store timestamps at fixed UTC precision and use UUID ordering as a stable tie-break for equal creation times.
- Enforce lengths using Unicode code points (not UTF-16 code units) consistently in shared UI validation and API validation. This resolves the remaining character-count detail in the architecture recommendation; backend remains authoritative.
- Use a stable error envelope and the HTTP mappings in Section 6; field-validation details may differ in exact wording but not in safe/accurate outcome.

### Risks and mitigations

| Risk | Mitigation |
|---|---|
| A native SQLite dependency may lack a Node 22 Windows binary or require build tools. | Prototype install on the supported Windows runtime before approval of exact package versions; commit lockfile; document prerequisites; do not substitute volatile in-memory storage. |
| Secure cookies complicate local HTTP-only development. | Use trusted HTTPS localhost certificates and test actual browser cookie flags; never add a development `Secure=false` exception. |
| Session adapters vary in maintenance and absolute-expiry support. | Select a maintained compatible SQLite store, retain the application's explicit absolute-expiry check, test store failure/expiry/rotation, and reject `MemoryStore`. |
| CSRF or session rotation can break login/registration flows if token issuance is stale. | Integration-test anonymous token acquisition, registration/login session regeneration, token refresh, logout invalidation, and rejected cross-origin mutation. |
| A complete unpaginated feed can grow without a numeric performance target. | Keep the mandated complete feed; index sort keys and measure with realistic seed data. Revisit pagination only with approved requirements change. |
| 20,000 Unicode code points can exceed naive JSON byte assumptions. | Bound request bytes separately from field character counts, test multi-byte text at the boundary, and adjust the proposed 128 KiB parser limit if required. |
| SQLite is single-writer and the demo has no numeric availability target. | Keep transactions short, configure busy timeout, test concurrent writes, and document local single-instance scope. |
| Email normalization and session secrets can be mishandled through logs/config. | Normalize consistently, enforce uniqueness in the DB, validate configuration at startup, redact sensitive fields, and add secret-scanning to verification. |

## 12. Requirements traceability

| Architecture element | Requirements addressed |
|---|---|
| React/Vite pages and state handling | FR-007–011, FR-015, FR-020–022; NFR-004 |
| Shared schemas plus authoritative API validation | FR-001–003, FR-011–014, FR-017, FR-019; NFR-001, NFR-003 |
| Scrypt and registration/login flows | FR-001–003; AS-001–003; NFR-001 |
| SQLite server-side sessions, cookie flags, expiry, rotation, logout | FR-004–006; AS-003–004; NFR-001 |
| CSRF and Origin checks | FR-004–006, FR-013–017; NFR-001 |
| Post schema, transactions, constraints, migrations, persistence | FR-012–018; AS-007–011; NFR-003, NFR-007 |
| Ownership-filtered service/repository operations | FR-013–014, FR-016–020; AS-008–012; NFR-001–002 |
| Plain-text rendering and limited response DTOs | FR-007, FR-009–010, FR-013–014; AS-005–006, AS-013; NFR-001–002 |
| Explicit API errors, stateful UI states, request IDs | FR-003, FR-006–007, FR-011, FR-013–022; AS-005–013; NFR-003–004 |
| Unit/integration/E2E and dependency checks | FR-018–022; AS-001–014; NFR-006, NFR-008 |
| Local migration, seed, run, and test workflow | NFR-005–008 |

## 13. Review boundary

This document records the approved architecture, not an implementation report. It does not create application source, install dependencies, run migrations, seed data, execute tests, or claim that the proposed commands work. Proposed stack details and local commands remain subject to validation during implementation.
