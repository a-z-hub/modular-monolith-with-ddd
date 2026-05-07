# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a **Modular Monolith** built with .NET 10.0 using **Domain-Driven Design (DDD)** principles. The application models a meeting management domain (inspired by Meetup.com) with features like meeting groups, meetings, registrations, payments, and user access management.

Key goals: demonstrate production-ready implementation of DDD tactical patterns, CQRS, modular architecture, comprehensive testing, and architectural decision logging.

## Prerequisites

- **.NET SDK 10.0** — Check with `dotnet --version`
- **Docker** — Required for integration tests and local database
- **SQL Server** — Via Docker or local instance (integration tests use SQL Server 2022)

## Quick Start

```bash
# Build the solution
./build.sh Compile

# Run API locally
cd API/CompanyName.MyMeetings.API
dotnet run

# API runs on http://localhost:5000 (or http://localhost:5001 for HTTPS)
```

For integration tests and full local development, start the SQL Server container:

```bash
docker-compose up -d  # from repository root
```

## Architecture

### High-Level Structure

```
API (HttpEndpoints)
    ↓
Modules (independent business domains)
    ├── Administration
    ├── Meetings
    ├── Payments
    ├── Registrations
    └── UserAccess
    ↓
BuildingBlocks (shared DDD patterns, infrastructure, testing utilities)
    ├── Application (CQRS handlers, command/query processing)
    ├── Domain (base entities, value objects, rules)
    └── Infrastructure (database, events, dependency injection)
```

### Module Structure

Each module follows **Clean Architecture** with these layers (as separate assemblies):

- **Domain** — Bounded Context implementation with Aggregate Roots, Entities, Value Objects, Domain Rules, Events. No external dependencies (POCOs).
- **Application** — CQRS request processing: CommandHandlers, QueryHandlers, Application Services. Depends on Domain.
- **Infrastructure** — Database access (repositories), Event Bus integration, configuration, dependency injection setup. Depends on Application and Domain.
- **IntegrationEvents** — Contracts published to event bus. Only assembly other modules can depend on.
- **Tests** — UnitTests, IntegrationTests, ArchTests (enforces architecture rules).

### Key Patterns

**CQRS**: Commands (write operations via domain model) and Queries (read operations via raw SQL on views) are processed by separate handlers.

**Decorator Pattern**: Cross-cutting concerns (logging, validation, unit-of-work) applied via decorator chain:
```
LoggingCommandHandlerDecorator
  → ValidationCommandHandlerDecorator
    → UnitOfWorkCommandHandlerDecorator
      → ActualCommandHandler
```

**Domain Model Principles**:
- High encapsulation: `private` by default, `internal` for module internals, `public` only at edges
- Rich in behavior: business logic in domain entities and aggregates, not anemic DTOs
- Persistence ignorance: no ORM/database imports in Domain layer
- Value Objects group primitives and enforce invariants
- Business language in naming (ubiquitous language)
- Testable design (unit tests without database or containers)

**Module Integration**:
- Each module exposed via interface (e.g., `IMeetingsModule`) with three methods:
  - `ExecuteCommandAsync<TResult>(ICommand<TResult>)` — command with result
  - `ExecuteCommandAsync(ICommand)` — command without result
  - `ExecuteQueryAsync<TResult>(IQuery<TResult>)` — query
- Modules communicate asynchronously via IntegrationEvents (event bus)
- No synchronous cross-module calls except during initialization

**Module Initialization**: Each module has static `Initialize(connectionString, executionContextAccessor, logger, ...)` method called from API startup. Composition Root created with Inversion-of-Control container (Autofac).

## Build & Test Commands

All commands use the NUKE build system. Run from repository root:

```bash
# Build
./build.sh Compile

# Unit tests
./build.sh UnitTests

# Architecture tests (enforce module boundaries, naming rules, dependency rules)
./build.sh ArchitectureTests

# Build + unit tests + arch tests
./build.sh BuildAndUnitTests

# Integration tests (requires Docker + SQL Server)
./build.sh IntegrationTests
```

For single test file (from src directory):
```bash
dotnet test <path-to-csproj> --filter "ClassName"
```

## Local Development Setup

**Connection String** (for integration tests and local API):
```
Server=127.0.0.1,1401;Database=CompanyName_MyMeetings;User=sa;Password=123qwe!@#QWE;Encrypt=False;
```

