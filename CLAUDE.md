# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is **Claude Code Action** (`@anthropic-ai/claude-code-action` v1.0.0), a GitHub Action that enables Claude to interact with GitHub PRs and issues. It supports automated PR reviews, issue triage, @claude mentions, and custom automation workflows.

## Development Tools

- **Runtime**: Bun 1.2.11
- **Language**: TypeScript 5.8.3 (strict mode)
- **Package Manager**: Bun
- **Formatter**: Prettier 3.5.3

## Common Development Tasks

```bash
# Run all tests
bun test

# Run specific test file
bun test test/modes/tag.test.ts

# Type checking
bun run typecheck

# Code formatting
bun run format          # Format code with Prettier
bun run format:check    # Check code formatting

# Install dependencies
bun install
```

## Architecture Overview

The action operates in two main phases:

### Phase 1: Preparation (`src/entrypoints/prepare.ts`)

1. **Authentication Setup**: Establishes GitHub token via OIDC or GitHub App
2. **Permission Validation**: Verifies actor has write permissions
3. **Trigger Detection**: Uses mode-specific logic to determine if Claude should respond
4. **Context Creation**: Prepares GitHub context and initial tracking comment

### Phase 2: Execution (`base-action/`)

The `base-action/` directory contains the core Claude Code execution logic, published separately as `@anthropic-ai/claude-code-base-action` for standalone use.

1. **MCP Server Setup**: Installs and configures GitHub MCP servers for tool access
2. **Prompt Generation**: Creates context-rich prompts from GitHub data
3. **Claude Integration**: Executes via multiple providers (Anthropic API, AWS Bedrock, Google Vertex AI, Microsoft Foundry)
4. **Result Processing**: Updates comments and creates branches/PRs as needed

## Project Structure

```
claude-code-action/
├── src/
│   ├── entrypoints/              # Action entry points
│   │   ├── prepare.ts            # Main preparation logic
│   │   ├── update-comment-link.ts # Post-execution comment updates
│   │   ├── format-turns.ts       # Claude conversation formatting
│   │   ├── cleanup-ssh-signing.ts # SSH key cleanup
│   │   └── collect-inputs.ts     # Input collection/validation
│   ├── github/                   # GitHub integration layer
│   │   ├── api/                  # REST/GraphQL clients
│   │   ├── data/                 # Data fetching and formatting
│   │   │   ├── fetcher.ts        # GraphQL data fetching
│   │   │   └── formatter.ts      # Data formatting for Claude
│   │   ├── operations/           # Branch, comment, git operations
│   │   │   ├── branch.ts         # Branch creation/cleanup
│   │   │   └── comments/         # Comment management
│   │   ├── validation/           # Permission and trigger validation
│   │   │   ├── permissions.ts    # Write access verification
│   │   │   ├── actor.ts          # Human vs bot detection
│   │   │   └── trigger.ts        # Trigger phrase matching
│   │   ├── context.ts            # Unified GitHub event handling
│   │   ├── token.ts              # OIDC/GitHub App authentication
│   │   └── utils/                # Image downloading, sanitization
│   ├── modes/                    # Execution modes
│   │   ├── tag/                  # @claude mention mode
│   │   ├── agent/                # Direct automation mode
│   │   ├── registry.ts           # Mode selection logic
│   │   └── detector.ts           # Mode auto-detection
│   ├── mcp/                      # MCP server implementations
│   │   ├── github-comment-server.ts       # Comment updates
│   │   ├── github-file-ops-server.ts      # File operations via API
│   │   ├── github-actions-server.ts       # CI/workflow access
│   │   ├── github-inline-comment-server.ts # PR line comments
│   │   ├── install-mcp-server.ts          # Auto-installation
│   │   └── path-validation.ts             # Path security validation
│   ├── create-prompt/            # Prompt generation
│   ├── prepare/                  # Preparation orchestration
│   └── utils/                    # Shared utilities
│       ├── retry.ts              # Exponential backoff retry
│       └── branch-template.ts    # Branch name generation
├── base-action/                  # Standalone execution layer
│   ├── src/
│   │   ├── index.ts              # Main entrypoint
│   │   ├── run-claude.ts         # Execute Claude (named pipes)
│   │   ├── run-claude-sdk.ts     # SDK-based execution
│   │   ├── prepare-prompt.ts     # Prompt preparation
│   │   ├── validate-env.ts       # Environment validation
│   │   ├── setup-claude-code-settings.ts # Settings setup
│   │   ├── install-plugins.ts    # Plugin installation
│   │   └── parse-sdk-options.ts  # SDK options parsing
│   ├── test/                     # Base-action tests
│   ├── action.yml                # Standalone action definition
│   ├── package.json              # Minimal dependencies
│   └── CLAUDE.md                 # Base-action documentation
├── test/                         # Main action tests (26+ files)
├── examples/                     # Example workflow files
├── docs/                         # Comprehensive documentation
├── .claude/                      # Custom Claude agents/commands
│   ├── agents/                   # Reusable review agents
│   ├── commands/                 # Custom slash commands
│   └── settings.json             # Claude Code settings
├── .github/workflows/            # CI/CD workflows
├── action.yml                    # Main action definition (40+ inputs)
├── package.json                  # Dependencies
└── tsconfig.json                 # TypeScript configuration
```

