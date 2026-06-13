# Using AI Coding Agents with API Platform

AI coding agents — Claude Code, Cursor, GitHub Copilot, OpenAI Codex, Gemini, and others — are
trained on snapshots of public code. Those snapshots lag behind the framework: an agent may suggest
the `#[ApiFilter]` attribute, legacy controllers, or a serialization pattern that API Platform 4.x
has since replaced. The result is plausible-looking code that does not match the current canonical
way of doing things.

This guide describes two complementary ways to keep an agent on current API Platform 4.x APIs while
it works in your project:

- The [API Platform skillset](#claude-code-the-api-platform-skillset), a Claude Code plugin that
  teaches Claude the canonical patterns, verified against `api-platform/core`.
- An agent-agnostic [`AGENTS.md`](#other-agents-cursor-github-copilot-openai-codex-gemini) file that
  points any agent at the official documentation as the source of truth.

> **Note:** This guide covers _dev-time_ guidance — keeping the agent that writes your code on
> current APIs. Exposing your running API _to_ AI agents as callable tools is a different concern,
> covered by the [MCP integration](mcp.md).

## Claude Code: the API Platform skillset

The [API Platform skillset](https://github.com/api-platform/skillset) is a
[Claude Code](https://docs.claude.com/en/docs/claude-code/overview) plugin that ships 15 skills for
API Platform 4.x development. Each skill teaches the current canonical way to do one thing, verified
against `api-platform/core`, and covers both the Symfony and Laravel integrations.

A skill is more than a documentation link: it is a focused set of instructions, code patterns, and
constraints that Claude loads on demand. When you ask Claude to add a filter, the `api-filter` skill
loads and steers it toward `QueryParameter` rather than the legacy `#[ApiFilter]` attribute.

### Installing the skillset

The plugin is distributed through a Claude Code marketplace. Add the marketplace, then install the
plugin:

```console
/plugin marketplace add api-platform/skillset
/plugin install api-platform@api-platform-skillset
```

These are [Claude Code slash commands](https://docs.claude.com/en/docs/claude-code/slash-commands) —
run them from inside a Claude Code session, not from your shell.

### How skills load

Skills are namespaced as `api-platform:<skill>` (for example `api-platform:api-filter`). You do not
invoke them manually: Claude loads a skill automatically when the task at hand is relevant to it. No
configuration is required once the plugin is installed.

### Available skills

| Skill                  | Covers                                                                                       |
| ---------------------- | -------------------------------------------------------------------------------------------- |
| `api-resource`         | Resources, DTOs, Object Mapper, nested sub-resources, custom operations                      |
| `api-filter`           | Collection filters with `QueryParameter`, legacy `#[ApiFilter]` migration                    |
| `state-provider`       | Custom read logic, decorating Doctrine providers, computed fields                            |
| `state-processor`      | Custom write logic, soft-delete, file downloads, side effects                                |
| `operations`           | Operation security expressions, validation groups, parameter validation, deprecation         |
| `securing-collections` | Multi-tenant isolation with Doctrine extensions and link handlers                            |
| `custom-validator`     | Custom validation constraints for business rules                                             |
| `serialization-groups` | Serialization contexts and `#[Groups]`, with guidance on when DTOs are better                |
| `pagination`           | Page-based, partial, and cursor pagination                                                   |
| `errors`               | RFC 7807 Problem Details, `#[ErrorResource]`, exception-to-status mapping                    |
| `graphql`              | GraphQL operations, resolvers, Relay pagination                                              |
| `mercure`              | Real-time updates over Mercure (Symfony only)                                                |
| `api-platform-mcp`     | Exposing resources to AI agents via MCP: `#[McpTool]`, `McpToolCollection`, `#[McpResource]` |
| `api-docs`             | OpenAPI customization, hiding operations, factory decoration                                 |
| `api-test`             | Functional tests with `ApiTestCase` (Symfony) and HTTP tests (Laravel)                       |

### Keeping the skillset up to date

The canonical patterns evolve with each release. Update the marketplace to pull the latest skills:

```console
/plugin marketplace update api-platform-skillset
```

## Other agents: Cursor, GitHub Copilot, OpenAI Codex, Gemini

The skillset is a Claude Code plugin: its `SKILL.md` plus `.claude-plugin/` marketplace format is
specific to Claude Code, so it does not load into other agents. For Cursor, GitHub Copilot, OpenAI
Codex, Gemini, and any other agent, the portable option is an `AGENTS.md` file at the root of your
project.

[`AGENTS.md`](https://agents.md/) is a convention — a plain Markdown file that most coding agents
read for project-specific instructions. It is lighter-weight than the Claude skillset: it does not
ship verified code patterns, it points the agent at where the current truth lives. The single most
useful instruction is to treat the official documentation as authoritative over training data:

```markdown
# AGENTS.md

This project uses [API Platform](https://api-platform.com) 4.x.

The official documentation at <https://api-platform.com/docs> is the source of truth. Prefer it over
patterns from your training data, which may describe older versions of the framework.

In particular:

- Declare collection filters with `QueryParameter`, not the legacy `#[ApiFilter]` attribute.
- Read and write logic belongs in state providers and state processors, not in custom controllers.
- Check the documentation for the current canonical pattern before generating API Platform code.
```

Tailor the bullet list to the conventions your project cares about. The point is to redirect the
agent to current documentation rather than letting it rely on stale training data.

Claude Code also reads a `CLAUDE.md` file. If you maintain both an `AGENTS.md` and use Claude Code,
keep a thin `CLAUDE.md` that imports the shared file rather than duplicating its content:

```markdown
# CLAUDE.md

@AGENTS.md
```

This keeps a single source of project guidance for every agent. For Claude Code specifically, the
[skillset](#claude-code-the-api-platform-skillset) remains the richer option and complements
`AGENTS.md`.

## Related: exposing your API to agents with MCP

This guide is about the agent that _writes_ your code. A separate concern is letting an AI agent
_call_ your running API at runtime — searching your books, creating an order, reading a resource.
That is what the [Model Context Protocol (MCP) integration](mcp.md) provides: it turns your API
Platform resources into MCP tools and resources an agent can discover and invoke.

The two are independent and can be used together:

- The skillset and `AGENTS.md` shape the code an agent produces during development.
- MCP exposes your deployed API as callable tools at runtime.

The `api-platform-mcp` skill in the skillset covers writing those MCP tool definitions; see
[Exposing Your API to AI Agents](mcp.md) for the full MCP reference.
