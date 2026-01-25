# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Development Tools

- Runtime: Bun 1.2.11
- TypeScript with strict configuration

## Common Development Tasks

### Available npm/bun scripts from package.json:

```bash
# Test
bun test

# Formatting
bun run format          # Format code with prettier
bun run format:check    # Check code formatting

# Type checking
bun run typecheck       # Run TypeScript type checker
```

## Architecture Overview

This is a GitHub Action that enables Claude to interact with GitHub PRs and issues. The action operates in two main phases:

### Phase 1: Preparation (`src/entrypoints/prepare.ts`)

1. **Authentication Setup**: Establishes GitHub token via OIDC or GitHub App
2. **Permission Validation**: Verifies actor has write permissions
3. **Trigger Detection**: Uses mode-specific logic to determine if Claude should respond
4. **Context Creation**: Prepares GitHub context and initial tracking comment

### Phase 2: Execution (`base-action/`)

The `base-action/` directory contains the core Claude Code execution logic, which serves a dual purpose:

- **Standalone Action**: Published separately as `@anthropic-ai/claude-code-base-action` for direct use
- **Inner Logic**: Used internally by this GitHub Action after preparation phase completes

Execution steps:

1. **MCP Server Setup**: Installs and configures GitHub MCP server for tool access
2. **Prompt Generation**: Creates context-rich prompts from GitHub data
3. **Claude Integration**: Executes via multiple providers (Anthropic API, AWS Bedrock, Google Vertex AI)
4. **Result Processing**: Updates comments and creates branches/PRs as needed

### Key Architectural Components

#### Mode System (`src/modes/`)

- **Tag Mode** (`tag/`): Responds to `@claude` mentions and issue assignments
- **Agent Mode** (`agent/`): Direct execution when explicit prompt is provided
- Extensible registry pattern in `modes/registry.ts`

#### GitHub Integration (`src/github/`)

- **Context Parsing** (`context.ts`): Unified GitHub event handling
- **Data Fetching** (`data/fetcher.ts`): Retrieves PR/issue data via GraphQL/REST
- **Data Formatting** (`data/formatter.ts`): Converts GitHub data to Claude-readable format
- **Branch Operations** (`operations/branch.ts`): Handles branch creation and cleanup
- **Comment Management** (`operations/comments/`): Creates and updates tracking comments

#### MCP Server Integration (`src/mcp/`)

- **GitHub Actions Server** (`github-actions-server.ts`): Workflow and CI access
- **GitHub Comment Server** (`github-comment-server.ts`): Comment operations
- **GitHub File Operations** (`github-file-ops-server.ts`): File system access
- Auto-installation and configuration in `install-mcp-server.ts`

#### Authentication & Security (`src/github/`)

- **Token Management** (`token.ts`): OIDC token exchange and GitHub App authentication
- **Permission Validation** (`validation/permissions.ts`): Write access verification
- **Actor Validation** (`validation/actor.ts`): Human vs bot detection

### Project Structure

```
src/
├── entrypoints/           # Action entry points
│   ├── prepare.ts         # Main preparation logic
│   ├── update-comment-link.ts  # Post-execution comment updates
│   └── format-turns.ts    # Claude conversation formatting
├── github/               # GitHub integration layer
│   ├── api/              # REST/GraphQL clients
│   ├── data/             # Data fetching and formatting
│   ├── operations/       # Branch, comment, git operations
│   ├── validation/       # Permission and trigger validation
│   └── utils/            # Image downloading, sanitization
├── modes/                # Execution modes
│   ├── tag/              # @claude mention mode
│   ├── agent/            # Automation mode
│   └── registry.ts       # Mode selection logic
├── mcp/                  # MCP server implementations
├── prepare/              # Preparation orchestration
└── utils/                # Shared utilities
```

## Important Implementation Notes

### Authentication Flow

- Uses GitHub OIDC token exchange for secure authentication
- Supports custom GitHub Apps via `APP_ID` and `APP_PRIVATE_KEY`
- Falls back to official Claude GitHub App if no custom app provided

### MCP Server Architecture

- Each MCP server has specific GitHub API access patterns
- Servers are auto-installed in `~/.claude/mcp/github-{type}-server/`
- Configuration merged with user-provided MCP config via `mcp_config` input

### Mode System Design

- Modes implement `Mode` interface with `shouldTrigger()` and `prepare()` methods
- Registry validates mode compatibility with GitHub event types
- Agent mode triggers when explicit prompt is provided

### Comment Threading

- Single tracking comment updated throughout execution
- Progress indicated via dynamic checkboxes
- Links to job runs and created branches/PRs
- Sticky comment option for consolidated PR comments

## Code Conventions

