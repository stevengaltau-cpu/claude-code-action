# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Development Tools

- **Runtime**: Bun 1.2.11
- **TypeScript**: Strict configuration with `moduleResolution: "bundler"`
- **Package Manager**: Bun (uses `bun.lock`)
- **Key Dependencies**:
  - `@anthropic-ai/claude-agent-sdk` v0.2.3 - Claude Agent SDK integration
  - `@modelcontextprotocol/sdk` - MCP server implementation
  - `@actions/core`, `@actions/github` - GitHub Actions integration
  - `@octokit/rest`, `@octokit/graphql` - GitHub API clients
  - `zod` - Runtime type validation

## Common Development Tasks

```bash
# Run tests
bun test

# Format code with Prettier
bun run format
bun run format:check    # Check formatting only

# Type checking
bun run typecheck       # Run TypeScript type checker

# Install git hooks
bun run install-hooks
```

## Architecture Overview

This is a GitHub Action that enables Claude to interact with GitHub PRs and issues. The project consists of two actions:

1. **Main Action** (`action.yml`): Full-featured GitHub automation with PR/issue context
2. **Base Action** (`base-action/`): Standalone Claude Code execution, published as `@anthropic-ai/claude-code-base-action`

### Execution Flow

#### Phase 1: Preparation (`src/entrypoints/prepare.ts`)

1. **Authentication Setup**: Establishes GitHub token via OIDC or GitHub App
2. **Permission Validation**: Verifies actor has write permissions
3. **Trigger Detection**: Uses mode-specific logic to determine if Claude should respond
4. **Context Creation**: Prepares GitHub context and initial tracking comment
5. **Branch Setup**: Creates working branches using configurable templates

#### Phase 2: Execution (`base-action/src/index.ts`)

1. **Environment Validation**: Validates required environment variables
2. **Settings Setup**: Configures Claude Code settings from JSON or file
3. **Plugin Installation**: Installs Claude Code plugins from marketplaces
4. **Prompt Preparation**: Loads and processes prompts
5. **Claude Execution**: Runs Claude Code via SDK with configured options
6. **Result Processing**: Updates comments, creates branches/PRs

### Key Architectural Components

#### Mode System (`src/modes/`)

Extensible system for different interaction modes:

- **Tag Mode** (`tag/`): Responds to `@claude` mentions, issue assignments, and label triggers
- **Agent Mode** (`agent/`): Direct execution when explicit prompt is provided
- **Registry** (`registry.ts`): Mode selection based on GitHub event context
- **Detector** (`detector.ts`): Determines appropriate mode from inputs

Modes implement the `Mode` interface with:

- `shouldTrigger()`: Determines if mode should activate
- `prepareContext()`: Prepares mode-specific context
- `getAllowedTools()` / `getDisallowedTools()`: Tool access control
- `generatePrompt()`: Creates Claude prompt
- `prepare()`: Sets up GitHub environment

#### GitHub Integration (`src/github/`)

- **Context Parsing** (`context.ts`): Unified GitHub event handling with discriminated unions
- **API Clients** (`api/client.ts`): REST and GraphQL client creation
- **Data Layer**:
  - `data/fetcher.ts`: Retrieves PR/issue data via GraphQL/REST
  - `data/formatter.ts`: Converts GitHub data to Claude-readable format
- **Operations**:
  - `operations/branch.ts`: Branch creation with template support
  - `operations/branch-cleanup.ts`: Stale branch cleanup
  - `operations/comments/`: Comment creation and updates
  - `operations/git-config.ts`: Git configuration and SSH signing
- **Validation**:
  - `validation/permissions.ts`: Write access verification
  - `validation/actor.ts`: Human vs bot detection
  - `validation/trigger.ts`: Trigger phrase detection
- **Utilities**:
  - `utils/sanitizer.ts`: Content sanitization for security
  - `utils/image-downloader.ts`: Image handling for Claude

#### MCP Server Integration (`src/mcp/`)

Custom MCP servers providing GitHub API access:

- **GitHub Actions Server** (`github-actions-server.ts`): Workflow and CI access
- **GitHub Comment Server** (`github-comment-server.ts`): Comment CRUD operations
- **GitHub Inline Comment Server** (`github-inline-comment-server.ts`): PR inline review comments
- **GitHub File Operations** (`github-file-ops-server.ts`): File system access with security
- **Path Validation** (`path-validation.ts`): Prevents path traversal attacks
- **Installation** (`install-mcp-server.ts`): Auto-installation and configuration

#### Authentication & Security (`src/github/`)

- **Token Management** (`token.ts`): OIDC token exchange and GitHub App authentication
- **Input Sanitization**: Prevents token exposure in outputs
- **Path Validation**: Symlink and traversal attack prevention
- **Plugin Validation**: Command injection prevention in plugin names

### Project Structure

