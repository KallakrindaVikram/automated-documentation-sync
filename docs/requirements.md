# Blog Platform Requirements Baseline

**Status:** Approved baseline

**Discovery date:** 2026-10-21

**Source of truth:** The eight linked Confluence pages below and the decisions recorded in this draft.

**Approval:** Explicitly approved by the user after the content-length limit was revised to 20,000 characters.

## 1. Purpose and scope

The Blog Platform is a demonstration web application for account registration and authentication, public discovery and reading of published posts, and authenticated users' management of their own posts. It comprises a web frontend, backend API, and persistent data store; technology choices are deferred to architecture.

The initial release includes:

- Registration, login, logout, and defined session behavior.
- A public feed and public individual-post reading.
- Authenticated creation, editing, and permanent deletion of a user's own posts.
- A private, paginated list of the authenticated user's posts.
- Server-side validation and authorization, safe content rendering, clear operation outcomes, and automated unit and integration testing.

No administrator role is in scope.

## 2. Roles and permissions

| Role | Permissions |
|---|---|
| Visitor (not authenticated) | Browse the public feed and read published posts. Register and log in. Cannot use protected post-management features. |
| Registered user (authenticated) | All visitor permissions; create posts; view only their own posts in My Blogs; edit or delete only posts they own; log out. |

The backend is authoritative for identity and ownership. Client-provided user or author identifiers must never grant access or change ownership.

## 3. Functional requirements

### 3.1 Authentication and sessions

- **FR-001 — Register:** Provide registration fields for display name, email, and password. Reject missing fields, invalid email formats, and an email already registered without regard to letter case. Display names are not required to be unique. A successful registration automatically signs in the new user.
- **FR-002 — Password policy and storage:** Require a password of at least 8 characters, with no mandatory character-class rules. Store passwords only as salted hashes using an appropriate password-hashing algorithm. Never store or log plaintext passwords.
- **FR-003 — Login:** Authenticate using email and password. Missing values receive validation feedback. Invalid credentials fail without creating an authenticated session, and the response does not reveal whether the email is registered.
- **FR-004 — Session establishment and identity:** Use server-side sessions. Session cookies must have the `Secure`, `HttpOnly`, and `SameSite` attributes configured. Derive the current user from the authenticated session for every protected operation; never trust a client-supplied user ID.
- **FR-005 — Session expiration:** Expire a session after 30 minutes of inactivity or 24 hours after creation, whichever occurs first. Reject protected requests after expiry and ask the user to authenticate again where appropriate.
- **FR-006 — Logout:** End and invalidate the current session. Protected requests using a logged-out session are denied. Repeated logout is handled safely, and the UI navigates to a suitable public or login page with appropriate feedback.

### 3.2 Public discovery and reading

- **FR-007 — Public feed:** Allow visitors and authenticated users to browse published posts without signing in. Display each post's title, author's public display name, publication date, and a preview consisting of the first 200 characters of plain-text content. Show distinct loading, successful-empty, and retrieval-error states; a retrieval failure must not appear as an empty feed.
- **FR-008 — Feed order and size:** Display the complete feed without pagination, newest first, using a deterministic tie-break when publication times match.
- **FR-009 — Read a post:** Allow direct navigation to a published post URL. Show its title, full content, public author display name, and publication date. Provide a route back to the feed.
- **FR-010 — Safe public data:** Do not expose password hashes, session data, or private account information through feed or post responses.
- **FR-011 — Missing and invalid post:** A nonexistent post returns an appropriate Not Found result and a helpful UI message. A malformed identifier is handled without an unhandled exception. Not Found is distinct from retrieval or server failure.

### 3.3 Create, edit, and delete

