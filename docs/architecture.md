# Elsa Workflows Brownfield Architecture Document

## Introduction

This document captures the **CURRENT STATE** of the Elsa Workflows v3 codebase, including architectural patterns, real-world implementations, and practical guidance for both using and contributing to the project. It serves as a comprehensive reference for developers working with Elsa, particularly those integrating it into their applications or contributing to the open-source project.

### Document Purpose

- **For Users**: Understand how to embed Elsa in your applications (like Bestedo CLI) for workflow orchestration
- **For Contributors**: Navigate the codebase structure, understand design patterns, and make meaningful contributions
- **For Learners**: Deep-dive into a production-grade .NET workflow engine architecture

### Document Scope

Comprehensive documentation of the entire Elsa Workflows system, covering:
- Core workflow execution engine
- Persistence and state management
- Expression languages (C#, JavaScript, Python, Liquid)
- Runtime services (bookmarks, triggers, actor model)
- HTTP and integration capabilities
- Visual designer integration
- Extension points and customization

### Change Log

| Date       | Version | Description                      | Author        |
|------------|---------|----------------------------------|---------------|
| 2025-11-11 | 1.0     | Initial brownfield analysis      | BMad Master   |

---

## Quick Reference - Key Files and Entry Points

### Critical Files for Understanding the System

**Core Abstractions:**
- **Activity Interface**: `src/modules/Elsa.Workflows.Core/Contracts/IActivity.cs`
- **Base Activity Class**: `src/modules/Elsa.Workflows.Core/Abstractions/CodeActivity.cs`
- **Execution Contexts**: `src/modules/Elsa.Workflows.Core/Contexts/`
  - `ActivityExecutionContext.cs` - Activity-level state
  - `WorkflowExecutionContext.cs` - Workflow-level state
- **Pipeline Interface**: `src/modules/Elsa.Workflows.Core/Contracts/IWorkflowExecutionPipeline.cs`

**Persistence:**
- **Definition Store**: `src/modules/Elsa.Workflows.Management/Contracts/IWorkflowDefinitionStore.cs`
- **Instance Store**: `src/modules/Elsa.Workflows.Management/Contracts/IWorkflowInstanceStore.cs`
- **Workflow State**: `src/modules/Elsa.Workflows.Core/State/WorkflowState.cs`

**Expression System:**
- **Handler Interface**: `src/modules/Elsa.Expressions/Contracts/IExpressionHandler.cs`
- **Expression Modules**: `src/modules/Elsa.Expressions.{CSharp,JavaScript,Python,Liquid}/`

**Feature System:**
- **Feature Base**: `src/common/Elsa.Features/Abstractions/FeatureBase.cs`
- **Module Registration**: Feature-based architecture with dependency injection

**Sample Application:**
- **Entry Point**: `src/apps/Elsa.ServerAndStudio.Web/Program.cs` - Shows complete Elsa configuration

**Example Activities:**
- **HTTP Request**: `src/modules/Elsa.Http/Activities/SendHttpRequest.cs`
- **Flowchart**: `src/modules/Elsa.Workflows.Core/Activities/Flowchart/`
- **Sequence**: `src/modules/Elsa.Workflows.Core/Activities/Sequence.cs`

---

## High Level Architecture

### Technical Summary

Elsa Workflows is a comprehensive .NET workflow engine built on .NET 9 with modern C# features. It follows a layered, modular architecture with clear separation of concerns:

- **Core Layer**: Workflow execution engine, activity model, state management
- **Management Layer**: Workflow definition and instance persistence
- **Runtime Layer**: Long-running workflow support, bookmarks, triggers, actor model
- **Expression Layer**: Dynamic expression evaluation in multiple languages
- **API Layer**: REST endpoints for workflow management
- **Integration Layers**: HTTP, messaging, scheduling, etc.

### Actual Tech Stack

| Category             | Technology                    | Version     | Notes                                    |
|----------------------|-------------------------------|-------------|------------------------------------------|
| **Runtime**          | .NET                          | 9.0.6       | Latest LTS, C# 13, implicit usings      |
| **Language**         | C#                            | latest      | Nullable reference types enabled         |
| **Build System**     | NUKE                          | -           | `build.sh` / `build.cmd`                |
| **Testing**          | xUnit                         | 2.9.3       | Unit, integration, component tests       |
| **DI Container**     | Microsoft.Extensions.DI       | 9.0.0       | Built-in ASP.NET Core DI                |
| **Actor Model**      | Proto.Actor                   | 1.7.0       | For distributed workflow execution       |
| **Messaging**        | MassTransit                   | 8.4.1       | RabbitMQ, Azure Service Bus support     |
| **Scheduling**       | Quartz.NET                    | 3.14.0      | Cron-based workflow scheduling          |
| **Web Framework**    | ASP.NET Core                  | 9.0.6       | For API and designer hosting            |
| **API Framework**    | FastEndpoints                 | 6.2.0       | High-performance REST endpoints         |
| **Persistence**      | Multiple options:             |             |                                          |
|                      | - Entity Framework Core       | 9.0.6       | SQL Server, PostgreSQL, MySQL, SQLite   |
|                      | - MongoDB.Driver              | 3.4.0       | NoSQL option                            |
|                      | - Dapper                      | 2.1.66      | Lightweight SQL option                  |
| **Expression Eval**  | Multiple languages:           |             |                                          |
|                      | - Microsoft.CodeAnalysis      | 4.14.0      | C# scripting                            |
|                      | - Jint                        | 4.3.0       | JavaScript engine                       |
|                      | - pythonnet                   | 3.1.0       | Python integration                      |
|                      | - Fluid.Core                  | 2.24.0      | Liquid templates                        |
| **Designer**         | Elsa Studio (Blazor WASM)     | 3.6.0       | Separate project, embeddable            |
| **Serialization**    | System.Text.Json              | 9.0.0       | Primary JSON serializer                 |
|                      | Newtonsoft.Json               | 13.0.3      | Fallback for compatibility              |

### Repository Structure Reality Check

- **Type**: Monorepo with layered module organization
- **Package Manager**: NuGet with Central Package Management (CPM)
- **Build Tool**: NUKE for cross-platform builds
- **Structure**: Clear separation between core, modules, apps, tests

---

## Source Tree and Module Organization

### Project Structure (Actual)

```
elsa-core/
├── src/
│   ├── modules/                # Core modules - business logic
│   │   ├── Elsa.Workflows.Core/           # ⭐ Workflow execution engine
│   │   ├── Elsa.Workflows.Management/     # ⭐ Definition/instance persistence
│   │   ├── Elsa.Workflows.Runtime/        # ⭐ Long-running workflows, bookmarks
│   │   ├── Elsa.Workflows.Api/            # ⭐ REST API endpoints
│   │   ├── Elsa.Expressions.*/            # Expression language modules
│   │   ├── Elsa.Http/                     # HTTP activities and endpoints
│   │   ├── Elsa.Identity/                 # Authentication/authorization
│   │   ├── Elsa.Scheduling/               # Quartz.NET integration
│   │   └── [28 other modules]/            # Domain-specific modules
│   ├── common/                # Shared libraries
│   │   ├── Elsa.Features/                 # ⭐ Module/feature system
│   │   ├── Elsa.Mediator/                 # ⭐ CQRS-style messaging
│   │   ├── Elsa.Api.Common/               # Shared API infrastructure
│   │   └── Elsa.Testing.Shared/           # Testing utilities
│   ├── apps/                  # Reference applications
│   │   ├── Elsa.ServerAndStudio.Web/      # ⭐ Combined server + designer
│   │   ├── Elsa.Server.Web/               # Workflow server only
│   │   └── Elsa.Studio.Web/               # Designer only
│   └── clients/               # API clients
│       └── Elsa.Api.Client/               # C# client for Elsa API
├── test/
│   ├── unit/                  # Fast, isolated tests
│   ├── integration/           # Integration tests (DB, external deps)
│   ├── component/             # End-to-end component tests
│   └── performance/           # Performance benchmarks
├── build/                     # NUKE build system
│   └── Build.cs               # ⭐ Build targets and configuration
├── Directory.Build.props      # ⭐ Shared MSBuild properties
├── Directory.Packages.props   # ⭐ Central package management
└── Elsa.sln                   # ⭐ Main solution file
```

### Key Modules and Their Purpose

**Core Infrastructure:**
- **Elsa.Workflows.Core**: Activity model, execution engine, pipelines, state management
- **Elsa.Workflows.Management**: Workflow definition/instance CRUD operations
- **Elsa.Workflows.Runtime**: Bookmarks, triggers, distributed execution, actor model
- **Elsa.Workflows.Api**: REST API for workflow operations

**Expression Languages:**
- **Elsa.Expressions**: Base expression system and contracts
- **Elsa.Expressions.CSharp**: C# scripting via Roslyn
- **Elsa.Expressions.JavaScript**: JavaScript via Jint engine
- **Elsa.Expressions.Python**: Python via pythonnet
- **Elsa.Expressions.Liquid**: Liquid templates

**Integration Modules:**
- **Elsa.Http**: HTTP activities (SendHttpRequest, HttpEndpoint, WriteHttpResponse)
- **Elsa.Scheduling**: Quartz.NET for cron-based triggers
- **Elsa.Caching**: Caching abstractions and implementations
- **Elsa.MongoDb**: MongoDB persistence provider

**Common Libraries:**
- **Elsa.Features**: Feature-based module registration system
- **Elsa.Mediator**: CQRS pattern implementation (commands, notifications)
- **Elsa.Api.Common**: Shared API infrastructure (FastEndpoints)

---

## Core Abstractions

### Activity Model

**IActivity Interface** (`src/modules/Elsa.Workflows.Core/Contracts/IActivity.cs`):

```csharp
public interface IActivity
{
    string Id { get; set; }              // Unique within activity collection
    string NodeId { get; set; }          // Unique within workflow graph
    string? Name { get; set; }           // Optional friendly name
    string Type { get; set; }            // Activity type identifier
    int Version { get; set; }            // Activity version

    ValueTask<bool> CanExecuteAsync(ActivityExecutionContext context);
    ValueTask ExecuteAsync(ActivityExecutionContext context);
}
```

**CodeActivity Base Class** (`src/modules/Elsa.Workflows.Core/Abstractions/CodeActivity.cs`):

The recommended base class for custom activities with auto-complete behavior:

```csharp
public abstract class CodeActivity : Activity
{
    protected CodeActivity()
    {
        Behaviors.Add<AutoCompleteBehavior>(this);  // Auto-completes when ExecuteAsync returns
    }
}

// Generic version with typed result
public abstract class CodeActivity<T> : CodeActivity, IActivityWithResult<T>
{
    [Output]
    public Output<T>? Result { get; set; }
}
```

**Real-World Example** - HTTP Request Activity (`src/modules/Elsa.Http/Activities/SendHttpRequest.cs`):

```csharp
[Activity("Elsa", "HTTP", "Send an HTTP request.", DisplayName = "HTTP Request")]
public class SendHttpRequest : SendHttpRequestBase
{
    [Input(Description = "Expected status codes to handle")]
    public ICollection<HttpStatusCodeCase> ExpectedStatusCodes { get; set; }

    [Port]
    public IActivity? UnmatchedStatusCode { get; set; }

    protected override async ValueTask HandleResponseAsync(
        ActivityExecutionContext context,
        HttpResponseMessage response)
    {
        var statusCode = (int)response.StatusCode;
        var matchingCase = ExpectedStatusCodes.FirstOrDefault(x => x.StatusCode == statusCode);
        var activity = matchingCase?.Activity ?? UnmatchedStatusCode;

        await context.ScheduleActivityAsync(activity, OnChildActivityCompletedAsync);
    }
}
```

### Execution Contexts

**ActivityExecutionContext** (`src/modules/Elsa.Workflows.Core/Contexts/ActivityExecutionContext.cs`):

Activity-level execution state with:
- Access to parent workflow context
- Activity-specific state storage
- Expression evaluation context
- Scheduling capabilities for child activities
- Input/output handling

Key properties:
```csharp
public partial class ActivityExecutionContext
{
    public string Id { get; set; }
    public WorkflowExecutionContext WorkflowExecutionContext { get; }
    public ActivityExecutionContext? ParentActivityExecutionContext { get; set; }
    public IActivity Activity { get; }
    public ActivityStatus Status { get; set; }
    public ExpressionExecutionContext ExpressionExecutionContext { get; }

    // State management
    public ChangeTrackingDictionary<string, object> ActivityState { get; }
    public ChangeTrackingDictionary<string, object> ActivityInput { get; }

    // Scheduling
    public ValueTask ScheduleActivityAsync(IActivity? activity, ...);
    public ValueTask CompleteActivityAsync();
}
```

**WorkflowExecutionContext** (`src/modules/Elsa.Workflows.Core/Contexts/WorkflowExecutionContext.cs`):

Workflow-level execution state with:
- Workflow graph and root activity
- Bookmarks for resumption
- Activity scheduler
- Workflow-level properties and variables
- Status tracking

Key properties:
```csharp
public partial class WorkflowExecutionContext
{
    public string Id { get; set; }
    public string? CorrelationId { get; set; }
    public WorkflowGraph WorkflowGraph { get; }
    public WorkflowStatus Status { get; set; }
    public WorkflowSubStatus SubStatus { get; set; }

    // Bookmarks for long-running workflows
    public ICollection<Bookmark> Bookmarks { get; }

    // Activity scheduling
    public IActivityScheduler Scheduler { get; }

    // Input/Output
    public IDictionary<string, object> Input { get; }
    public IDictionary<string, object> Output { get; }

    // Factory methods
    public static async Task<WorkflowExecutionContext> CreateAsync(...);
}
```

### Pipeline Architecture

**IWorkflowExecutionPipeline** (`src/modules/Elsa.Workflows.Core/Contracts/IWorkflowExecutionPipeline.cs`):

Middleware-based execution pipeline:

```csharp
public interface IWorkflowExecutionPipeline
{
    WorkflowMiddlewareDelegate Pipeline { get; }
    Task ExecuteAsync(WorkflowExecutionContext context);
}

// Middleware delegate signature
public delegate ValueTask WorkflowMiddlewareDelegate(WorkflowExecutionContext context);
```

**Pipeline Composition**: Workflow execution flows through middleware components:
1. **Workflow Pipeline**: Pre/post workflow execution hooks
2. **Activity Pipeline**: Evaluation → Execution → Bookmarking → Logging

**Custom Middleware Example**:
```csharp
public class MyMiddleware : IActivityExecutionMiddleware
{
    public async ValueTask InvokeAsync(
        ActivityExecutionContext context,
        ActivityMiddlewareDelegate next)
    {
        // Before activity execution
        await next(context);
        // After activity execution
    }
}

// Register in feature
feature.WithActivityExecutionPipeline(pipeline =>
    pipeline.UseMiddleware<MyMiddleware>());
```

---

## Data Models and State Management

### Workflow State

**WorkflowState** (`src/modules/Elsa.Workflows.Core/State/WorkflowState.cs`):

Serializable snapshot of workflow execution:

```csharp
public class WorkflowState
{
    public string Id { get; set; }
    public string DefinitionId { get; set; }
    public string DefinitionVersionId { get; set; }
    public int DefinitionVersion { get; set; }

    // Status
    public WorkflowStatus Status { get; set; }
    public WorkflowSubStatus SubStatus { get; set; }

    // Long-running workflow support
    public ICollection<Bookmark> Bookmarks { get; set; }
    public ICollection<ActivityIncident> Incidents { get; set; }

    // Execution state
    public ICollection<ActivityExecutionContextState> ActivityExecutionContexts { get; set; }
    public ICollection<ActivityWorkItemState> ScheduledActivities { get; set; }
    public ICollection<CompletionCallbackState> CompletionCallbacks { get; set; }

    // Data
    public IDictionary<string, object> Input { get; set; }
    public IDictionary<string, object> Output { get; set; }
    public IDictionary<string, object> Properties { get; set; }
}
```

### Bookmarks

**Bookmark Model** (`src/modules/Elsa.Workflows.Core/Models/Bookmark.cs`):

Bookmarks enable workflow resumption after external events:

```csharp
public class Bookmark
{
    public string Id { get; set; }
    public string Name { get; set; }              // Bookmark type/name
    public string Hash { get; set; }              // Unique hash for matching
    public object? Payload { get; set; }          // Activity-specific data
    public string ActivityNodeId { get; set; }    // Which activity created it
    public string? CorrelationId { get; set; }    // For workflow correlation
}
```

**Usage Pattern**: Activities create bookmarks and suspend; external events resume via bookmark matching.

---

## Persistence Layer

### Store Abstractions

**IWorkflowDefinitionStore** (`src/modules/Elsa.Workflows.Management/Contracts/IWorkflowDefinitionStore.cs`):

Repository pattern for workflow definitions:

```csharp
public interface IWorkflowDefinitionStore
{
    Task<WorkflowDefinition?> FindAsync(WorkflowDefinitionFilter filter, ...);
    Task<Page<WorkflowDefinition>> FindManyAsync(WorkflowDefinitionFilter filter, PageArgs pageArgs, ...);
    Task SaveAsync(WorkflowDefinition definition, ...);
    Task<long> DeleteAsync(WorkflowDefinitionFilter filter, ...);
    Task<bool> AnyAsync(WorkflowDefinitionFilter filter, ...);
}
```

**IWorkflowInstanceStore** (`src/modules/Elsa.Workflows.Management/Contracts/IWorkflowInstanceStore.cs`):

Repository pattern for workflow instances:

```csharp
public interface IWorkflowInstanceStore
{
    ValueTask<WorkflowInstance?> FindAsync(WorkflowInstanceFilter filter, ...);
    ValueTask<Page<WorkflowInstance>> FindManyAsync(WorkflowInstanceFilter filter, PageArgs pageArgs, ...);
    ValueTask SaveAsync(WorkflowInstance instance, ...);
    ValueTask<long> DeleteAsync(WorkflowInstanceFilter filter, ...);
}
```

### Persistence Implementations

**Available Providers**:
- **Entity Framework Core**: SQL Server, PostgreSQL, MySQL, SQLite, Oracle
  - Location: `src/modules/Elsa.EntityFrameworkCore.*/`
- **MongoDB**: NoSQL option
  - Location: `src/modules/Elsa.MongoDb/`
- **Dapper**: Lightweight SQL (in development)
  - Location: `src/modules/Elsa.Dapper.*/`

**Filter-Based Queries**: All stores use filter objects for type-safe, composable queries:

```csharp
var filter = new WorkflowDefinitionFilter
{
    DefinitionId = "my-workflow",
    VersionOptions = VersionOptions.Latest
};
var definition = await definitionStore.FindAsync(filter);
```

---

## Expression System

### Expression Handler Interface

**IExpressionHandler** (`src/modules/Elsa.Expressions/Contracts/IExpressionHandler.cs`):

```csharp
public interface IExpressionHandler
{
    ValueTask<object?> EvaluateAsync(
        Expression expression,
        Type returnType,
        ExpressionExecutionContext context,
        ExpressionEvaluatorOptions options);
}
```

### Expression Languages

**1. C# Expressions** (`src/modules/Elsa.Expressions.CSharp/`):
- Uses Roslyn (Microsoft.CodeAnalysis.CSharp.Scripting)
- Full C# language support
- Access to workflow variables and context

**2. JavaScript Expressions** (`src/modules/Elsa.Expressions.JavaScript/`):
- Uses Jint JavaScript engine
- ECMAScript 5.1 compatible
- Configurable CLR access

**3. Python Expressions** (`src/modules/Elsa.Expressions.Python/`):
- Uses pythonnet for Python integration
- Requires Python runtime configuration
- Access to Python standard library

**4. Liquid Templates** (`src/modules/Elsa.Expressions.Liquid/`):
- Uses Fluid.Core for Liquid template engine
- Safe for user-generated content
- Common for text generation workflows

### Expression Syntax

Activities can use expressions in their inputs:

```csharp
[Activity("MyNamespace", "MyCategory")]
public class MyActivity : CodeActivity
{
    [Input(Description = "Dynamic input")]
    public Input<string> MyInput { get; set; } = new(new CSharpExpression("\"Hello \" + variable"));
}
```

---

## Module/Feature System

### Feature Base Class

**FeatureBase** (`src/common/Elsa.Features/Abstractions/FeatureBase.cs`):

```csharp
public abstract class FeatureBase : IFeature
{
    protected FeatureBase(IModule module)
    {
        Module = module;
    }

    public IModule Module { get; }
    public IServiceCollection Services => Module.Services;

    public virtual void Configure() { }
    public virtual void Apply() { }
    public virtual void ConfigureHostedServices() { }
}
```

### Feature Registration Pattern

**Custom Feature Example**:

```csharp
public class MyWorkflowFeature : FeatureBase
{
    public MyWorkflowFeature(IModule module) : base(module) { }

    public override void Apply()
    {
        // Register services
        Services.AddSingleton<IMyService, MyService>();
        Services.AddScoped<IMyRepository, MyRepository>();

        // Register activities
        Services.AddActivity<MyCustomActivity>();
    }

    public override void Configure()
    {
        // Configure feature behavior
        Module.Configure<MyOptions>(options =>
        {
            options.Setting = "value";
        });
    }
}

// In Program.cs
services.AddElsa(elsa =>
{
    elsa.AddFeature<MyWorkflowFeature>();
});
```

### Feature Dependencies

Features can declare dependencies:

```csharp
[DependsOn(typeof(WorkflowsCoreFeature))]
[DependsOn(typeof(WorkflowsApiFeature))]
public class MyFeature : FeatureBase
{
    // This feature requires core and API features
}
```

---

## Runtime Services

### Bookmark System

**IBookmarkResumer** (`src/modules/Elsa.Workflows.Runtime/Contracts/IBookmarkResumer.cs`):

Resumes workflows via bookmark matching:

```csharp
public interface IBookmarkResumer
{
    Task<ResumeBookmarkResult> ResumeAsync(
        string bookmarkId,
        ResumeBookmarkOptions? options = null,
        CancellationToken cancellationToken = default);
}
```

**Bookmark Flow**:
1. Activity creates bookmark during execution
2. Workflow suspends and saves state
3. External event triggers bookmark resolution
4. Runtime resumes workflow at bookmarked activity
5. Activity receives payload and continues

### Trigger System

**ITriggerIndexer** (`src/modules/Elsa.Workflows.Runtime/Contracts/ITriggerIndexer.cs`):

Indexes workflow triggers for automatic workflow invocation:

```csharp
public interface ITriggerIndexer
{
    Task IndexTriggersAsync(CancellationToken cancellationToken = default);
}
```

**Trigger Flow**:
1. Workflow definition contains trigger activities (e.g., HttpEndpoint, CronEvent)
2. Trigger indexer discovers and registers triggers
3. External events match triggers to workflow definitions
4. Runtime starts new workflow instances automatically

### Actor Model Integration

**Proto.Actor** (`src/modules/Elsa.Workflows.Runtime.ProtoActor/`):

Elsa integrates Proto.Actor for distributed workflow execution:

- **Virtual Actors**: Each workflow instance is an actor
- **Location Transparency**: Workflows can run on any node in cluster
- **Scalability**: Distribute workflows across multiple servers
- **Fault Tolerance**: Actor supervision and recovery

**Configuration** (from `Program.cs`):

```csharp
services.AddElsa(elsa =>
{
    elsa.UseWorkflowRuntime(runtime =>
    {
        // Actor model is enabled via Proto.Actor module
        runtime.UseProtoActor(protoActor =>
        {
            protoActor.PersistenceProvider = ...;
            protoActor.ClusterProvider = ...;
        });
    });
});
```

---

## API Layer

### FastEndpoints

**Elsa uses FastEndpoints** for high-performance REST APIs (`src/modules/Elsa.Workflows.Api/`):

```csharp
public class ExecuteWorkflowEndpoint : ElsaEndpoint<ExecuteWorkflowRequest, WorkflowState>
{
    public override void Configure()
    {
        Post("/workflow-definitions/{definitionId}/execute");
        ConfigurePermissions("execute:workflow");
    }

    public override async Task<WorkflowState> ExecuteAsync(
        ExecuteWorkflowRequest request,
        CancellationToken ct)
    {
        var workflowRuntime = Resolve<IWorkflowRuntime>();
        var result = await workflowRuntime.StartWorkflowAsync(
            request.DefinitionId,
            new StartWorkflowRuntimeParams
            {
                Input = request.Input,
                CorrelationId = request.CorrelationId
            },
            ct);

        return result.WorkflowState;
    }
}
```

### API Structure

**Key Endpoint Groups**:
- **Workflow Definitions**: CRUD operations on definitions
- **Workflow Instances**: Query and manage running instances
- **Workflow Execution**: Start, resume, cancel workflows
- **Activity Execution**: Execute individual activities
- **Bookmarks**: Query and resume bookmarks

---

## Embedding Elsa in Applications

### Console Application Example (for Bestedo)

**Basic Setup**:

```csharp
using Elsa.Extensions;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;

var builder = Host.CreateApplicationBuilder(args);

builder.Services.AddElsa(elsa =>
{
    elsa
        .UseWorkflowManagement()      // Workflow definition/instance management
        .UseWorkflowRuntime()          // Runtime services (bookmarks, triggers)
        .UseCSharp()                   // C# expressions
        .UseJavaScript()               // JavaScript expressions
        .AddActivitiesFrom<Program>()  // Auto-discover custom activities
        .AddWorkflowsFrom<Program>();  // Auto-discover coded workflows
});

var host = builder.Build();
await host.RunAsync();
```

**Running Workflows Programmatically**:

```csharp
// Get workflow runtime
var workflowRuntime = serviceProvider.GetRequiredService<IWorkflowRuntime>();

// Start workflow
var result = await workflowRuntime.StartWorkflowAsync(
    "my-workflow-definition-id",
    new StartWorkflowRuntimeParams
    {
        Input = new Dictionary<string, object>
        {
            ["taskId"] = "PROJ-123",
            ["parameters"] = new { ... }
        }
    });

// Check status
if (result.WorkflowState.Status == WorkflowStatus.Finished)
{
    Console.WriteLine("Workflow completed successfully");
    var output = result.WorkflowState.Output;
}
```

### ASP.NET Core Application

**Full Configuration** (from `src/apps/Elsa.ServerAndStudio.Web/Program.cs`):

```csharp
services.AddElsa(elsa =>
{
    elsa
        .UseIdentity(identity =>
        {
            identity.UseConfigurationBasedUserProvider(options => ...);
        })
        .UseDefaultAuthentication()
        .UseWorkflowManagement(management => management.UseCache())
        .UseWorkflowRuntime(runtime =>
        {
            runtime.UseCache();
            runtime.DistributedLockProvider = _ => new FileDistributedSynchronizationProvider(...);
        })
        .UseJavaScript()
        .UseLiquid()
        .UseCSharp()
        .UsePython()
        .UseHttp(http =>
        {
            http.ConfigureHttpOptions = options =>
            {
                options.BaseUrl = new Uri(configuration["Http:BaseUrl"]!);
            };
        })
        .UseWorkflowsApi()
        .AddActivitiesFrom<Program>()
        .AddWorkflowsFrom<Program>();
});

// Configure middleware
app.UseWorkflowsApi();
app.UseWorkflows();
```

---

## Development and Build Process

### NUKE Build System

**Build Commands** (`build.sh` / `build.cmd`):

```bash
# Default build (Compile target)
./build.sh

# Clean and compile
./build.sh Clean Compile

# Run all tests
./build.sh Test

# Create NuGet packages
./build.sh Pack

# Release configuration
./build.sh --configuration Release

# Code coverage
./build.sh Test --analyse-code
```

**Build Configuration** (`build/Build.cs`):

```csharp
partial class Build : NukeBuild, ITest, IPack
{
    public static int Main() => Execute<Build>(x => ((ICompile)x).Compile);

    Target Clean => _ => _
        .Before<IRestore>(x => x.Restore)
        .Executes(() => { ... });

    public IEnumerable<Project> TestProjects =>
        Solution.AllProjects.Where(x => x.Name.EndsWith("Tests"));
}
```

### Running Applications

**1. Combined Server + Designer** (recommended for development):
```bash
dotnet run --project src/apps/Elsa.ServerAndStudio.Web/Elsa.ServerAndStudio.Web.csproj
# Navigate to http://localhost:5000
# Login: admin / password
```

**2. Workflow Server Only**:
```bash
dotnet run --project src/apps/Elsa.Server.Web/Elsa.Server.Web.csproj
```

**3. Visual Designer Only**:
```bash
dotnet run --project src/apps/Elsa.Studio.Web/Elsa.Studio.Web.csproj
```

### Docker

```bash
docker pull elsaworkflows/elsa-server-and-studio-v3:latest
docker run -t -i \
  -e ASPNETCORE_ENVIRONMENT='Development' \
  -e HTTP_PORTS=8080 \
  -e HTTP__BASEURL=http://localhost:13000 \
  -p 13000:8080 \
  elsaworkflows/elsa-server-and-studio-v3:latest
```

---

## Testing Infrastructure

### Test Organization

**Structure** (`test/` directory):
- **unit/**: Fast, isolated unit tests (no external dependencies)
- **integration/**: Integration tests (databases, message queues)
- **component/**: End-to-end component tests
- **performance/**: Performance benchmarks (BenchmarkDotNet)

**Test Project Naming**: `[ProjectName].Tests` or `[ProjectName].UnitTests`

### Running Tests

**All tests**:
```bash
./build.sh Test
```

**Specific project**:
```bash
dotnet test test/unit/Elsa.Workflows.Core.UnitTests/Elsa.Workflows.Core.UnitTests.csproj
```

**Single test**:
```bash
dotnet test --filter "FullyQualifiedName=Namespace.ClassName.TestMethodName"
```

**Pattern matching**:
```bash
dotnet test --filter "FullyQualifiedName~Workflows"
```

---

## Extension Points

### Creating Custom Activities

**Step 1: Define Activity Class**:

```csharp
using Elsa.Workflows;
using Elsa.Workflows.Attributes;

[Activity("MyNamespace", "MyCategory", "Description of what it does")]
public class FetchTaskActivity : CodeActivity<TaskData>
{
    [Input(Description = "Task ID to fetch")]
    public Input<string> TaskId { get; set; } = default!;

    [Input(Description = "Task source provider")]
    public Input<string> Provider { get; set; } = default!;

    protected override async ValueTask ExecuteAsync(ActivityExecutionContext context)
    {
        var taskId = TaskId.Get(context);
        var provider = Provider.Get(context);

        // Your custom logic here
        var taskService = context.GetRequiredService<ITaskService>();
        var taskData = await taskService.FetchTaskAsync(taskId, provider);

        // Set output
        context.SetResult(taskData);
    }
}
```

**Step 2: Register Activity**:

Activities are auto-discovered via `AddActivitiesFrom<T>()`:

```csharp
services.AddElsa(elsa =>
{
    elsa.AddActivitiesFrom<Program>();  // Scans assembly for activities
});
```

Or register explicitly:

```csharp
services.AddActivity<FetchTaskActivity>();
```

### Creating Custom Expression Languages

**Implement IExpressionHandler**:

```csharp
public class MyExpressionHandler : IExpressionHandler
{
    public async ValueTask<object?> EvaluateAsync(
        Expression expression,
        Type returnType,
        ExpressionExecutionContext context,
        ExpressionEvaluatorOptions options)
    {
        var expressionString = expression.Value.ToString();

        // Your custom evaluation logic
        var result = await EvaluateMyLanguageAsync(expressionString, context);

        // Convert to expected return type
        return Convert.ChangeType(result, returnType);
    }
}

// Register
services.AddExpressionHandler<MyExpressionHandler>("mylang");
```

### Creating Custom Middleware

**Activity Middleware Example**:

```csharp
public class LoggingMiddleware : IActivityExecutionMiddleware
{
    private readonly ILogger<LoggingMiddleware> _logger;

    public LoggingMiddleware(ILogger<LoggingMiddleware> logger)
    {
        _logger = logger;
    }

    public async ValueTask InvokeAsync(
        ActivityExecutionContext context,
        ActivityMiddlewareDelegate next)
    {
        _logger.LogInformation("Executing activity: {ActivityType}", context.Activity.Type);
        var sw = Stopwatch.StartNew();

        await next(context);

        sw.Stop();
        _logger.LogInformation("Activity {ActivityType} completed in {Duration}ms",
            context.Activity.Type, sw.ElapsedMilliseconds);
    }
}

// Register
public class MyFeature : FeatureBase
{
    public override void Apply()
    {
        Module.WithActivityExecutionPipeline(pipeline =>
            pipeline.UseMiddleware<LoggingMiddleware>());
    }
}
```

### Custom Persistence Providers

**Implement Store Interfaces**:

```csharp
public class MyCustomWorkflowDefinitionStore : IWorkflowDefinitionStore
{
    public async Task<WorkflowDefinition?> FindAsync(
        WorkflowDefinitionFilter filter,
        CancellationToken cancellationToken = default)
    {
        // Your custom persistence logic
    }

    public async Task SaveAsync(
        WorkflowDefinition definition,
        CancellationToken cancellationToken = default)
    {
        // Your custom persistence logic
    }

    // Implement other methods...
}

// Register
services.AddScoped<IWorkflowDefinitionStore, MyCustomWorkflowDefinitionStore>();
```

---

## Integration Points

### HTTP Integration

**HTTP Activities** (`src/modules/Elsa.Http/`):
- **SendHttpRequest**: Make HTTP calls to external APIs
- **HttpEndpoint**: Expose workflows as HTTP endpoints
- **WriteHttpResponse**: Return HTTP responses from workflows
- **DownloadHttpFile**: Download files via HTTP

**Usage Example**:

```csharp
var workflow = new WorkflowBuilder()
    .WithHttpEndpoint("/api/webhook", HttpMethods.Post)
    .Then<ProcessWebhookData>()
    .Then<SendHttpRequest>(activity =>
    {
        activity.Url = new("https://api.example.com/notify");
        activity.Method = new(HttpMethods.Post);
        activity.Content = new(/* payload */);
    })
    .Build();
