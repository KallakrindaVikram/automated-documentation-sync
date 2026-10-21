# Blog Platform — Copilot SDLC Agent Starter

This package contains eight specialist GitHub Copilot agents. The orchestrator is intentionally not included yet; first test each specialist independently.

## Install
1. Extract this ZIP into the root of `automated-documentation-sync`, alongside `README.md` and `.git`.
2. Review the `.github` files and check Git status before committing.
3. Open Copilot Chat in VS Code and select the `requirements` agent.
4. Run `.github/prompts/test-requirements-agent.prompt.md` as a starting test.

The agents only have access to Confluence if the configured MCP tools are available in that Copilot session. They do not automatically commit, push, merge, deploy, or create PRs. Keep test runs that modify files on disposable branches. Do not commit secrets or real user data.

## Test order
Requirements → Architecture → Design Review → Implementation Planning → Implementation → Code Review → Verification → Release/PR Preparation.