- **FR-012 — Create and publish:** Authenticated users can submit a title and content. Both are required; leading and trailing whitespace is trimmed before persistence, and whitespace-only values are rejected. Enforce a maximum title length of 120 characters and content length of 20,000 characters on the backend. Creating a post publishes it immediately; drafts are not included.
- **FR-013 — Create ownership and confirmation:** The backend associates a new post with the authenticated user, ignoring any author identity supplied by the client. Confirm publication and provide a way to view the post only after persistence succeeds. Prevent accidental repeat submission while a request is in progress and avoid unintended duplicates on retry.
- **FR-014 — Edit owned post:** An authenticated owner can load the current title and content and update them. Apply FR-012's required-field, whitespace, and length rules. Update the last-updated timestamp; preserve the original author and creation timestamp. The client cannot change the author. After a successful edit, navigate to the post detail.
- **FR-015 — Warn about unsaved edits:** Warn the user before they leave an edit form with unsaved changes.
- **FR-016 — Delete owned post:** An authenticated owner can request deletion only after confirming in a native confirmation dialog. Deletion is permanent at the application level. A cancelled deletion leaves the post unchanged. Report success only after the backend confirms persistence; after successful deletion, return to My Blogs.
- **FR-017 — Ownership and resource responses:** Enforce ownership on every backend edit and delete request, regardless of frontend controls. Return **403 Forbidden** when the post exists but belongs to another user; return **404 Not Found** when the post does not exist. Denied or failed operations leave data unchanged.
- **FR-018 — Persistence and data integrity:** Successfully created and edited posts survive application restarts. Successfully deleted posts are no longer available through the feed, detail route, or My Blogs. Failed writes do not leave partially created or modified records or produce success-shaped UI.

### 3.4 My Blogs and navigation

- **FR-019 — My Blogs access and isolation:** Require authentication for My Blogs. Return only posts belonging to the session's authenticated user; ignore client-supplied user identifiers. A user's posts must not appear in another user's list, including after switching accounts.
- **FR-020 — My Blogs display:** Sort newest first with a deterministic tie-break, paginate at 10 posts per page, and display publication and last-updated dates. Provide ways to create, open, edit, and initiate deletion of the user's own posts.
- **FR-021 — Empty, loading, and retrieval states:** Show a helpful empty state with a create-post path when the request succeeds with no posts. Show loading while data is fetched. Show an error and retry option on retrieval failure; never misrepresent a failed request as an empty result.
- **FR-022 — Navigation and mutation feedback:** Provide clear navigation between the feed, authentication pages, post detail, post creation, and My Blogs. After successful creation or editing, navigate to post detail; after successful deletion, return to My Blogs. Display success feedback only after successful persistence. Unknown routes show an appropriate not-found page or message.

## 4. Validation, errors, and response behavior

- Validate all important fields, request bodies, and identifiers on the backend even when the frontend performs validation.
- Reject missing required fields, invalid email formats, whitespace-only post fields, and values over the agreed post-length limits. Identify invalid fields where practical and give actionable feedback.
- Enforce case-insensitive uniqueness for email addresses without creating duplicate accounts.
- Use a generic login failure for invalid credentials; do not disclose account existence.
- Distinguish validation failures, authentication failures, authorization failures, missing resources, loading, empty results, and unexpected server/network failures as applicable.
- Do not expose passwords, password hashes, session secrets, database details, SQL statements, stack traces, or other sensitive implementation details in responses or ordinary logs.
- A failed create, edit, or delete is never reported as successful. Failed edits preserve the saved content; failed deletes leave the post available.
- Use safe parameterized queries or an equivalent safe data-access mechanism. Handle malformed requests without crashing.

## 5. Data and content rules

At minimum, a user record includes a unique identifier, display name, case-insensitively unique email, and password hash. A post includes a unique identifier, title, full content, author reference, creation timestamp, and last-updated timestamp. Posts are published immediately; no draft workflow is required.

Persist timestamps in UTC and display them in the viewer's local timezone using a consistent presentation. Obtain public author names from the associated user record. Render all user-provided title and content safely so it cannot execute arbitrary HTML or JavaScript.

The exact database schema and timestamp representation remain architecture decisions, subject to these requirements.

## 6. Non-functional requirements

- **NFR-001 — Security:** Apply backend authentication and authorization; securely hash passwords; protect session cookies; validate server-side; prevent unsafe database access and script execution; protect secrets; and avoid sensitive information in errors and logs.
- **NFR-002 — Privacy:** Expose only public author display names on public pages. Keep My Blogs limited to the authenticated user's content. Never return authentication secrets or unrelated private account information.
- **NFR-003 — Reliability and integrity:** Handle missing records, invalid input, failed requests, and persistence errors gracefully. Successful writes persist, and failed operations do not leave partial changes or misleading UI state.
- **NFR-004 — Usability:** Provide clear navigation, validation messages, and relevant loading, empty, success, not-found, and error states. Ask for confirmation before deletion.
- **NFR-005 — Maintainability:** Separate frontend, backend, and persistence responsibilities clearly. Avoid unnecessary duplication of shared validation and authorization logic.
- **NFR-006 — Testability:** Core logic and API behavior must be testable through automated tests, including successful and important failure cases. Document commands needed to build and test the application.
- **NFR-007 — Local development:** Make the application straightforward to install, initialize, seed with demonstration data, run, and test using documented commands.
- **NFR-008 — Dependency safety:** Select reputable dependencies with compatible versions and available vulnerability checks. Record unresolved dependency findings during verification.
- **NFR-009 — Performance:** No numeric performance SLA is required for the initial demonstration release. The complete, unpaginated public feed and 10-item My Blogs pagination behavior remain functional requirements.