**Environment Variables** (if not using docker-compose):
```bash
export MEETINGS_CONNECTION_STRING="Server=localhost;Database=CompanyName_MyMeetings;User=sa;Password=..."
export ASPNETCORE_ENVIRONMENT="Development"
```

**Docker Compose** starts:
- SQL Server on port 1401 (internal port 1433)
- Database pre-seeded with migrations

After running `docker-compose up -d`, wait ~10 seconds for SQL Server to be ready before running tests.

## Directory Layout

```
src/
├── API/
│   └── CompanyName.MyMeetings.API/          # HTTP API entry point
├── Modules/
│   ├── Administration/   {Application, Domain, Infrastructure, IntegrationEvents, Tests}
│   ├── Meetings/         {Application, Domain, Infrastructure, IntegrationEvents, Tests}
│   ├── Payments/         {Application, Domain, Infrastructure, IntegrationEvents, Tests}
│   ├── Registrations/    {Application, Domain, Infrastructure, IntegrationEvents, Tests}
│   └── UserAccess/       {Application, Domain, Infrastructure, IntegrationEvents, Tests}
├── BuildingBlocks/
│   ├── Application/      # Shared CQRS, mediator, base handlers
│   ├── Domain/           # Base Entity, ValueObject, IAggregateRoot, DomainEvents
│   └── Infrastructure/   # DbContext, repositories, event bus, DI configuration
├── Database/
│   ├── CompanyName.MyMeetings.Database/     # SQL scripts, views
│   └── DatabaseMigrator/                     # DbUp migrations
└── Tests/
    ├── ArchTests/        # Architecture compliance tests
    └── IntegrationTests/ # System-level integration tests
```

## Key Files & Concepts

**Directory.Build.props** — .NET version (net10.0), shared compilation settings, code analysis rules.

**Directory.Packages.props** — Centralized NuGet package versions.

**Domain Aggregate Pattern** (e.g., `Modules/Meetings/Domain/MeetingGroups/MeetingGroup.cs`):
```csharp
public class MeetingGroup : Entity, IAggregateRoot
{
    private string _name;  // private by default
    private List<MeetingGroupMember> _members;

    // Only static factory or private constructors
    internal static MeetingGroup CreateBasedOnProposal(...) { ... }

    // Methods enforce business rules via DomainRules
    public Meeting CreateMeeting(...) {
        this.CheckRule(new MeetingCanBeOrganizedOnlyByPayedGroupRule(...));
        return new Meeting(...);
    }
}
```

**CQRS Handler** (e.g., `Modules/Meetings/Application/MeetingGroups/CreateNewMeetingGroupCommandHandler.cs`):
- Command: contains request data, has an Id for tracing
- Handler: processes command, updates repositories, publishes domain events
- Decorators add logging, validation, unit-of-work management

**Repository Pattern**: Repositories abstract data access; queries use Dapper with raw SQL on database views for performance.

## Development Workflow

1. **Identify the module** — which business domain does the feature belong to?
2. **Start with Domain** — define aggregates, value objects, and business rules
3. **Add Application layer** — commands/queries and their handlers
4. **Implement Infrastructure** — repositories, DbUp migrations, DI configuration
5. **Module initialization** — ensure `Initialize()` is called from API
6. **Write tests** — unit tests (domain logic), integration tests (full flow)
7. **Enforce integration** — use IntegrationEvents for cross-module async communication

## Common Patterns to Follow

- **Internal by default**: Mark types `internal` unless part of module's public interface
- **Immutable after creation**: Aggregate state changes only via public methods
- **Business rule validation**: Use `CheckRule(IDomainRule)` for invariants
- **Decorator registration**: Cross-cutting concerns added via Autofac decorators
- **Event sourcing**: Consider for audit trail; events published to IntegrationEvents bus
- **SQL views for reads**: Read models built as database views queried with Dapper

## Testing Strategy

**Unit Tests** — Domain logic, no database. Test aggregates and value objects directly.

**Integration Tests** — Full module flow with real database. Use TestFixture base class for setup.

**Architecture Tests** — Assert module boundaries (no circular dependencies), naming conventions (e.g., handlers end in `CommandHandler`), layer dependencies (Application depends on Domain, not Infrastructure).

