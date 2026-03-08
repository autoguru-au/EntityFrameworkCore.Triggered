# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

EntityFrameworkCore.Triggered is a library that adds trigger support to EF Core. Triggers respond to entity changes before and after they are committed to the database, similar to SQL database triggers but implemented in C#.

## Build Commands

```bash
# Build (v3 is the default configuration on this branch)
dotnet build EntityFrameworkCore.Triggered.sln

# Run all tests
dotnet test EntityFrameworkCore.Triggered.sln

# Run a single test project
dotnet test test/EntityFrameworkCore.Triggered.Tests

# Run a specific test by filter
dotnet test test/EntityFrameworkCore.Triggered.Tests --filter "FullyQualifiedName~TriggerSessionTests"

# Build only core libraries (no samples)
dotnet build EntityFrameworkCore.Triggered.Core.slnf

# Build samples only
dotnet build EntityFrameworkCore.Triggered.Samples.slnf
```

## Multi-Version Build System

The repo supports 3 major versions via `$(EFCoreTriggeredVersion)` in `Directory.Build.props`:
- **V1**: EF Core 3.1, `netstandard2.0`, config `ReleaseV1`/`DebugV1`
- **V2**: EF Core 5.0, `netstandard2.1`, config `ReleaseV2`/`DebugV2`
- **V3** (current branch): EF Core 6.0+, `net6.0`, config `Release`/`Debug`

Source files use `#if` directives with `EFCORETRIGGERED_V1`, `EFCORETRIGGERED_V2`, `EFCORETRIGGERED_V3` for version-specific code.

## Build Settings

- `TreatWarningsAsErrors` is enabled globally
- Nullable reference types are enabled globally (`<Nullable>enable</Nullable>`)
- Language version is C# 9.0
- Strong-name signing is used (`EntityFrameworkCore.Triggered.snk`)

## Architecture

### NuGet Packages (src/)

- **Abstractions** — Trigger interfaces only (`IBeforeSaveTrigger<T>`, `IAfterSaveTrigger<T>`, `IAfterSaveFailedTrigger<T>`, `ITriggerContext<T>`, lifecycle triggers)
- **EntityFrameworkCore.Triggered** — Core implementation: trigger session management, SaveChanges interception, cascading support
- **Extensions** — Assembly scanning for auto-discovery of triggers via `AddAssemblyTriggers()`
- **Transactions** / **Transactions.Abstractions** — Transaction-scoped triggers (`IBeforeCommitTrigger<T>`, `IAfterCommitTrigger<T>`, etc.)

### Core Flow

1. **Registration**: Triggers are registered via `options.UseTriggers(t => t.AddTrigger<MyTrigger>())` on `DbContextOptionsBuilder`
2. **Interception**: `TriggerSessionSaveChangesInterceptor` hooks into EF Core's `SaveChanges`/`SaveChangesAsync`
3. **Execution order**: BeforeSaveStarting → BeforeSave (with cascading) → CaptureChanges → BeforeSaveCompleted → [actual SaveChanges] → AfterSaveStarting → AfterSave → AfterSaveCompleted
4. **Cascading**: BeforeSave triggers can modify entities, causing additional trigger invocations. Controlled by `ICascadeStrategy` with a configurable max cycle count (default 100).

### Key Internal Types

- `TriggerSession` (`TriggerSession.cs`) — Orchestrates trigger execution for a single SaveChanges call
- `TriggerService` (`TriggerService.cs`) — Factory for trigger sessions, manages current session state
- `TriggerSessionSaveChangesInterceptor` (`Internal/`) — EF Core `ISaveChangesInterceptor` implementation
- `TriggersOptionExtension` (`Infrastructure/Internal/`) — EF Core options extension that registers all trigger services
- `TriggerContextTracker` — Wraps EF Core's ChangeTracker to produce `TriggerContext<T>` instances

### Test Projects (test/)

- **Tests** — Unit tests for the core library (xUnit + ScenarioTests.XUnit)
- **Extensions.Tests** — Tests for assembly scanning
- **IntegrationTests** — End-to-end tests
- **Transactions.Tests** — Transaction trigger tests

Tests use EF Core InMemory and SQLite providers.