## 7. Out of scope

The initial release excludes comments and replies; likes, follows, and social feeds; image uploads; rich-text editing; drafts and scheduled publishing; password reset and social login; administrator dashboards and role management; user profile changes; post recovery, trash, version history, and collaborative editing; cloud deployment; and external AI integrations. Any scope expansion requires explicit agreement before implementation.

## 8. Testable acceptance scenarios

These scenarios are the minimum outcome-oriented acceptance set. Detailed unit and integration tests may split them further.

| ID | Given / When / Then | Requirement and source trace |
|---|---|---|
| **AS-001 Registration** | Given valid new-user details, when the visitor registers, then an account is created with a salted password hash and the user is signed in. | FR-001–002; Authentication US-001 AC-01, AC-05, AC-07 |
| **AS-002 Registration rejection** | Given missing fields, malformed email, a case-insensitive duplicate email, or a password under 8 characters, when registration is submitted, then the backend rejects it with useful safe feedback and creates no account. | FR-001–002; Authentication US-001 AC-02–06, AC-08–10 |
| **AS-003 Login** | Given correct credentials, when the user logs in, then a server-side session is established. Given invalid or missing credentials, then login fails, no session is established, and invalid credentials do not reveal account existence. | FR-003–004; Authentication US-002 AC-01–08 |
| **AS-004 Logout and expiry** | Given an authenticated session, when the user logs out, then it is invalidated. Given 30 minutes idle or 24 hours elapsed, whichever comes first, protected access is denied and reauthentication is required. | FR-005–006; Authentication US-003 AC-01–06; US-004 AC-01–06 |
| **AS-005 Public feed** | Given published posts, when a visitor opens the feed, then all are listed newest first with deterministic tie-breaks and the required title, public author, publication date, and 200-character plain-text preview. Given no posts, show an empty state; given a retrieval failure, show an error instead. | FR-007–008, FR-010; Discovery US-005 AC-01–10 |
| **AS-006 Read and missing post** | Given a published post, when a visitor opens it directly or from the feed, then its required details display and content is safe to render. Given a missing or malformed ID, show an appropriate not-found/validation outcome; distinguish this from retrieval failure. | FR-009–011; Discovery US-006 AC-01–09, US-007 AC-01–05 |
| **AS-007 Create post** | Given an authenticated user and valid title/content within limits, when the user submits, then a post is persisted, immediately published under that user, and the user receives a success confirmation and link to its detail page. | FR-012–013, FR-018, FR-022; Create US-008 AC-01–15, US-010 AC-01, AC-05–07 |
| **AS-008 Reject invalid create** | Given missing, whitespace-only, or over-limit content, an unauthenticated request, or a spoofed author ID, when create is attempted, then the backend rejects it without a record, identity spoofing, or false success. | FR-012–013; Create US-009 AC-01–08, US-010 AC-02–07 |
| **AS-009 Edit own post** | Given an authenticated owner, when they load and save valid edits, then the updated content persists, the last-updated timestamp changes, original author/creation time remain unchanged, and the user returns to post detail. Warn before discarding unsaved changes. | FR-014–015, FR-018, FR-022; Edit US-011 AC-01–08, US-012 AC-01, US-013 AC-01–09 |
| **AS-010 Enforce edit/delete ownership** | Given an existing post owned by another user, when an edit or delete is attempted, then return 403 and leave it unchanged. Given a nonexistent post, return 404. An unauthenticated request is denied. | FR-017; Edit US-011 AC-09–12, Authorization and Data Integrity; Delete US-014 AC-07–09, US-015 AC-04–05 |
| **AS-011 Delete** | Given an owner requests deletion, when they cancel native confirmation, then the post remains. When they confirm and persistence succeeds, deletion is permanent, success is reported, and the user returns to My Blogs. A persistence failure does not report success or hide the post permanently. | FR-016–018, FR-022; Delete US-014 AC-01–14, US-015 AC-01–08 |
| **AS-012 My Blogs** | Given an authenticated user, when they open My Blogs, then only their posts are shown newest first, 10 per page, with publication and last-updated dates. Empty, loading, and failed-retrieval states are distinct. Successful mutations appear after refresh; account switching does not expose previous-user data. | FR-019–021; My Blogs US-016 AC-01–12, US-017 AC-01–07 |
| **AS-013 Safe failure and rendering** | Given malformed input, unexpected API/database failure, or user-provided script-like content, when the application processes or displays it, then it handles errors without crashing or leaking internals, never reports failed writes as successful, and does not execute the content. | FR-010–011, FR-018, validation/error behavior, NFR-001–003; Security and Quality US-018 AC-01–10, US-019 AC-01–09, US-020 AC-01–08 |
| **AS-014 Quality evidence** | Given a candidate implementation, when its documented build and automated test commands run, then unit, integration, negative, authentication, persistence, and ownership behavior have recorded results; limitations and untested behavior are disclosed. | NFR-006–008; Security and Quality US-021 AC-01–09 |

