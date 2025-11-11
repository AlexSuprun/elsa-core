# Elsa Workflows - User Glossary

**Audience**: Developers building workflows with Elsa
**Purpose**: Quick reference for terms, entities, and concepts when using Elsa
**Companion Document**: See [Brownfield Architecture Document](./architecture.md) for detailed explanations

---

## Table of Contents

- [Core Terms](#core-terms)
- [Domain Entities](#domain-entities)
- [Visual Guides](#visual-guides)
  - [System Overview](#system-overview)
  - [Workflow Lifecycle](#workflow-lifecycle)
  - [Activity Execution](#activity-execution)
  - [Long-Running Workflows](#long-running-workflows)
  - [Expression System](#expression-system)
  - [Variable Scoping](#variable-scoping)
  - [HTTP Workflows](#http-workflows)
  - [Composite Activities](#composite-activities)
  - [Error Handling](#error-handling)
  - [Workflow Versioning](#workflow-versioning)
  - [Triggers and Events](#triggers-and-events)
  - [Workflow Correlation](#workflow-correlation)
  - [Activity Ports](#activity-ports)
  - [State Persistence](#state-persistence)
  - [Workflow Cancellation](#workflow-cancellation)
- [Cross-References](#cross-references)

---

## Core Terms

### Activity
A single unit of work in a workflow. Activities are the building blocks that perform operations like sending HTTP requests, writing to logs, performing calculations, or making decisions.

**Example**: `SendHttpRequest`, `WriteLine`, `SetVariable`, `If`

**See**: [Architecture Doc - Activity Model](./architecture.md#activity-model)

---

### Activity Node
The representation of an activity within a workflow graph. Each activity instance becomes a node with a unique `NodeId` in the workflow structure.

**Properties**:
- `NodeId` - Unique identifier within workflow graph
- `Activity` - The actual activity instance
- Connections to other nodes

---

### Bookmark
A resumption point in a workflow that allows it to pause and wait for an external event. When the event occurs, the workflow resumes from the bookmarked activity.

**Use Cases**:
- Waiting for HTTP requests (webhooks)
- Waiting for user input
- Waiting for message queue events
- Waiting for scheduled time

**Example**: An `HttpEndpoint` activity creates a bookmark, suspending the workflow until an HTTP request arrives.

**See**: [Architecture Doc - Bookmark System](./architecture.md#bookmark-system)

---

### Code Activity
A C# class that implements workflow logic. The recommended way to create custom activities by inheriting from `CodeActivity` or `CodeActivity<T>`.

**Example**:
```csharp
[Activity("MyNamespace", "MyCategory", "Fetches user data")]
public class FetchUserActivity : CodeActivity<User>
{
    [Input] public Input<string> UserId { get; set; }

    protected override async ValueTask ExecuteAsync(ActivityExecutionContext context)
    {
        var userId = UserId.Get(context);
        var user = await _userService.GetUserAsync(userId);
        context.SetResult(user);
    }
}
```

**See**: [Architecture Doc - Creating Custom Activities](./architecture.md#creating-custom-activities)

---

### Composite Activity
An activity that contains other activities (children). Examples include `Sequence`, `Flowchart`, `ForEach`, `While`, and `Parallel`.

**Use**: Organize complex workflows into logical units, create reusable workflow components.

---

### Correlation
A mechanism to match incoming events (like HTTP requests or messages) with the correct workflow instance. Uses a `CorrelationId` to identify related workflow executions.

**Example**: Order processing workflow where all events for order "ORD-123" are correlated using that order ID.

---

### Expression
A dynamic value that is evaluated at runtime. Elsa supports multiple expression languages:
- **C#**: `new CSharpExpression("input.Name.ToUpper()")`
- **JavaScript**: `new JavaScriptExpression("input.name.toUpperCase()")`
- **Python**: `new PythonExpression("input['name'].upper()")`
- **Liquid**: `new LiquidExpression("{{ input.name | upcase }}")`

**See**: [Architecture Doc - Expression System](./architecture.md#expression-system)

---

### Flowchart
A composite activity that executes child activities based on connections and conditions. Supports branching, looping, and parallel execution paths.

**Use**: Visual workflows, complex branching logic, decision trees.

---

### Input
A property on an activity that accepts a value from the workflow. Decorated with `[Input]` attribute. Can accept literal values or expressions.

**Example**:
```csharp
[Input(Description = "The URL to send request to")]
public Input<string> Url { get; set; }
```

---

### Long-Running Workflow
A workflow that can pause execution and resume later, typically using bookmarks. Unlike short-running workflows that complete immediately, long-running workflows can span hours, days, or weeks.

**Examples**:
- Order fulfillment (wait for payment, shipping, delivery)
- Approval processes (wait for manager approval)
- Scheduled tasks (wait until specific time)

**See**: [Architecture Doc - Long-Running Workflows](./architecture.md#long-running-workflows)

---

### Outcome
A named exit point from an activity. Activities can have multiple outcomes representing different execution paths (e.g., "Done", "True", "False", "Success", "Failed").

**Example**: An `If` activity has two outcomes: "True" and "False"

---

### Output
A property on an activity that produces a value. Decorated with `[Output]` attribute. Other activities can reference this output as input.

**Example**:
```csharp
[Output(Description = "The HTTP response")]
public Output<HttpResponseMessage> Response { get; set; }
```

---

### Port
A connection point on an activity that can be linked to another activity. Represented by activity properties decorated with `[Port]` attribute.

**Example**:
```csharp
[Port]
public IActivity? Then { get; set; }  // Next activity to execute
```

---

### Sequence
A composite activity that executes child activities in order, one after another. Simplest form of activity composition.

**Use**: Linear workflows, step-by-step processes.

---

### Trigger
An activity that starts a workflow automatically when an event occurs. Common triggers include:
- **HttpEndpoint**: Starts on HTTP request
- **Timer**: Starts on schedule (cron expression)
- **MessageReceived**: Starts on message queue event

**See**: [Architecture Doc - Trigger System](./architecture.md#trigger-system)

---

### Variable
A named storage location for workflow data. Variables are scoped to their containing activity (typically `Sequence` or `Flowchart`).

**Example**:
```csharp
var sequence = new Sequence
{
    Variables = { new Variable<string>("userName") },
    Activities = { /* activities that use userName */ }
};
```

---

### Workflow
A graph of connected activities that performs business logic. Can be defined in code, JSON, or visually through the designer.

**Types**:
- **Short-Running**: Completes immediately
- **Long-Running**: Can pause and resume
- **Singleton**: Only one instance can run at a time
- **Burst**: Multiple instances can run simultaneously

---

### Workflow Definition
The blueprint or template for a workflow. Contains the structure (activities, connections) but not execution state.

**Properties**:
- `DefinitionId` - Unique identifier for this workflow
- `Version` - Version number
- `IsLatest` - Whether this is the latest version
- `IsPublished` - Whether this version is published

---

### Workflow Instance
A running or completed execution of a workflow definition. Contains execution state, variables, bookmarks, and history.

**Properties**:
- `Id` - Unique instance identifier
- `DefinitionId` - Which definition this is an instance of
- `DefinitionVersion` - Which version was used
- `Status` - Current status (Running, Suspended, Finished, Faulted)
- `CorrelationId` - Optional correlation identifier

---

### Workflow Status
The execution state of a workflow instance:
- **Pending**: Created but not yet started
- **Running**: Currently executing
- **Suspended**: Paused, waiting for external event (bookmark)
- **Finished**: Completed successfully
- **Faulted**: Terminated due to error
- **Canceled**: Manually canceled

---

## Domain Entities

### Activity Entity

**Purpose**: Represents a single unit of work in a workflow

**Key Properties**:
- `Id` (string) - Unique within activity collection
- `NodeId` (string) - Unique within workflow graph
- `Name` (string?) - Optional friendly name
- `Type` (string) - Activity type identifier
- `Version` (int) - Activity version

**Key Methods**:
- `CanExecuteAsync()` - Check if activity can execute
- `ExecuteAsync()` - Execute the activity logic

**Relationships**:
- Contains: Input properties, Output properties
- Part of: Workflow graph
- Executed by: Activity execution pipeline

---

### Workflow Entity

**Purpose**: Container for workflow structure and execution

**Key Properties**:
- `Root` (IActivity) - Root activity of the workflow
- `Variables` (IEnumerable<Variable>) - Workflow-level variables
- `CustomProperties` (IDictionary) - Metadata

**Types**:
- Programmatic (C# code)
- JSON-defined
- Designer-created

**Lifecycle**: Definition → Instance → Execution → Completion/Suspension

---

### WorkflowDefinition Entity

**Purpose**: Persistent blueprint for workflows

**Key Properties**:
- `DefinitionId` (string) - Unique identifier
- `Version` (int) - Version number
- `IsLatest` (bool) - Latest version flag
- `IsPublished` (bool) - Published status
- `Name` (string?) - Display name
- `Description` (string?) - Description
- `CreatedAt` (DateTimeOffset) - Creation timestamp
- `MaterializerName` (string) - How to deserialize

**Storage**: Persisted via `IWorkflowDefinitionStore`

---

### WorkflowInstance Entity

**Purpose**: Represents a running or completed workflow execution

**Key Properties**:
- `Id` (string) - Unique instance identifier
- `DefinitionId` (string) - Which definition
- `DefinitionVersion` (int) - Which version
- `Status` (WorkflowStatus) - Current status
- `SubStatus` (WorkflowSubStatus) - Detailed status
- `CorrelationId` (string?) - For event correlation
- `CreatedAt` (DateTimeOffset) - Start time
- `FinishedAt` (DateTimeOffset?) - End time
- `WorkflowState` (WorkflowState) - Serialized state

**Storage**: Persisted via `IWorkflowInstanceStore`

---

### Bookmark Entity

**Purpose**: Resumption point for suspended workflows

**Key Properties**:
- `Id` (string) - Unique bookmark identifier
- `Name` (string) - Bookmark type/name
- `Hash` (string) - For matching with events
- `Payload` (object?) - Activity-specific data
- `ActivityNodeId` (string) - Which activity created it
- `CorrelationId` (string?) - For correlation

**Lifecycle**:
1. Activity creates bookmark
2. Workflow suspends
3. External event matches bookmark hash
4. Workflow resumes at bookmarked activity

---

### Input<T> Entity

**Purpose**: Activity input property wrapper

**Properties**:
- `MemoryBlockReference` - Where value is stored
- `Expression` - Dynamic expression for value

**Methods**:
- `Get(context)` - Retrieve value with evaluation
- `Set(value)` - Set literal value

**Example**:
```csharp
[Input]
public Input<string> Message { get; set; } = new("Hello");

// In activity:
var message = Message.Get(context);
```

---

### Output<T> Entity

**Purpose**: Activity output property wrapper

**Properties**:
- `MemoryBlockReference` - Where to store value

**Methods**:
- `Set(context, value)` - Set output value

**Example**:
```csharp
[Output]
public Output<HttpResponse> Response { get; set; }

// In activity:
Response.Set(context, httpResponse);
```

---

### Variable Entity

**Purpose**: Named storage for workflow data

**Key Properties**:
- `Id` (string) - Unique identifier
- `Name` (string?) - Variable name
- `Value` (object?) - Current value
- `StorageDriverType` (Type?) - How to persist

**Scoping**: Variables are scoped to their container (Sequence, Flowchart, Workflow)

**Access**:
```csharp
// Set variable
context.SetVariable("userName", "John");

// Get variable
var userName = context.GetVariable<string>("userName");

// In expressions
new CSharpExpression("userName.ToUpper()")
```

---

### Expression Entity

**Purpose**: Dynamic value evaluated at runtime

**Properties**:
- `Type` (string) - Expression language (C#, JavaScript, Python, Liquid)
- `Value` (object) - Expression string/object

**Languages**:
- **C#**: Full C# scripting via Roslyn
- **JavaScript**: ECMAScript via Jint engine
- **Python**: Python via pythonnet
- **Liquid**: Template engine via Fluid

**Example**:
```csharp
var expr = new CSharpExpression("input.Name + \" \" + input.LastName");
```

---

### Trigger Entity

**Purpose**: Automatically starts workflows on events

**Common Trigger Activities**:
- `HttpEndpoint` - HTTP request arrives
- `Timer` - Scheduled time/cron
- `MessageReceived` - Message queue event

**Mechanism**:
1. Trigger indexer discovers triggers in workflows
2. Runtime registers triggers
3. Events match triggers to workflow definitions
4. New workflow instances start automatically

---

### Connection Entity

**Purpose**: Links activities in a flowchart

**Properties**:
- `Source` (Endpoint) - Source activity and port
- `Target` (Endpoint) - Target activity and port

**Example**: Connect activity A's "Done" outcome to activity B

---

## Visual Guides

### System Overview

```mermaid
graph TB
    User["👤 User"]
    WD["📄 Workflow Definition<br/>(Blueprint)"]
    WI["⚙️ Workflow Instance<br/>(Execution)"]
    Runtime["🏃 Workflow Runtime"]
    Activities["🔧 Activities<br/>(SendHttp, WriteLine, etc.)"]
    Triggers["⏰ Triggers<br/>(HttpEndpoint, Timer)"]
    Bookmarks["📌 Bookmarks<br/>(Suspension Points)"]
    Storage["💾 Persistence<br/>(Database)"]

    User -->|"1. Create/Design"| WD
    User -->|"2. Start Manually"| Runtime
    Triggers -->|"Auto-Start"| Runtime
    Runtime -->|"3. Instantiate"| WI
    WI -->|"4. Execute"| Activities
    WI -->|"5. Suspend"| Bookmarks
    WI -.->|"Save State"| Storage
    Storage -.->|"Load State"| WI
    Bookmarks -->|"6. Resume"| Runtime

    style User fill:#e1f5fe
    style WD fill:#fff3e0
    style WI fill:#e8f5e9
    style Runtime fill:#f3e5f5
    style Activities fill:#fce4ec
    style Triggers fill:#fff9c4
    style Bookmarks fill:#ffebee
    style Storage fill:#e0f2f1
```

**Description**: High-level overview of how users interact with Elsa workflows, from definition to execution.

---

### Workflow Lifecycle

```mermaid
stateDiagram-v2
    [*] --> Created: Define Workflow
    Created --> Published: Publish
    Published --> Running: Start Instance
    Running --> Suspended: Create Bookmark
    Running --> Finished: Complete
    Running --> Faulted: Error Occurs
    Running --> Canceled: Cancel Request
    Suspended --> Running: Resume Bookmark
    Suspended --> Canceled: Cancel Request
    Finished --> [*]
    Faulted --> [*]
    Canceled --> [*]

    note right of Created
        Workflow Definition
        created but not active
    end note

    note right of Published
        Ready to be
        instantiated
    end note

    note right of Running
        Actively executing
        activities
    end note

    note right of Suspended
        Waiting for external
        event (bookmark)
    end note
```

**Description**: Complete lifecycle of a workflow from definition to completion, including suspension for long-running scenarios.

---

### Activity Execution

```mermaid
sequenceDiagram
    participant User
    participant Runtime
    participant Workflow
    participant Activity
    participant Pipeline

    User->>Runtime: StartWorkflowAsync()
    Runtime->>Workflow: Create Instance
    Runtime->>Workflow: ExecuteAsync()

    loop For each scheduled activity
        Workflow->>Pipeline: Execute Activity Pipeline
        Pipeline->>Activity: CanExecuteAsync()
        Activity-->>Pipeline: true
        Pipeline->>Activity: ExecuteAsync()

        alt Activity completes
            Activity-->>Pipeline: Complete
            Pipeline-->>Workflow: Activity Done
        else Activity creates bookmark
            Activity->>Workflow: CreateBookmark()
            Activity-->>Pipeline: Suspend
            Pipeline-->>Workflow: Suspended
        end
    end

    alt Workflow Finished
        Workflow-->>Runtime: Status: Finished
        Runtime-->>User: WorkflowState
    else Workflow Suspended
        Workflow->>Runtime: Save State
        Runtime-->>User: WorkflowState (Suspended)
    end
```

**Description**: Sequence of events when executing activities in a workflow, showing completion and suspension paths.

---

### Long-Running Workflows

```mermaid
graph TB
    Start["🚀 Start Workflow"]
    A1["Activity 1<br/>Process Order"]
    Bookmark["📌 Create Bookmark<br/>Wait for Payment"]
    Suspend["💤 Suspend Workflow<br/>Save State"]
    External["⏳ External Event<br/>Payment Received"]
    Resume["▶️ Resume Workflow"]
    A2["Activity 2<br/>Ship Order"]
    End["✅ Finish Workflow"]

    Start --> A1
    A1 --> Bookmark
    Bookmark --> Suspend
    Suspend -.->|"Hours/Days Later"| External
    External --> Resume
    Resume --> A2
    A2 --> End

    style Start fill:#e8f5e9
    style Bookmark fill:#fff3e0
    style Suspend fill:#ffebee
    style External fill:#e1f5fe
    style Resume fill:#f3e5f5
    style End fill:#c8e6c9
```

**Description**: How workflows pause with bookmarks and resume when external events occur.

**Use Cases**:
- Order fulfillment (wait for payment)
- Approval workflows (wait for manager)
- Scheduled tasks (wait until time)
- Webhook handlers (wait for callback)

---

### Expression System

```mermaid
graph LR
    Activity["🔧 Activity<br/>Input Property"]
    ExprType{"Expression<br/>Type?"}
    CSharp["C# Handler<br/>Roslyn Compiler"]
    JS["JavaScript Handler<br/>Jint Engine"]
    Python["Python Handler<br/>pythonnet"]
    Liquid["Liquid Handler<br/>Fluid Engine"]
    Result["✅ Evaluated<br/>Result"]

    Activity --> ExprType
    ExprType -->|"C#"| CSharp
    ExprType -->|"JavaScript"| JS
    ExprType -->|"Python"| Python
    ExprType -->|"Liquid"| Liquid
    CSharp --> Result
    JS --> Result
    Python --> Result
    Liquid --> Result

    style Activity fill:#e1f5fe
    style ExprType fill:#fff3e0
    style CSharp fill:#e8f5e9
    style JS fill:#fff9c4
    style Python fill:#e1bee7
    style Liquid fill:#b2dfdb
    style Result fill:#c8e6c9
```

**Description**: How expressions are evaluated at runtime using different language handlers.

**Example Usage**:
```csharp
// C# Expression
new Input<string>(new CSharpExpression("input.Name.ToUpper()"))

// JavaScript Expression
new Input<string>(new JavaScriptExpression("input.name.toUpperCase()"))

// Python Expression
new Input<string>(new PythonExpression("input['name'].upper()"))

// Liquid Expression
new Input<string>(new LiquidExpression("{{ input.name | upcase }}"))
```

---

### Variable Scoping

```mermaid
graph TB
    Workflow["🔄 Workflow<br/>Scope: workflowVar"]
    Seq1["📋 Sequence 1<br/>Scope: seq1Var"]
    Seq2["📋 Sequence 2<br/>Scope: seq2Var"]
    A1["Activity 1<br/>Can access:<br/>workflowVar, seq1Var"]
    A2["Activity 2<br/>Can access:<br/>workflowVar, seq1Var"]
    A3["Activity 3<br/>Can access:<br/>workflowVar, seq2Var"]

    Workflow --> Seq1
    Workflow --> Seq2
    Seq1 --> A1
    Seq1 --> A2
    Seq2 --> A3

    style Workflow fill:#e8f5e9
    style Seq1 fill:#fff3e0
    style Seq2 fill:#fff3e0
    style A1 fill:#e1f5fe
    style A2 fill:#e1f5fe
    style A3 fill:#e1f5fe
```

**Description**: Variables are scoped hierarchically. Child activities can access parent variables, but not sibling variables.

**Example**:
```csharp
var workflow = new Workflow
{
    Variables = { new Variable<string>("workflowVar") },
    Root = new Sequence
    {
        Variables = { new Variable<int>("seqVar") },
        Activities =
        {
            // Can access both workflowVar and seqVar
            new WriteLine { Text = new(ctx => ctx.GetVariable<string>("workflowVar")) },
            new WriteLine { Text = new(ctx => ctx.GetVariable<int>("seqVar").ToString()) }
        }
    }
};
```

---

### HTTP Workflows

```mermaid
sequenceDiagram
    participant Client
    participant HttpEndpoint
    participant Workflow
    participant SendHttp
    participant ExternalAPI
    participant WriteResponse

    Client->>HttpEndpoint: POST /api/webhook
    HttpEndpoint->>Workflow: Start Workflow Instance
    Workflow->>SendHttp: Execute SendHttpRequest
    SendHttp->>ExternalAPI: POST /external/api
    ExternalAPI-->>SendHttp: 200 OK Response
    SendHttp-->>Workflow: Response Data
    Workflow->>WriteResponse: Write HTTP Response
    WriteResponse-->>HttpEndpoint: Response Content
    HttpEndpoint-->>Client: 200 OK + Body
```

**Description**: Typical HTTP workflow pattern: receive request, process, call external API, return response.

**Example Workflow**:
```csharp
var workflow = new WorkflowBuilder()
    .WithHttpEndpoint("/api/process", HttpMethods.Post)
    .Then<ParseJsonActivity>()
    .Then<SendHttpRequest>(req =>
    {
        req.Url = new("https://api.external.com/data");
        req.Method = new(HttpMethods.Post);
    })
    .Then<WriteHttpResponse>(resp =>
    {
        resp.Content = new("Processed successfully");
    })
    .Build();
```

---

### Composite Activities

```mermaid
graph TB
    Root["🔄 Workflow Root"]
    Seq["📋 Sequence"]
    FC["🔀 Flowchart"]
    FE["🔁 ForEach"]

    A1["Activity 1"]
    A2["Activity 2"]
    A3["Activity 3"]

    Dec["💠 Decision"]
    B1["Activity B1"]
    B2["Activity B2"]

    Item1["Item Activity"]

    Root --> Seq
    Root --> FC
    Root --> FE

    Seq --> A1
    A1 --> A2
    A2 --> A3

    FC --> Dec
    Dec -->|"True"| B1
    Dec -->|"False"| B2

    FE --> Item1

    style Root fill:#e8f5e9
    style Seq fill:#fff3e0
    style FC fill:#e1f5fe
    style FE fill:#f3e5f5
    style Dec fill:#ffebee
```

**Description**: Composite activities contain other activities. Common types: Sequence (linear), Flowchart (branching), ForEach (iteration).

**Usage**:
- **Sequence**: Execute activities in order
- **Flowchart**: Complex branching logic, loops
- **ForEach**: Process collection of items
- **Parallel**: Execute activities simultaneously
- **While**: Loop while condition is true

---

### Error Handling

```mermaid
graph TB
    Activity["🔧 Activity Executes"]
    Success{"Success?"}
    Complete["✅ Complete Activity"]
    Error["❌ Error Occurs"]

    TryCatch{"In<br/>TryCatch?"}
    CatchActivity["🔧 Catch Activity<br/>Handle Error"]
    FaultFlow["⚠️ Fault Flow"]

    Incident["📝 Create Incident"]
    WorkflowFault["💥 Workflow Faulted"]

    Activity --> Success
    Success -->|"Yes"| Complete
    Success -->|"No"| Error

    Error --> TryCatch
    TryCatch -->|"Yes"| CatchActivity
    TryCatch -->|"No"| Incident

    CatchActivity --> Complete
    Incident --> FaultFlow
    FaultFlow --> WorkflowFault

    style Activity fill:#e1f5fe
    style Success fill:#fff3e0
    style Complete fill:#c8e6c9
    style Error fill:#ffcdd2
    style CatchActivity fill:#fff9c4
    style Incident fill:#ffebee
    style WorkflowFault fill:#ef5350
```

**Description**: Error handling flow in workflows, with optional TryCatch activities for graceful error recovery.

**Example**:
```csharp
new TryCatch
{
    Try = new SendHttpRequest { Url = new("https://api.example.com") },
    Catches =
    {
        new Catch<HttpRequestException>
        {
            Activity = new WriteLine { Text = new("HTTP request failed") }
        }
    }
}
```

---

### Workflow Versioning

```mermaid
graph LR
    V1["📄 Version 1<br/>IsPublished: true<br/>IsLatest: false"]
    V2["📄 Version 2<br/>IsPublished: true<br/>IsLatest: false"]
    V3["📄 Version 3<br/>IsPublished: true<br/>IsLatest: true"]
    Draft["📝 Draft<br/>IsPublished: false<br/>IsLatest: false"]

    I1["⚙️ Instance 1<br/>Version: 1"]
    I2["⚙️ Instance 2<br/>Version: 2"]
    I3["⚙️ Instance 3<br/>Version: 3"]

    V1 --> I1
    V2 --> I2
    V3 --> I3
    V3 -.->|"New Instances"| I3
    Draft -.->|"Publish"| V3

    style V1 fill:#e0e0e0
    style V2 fill:#e0e0e0
    style V3 fill:#c8e6c9
    style Draft fill:#fff3e0
    style I1 fill:#e1f5fe
    style I2 fill:#e1f5fe
    style I3 fill:#e1f5fe
```

**Description**: Multiple versions of a workflow definition can coexist. Running instances use their original version.

**Key Points**:
- Each version is immutable
- `IsLatest` flag marks newest version
- `IsPublished` controls availability
- Running instances don't auto-upgrade
- Draft versions can be edited

---

### Triggers and Events

```mermaid
graph TB
    WD1["📄 Workflow Definition 1<br/>Trigger: HttpEndpoint /api/order"]
    WD2["📄 Workflow Definition 2<br/>Trigger: Timer (daily at 9am)"]
    WD3["📄 Workflow Definition 3<br/>Trigger: MessageReceived (queue)"]

    Indexer["🔍 Trigger Indexer"]
    Registry["📋 Trigger Registry"]

    HTTP["🌐 HTTP Request<br/>/api/order"]
    Timer["⏰ Scheduled Time<br/>9:00 AM"]
    Message["📨 Message Arrives<br/>Queue Event"]

    Runtime["🏃 Workflow Runtime"]

    WD1 --> Indexer
    WD2 --> Indexer
    WD3 --> Indexer
    Indexer --> Registry

    HTTP --> Registry
    Timer --> Registry
    Message --> Registry

    Registry -->|"Match & Start"| Runtime
    Runtime -->|"New Instance"| WD1
    Runtime -->|"New Instance"| WD2
    Runtime -->|"New Instance"| WD3

    style Indexer fill:#fff3e0
    style Registry fill:#e8f5e9
    style Runtime fill:#e1f5fe
```

**Description**: How triggers automatically start workflow instances when events occur.

**Trigger Types**:
- **HttpEndpoint**: HTTP requests
- **Timer**: Scheduled/cron-based
- **MessageReceived**: Message queue events
- **Custom**: Implement `IBookmarkHandler`

---

### Workflow Correlation

```mermaid
sequenceDiagram
    participant Order1
    participant Order2
    participant Runtime
    participant WF1 as Workflow<br/>ORD-123
    participant WF2 as Workflow<br/>ORD-456

    Order1->>Runtime: Event (CorrelationId: ORD-123)
    Runtime->>WF1: Resume (matches ORD-123)

    Order2->>Runtime: Event (CorrelationId: ORD-456)
    Runtime->>WF2: Resume (matches ORD-456)

    Order1->>Runtime: Another Event (ORD-123)
    Runtime->>WF1: Resume again (same workflow)

    Note over Runtime: Correlation ensures events<br/>reach correct workflow instance
```

**Description**: Correlation matches events with the correct workflow instance using CorrelationId.

**Use Cases**:
- Order processing (all events for order ABC go to same workflow)
- User workflows (all events for user 123 go to same workflow)
- Multi-step processes (link related events together)

**Example**:
```csharp
// Start workflow with correlation
await runtime.StartWorkflowAsync(
    "order-workflow",
    new StartWorkflowRuntimeParams
    {
        CorrelationId = "ORD-123",
        Input = new { orderId = "ORD-123" }
    });

// Later, resume with same correlation
await runtime.ResumeBookmarkAsync(
    bookmarkId,
    new ResumeBookmarkOptions
    {
        CorrelationId = "ORD-123"
    });
```

---

### Activity Ports

```mermaid
graph LR
    If["💠 If Activity"]
    True["✅ True Port"]
    False["❌ False Port"]
    Done["✔️ Done Port"]

    A1["Activity A"]
    A2["Activity B"]
    A3["Activity C"]

    If -->|"Condition = true"| True
    If -->|"Condition = false"| False
    If -->|"Always"| Done

    True --> A1
    False --> A2
    Done --> A3

    style If fill:#e1f5fe
    style True fill:#c8e6c9
    style False fill:#ffcdd2
    style Done fill:#fff3e0
```

**Description**: Activities can have multiple ports (exit points) for different execution paths.

**Common Ports**:
- **Done**: Default completion port
- **True/False**: Conditional activities
- **Success/Failed**: Error handling
- **Custom**: Activity-specific outcomes

**Example**:
```csharp
var ifActivity = new If
{
    Condition = new(ctx => ctx.GetVariable<int>("age") >= 18),
    Then = new WriteLine { Text = new("Adult") },
    Else = new WriteLine { Text = new("Minor") }
};
```

---

### State Persistence

```mermaid
graph TB
    WF["⚙️ Workflow Instance<br/>Running"]

    Extract["📤 Extract State"]
    WS["📦 WorkflowState<br/>(Serializable)"]

    Store["💾 Persistence Store<br/>(Database)"]

    Load["📥 Load State"]
    Restore["🔄 Restore Workflow"]

    WF -->|"Suspend/Complete"| Extract
    Extract --> WS
    WS --> Store

    Store -->|"Resume Later"| Load
    Load --> WS
    WS --> Restore
    Restore -->|"Continue Execution"| WF

    style WF fill:#e8f5e9
    style Extract fill:#fff3e0
    style WS fill:#e1f5fe
    style Store fill:#e0f2f1
    style Load fill:#f3e5f5
    style Restore fill:#fff9c4
```

**Description**: How workflow state is persisted and restored for long-running workflows.

**WorkflowState Contains**:
- Workflow status and metadata
- Activity execution contexts (call stack)
- Scheduled activities
- Bookmarks
- Variables and properties
- Incidents (errors)

**Persistence Providers**:
- Entity Framework Core (SQL)
- MongoDB (NoSQL)
- Dapper (lightweight SQL)
- Custom (implement `IWorkflowInstanceStore`)

---

### Workflow Cancellation

```mermaid
stateDiagram-v2
    [*] --> Running
    Running --> Canceling: Cancel Request
    Canceling --> CleanupActivities: Cleanup Phase
    CleanupActivities --> Canceled: All Cleaned Up
    Canceled --> [*]

    Running --> Suspended: Bookmark Created
    Suspended --> Canceling: Cancel Request

    note right of Canceling
        Cancellation token
        propagates to activities
    end note

    note right of CleanupActivities
        Activities can perform
        cleanup operations
    end note
```

**Description**: Workflow cancellation flow, allowing graceful cleanup before termination.

**Cancellation Methods**:
```csharp
// Cancel via runtime
await runtime.CancelWorkflowAsync(workflowInstanceId);

// Check cancellation in activity
protected override async ValueTask ExecuteAsync(ActivityExecutionContext context)
{
    if (context.CancellationToken.IsCancellationRequested)
    {
        // Cleanup and exit
        return;
    }

    // Normal execution
}
```

---

## Cross-References

### Architecture Document Sections

For detailed explanations, see the [Brownfield Architecture Document](./architecture.md):

- **Activity Model**: [Core Abstractions - Activity Model](./architecture.md#activity-model)
- **Execution Contexts**: [Core Abstractions - Execution Contexts](./architecture.md#execution-contexts)
- **Bookmark System**: [Runtime Services - Bookmark System](./architecture.md#bookmark-system)
- **Trigger System**: [Runtime Services - Trigger System](./architecture.md#trigger-system)
- **Expression System**: [Expression System](./architecture.md#expression-system)
- **Persistence Layer**: [Persistence Layer](./architecture.md#persistence-layer)
- **Custom Activities**: [Extension Points - Creating Custom Activities](./architecture.md#creating-custom-activities)
- **Embedding Elsa**: [Embedding Elsa in Applications](./architecture.md#embedding-elsa-in-applications)

---

## Quick Reference Card

### Common Workflow Patterns

**Simple Sequential Workflow**:
```csharp
var workflow = new Sequence
{
    Activities =
    {
        new WriteLine { Text = new("Step 1") },
        new WriteLine { Text = new("Step 2") },
        new WriteLine { Text = new("Step 3") }
    }
};
```

**Conditional Workflow**:
```csharp
var workflow = new If
{
    Condition = new(ctx => ctx.GetInput<int>("age") >= 18),
    Then = new WriteLine { Text = new("Adult") },
    Else = new WriteLine { Text = new("Minor") }
};
```

**HTTP Webhook Workflow**:
```csharp
var workflow = new Sequence
{
    Activities =
    {
        new HttpEndpoint { Path = new("/webhook"), Method = new(HttpMethods.Post) },
        new ProcessDataActivity(),
        new WriteHttpResponse { Content = new("Success") }
    }
};
```

**Long-Running Approval Workflow**:
```csharp
var workflow = new Sequence
{
    Activities =
    {
        new SendNotification { Message = new("Approval needed") },
        new WaitForApproval(),  // Creates bookmark, suspends
        new ProcessApproval()
    }
};
```

---

**Document Version**: 1.0
**Last Updated**: 2025-11-11
**For**: Elsa Workflows v3