## Key Components

### Mode System (`src/modes/`)

- **Tag Mode** (`tag/`): Responds to `@claude` mentions, issue assignments, and labels
- **Agent Mode** (`agent/`): Direct execution when explicit `prompt` input is provided
- Modes implement interface with `shouldTrigger()`, `prepare()`, and `generatePrompt()` methods
- Auto-detection via `detector.ts` based on event type and inputs

### MCP Servers (`src/mcp/`)

Four MCP servers provide GitHub API access:

| Server                            | Tool                                                            | Purpose                                |
| --------------------------------- | --------------------------------------------------------------- | -------------------------------------- |
| `github-comment-server.ts`        | `update_claude_comment`                                         | Update tracking comments with progress |
| `github-file-ops-server.ts`       | `read_file`, `commit_files`, `delete_files`                     | File operations via GitHub API         |
| `github-actions-server.ts`        | `get_ci_status`, `get_workflow_run_details`, `download_job_log` | CI/workflow information                |
| `github-inline-comment-server.ts` | `create_inline_comment`                                         | Line-specific PR comments              |

Servers are auto-installed in `~/.claude/mcp/github-{type}-server/` via `install-mcp-server.ts`.

### GitHub Integration (`src/github/`)

- **Context** (`context.ts`): Handles 5 event types with discriminated unions
- **Data Fetcher** (`data/fetcher.ts`): GraphQL queries for PR/issue data
- **Data Formatter** (`data/formatter.ts`): Converts to Claude-readable format
- **Token Management** (`token.ts`): OIDC exchange and GitHub App auth
- **Validation**: Permission checks, trigger matching, actor detection

### Provider Support

The action supports multiple AI providers:

| Provider          | Configuration       | Authentication            |
| ----------------- | ------------------- | ------------------------- |
| Anthropic API     | Default             | `anthropic_api_key` input |
| AWS Bedrock       | `use_bedrock: true` | OIDC authentication       |
| Google Vertex AI  | `use_vertex: true`  | OIDC authentication       |
| Microsoft Foundry | `use_foundry: true` | OIDC authentication       |

## Testing

### Test Structure

- **Unit tests**: Mode logic, trigger validation, permissions, data formatting
- **Integration tests**: Full workflow scenarios, comment operations
- **MCP server tests**: Server implementation testing
- **34 total test files** across main and base-action

### Running Tests

```bash
# All tests
bun test

# Specific test file
bun test test/modes/tag.test.ts

# Base-action tests
bun test base-action/test/
```

## Custom Claude Assets

### Agents (`.claude/agents/`)

Reusable review agents for specific domains:

- `code-quality-reviewer.md`
- `security-code-reviewer.md`
- `performance-reviewer.md`
- `test-coverage-reviewer.md`
- `documentation-accuracy-reviewer.md`

### Commands (`.claude/commands/`)

Custom slash commands:

- `label-issue.md` - Issue labeling
- `review-pr.md` - PR review
- `commit-and-pr.md` - Commit and PR creation

## GitHub Workflows

### CI/CD (`.github/workflows/`)

| Workflow               | Purpose                          |
| ---------------------- | -------------------------------- |
| `ci.yml`               | Type checking, formatting, tests |
| `release.yml`          | Release automation               |
| `sync-base-action.yml` | Sync base-action changes         |

### Testing Workflows

| Workflow                      | Purpose                      |
| ----------------------------- | ---------------------------- |
| `test-base-action.yml`        | Tests base-action execution  |
| `test-mcp-servers.yml`        | MCP server integration tests |
| `test-settings.yml`           | Settings configuration tests |
| `test-structured-output.yml`  | Structured output validation |
| `test-custom-executables.yml` | Custom executable paths      |

### Example/Demo Workflows

| Workflow            | Purpose                      |
| ------------------- | ---------------------------- |
| `claude.yml`        | Responds to @claude mentions |
| `claude-review.yml` | PR review automation         |
| `issue-triage.yml`  | Issue triage automation      |

## Example Workflows (`examples/`)

Ready-to-use workflow templates:

- `claude.yml` - Basic @claude mention setup
- `pr-review-comprehensive.yml` - Full PR review with progress
- `pr-review-filtered-authors.yml` - Review specific authors
- `pr-review-filtered-paths.yml` - Review specific paths
- `issue-triage.yml` - Automatic issue categorization
- `issue-deduplication.yml` - Find duplicate issues
- `ci-failure-auto-fix.yml` - Auto-fix CI failures
- `test-failure-analysis.yml` - Analyze test failures
- `manual-code-analysis.yml` - On-demand code analysis

## Code Conventions

### TypeScript

- Strict mode with `noUnusedLocals` and `noUnusedParameters`
- Bun-specific configuration with `moduleResolution: "bundler"`
- Use discriminated unions for GitHub context types
- Prefer explicit error handling with detailed messages

### Patterns

- **Retry logic**: Use `utils/retry.ts` for GitHub API operations
- **Path validation**: Always validate file paths in MCP servers
- **Branch naming**: Alphanumeric with configurable templates
- **Content sanitization**: Remove HTML, prevent injection attacks

### Security

- Branch name validation (prevents command injection)
- Path validation (prevents directory traversal)
- Content sanitization (HTML removal, entity encoding)
- Comment filtering by timestamp (prevents post-trigger edits)
- SSH signing support for commits

## Dependencies

### Main Action

```json
{
  "@actions/core": "^1.10.1",
  "@actions/github": "^6.0.1",
  "@anthropic-ai/claude-agent-sdk": "^0.2.3",
  "@modelcontextprotocol/sdk": "^1.11.0",
  "@octokit/graphql": "^8.2.2",
  "@octokit/rest": "^21.1.1",
  "@octokit/webhooks-types": "^7.6.1",
  "node-fetch": "^3.3.2",
  "shell-quote": "^1.8.3",
  "zod": "^3.24.4"
}
```

### Base-Action (Minimal)

```json
{
  "@actions/core": "^1.10.1",
  "@anthropic-ai/claude-agent-sdk": "^0.2.3",
  "shell-quote": "^1.8.3"
}
```

## Important Notes

### Authentication Flow

1. Uses GitHub OIDC token exchange for secure authentication
2. Supports custom GitHub Apps via `APP_ID` and `APP_PRIVATE_KEY`
3. Falls back to official Claude GitHub App if no custom app provided

### Comment Threading

- Single tracking comment updated throughout execution
- Progress indicated via dynamic checkboxes
- Links to job runs and created branches/PRs
- Sticky comment option for consolidated PR comments

### Tool Restrictions

Default allowed tools include:

- File operations: `Edit`, `MultiEdit`, `Glob`, `Grep`, `LS`, `Read`, `Write`
- Git operations: `Bash(git add)`, `Bash(git commit)`, `Bash(git push)`, etc.
- MCP tools: `mcp__github_comment__update_claude_comment`, etc.

Tool access can be customized via `allowed_tools` and `disallowed_tools` inputs.
