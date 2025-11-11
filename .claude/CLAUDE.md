# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Elsa Workflows is a powerful .NET workflow engine that enables workflow execution within any .NET application. It supports C#-coded workflows, visual designer workflows, and JSON-defined workflows. The project is built on .NET 9 with C# latest features and uses implicit usings and nullable reference types throughout.

## Build & Development Commands

### Building the Project

**NUKE Build System**: This project uses NUKE for builds. All build commands should use the build scripts:

```bash
# macOS/Linux
./build.sh [target] [options]

# Windows
.\build.cmd [target] [options]
```

**Available Build Targets:**
- `Compile` (default) - Builds the solution
- `Clean` - Cleans bin/obj directories and artifacts
- `Restore` - Restores NuGet packages
- `Test` - Runs all tests (unit, integration, component, performance)
- `Pack` - Creates NuGet packages

**Common Build Commands:**
```bash
# Build with default target (Compile)
./build.sh

# Clean and compile
./build.sh Clean Compile

# Run tests
./build.sh Test

# Create packages
./build.sh Pack

# Build with Release configuration
./build.sh --configuration Release

# Run tests with code coverage
./build.sh Test --analyse-code
```

### Running Tests

**Test Organization** (in `test/` directory):
- `unit/` - Fast, isolated unit tests
- `integration/` - Integration tests requiring external dependencies
- `component/` - End-to-end component tests
- `performance/` - Performance benchmark tests

**Run specific test projects:**
```bash
# Run specific test project with dotnet
dotnet test test/unit/Elsa.Workflows.Core.UnitTests/Elsa.Workflows.Core.UnitTests.csproj

# Run single test
dotnet test --filter "FullyQualifiedName=Namespace.ClassName.TestMethodName"

# Run tests matching pattern
dotnet test --filter "FullyQualifiedName~Workflows"
```

### Running Applications

**Reference Applications** (in `src/apps/`):

1. **Elsa.Server.Web** - Workflow server only
   ```bash
   dotnet run --project src/apps/Elsa.Server.Web/Elsa.Server.Web.csproj
   ```

2. **Elsa.ServerAndStudio.Web** - Combined server + visual designer (recommended for development)
   ```bash
   dotnet run --project src/apps/Elsa.ServerAndStudio.Web/Elsa.ServerAndStudio.Web.csproj
   ```
   Default credentials: `admin` / `password`

3. **Elsa.Studio.Web** - Designer only (requires separate server)
   ```bash
   dotnet run --project src/apps/Elsa.Studio.Web/Elsa.Studio.Web.csproj
   ```

### Docker

```bash
# Pull and run latest image
docker pull elsaworkflows/elsa-server-and-studio-v3:latest
docker run -t -i -e ASPNETCORE_ENVIRONMENT='Development' -e HTTP_PORTS=8080 -e HTTP__BASEURL=http://localhost:13000 -p 13000:8080 elsaworkflows/elsa-server-and-studio-v3:latest
```

## Architecture Overview

### Module Organization

**Layered Module Structure** (in `src/modules/`):

- **Elsa.Workflows.Core** - Core workflow engine (execution, activities, state management)
- **Elsa.Workflows.Management** - Workflow definition and instance persistence
- **Elsa.Workflows.Runtime** - Runtime services, bookmarks, triggers, actor model
- **Elsa.Workflows.Api** - REST API endpoints for workflow management
- **Elsa.Expressions.[Language]** - Expression evaluation (CSharp, JavaScript, Python, Liquid)
- **Elsa.Http** - HTTP activities and endpoints
- **Elsa.Identity** - Authentication and authorization
- **Elsa.[Other]** - Additional domain modules (Email, Caching, etc.)

**Common Libraries** (in `src/common/`):
- **Elsa.Features** - Module/feature registration system
- **Elsa.Mediator** - CQRS-style commands and notifications
- **Elsa.Api.Common** - Shared API infrastructure
- **Elsa.Testing.Shared** - Testing utilities

### Core Abstractions

**Activity Model:**
- Activities are the atomic units of work in workflows
- Base class: `CodeActivity` for simple activities with auto-complete behavior
- Activities have inputs (decorated with `[Input]`) and outputs (decorated with `[Output]`)
- Activities execute via `ExecuteAsync(ActivityExecutionContext context)`
- Example location: `src/modules/Elsa.Workflows.Core/Abstractions/CodeActivity.cs`

**Execution Context:**
- `WorkflowExecutionContext` - Workflow-level execution state (bookmarks, variables, status)
- `ActivityExecutionContext` - Activity-level execution state (parent/child, input/output)

**State Management:**
- `WorkflowState` - Serializable state snapshot
- `WorkflowInstance` - Persisted instance entity
- `WorkflowDefinition` - Persisted definition entity
- State is extracted via `IWorkflowStateExtractor` and persisted via stores

**Pipeline Architecture:**
- Workflow execution flows through middleware pipelines
- `IWorkflowExecutionPipeline` - Workflow-level pipeline
- `IActivityExecutionPipeline` - Activity-level pipeline (evaluation, execution, bookmarking, logging)