- Use Bun-specific TypeScript configuration with `moduleResolution: "bundler"`
- Strict TypeScript with `noUnusedLocals` and `noUnusedParameters` enabled
- Prefer explicit error handling with detailed error messages
- Use discriminated unions for GitHub context types
- Implement retry logic for GitHub API operations via `utils/retry.ts`

---

## Skills

The following skills enhance Claude's capabilities for this project. Each skill provides specialized expertise that activates based on context.

### Language Specialists

- **TypeScript Pro**: Advanced type systems, generics, type guards, utility types, tRPC integration, monorepo setup. Use strict mode, branded types, discriminated unions, `satisfies` operator. Avoid explicit `any`, prefer const objects over enums.

- **JavaScript Pro**: Modern ES2023+ features, async patterns, browser APIs, Node.js development, module systems, performance optimization.

- **Python Pro**: Python 3.11+ with type hints, pytest, async/await, dataclasses, mypy configuration, production-grade patterns.

- **Go Pro**: Goroutines, channels, Go generics, gRPC integration, concurrent programming, microservices architecture.

- **Rust Engineer**: Memory safety, ownership patterns, lifetimes, traits, async/await with tokio, zero-cost abstractions.

- **SQL Pro**: Complex queries, window functions, CTEs, indexing strategies, query plan analysis, database schema design.

- **C++ Pro**: Modern C++20/23, RAII, templates, concepts, ranges, coroutines, SIMD optimization, memory management.

- **Swift Expert**: Swift 5.9+, SwiftUI, async/await concurrency, protocol-oriented programming, actors, server-side Swift.

- **Kotlin Specialist**: Coroutines, Flow API, KMP projects, Jetpack Compose, Ktor servers, DSL design, sealed classes.

- **C# Developer**: .NET 8+, ASP.NET Core APIs, Blazor, Entity Framework Core, minimal APIs, async patterns, CQRS with MediatR.

- **PHP Pro**: PHP 8.3+, Laravel, Symfony, strict typing, PHPStan level 9, async patterns with Swoole, PSR standards.

- **Java Architect**: Spring Boot 3.x, WebFlux, JPA optimization, Spring Security, cloud-native patterns, reactive programming.

### Backend Frameworks

- **NestJS Expert**: Modular architecture, dependency injection, modules, controllers, services, DTOs, guards, interceptors, TypeORM/Prisma integration.

- **Django Expert**: Django models, ORM optimization, DRF serializers, viewsets, JWT authentication, REST APIs.

- **FastAPI Expert**: High-performance async Python APIs, Pydantic V2, async SQLAlchemy, JWT authentication, WebSockets, OpenAPI documentation.

- **Spring Boot Engineer**: Spring Boot 3.x, Spring Data JPA, Spring Security 6, WebFlux, Spring Cloud integration.

- **Laravel Specialist**: Laravel 10+, Eloquent ORM, API resources, Livewire components, Sanctum authentication, Horizon queues.

- **Rails Expert**: Rails 7+, Hotwire, Turbo Frames/Streams, Action Cable, Active Record optimization, Sidekiq.

- **.NET Core Expert**: .NET 8, minimal APIs, clean architecture, Entity Framework Core, CQRS with MediatR, JWT authentication, AOT compilation.

### Frontend & Mobile

- **React Expert**: React 18+, Server Components, hooks patterns, state management, performance optimization, Suspense boundaries, React 19 features.

- **Next.js Developer**: Next.js 14+ App Router, server components, server actions, full-stack features, SEO implementation, production deployment.

- **Vue Expert**: Vue 3 Composition API, Pinia, Nuxt 3, Quasar, TypeScript, PWA, Capacitor mobile apps, Vite configuration.

- **Angular Architect**: Angular 17+ standalone components, signals, RxJS patterns, NgRx state management, performance optimization.

- **React Native Expert**: Cross-platform mobile apps, Expo, navigation patterns, platform-specific code, native modules, FlatList optimization.

- **Flutter Expert**: Flutter 3+, Dart, Riverpod/Bloc state management, GoRouter navigation, platform-specific implementations.

### Infrastructure & Cloud

- **Kubernetes Specialist**: K8s deployments, Helm charts, RBAC policies, NetworkPolicies, storage configuration, cluster management, performance optimization.

- **Terraform Engineer**: Infrastructure as Code, multi-cloud provisioning, module development, state management, provider configuration, infrastructure testing.

- **Cloud Architect**: AWS/Azure/GCP architecture, Well-Architected Framework, cost optimization, disaster recovery, landing zones, serverless design.

- **Postgres Pro**: PostgreSQL optimization, EXPLAIN analysis, JSONB operations, extensions, VACUUM tuning, replication, performance monitoring.

- **Database Optimizer**: Index design, query rewrites, configuration tuning, partitioning strategies, lock contention resolution.

### API & Architecture

