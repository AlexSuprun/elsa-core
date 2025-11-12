# Elsa Workflows - Quick Reference Cheat Sheet

**Version**: Elsa v3
**Purpose**: Fast lookup for built-in activities, patterns, and common tasks
**Audience**: Developers building workflows with Elsa

---

## Table of Contents

- [Composite Activities](#composite-activities)
- [Control Flow](#control-flow)
- [Variables & Data](#variables--data)
- [Input/Output](#inputoutput)
- [HTTP Activities](#http-activities)
- [Scheduling & Timing](#scheduling--timing)
- [Workflow Orchestration](#workflow-orchestration)
- [Flowchart Activities](#flowchart-activities)
- [Other Activities](#other-activities)
- [Triggers](#triggers)
- [Expression Languages](#expression-languages)
- [Persistence Options](#persistence-options)
- [Common Patterns](#common-patterns)
- [Decision Trees](#decision-trees)

---

## Composite Activities

Activities that contain other activities.

### Sequence
**Purpose**: Execute activities in linear order, one after another
**Use When**: Simple step-by-step workflows
**Example**:
```csharp
new Sequence
{
    Activities =
    {
        new WriteLine { Text = new("Step 1") },
        new WriteLine { Text = new("Step 2") },
        new WriteLine { Text = new("Step 3") }
    }
}
```

---

### Flowchart
**Purpose**: Complex workflows with branching, loops, and parallel paths
**Use When**: Visual workflows, complex decision trees, state machines
**Example**:
```csharp
new Flowchart
{
    Start = startNode,
    Activities = { activity1, activity2, decision },
    Connections =
    {
        new Connection(startNode, activity1),
        new Connection(activity1, decision)
    }
}
```

---

### ForEach
**Purpose**: Iterate over a collection
**Use When**: Processing lists, arrays, or enumerables
**Inputs**: `Items` (collection)
**Outputs**: `CurrentIndex`, `CurrentValue`
**Example**:
```csharp
new ForEach<string>
{
    Items = new(new[] { "apple", "banana", "cherry" }),
    Body = new WriteLine
    {
        Text = new(context => context.GetVariable<string>("CurrentValue"))
    }
}
```

---

### For
**Purpose**: Traditional for-loop with counter
**Use When**: Iteration with index control
**Inputs**: `Start`, `End`, `Step`
**Outputs**: `CurrentValue`
**Example**:
```csharp
new For
{
    Start = new(0),
    End = new(10),
    Step = new(1),
    Body = new WriteLine { Text = new(context => $"Index: {context.GetVariable<int>("CurrentValue")}") }
}
```

---

### While
**Purpose**: Loop while condition is true
**Use When**: Conditional iteration
**Inputs**: `Condition`
**Example**:
```csharp
new While
{
    Condition = new(context => context.GetVariable<int>("counter") < 10),
    Body = new Sequence
    {
        Activities =
        {
            new WriteLine { Text = new("Looping...") },
            new SetVariable { Variable = "counter", Value = new(context => context.GetVariable<int>("counter") + 1) }
        }
    }
}
```

---

### Parallel
**Purpose**: Execute multiple activities simultaneously
**Use When**: Independent operations that can run concurrently
**Inputs**: `CompletionMode` (WaitAll, WaitAny)
**Example**:
```csharp
new Parallel
{
    CompletionMode = ParallelCompletionMode.WaitAll,
    Branches =
    {
        new WriteLine { Text = new("Branch 1") },
        new WriteLine { Text = new("Branch 2") },
        new SendHttpRequest { Url = new("https://api.example.com") }
    }
}
```

---

### ParallelForEach
**Purpose**: Process collection items in parallel
**Use When**: High-volume data processing, independent item operations
**Inputs**: `Items`, `MaxDegreeOfParallelism`
**Example**:
```csharp
new ParallelForEach<string>
{
    Items = new(urls),
    MaxDegreeOfParallelism = new(5),
    Body = new SendHttpRequest
    {
        Url = new(context => context.GetVariable<string>("CurrentValue"))
    }
}
```

---

## Control Flow

### If
**Purpose**: Conditional branching (if/then/else)
**Inputs**: `Condition` (bool)
**Outputs**: `Result` (bool)
**Ports**: `Then`, `Else`
**Example**:
```csharp
new If
{
    Condition = new(context => context.GetInput<int>("age") >= 18),
    Then = new WriteLine { Text = new("Adult") },
    Else = new WriteLine { Text = new("Minor") }
}
```

---

### Switch
**Purpose**: Multi-way branching based on conditions
**Inputs**: `Cases` (collection), `Mode` (MatchFirst/MatchAll)
**Ports**: `Default` (no match)
**Example**:
```csharp
new Switch
{
    Cases =
    {
        new SwitchCase { Condition = new(context => context.GetInput<string>("status") == "pending"), Activity = handlePending },
        new SwitchCase { Condition = new(context => context.GetInput<string>("status") == "approved"), Activity = handleApproved },
        new SwitchCase { Condition = new(context => context.GetInput<string>("status") == "rejected"), Activity = handleRejected }
    },
    Default = handleUnknown
}
```

---

### Break
**Purpose**: Exit from a loop (ForEach, For, While)
**Use When**: Early loop termination
**Example**:
```csharp
new ForEach<int>
{
    Items = new(numbers),
    Body = new If
    {
        Condition = new(context => context.GetVariable<int>("CurrentValue") == 5),
        Then = new Break()  // Exit loop when value is 5
    }
}
```

---

### Complete
**Purpose**: Complete workflow execution immediately
**Use When**: Early workflow termination with success
**Example**:
```csharp
new Complete()  // Workflow finishes here, skipping remaining activities
```

---

### Finish
**Purpose**: Mark workflow as finished (deprecated, use Complete)
**Use When**: Legacy workflows

---

### Fault
**Purpose**: Throw a fault/error in the workflow
**Inputs**: `Message`
**Use When**: Triggering error handling
**Example**:
```csharp
new Fault
{
    Message = new("Payment processing failed")
}
```

---

### End
**Purpose**: Mark the end of a workflow path (no effect on execution)
**Use When**: Visual indicator in flowcharts

---

## Variables & Data

### SetVariable
**Purpose**: Set or update a variable value
**Inputs**: `Variable` (name), `Value`
**Example**:
```csharp
new SetVariable
{
    Variable = "userName",
    Value = new("John Doe")
}

// Or with expression
new SetVariable
{
    Variable = "fullName",
    Value = new(context => $"{context.GetVariable<string>("firstName")} {context.GetVariable<string>("lastName")}")
}
```

---

### SetName
**Purpose**: Set workflow instance name
**Inputs**: `Value` (name)
**Use When**: Dynamic workflow naming
**Example**:
```csharp
new SetName
{
    Value = new(context => $"Order-{context.GetInput<string>("orderId")}")
}
```

---

### Correlate
**Purpose**: Set or update workflow correlation ID
**Inputs**: `CorrelationId`
**Use When**: Linking related workflow events
**Example**:
```csharp
new Correlate
{
    CorrelationId = new(context => context.GetInput<string>("orderId"))
}
```

---

## Input/Output

### WriteLine
**Purpose**: Write text to standard output
**Inputs**: `Text`
**Use When**: Console logging, debugging
**Example**:
```csharp
new WriteLine
{
    Text = new("Hello, Elsa!")
}

// With variable
new WriteLine
{
    Text = new(context => $"User: {context.GetVariable<string>("userName")}")
}
```

---

### ReadLine
**Purpose**: Read text from standard input
**Outputs**: `Result` (string)
**Use When**: Console input (testing/debugging)
**Example**:
```csharp
new ReadLine
{
    Result = new("userInput")
}
```

---

## HTTP Activities

### HttpEndpoint
**Purpose**: Create HTTP endpoint that triggers workflow
**Inputs**: `Path`, `Method`, `ReadContent`
**Outputs**: `Body`, `Headers`, `QueryString`
**Use When**: Webhooks, REST API endpoints
**Example**:
```csharp
new HttpEndpoint
{
    Path = new("/api/orders"),
    Method = new(HttpMethods.Post),
    ReadContent = new(true)
}
```

---

### SendHttpRequest
**Purpose**: Send HTTP request to external API
**Inputs**: `Url`, `Method`, `Content`, `Headers`, `Authorization`
**Outputs**: `Response`, `StatusCode`, `Content`
**Ports**: Status code ports (200, 404, etc.), `UnmatchedStatusCode`, `FailedToConnect`, `Timeout`
**Example**:
```csharp
new SendHttpRequest
{
    Url = new("https://api.example.com/users"),
    Method = new(HttpMethods.Get),
    Headers = new(new Dictionary<string, string>
    {
        ["Authorization"] = "Bearer token123"
    }),
    ExpectedStatusCodes =
    {
        new HttpStatusCodeCase
        {
            StatusCode = 200,
            Activity = new WriteLine { Text = new("Success!") }
        },
        new HttpStatusCodeCase
        {
            StatusCode = 404,
            Activity = new WriteLine { Text = new("Not Found") }
        }
    },
    UnmatchedStatusCode = new WriteLine { Text = new("Unexpected status") }
}
```

---

### WriteHttpResponse
**Purpose**: Write HTTP response in workflow
**Inputs**: `Content`, `ContentType`, `StatusCode`, `Headers`
**Use When**: Responding to HttpEndpoint
**Example**:
```csharp
new WriteHttpResponse
{
    Content = new("{\"status\": \"success\"}"),
    ContentType = new("application/json"),
    StatusCode = new(200)
}
```

---

### WriteFileHttpResponse
**Purpose**: Send file as HTTP response
**Inputs**: `File`, `ContentType`, `DownloadFileName`
**Use When**: File downloads
**Example**:
```csharp
new WriteFileHttpResponse
{
    File = new(fileBytes),
    ContentType = new("application/pdf"),
    DownloadFileName = new("report.pdf")
}
```

---

### DownloadHttpFile
**Purpose**: Download file from URL
**Inputs**: `Url`, `Filename`
**Outputs**: `File`, `Content`
**Example**:
```csharp
new DownloadHttpFile
{
    Url = new("https://example.com/file.pdf"),
    Filename = new("/downloads/file.pdf")
}
```

---

## Scheduling & Timing

### Timer
**Purpose**: Trigger workflow at specific time
**Inputs**: `ExecuteAt` (DateTimeOffset)
**Use When**: One-time scheduled execution
**Example**:
```csharp
new Timer
{
    ExecuteAt = new(DateTimeOffset.UtcNow.AddHours(24))
}
```

---

### Cron
**Purpose**: Trigger workflow on cron schedule
**Inputs**: `CronExpression`
**Use When**: Recurring scheduled tasks
**Example**:
```csharp
new Cron
{
    CronExpression = new("0 9 * * *")  // Daily at 9 AM
}

// Common patterns:
// "*/5 * * * *"     - Every 5 minutes
// "0 0 * * *"      - Daily at midnight
// "0 0 * * 0"      - Weekly on Sunday midnight
// "0 0 1 * *"      - Monthly on 1st at midnight
```

---

### StartAt
**Purpose**: Delay workflow start until specific time
**Inputs**: `ExecuteAt`
**Use When**: Deferred workflow execution
**Example**:
```csharp
new StartAt
{
    ExecuteAt = new(DateTimeOffset.Parse("2025-12-31T23:59:59Z"))
}
```

---

### Delay
**Purpose**: Pause workflow for duration
**Inputs**: `Duration` (TimeSpan)
**Use When**: Waiting between operations
**Example**:
```csharp
new Delay
{
    Duration = new(TimeSpan.FromMinutes(5))
}

// Or dynamic
new Delay
{
    Duration = new(context => TimeSpan.FromSeconds(context.GetInput<int>("waitSeconds")))
}
```

---

## Workflow Orchestration

### DispatchWorkflow
**Purpose**: Start another workflow asynchronously (fire-and-forget)
**Inputs**: `WorkflowDefinitionId`, `Input`, `CorrelationId`
**Use When**: Parent-child workflows, event-driven architectures
**Example**:
```csharp
new DispatchWorkflow
{
    WorkflowDefinitionId = new("order-processing"),
    Input = new(new Dictionary<string, object>
    {
        ["orderId"] = "ORD-123",
        ["customerId"] = "CUST-456"
    }),
    CorrelationId = new("ORD-123")
}
```

---

### ExecuteWorkflow
**Purpose**: Execute another workflow and wait for completion
**Inputs**: `WorkflowDefinitionId`, `Input`
**Outputs**: `Result` (output from child workflow)
**Use When**: Reusable sub-workflows, workflow composition
**Example**:
```csharp
new ExecuteWorkflow
{
    WorkflowDefinitionId = new("validate-order"),
    Input = new(new Dictionary<string, object>
    {
        ["order"] = orderData
    }),
    Result = new("validationResult")
}
```

---

### Event
**Purpose**: Wait for external event (creates bookmark)
**Inputs**: `EventName`
**Outputs**: `Payload`
**Use When**: Waiting for callbacks, webhooks, messages
**Example**:
```csharp
new Event
{
    EventName = new("PaymentReceived"),
    Payload = new("paymentData")
}
```

---

### PublishEvent
**Purpose**: Publish event to resume waiting workflows
**Inputs**: `EventName`, `Payload`, `CorrelationId`
**Use When**: Triggering event-based workflows
**Example**:
```csharp
new PublishEvent
{
    EventName = new("PaymentReceived"),
    Payload = new(paymentData),
    CorrelationId = new("ORD-123")
}
```

---

### BulkDispatchWorkflows
**Purpose**: Start multiple workflows in bulk
**Inputs**: `WorkflowDefinitionIds`, `Input`
**Use When**: Batch workflow processing
**Example**:
```csharp
new BulkDispatchWorkflows
{
    WorkflowDefinitionIds = new(new[] { "workflow1", "workflow2", "workflow3" }),
    Input = new(sharedData)
}
```

---

### RunTask
**Purpose**: Execute background task
**Inputs**: `TaskName`, `Payload`
**Use When**: Long-running background operations

---

## Flowchart Activities

Special activities used within Flowcharts.

### FlowDecision
**Purpose**: Decision node in flowchart (if/then)
**Inputs**: `Condition`
**Ports**: `True`, `False`
**Example**:
```csharp
var decision = new FlowDecision
{
    Condition = new(context => context.GetVariable<int>("amount") > 1000)
};
```

---

### FlowSwitch
**Purpose**: Multi-way decision node
**Inputs**: `Expression`, `Cases`
**Ports**: One port per case + Default
**Example**:
```csharp
var flowSwitch = new FlowSwitch
{
    Expression = new(context => context.GetVariable<string>("status")),
    Cases =
    {
        ["pending"] = pendingActivity,
        ["approved"] = approvedActivity,
        ["rejected"] = rejectedActivity
    },
    Default = unknownActivity
};
```

---

### FlowJoin
**Purpose**: Synchronization point (wait for multiple paths)
**Inputs**: `Mode` (WaitAll, WaitAny)
**Use When**: Merging parallel branches
**Example**:
```csharp
var join = new FlowJoin
{
    Mode = FlowJoinMode.WaitAll  // Wait for all incoming branches
};
```

---

### FlowFork
**Purpose**: Split execution into multiple parallel paths
**Use When**: Parallel branching in flowcharts

---

### FlowNode
**Purpose**: Container for activities in flowcharts
**Use When**: Wrapping activities for flowchart use

---

## Other Activities

### CreateZipArchive
**Module**: Elsa.IO.Compression
**Purpose**: Create ZIP archive from files
**Inputs**: `Files`, `ArchivePath`
**Example**:
```csharp
new CreateZipArchive
{
    Files = new(filePaths),
    ArchivePath = new("/archives/backup.zip")
}
```

---

### Fork
**Purpose**: Split execution into multiple branches (all execute)
**Use When**: Parallel non-blocking branches

---

### Inline
**Purpose**: Execute inline code
**Use When**: Quick custom logic without creating activity

---

### DynamicActivity
**Purpose**: Activity whose behavior is determined at runtime
**Use When**: Advanced scenarios, plugin systems

---

### Composite
**Purpose**: Base class for creating composite activities
**Use When**: Building reusable activity containers

---

### Container
**Purpose**: Generic container activity
**Use When**: Grouping activities

---

## Triggers

Activities that start workflows automatically.

### HttpEndpoint (Trigger)
**Starts When**: HTTP request arrives at path
**Example**: `/api/webhook`

### Timer (Trigger)
**Starts When**: Specified time reached
**Example**: Execute at 2025-01-01 00:00:00

### Cron (Trigger)
**Starts When**: Cron expression matches
**Example**: `0 9 * * *` (daily at 9 AM)

### Event (Trigger)
**Starts When**: Matching event published
**Example**: `PaymentReceived` event

---

## Expression Languages

### C# Expressions
**Syntax**: Roslyn C# scripting
**Performance**: Fastest (compiled)
**Use When**: Complex logic, type safety
**Example**:
```csharp
new Input<string>(new CSharpExpression("input.Name.ToUpper() + \" \" + input.Age.ToString()"))
```

**Available Context**:
- `input` - Workflow input
- Variables (by name)
- Activity outputs

---

### JavaScript Expressions
**Syntax**: ECMAScript 5.1 (Jint)
**Performance**: Medium
**Use When**: Simple logic, dynamic workflows
**Example**:
```csharp
new Input<string>(new JavaScriptExpression("input.name.toUpperCase() + ' ' + input.age"))
```

---

### Python Expressions
**Syntax**: Python via pythonnet
**Performance**: Slower (requires Python runtime)
**Use When**: Python integration, ML models
**Example**:
```csharp
new Input<string>(new PythonExpression("input['name'].upper() + ' ' + str(input['age'])"))
```

---

### Liquid Expressions
**Syntax**: Liquid templating
**Performance**: Medium
**Use When**: Text generation, emails, safe templates
**Example**:
```csharp
new Input<string>(new LiquidExpression("Hello {{ input.name | upcase }}!"))
```

---

## Persistence Options

### Entity Framework Core
**Databases**: SQL Server, PostgreSQL, MySQL, SQLite, Oracle
**Features**: Full-featured, migrations, LINQ
**Use When**: Relational databases, complex queries
**Setup**:
```csharp
services.AddElsa(elsa =>
{
    elsa.UseEntityFrameworkCore(ef =>
    {
        ef.UseSqlite("Data Source=elsa.db");
    });
});
```

---

### MongoDB
**Database**: MongoDB (NoSQL)
**Features**: Schemaless, horizontal scaling
**Use When**: Document storage, high scalability
**Setup**:
```csharp
services.AddElsa(elsa =>
{
    elsa.UseMongoDB("mongodb://localhost:27017/elsa");
});
```

---

### Dapper
**Databases**: Any SQL database
**Features**: Lightweight, performance-focused
**Use When**: Simple persistence, high performance
**Setup**:
```csharp
services.AddElsa(elsa =>
{
    elsa.UseDapper(dapper =>
    {
        dapper.UseSqlServer("connection-string");
    });
});
```

---

## Common Patterns

### 1. HTTP Webhook Handler
```csharp
new Sequence
{
    Activities =
    {
        new HttpEndpoint { Path = new("/webhook"), Method = new(HttpMethods.Post) },
        new SetVariable { Variable = "data", Value = new(context => context.GetInput<object>("Body")) },
        new SendHttpRequest { Url = new("https://api.example.com/process") },
        new WriteHttpResponse { Content = new("{\"status\": \"success\"}"), ContentType = new("application/json") }
    }
}
```

---

### 2. Approval Workflow (Long-Running)
```csharp
new Sequence
{
    Activities =
    {
        new SendHttpRequest { Url = new("https://api.example.com/notify-manager") },
        new Event { EventName = new("ApprovalReceived") },  // Creates bookmark, suspends
        new If
        {
            Condition = new(context => context.GetVariable<bool>("approved")),
            Then = new WriteLine { Text = new("Approved!") },
            Else = new WriteLine { Text = new("Rejected") }
        }
    }
}
```

---

### 3. Parallel API Calls
```csharp
new Parallel
{
    CompletionMode = ParallelCompletionMode.WaitAll,
    Branches =
    {
        new SendHttpRequest { Url = new("https://api1.example.com") },
        new SendHttpRequest { Url = new("https://api2.example.com") },
        new SendHttpRequest { Url = new("https://api3.example.com") }
    }
}
```

---

### 4. Retry with Delay
```csharp
new Sequence
{
    Variables = { new Variable<int>("retryCount") },
    Activities =
    {
        new While
        {
            Condition = new(context => context.GetVariable<int>("retryCount") < 3),
            Body = new Sequence
            {
                Activities =
                {
                    new SendHttpRequest { Url = new("https://api.example.com") },
                    new If
                    {
                        Condition = new(context => /* check if success */),
                        Then = new Break(),
                        Else = new Sequence
                        {
                            Activities =
                            {
                                new Delay { Duration = new(TimeSpan.FromSeconds(5)) },
                                new SetVariable { Variable = "retryCount", Value = new(context => context.GetVariable<int>("retryCount") + 1) }
                            }
                        }
                    }
                }
            }
        }
    }
}
```

---

### 5. Data Processing Pipeline
```csharp
new Sequence
{
    Activities =
    {
        new HttpEndpoint { Path = new("/upload"), Method = new(HttpMethods.Post) },
        new SetVariable { Variable = "data", Value = new(context => context.GetInput<string>("Body")) },
        new ForEach<DataItem>
        {
            Items = new(context => ParseData(context.GetVariable<string>("data"))),
            Body = new Sequence
            {
                Activities =
                {
                    new WriteLine { Text = new(context => $"Processing: {context.GetVariable<DataItem>("CurrentValue").Id}") },
                    new SendHttpRequest { Url = new("https://api.example.com/process") }
                }
            }
        },
        new WriteHttpResponse { Content = new("Processing complete") }
    }
}
```

---

### 6. Scheduled Daily Report
```csharp
new Sequence
{
    Activities =
    {
        new Cron { CronExpression = new("0 9 * * *") },  // Daily at 9 AM
        new SendHttpRequest { Url = new("https://api.example.com/get-data") },
        new SetVariable { Variable = "reportData", Value = new(context => context.GetVariable<object>("ResponseContent")) },
        new SendHttpRequest
        {
            Url = new("https://email-service.com/send"),
            Content = new(context => GenerateReport(context.GetVariable<object>("reportData")))
        }
    }
}
```

---

## Decision Trees

### When to Use Which Composite Activity?

```
Need to execute activities?
│
├─ In order? → Use Sequence
│
├─ With complex branching/loops? → Use Flowchart
│
├─ Over a collection?
│  ├─ Sequentially? → Use ForEach
│  └─ In parallel? → Use ParallelForEach
│
├─ With a counter? → Use For
│
├─ While condition is true? → Use While
│
└─ Independently in parallel? → Use Parallel
```

---

### When to Use Which Control Flow?

```
Need to make a decision?
│
├─ Two branches (yes/no)? → Use If
│
└─ Multiple conditions?
   ├─ Check first match? → Use Switch (Mode: MatchFirst)
   └─ Check all matches? → Use Switch (Mode: MatchAll)
```

---

### When to Use Which Trigger?

```
How should workflow start?
│
├─ On HTTP request? → Use HttpEndpoint
│
├─ At specific time? → Use Timer
│
├─ On schedule (recurring)? → Use Cron
│
├─ On event/message? → Use Event
│
└─ Manually via API? → Use WorkflowRuntime.StartWorkflowAsync()
```

---

### When to Use Which Persistence?

```
What database do you have?
│
├─ SQL Server/PostgreSQL/MySQL/SQLite?
│  ├─ Full ORM features needed? → Entity Framework Core
│  └─ Performance-focused? → Dapper
│
├─ MongoDB? → MongoDB provider
│
└─ Custom database? → Implement IWorkflowInstanceStore
```

---

### When to Use Which Expression Language?

```
What's your use case?
│
├─ Complex logic, type safety? → C# (fastest)
│
├─ Simple logic, dynamic workflows? → JavaScript
│
├─ Python integration, ML? → Python (slowest)
│
└─ Text generation, safe templates? → Liquid
```

---

## Quick Tips

### Variable Access
```csharp
// Get variable
var value = context.GetVariable<string>("variableName");

// Set variable
context.SetVariable("variableName", "value");

// In expression
new Input<string>(new CSharpExpression("variableName.ToUpper()"))
```

---

### Workflow Input/Output
```csharp
// Access workflow input
var orderId = context.GetInput<string>("orderId");

// Set workflow output
context.SetOutput("result", resultData);
```

---

### Activity Output
```csharp
// Define output in activity
[Output]
public Output<string> Result { get; set; }

// Set output
Result.Set(context, "output value");

// Access in another activity
var previousResult = context.GetVariable<string>("Result");
```

---

### Correlation
```csharp
// Start workflow with correlation
await runtime.StartWorkflowAsync("workflow-id", new StartWorkflowRuntimeParams
{
    CorrelationId = "ORD-123",
    Input = new Dictionary<string, object> { ["orderId"] = "ORD-123" }
});

// Set correlation in workflow
new Correlate { CorrelationId = new("ORD-123") }

// Resume with correlation
await runtime.ResumeBookmarkAsync(bookmarkId, new ResumeBookmarkOptions
{
    CorrelationId = "ORD-123"
});
```

---

### Error Handling
```csharp
// Throw fault
new Fault { Message = new("Something went wrong") }

// Handle with TryCatch (if available)
// Or use incident tracking via workflow state
```

---

## Performance Tips

1. **Use C# expressions** for performance-critical paths (compiled, not interpreted)
2. **ParallelForEach** for high-volume data processing
3. **Caching**: Enable for workflow definitions and activity descriptors
4. **Batch operations**: Use BulkDispatchWorkflows for multiple workflow starts
5. **Actor model**: Use Proto.Actor for horizontal scaling
6. **Lightweight persistence**: Consider Dapper for simple scenarios

---

## Common Gotchas

1. **Variable scope**: Variables are scoped to their container (Sequence, Flowchart)
2. **Bookmark payload**: Must be JSON-serializable and deterministic
3. **Auto-complete**: `CodeActivity` completes automatically; use `Activity` for bookmarks
4. **Expression context**: Not all variables are available in all expressions
5. **Correlation**: Required for long-running workflows with multiple events

---

**Document Version**: 1.0
**Last Updated**: 2025-11-11
**For**: Elsa Workflows v3

**Quick Links**:
- [Architecture Doc](./architecture.md)
- [User Glossary](./glossary-users.md)
- [Contributor Glossary](./glossary-contributors.md)