```
├── action.yml              # Main action definition
├── base-action/            # Standalone base action
│   ├── action.yml          # Base action definition
│   ├── src/
│   │   ├── index.ts        # Entry point
│   │   ├── run-claude.ts   # Claude execution logic
│   │   ├── run-claude-sdk.ts  # SDK integration
│   │   ├── install-plugins.ts # Plugin installation
│   │   ├── setup-claude-code-settings.ts
│   │   ├── validate-env.ts
│   │   └── prepare-prompt.ts
│   └── test/
├── src/
│   ├── entrypoints/        # Action entry points
│   │   ├── prepare.ts      # Main preparation logic
│   │   ├── update-comment-link.ts  # Post-execution updates
│   │   ├── format-turns.ts # Conversation formatting
│   │   ├── collect-inputs.ts # Input collection
│   │   └── cleanup-ssh-signing.ts # SSH key cleanup
│   ├── create-prompt/      # Prompt generation
│   │   ├── index.ts        # Prompt builder
│   │   └── types.ts        # Prompt types
│   ├── github/             # GitHub integration layer
│   │   ├── api/            # REST/GraphQL clients
│   │   │   ├── client.ts
│   │   │   ├── config.ts
│   │   │   └── queries/github.ts
│   │   ├── data/           # Data fetching and formatting
│   │   ├── operations/     # Branch, comment, git operations
│   │   ├── validation/     # Permission and trigger validation
│   │   ├── utils/          # Image downloading, sanitization
│   │   ├── context.ts      # Event context parsing
│   │   ├── constants.ts    # Bot IDs, limits
│   │   ├── token.ts        # Authentication
│   │   └── types.ts        # Type definitions
│   ├── modes/              # Execution modes
│   │   ├── tag/            # @claude mention mode
│   │   ├── agent/          # Automation mode
│   │   ├── registry.ts     # Mode selection
│   │   ├── detector.ts     # Mode detection
│   │   └── types.ts        # Mode interfaces
│   ├── mcp/                # MCP server implementations
│   ├── prepare/            # Preparation orchestration
│   └── utils/              # Shared utilities
│       ├── branch-template.ts  # Branch name templating
│       ├── extract-user-request.ts
│       └── retry.ts        # Retry logic with backoff
├── test/                   # Test files
├── examples/               # Example workflow configurations
├── docs/                   # Documentation
└── scripts/                # Development scripts
```

## Key Features

### Branch Name Templates

Customizable branch naming via `branch_name_template` input:

- Variables: `{{prefix}}`, `{{entityType}}`, `{{entityNumber}}`, `{{timestamp}}`, `{{sha}}`, `{{label}}`, `{{description}}`
- Default: `{{prefix}}{{entityType}}-{{entityNumber}}-{{timestamp}}`
- See `src/utils/branch-template.ts` for implementation

### Commit Signing

Two signing options:

1. **GitHub Commit Signature**: Via `use_commit_signing: true`
2. **SSH Signing**: Via `ssh_signing_key` input (takes precedence)

Cleanup handled by `src/entrypoints/cleanup-ssh-signing.ts`

### Cloud Provider Support

Multi-cloud deployment via environment variables:

- **Anthropic API**: Direct API access (default)
- **AWS Bedrock**: Set `use_bedrock: true`
- **Google Vertex AI**: Set `use_vertex: true`
- **Microsoft Foundry**: Set `use_foundry: true`

### Plugin System (`base-action/src/install-plugins.ts`)

- Install from custom marketplaces
- Validates plugin names against injection attacks
- Supports marketplace Git URLs and local paths

## Implementation Notes

### Authentication Flow

1. OIDC token exchange for secure authentication
2. Custom GitHub Apps via `APP_ID` and `APP_PRIVATE_KEY`
3. Falls back to official Claude GitHub App if no custom app provided
4. Token revocation on completion for security

### Security Practices

- **Path Validation**: `src/mcp/path-validation.ts` prevents traversal via symlinks
- **Content Sanitization**: `src/github/utils/sanitizer.ts` removes tokens from output
- **Plugin Validation**: Regex patterns prevent command injection
- **Input Validation**: Zod schemas for runtime type checking

### Error Handling Patterns

- Retry logic with exponential backoff (`src/utils/retry.ts`)
- Detailed error messages for common GitHub API issues
- Graceful degradation for optional features
- Non-blocking cleanup operations

### Comment Threading

- Single tracking comment updated throughout execution
- Progress indicated via dynamic checkboxes
- Links to job runs and created branches/PRs
- Sticky comment option via `use_sticky_comment`

## Code Conventions

- **TypeScript**: Strict mode with `noUnusedLocals` and `noUnusedParameters`
- **Discriminated Unions**: For GitHub context types (see `src/github/context.ts`)
- **Explicit Error Handling**: Detailed error messages with context
- **Async/Await**: Consistent async patterns throughout
- **Type Exports**: Use `export type` for type-only exports
- **Bun-specific**: Uses `import.meta.main` for entry point detection

## Testing

Tests use Bun's built-in test runner:

```bash
bun test                    # Run all tests
bun test path/to/file.test.ts  # Run specific test
```

Key test files:

- `test/modes/`: Mode-specific tests
- `test/data-*.test.ts`: Data fetching/formatting tests
- `test/sanitizer.test.ts`: Security-related tests
- `base-action/test/`: Base action tests
