---
description: Security conventions for the Blog Platform.
applyTo: "**/*.{js,jsx,ts,tsx,mjs,cjs,sql}"
---

- Never store passwords in plain text; use a suitable salted password-hashing algorithm.
- Enforce authentication and authorization on the server for every protected operation.
- Derive identity from the authenticated session; never trust client-supplied author/user IDs for authorization.
- Check ownership for every edit and delete operation.
- Validate inputs on the server and use safe parameterized database access.
- Render user-generated content as text or sanitize it with a suitable maintained sanitizer; do not inject untrusted HTML.
- Protect cookie-authenticated state-changing operations against CSRF.
- Rotate/regenerate session identifiers after authentication and invalidate sessions on logout.
- Keep secrets in environment variables or a suitable secret store; never commit secrets or log credentials/session tokens.
- Return safe error messages without stack traces or sensitive implementation details.
- Add tests for unauthenticated access and cross-user authorization failures.