## 9. Decision log

All proposals below were submitted for confirmation; the recorded answer is the user's decision, not an unapproved assumption.

| ID | Question and proposal/options | User answer | Resulting requirement | Traceability |
|---|---|---|---|---|
| D-01 | Password minimum/complexity; proposal was 12 characters without character-class rules. | Minimum 8 characters; no mandatory character-class rules. | FR-002 sets the password rule. | Authentication US-001 AC-06 |
| D-02 | Session mechanism; proposal was server-side sessions with secure cookie attributes. | Server-side sessions with Secure, HttpOnly, SameSite cookie settings and documented expiration. | FR-004 sets mechanism and attributes; D-03 sets expiry. | Authentication US-002 AC-05–07; US-004 AC-02–03 |
| D-03 | Session lifetime; proposal was 30-minute idle and 24-hour absolute expiry. | Accepted proposal. | FR-005 expires at the first of those limits. | Authentication US-004 AC-02–03 |
| D-04 | Auto-login, display-name uniqueness, and profile changes; proposal was no auto-login, non-unique names, no profile changes. | Auto-login; case-insensitive unique email; non-unique display names; profile changes out of scope. | FR-001; profile changes excluded. | Authentication US-001 AC-02, AC-05; Decisions Requiring Requirements Discovery |
| D-05 | Post limits and whitespace; initial proposal was title 120, content 10,000, trim outer whitespace. | Initially accepted the proposal, then during approval requested a 20,000-character content limit instead. | FR-012 sets title 120 and content 20,000; FR-014 applies the same limits to edits; outer whitespace is trimmed and blank values are rejected. | Create US-008 AC-08; US-009 AC-02–04; Edit US-011 AC-05–06 |
| D-06 | Feed order, pagination, preview; proposal was newest first, complete list, first 200 plain-text characters. | Accepted proposal. | FR-007–008 define feed behavior and deterministic tie-break. | Discovery US-005 AC-02, AC-09 |
| D-07 | Timestamp storage/display; proposal was UTC storage and viewer-local display. | Accepted proposal. | Section 5 requires UTC persistence and local display. | Discovery Data and Display Rules; Create Data Requirements |
| D-08 | Create/edit destination and unsaved edits; proposal was post detail after create/edit and warn before discarding. | Accepted proposal. | FR-014–015 and FR-022 set destinations and warning behavior. | Create US-010 AC-01; Edit US-013 AC-05–06 |
| D-09 | Response for another user's existing post; options were 403 or concealed 404. | 403 for another user's existing post; 404 if nonexistent. | FR-017 sets these distinct responses. | Edit Authorization and Data Integrity; Delete US-014 AC-07–09 |
| D-10 | Deletion policy/destination/dialog; proposal was permanent deletion, My Blogs, native confirmation. | Accepted proposal. | FR-016 and FR-022 set deletion behavior. | Delete Initial Deletion Policy; US-014 AC-04–06 |
| D-11 | My Blogs order, pagination, dates, mutation feedback; proposal was newest first, unpaginated, show dates. | Newest first; 10 per page; show publication and last-updated dates; explicit mutation success feedback. | FR-020–022 set list behavior and truthful feedback. | My Blogs US-016 AC-04, AC-08–10; US-017 |
| D-12 | Numeric performance target; proposal was no numeric SLA for demo. | No numeric performance SLA; qualitative NFRs remain. | NFR-009 sets this boundary. | Product Requirements §8; Security and Quality §6 |
| D-13 | Initial out-of-scope list; proposal was to confirm all listed exclusions and no profile edits. | Confirmed all listed exclusions; no profile editing. | Section 7 defines initial exclusions. | Product Requirements §§3, 7; Security and Quality §7 |

