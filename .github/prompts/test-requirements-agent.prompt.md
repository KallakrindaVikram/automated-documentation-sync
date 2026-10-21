---
name: test-requirements-agent
description: Independently test requirements discovery against the published Confluence pages.
agent: requirements
---

Start requirements discovery. Retrieve all eight Blog Platform Confluence pages through the configured MCP connection, identify unresolved decisions, and ask clarification questions. Do not write `docs/requirements.md` until the blocking questions have been answered and the user has reviewed the proposed baseline. Do not modify Confluence, commit, push, merge, or create a PR.