- **GraphQL Architect**: Schema design, resolvers with DataLoader, Apollo Federation, query optimization, real-time subscriptions.

- **API Designer**: RESTful API design, OpenAPI specifications, resource modeling, versioning strategies, pagination patterns, error handling.

- **WebSocket Engineer**: Real-time communication, bidirectional messaging, horizontal scaling with Redis, presence tracking, room management.

- **Microservices Architect**: Service boundaries, DDD, saga patterns, event sourcing, service mesh, distributed tracing.

- **MCP Developer**: Model Context Protocol development, TypeScript/Python SDKs, resource providers, tool functions.

- **Architecture Designer**: System design, choosing architectures, ADRs, scalability planning, design patterns.

### Quality & Testing

- **Test Master**: Testing strategy (unit, integration, E2E, performance, security), coverage analysis, automation frameworks.

- **Playwright Expert**: Browser automation, E2E testing, Page Object Model, test flakiness debugging, visual testing.

- **Code Reviewer**: PR reviews, code quality checks, refactoring suggestions, security vulnerability identification.

- **Code Documenter**: Docstrings, API documentation, OpenAPI/Swagger specs, JSDoc, doc portals, tutorials.

### DevOps & Operations

- **DevOps Engineer**: CI/CD pipelines, Docker, Kubernetes, GitOps, containerization, infrastructure as code.

- **Monitoring Expert**: Logging, metrics, tracing, alerting, Prometheus/Grafana dashboards, load testing, capacity planning.

- **SRE Engineer**: SLIs/SLOs, error budgets, incident management, toil reduction, chaos engineering, capacity planning.

- **Chaos Engineer**: Chaos experiments, failure injection, resilience testing, blast radius control, game days, antifragile systems.

- **CLI Developer**: Command-line tools, argument parsing, interactive prompts, progress indicators, shell completions.

### Security

- **Secure Code Guardian**: Authentication/authorization implementation, input validation, encryption, OWASP Top 10 prevention.

- **Security Reviewer**: Security audits, SAST scans, penetration testing, DevSecOps practices, cloud security reviews, vulnerability identification.

### Data & Machine Learning

- **Pandas Pro**: DataFrame manipulation, data cleaning, aggregation, merging, time series analysis, performance optimization.

- **Spark Engineer**: Apache Spark, PySpark, distributed data processing, Spark SQL, RDD operations, streaming analytics.

- **ML Pipeline**: ML pipelines, MLflow, Kubeflow, feature stores, experiment tracking, model lifecycle automation.

- **Prompt Engineer**: LLM prompt design, chain-of-thought, few-shot learning, structured outputs, evaluation frameworks.

- **RAG Architect**: RAG systems, vector databases, embeddings, semantic search, document retrieval, context augmentation.

- **Fine-Tuning Expert**: LLM fine-tuning, LoRA, QLoRA, PEFT, dataset preparation, model optimization.

### Platform Specialists

- **Salesforce Developer**: Apex, Lightning Web Components, SOQL, governor limits, bulk processing, platform events.

- **Shopify Expert**: Liquid templating, Storefront API, app development, checkout customization, Shopify Plus features.

- **WordPress Pro**: WordPress themes, plugins, Gutenberg blocks, WooCommerce, performance and security optimization.

- **Atlassian MCP**: Jira/Confluence integration via MCP, JQL queries, CQL queries, ticket creation, documentation updates.

### Specialized

- **Debugging Wizard**: Systematic debugging methodology - reproduce, isolate, hypothesize, test, fix, prevent. Never guess, test hypotheses systematically. Remove all debug code before committing.

- **Fullstack Guardian**: Implementing features across frontend and backend, building APIs with UI, end-to-end data flows.

- **Feature Forge**: Feature definition, requirements gathering, user stories, EARS format specifications.

- **Spec Miner**: Reverse-engineering specifications from existing code, legacy analysis, code archaeology.

- **Legacy Modernizer**: Modernizing legacy systems, strangler fig pattern, monolith decomposition, framework upgrades.

- **Embedded Systems**: Firmware for microcontrollers, STM32, ESP32, FreeRTOS, bare-metal, power optimization, real-time systems.

- **Game Developer**: Game systems, Unity, Unreal Engine, ECS, physics, networking, game performance optimization.

### Skill Activation

Skills activate automatically based on context. Combine skills for complex tasks:

- **New Feature**: Feature Forge → Architecture Designer → Fullstack Guardian + Framework Skills → Test Master → Code Reviewer → Security Reviewer → DevOps Engineer
- **Bug Fixing**: Debugging Wizard → Framework Skills → Test Master → Code Reviewer
- **Security Audit**: Security Reviewer → Secure Code Guardian → Code Reviewer
- **Legacy Migration**: Spec Miner → Legacy Modernizer → Architecture Designer → Test Master