```

### Messaging Integration

**MassTransit Support** (`src/modules/Elsa.MassTransit/`):
- RabbitMQ
- Azure Service Bus
- In-memory for testing

**Usage**:

```csharp
services.AddElsa(elsa =>
{
    elsa.UseMassTransit(massTransit =>
    {
        massTransit.UseRabbitMq(configuration["RabbitMq:Host"]!);
    });
});
```

### Scheduling Integration

**Quartz.NET Support** (`src/modules/Elsa.Scheduling/`):
- Cron-based workflow triggers
- Scheduled workflow execution
- Persistent job storage

---

## Known Patterns and Gotchas

### Auto-Complete Behavior

**Pattern**: Activities derived from `CodeActivity` automatically complete when `ExecuteAsync` returns.

**Gotcha**: If you need manual completion (e.g., long-running activities with bookmarks), use `Activity` base class instead:

```csharp
public class ManualCompletionActivity : Activity
{
    protected override async ValueTask ExecuteAsync(ActivityExecutionContext context)
    {
        // Create bookmark for later resumption
        context.CreateBookmark();

        // DO NOT call CompleteActivityAsync() - workflow will suspend here
    }
}
```

### Bookmark Hashing

**Pattern**: Bookmarks use hash-based matching for resumption.

**Gotcha**: Bookmark payload must be deterministic for same logical bookmark:

```csharp
// Good - deterministic
var payload = new { TaskId = "123", Provider = "github" };