### Persistence Layer

**Store Abstractions:**
- `IWorkflowDefinitionStore` - CRUD operations for workflow definitions
- `IWorkflowInstanceStore` - CRUD operations for workflow instances
- Repository pattern with filter-based queries
- Implementations available for Entity Framework Core, MongoDB, Dapper

### Expression System

**Expression Handlers:**
- `IExpressionHandler` - Interface for custom expression languages
- Built-in: C#, JavaScript, Python, Liquid templates
- Expressions evaluated at runtime from activity properties
- Location: `src/modules/Elsa.Expressions/Contracts/IExpressionHandler.cs`

### Module/Feature System

**Registration Pattern:**
```csharp
public class MyFeature : FeatureBase
{
    public override void Apply()
    {
        Services.AddSingleton<IMyService, MyService>();
    }
}

// In Startup/Program.cs
services.AddElsa(elsa =>
{
    elsa.AddFeature<MyFeature>();
});
```

- Features can declare dependencies: `[DependsOn(typeof(OtherFeature))]`
- Features configure services, pipelines, and activities
- Location: `src/common/Elsa.Features`

### Mediator Pattern (CQRS)

**Command/Event Flow:**
- Commands (`ICommand` / `ICommand<T>`) - Write operations
- Notifications (`INotification`) - Pub-sub events
- Handlers implement `ICommandHandler<T>` or `INotificationHandler<T>`
- Examples: `SaveWorkflowDefinitionCommand`, `WorkflowStarted` notification
- Location: `src/common/Elsa.Mediator`

## Extension Points

### Creating Custom Activities

1. **Inherit from CodeActivity:**
```csharp
[Activity("MyNamespace", "MyCategory", "Description")]
public class MyActivity : CodeActivity
{
    [Input] public Input<string> MyInput { get; set; }
    [Output] public Output<string> Result { get; set; }

    protected override async ValueTask ExecuteAsync(ActivityExecutionContext context)
    {
        var inputValue = MyInput.Get(context);
        // Your logic
        Result.Set(context, result);
    }
}
```

2. **Register in a feature or via activity providers** - Activities are auto-discovered via `IActivityProvider`

### Adding Custom Expression Languages

Implement `IExpressionHandler` interface:
```csharp
public class MyExpressionHandler : IExpressionHandler
{
    public string Language => "mylang";

    public async Task<object?> EvaluateAsync(
        Expression expression,
        Type returnType,
        ExpressionExecutionContext context)
    {
        // Evaluation logic
    }
}
```

### Creating Custom Middleware

Add to execution pipelines:
```csharp
public class MyMiddleware : IActivityExecutionMiddleware
{
    public async ValueTask InvokeAsync(ActivityExecutionContext context, ActivityMiddlewareDelegate next)
    {
        // Before execution
        await next(context);
        // After execution
    }
}

// Register
feature.WithActivityExecutionPipeline(pipeline =>
    pipeline.UseMiddleware<MyMiddleware>());
```

### Custom Persistence

Implement store interfaces:
- `IWorkflowDefinitionStore`
- `IWorkflowInstanceStore`

Pattern: Repository with filter-based queries and pagination support.

## Code Standards

### Language Features

- **Target:** .NET 9
- **Language:** C# latest with implicit usings and nullable reference types enabled
- **Warnings:** Documentation warnings (CS1591) and specific IL trimming warnings suppressed

### Project Structure

- **Directory.Build.props** - Shared MSBuild properties (package info, compiler settings)
- **Directory.Packages.props** - Central package management (CPM) with version centralization
- All projects use MIT license
- XML documentation generation enabled for all projects

### Testing

- Write unit tests for new activities and core logic
- Use integration tests for persistence and external dependencies
- Component tests for end-to-end scenarios
- Follow existing test project naming: `[ProjectName].Tests` or `[ProjectName].UnitTests`

## Development Workflow

### Branch Strategy

- **Trunk-based development** - Branch from `main`, submit PRs back to `main`
- Open an issue before starting work to align with project goals
- See CONTRIBUTING.md for detailed contribution guidelines

### Git Rules

- **No git rebase** - Use merge commits
- Follow conventional commit messages where possible
- Do not push to main directly - always use pull requests

### Key Files to Check

- **Directory.Build.props** - Build configuration and package settings
- **Directory.Packages.props** - NuGet package versions
- **Elsa.sln** - Main solution file
- **build/Build.cs** - NUKE build configuration
- **README.md** - Public documentation
- **CONTRIBUTING.md** - Contribution guidelines

## Important Notes

- Activities use `CodeActivity` base class for auto-complete behavior
- State management separates runtime context from persistent state
- Mediator pattern used extensively for cross-cutting concerns
- Features are the primary unit of modularity and configuration
- Expression languages evaluated lazily at runtime
- Pipelines provide extension points at workflow and activity levels
- Store abstractions enable pluggable persistence backends