## 10. Constraints, approved decisions, and remaining risks

### Constraints and approved decisions

- Keep the initial application simple and suitable for a demonstration; technology stack and schema are architecture decisions.
- The security and ownership requirements must be enforced on the backend, not solely by UI behavior.
- The product must be testable locally and have documented build/test commands.
- Decisions D-01 through D-13 above are approved discovery decisions, subject to approval of this overall baseline.

### Approved assumptions

No additional assumptions have been approved. Decisions D-01 through D-13 are explicit user decisions; the non-blocking design details below remain open rather than being silently assumed.

### Remaining design risks / non-blocking decisions

These details are not specified by the source pages or user decisions and must be resolved and documented by architecture/implementation without weakening this baseline:

- Exact email-format validation and canonicalization implementation, including consistency with case-insensitive uniqueness.
- Exact password-hashing algorithm and parameters; cookie `SameSite` value, path/domain scope, CSRF protections, and secure local-development handling.
- Exact Unicode character-counting rule for the 120-character title and 20,000-character content limits, and the deterministic secondary sort key for timestamp ties.
- Exact preview extraction/normalization rule for plain-text content and date/time presentation format.
- Exact validation and authentication HTTP status mappings where this baseline does not specify one. Ownership responses are explicitly 403/404 as defined in FR-017.
- Pagination navigation details for My Blogs and an appropriate feed size/latency as no numeric performance target is required.

Any product behavior change or scope expansion must be brought back for explicit approval; technical choices must be recorded in the architecture artifact.

## 11. Traceability to Confluence sources

The source pages were retrieved in full during discovery. The story/acceptance-criterion references in Sections 3 and 8 trace requirements to their source.

| Source | Linked page | Main trace coverage |
|---|---|---|
| Product overview and scope | [Blog Platform — Product Requirements](https://vikram-epam-practice.atlassian.net/wiki/spaces/ghcc/pages/21692417/Blog+Platform+Product+Requirements) | Product goals, roles, functional scope, security/data integrity, UX, exclusions, NFRs, delivery constraints |
| Authentication | [01 — Authentication](https://vikram-epam-practice.atlassian.net/wiki/spaces/ghcc/pages/21790721/01+Authentication+Registration+login+logout+session+behavior) | US-001–004: registration, login, logout, session behavior, authorization, validation, tests |
| Discovery | [02 — Blog Discovery](https://vikram-epam-practice.atlassian.net/wiki/spaces/ghcc/pages/21692426/02+Blog+Discovery+Blog+feed+individual+post+details+missing+posts) | US-005–007: public feed, post reading, navigation, missing/invalid resources, safe rendering |
| Creation | [03 — Create Blog Posts](https://vikram-epam-practice.atlassian.net/wiki/spaces/ghcc/pages/21987329/03+Create+Blog+Posts+Writing+validation+saving+and+publishing) | US-008–010: create/publish, validation, persistence, outcomes |
| Editing | [04 — Edit Blog Posts](https://vikram-epam-practice.atlassian.net/wiki/spaces/ghcc/pages/22020097/04+Edit+Blog+Posts+Editing+posts+and+enforcing+ownership) | US-011–013: edit, ownership, loading, validation, persistence, outcomes |
| Deletion | [05 — Delete Blog Posts](https://vikram-epam-practice.atlassian.net/wiki/spaces/ghcc/pages/21889026/05+Delete+Blog+Posts+Deletion+confirmation+ownership+restrictions) | US-014–015: confirmation, permanent deletion, ownership, outcomes |
| My Blogs | [06 — My Blogs](https://vikram-epam-practice.atlassian.net/wiki/spaces/ghcc/pages/22052865/06+My+Blogs+Viewing+your+own+posts+and+empty+states) | US-016–017: ownership-filtered list, empty/loading/error states, refresh, account isolation |
| Cross-cutting security and quality | [07 — Security and Quality](https://vikram-epam-practice.atlassian.net/wiki/spaces/ghcc/pages/22085633/07+Security+and+Quality+Authorization+validation+error+handling+tests+and+non-functional+requirements) | US-018–021: security, feedback, data integrity, quality evidence, NFRs, exclusions |

## 12. Baseline review and approval

**Approval status: Approved.** The user explicitly approved the baseline and requested the content-length limit be raised to 20,000 characters; that change is incorporated above.