// Bad - includes timestamp, different every time
var payload = new { TaskId = "123", Timestamp = DateTime.Now };
```

### Variable Scope

**Pattern**: Variables are scoped to their containing activity (Sequence, Flowchart, etc.).

**Gotcha**: Child activities can access parent variables, but not sibling variables:

```csharp
var sequence = new Sequence
{
    Variables = { new Variable<string>("sharedVar") },
    Activities =
    {
        new SetVariable { Variable = "sharedVar", Value = new("Hello") },
        new WriteLine { Text = new(context => context.GetVariable<string>("sharedVar")) }
    }
};
```

---

## Performance Considerations

### Actor Model for Scale

**When to use**: Workflows that need to scale horizontally across multiple servers.

**Trade-off**: Additional complexity and infrastructure (cluster provider, persistence).

**Typical scenarios**:
- High-volume workflow execution (thousands per second)
- Long-running workflows that span days/weeks
- Multi-tenant scenarios with isolation requirements

### Caching

**Cache Layers**:
- Workflow definitions (rarely change)
- Activity descriptors (static metadata)
- Expression evaluation results (when deterministic)

**Configuration**:

```csharp
services.AddElsa(elsa =>
{
    elsa
        .UseWorkflowManagement(management => management.UseCache())
        .UseWorkflowRuntime(runtime => runtime.UseCache())
        .UseHttp(http => http.UseCache());
});
```

### Expression Evaluation

**Performance**: C# expressions are fastest (compiled), JavaScript is moderate, Python is slowest.

**Recommendation**: Use C# expressions for performance-critical paths.

---

## Contributing Guidelines

### Opening Issues

Before contributing, open an issue to discuss:
- New features or significant changes
- Alignment with project goals
- Implementation approach

See `CONTRIBUTING.md` for detailed guidelines.

### Coding Standards

**Project-Wide Standards** (from `Directory.Build.props`):
- .NET 9 target
- C# latest with implicit usings
- Nullable reference types enabled
- XML documentation required
- MIT license

**Code Style**:
- Follow `.editorconfig` settings
- Use meaningful names (no abbreviations)
- Document public APIs
- Write tests for new features

### Branch Strategy

**Trunk-Based Development**:
- Branch from `main`
- Submit PRs back to `main`
- No `git rebase` - use merge commits
- Follow conventional commit messages

---

## Resources for Learning

### Documentation

- **Official Docs**: https://docs.elsaworkflows.io/
- **GitHub**: https://github.com/elsa-workflows/elsa-core
- **Discord**: https://discord.gg/hhChk5H472
- **Stack Overflow**: Tag `elsa-workflows`

### Key Files to Study

**For Activity Development**:
1. `src/modules/Elsa.Workflows.Core/Abstractions/CodeActivity.cs`
2. `src/modules/Elsa.Http/Activities/SendHttpRequest.cs`
3. `src/modules/Elsa.Workflows.Core/Activities/Sequence.cs`

**For Workflow Execution**:
1. `src/modules/Elsa.Workflows.Core/Contexts/WorkflowExecutionContext.cs`
2. `src/modules/Elsa.Workflows.Core/Pipelines/WorkflowExecution/`
3. `src/modules/Elsa.Workflows.Runtime/Services/DefaultWorkflowRuntime.cs`

**For Embedding Elsa**:
1. `src/apps/Elsa.ServerAndStudio.Web/Program.cs`
2. `src/modules/Elsa.Features/`
3. `README.md`

### Sample Workflows

**Location**: `src/samples/`

Study sample projects to understand:
- How to define workflows programmatically
- How to use activities
- How to handle long-running workflows
- How to integrate with external systems

---

## Appendix - Useful Commands

### Build Commands

```bash
# Build solution
./build.sh

# Clean build
./build.sh Clean Compile

# Run tests
./build.sh Test

# Create packages
./build.sh Pack

# Release build with tests and code coverage
./build.sh --configuration Release Test --analyse-code Pack
```

### Development Commands

```bash
# Restore packages
dotnet restore

# Build specific project
dotnet build src/modules/Elsa.Workflows.Core/Elsa.Workflows.Core.csproj

# Run specific application
dotnet run --project src/apps/Elsa.ServerAndStudio.Web/Elsa.ServerAndStudio.Web.csproj

# Run tests with filter
dotnet test --filter "FullyQualifiedName~Workflows"

# Watch mode for continuous testing
dotnet watch test --project test/unit/Elsa.Workflows.Core.UnitTests/
```

### Git Commands

```bash
# Check status
git status

# Create feature branch
git checkout -b feature/my-feature

# Commit changes
git add .
git commit -m "feat: add new activity for task fetching"

# Push and create PR
git push -u origin feature/my-feature
gh pr create --title "Add task fetching activity" --body "Implements #123"

# DO NOT use rebase
# Instead, use merge commits
git merge main  # NOT git rebase main
```

### Docker Commands

```bash
# Pull latest image
docker pull elsaworkflows/elsa-server-and-studio-v3:latest

# Run with environment variables
docker run -t -i \
  -e ASPNETCORE_ENVIRONMENT='Development' \
  -e HTTP_PORTS=8080 \
  -e HTTP__BASEURL=http://localhost:13000 \
  -p 13000:8080 \
  elsaworkflows/elsa-server-and-studio-v3:latest

# Build custom image
docker build -t my-elsa-app -f docker/Dockerfile .
```

---

## Conclusion

This document provides a comprehensive overview of the Elsa Workflows architecture as it exists today. It captures:
- ✅ Real-world project structure and organization
- ✅ Core abstractions and design patterns
- ✅ Persistence and state management
- ✅ Expression system architecture
- ✅ Runtime services and actor model integration
- ✅ Extension points for customization
- ✅ Practical guidance for both using and contributing

**Next Steps for Bestedo Integration**:

1. **Study Activity Creation**: Start by creating custom activities for TaskSource, CodeHosting, and AiAgent integrations
2. **Embed Elsa Runtime**: Replace hardcoded Plan/Build/Review workflows with Elsa workflow definitions
3. **Leverage Long-Running Workflows**: Use bookmarks for waiting on external events (PR approval, build completion)
4. **Use Expression Languages**: Allow dynamic workflow configuration via expressions
5. **Contribute Back**: As you build Bestedo-specific features, consider generalizing and contributing back to Elsa

**For Contributors**:

- Focus on the areas most relevant to your interests
- Start with small, focused contributions
- Engage with the community on Discord
- Study existing code before proposing changes

This is a living document and will be updated as Elsa evolves. Happy workflow building! 🚀
