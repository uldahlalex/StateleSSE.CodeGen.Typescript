# EventSourceGen

**Generate TypeScript EventSource clients from OpenAPI specifications with C# attributes.**

## Features

- ✅ **[EventSourceEndpoint] Attribute** - Explicitly mark SSE endpoints
- ✅ **TypeScript Code Generation** - Auto-generate typed EventSource functions
- ✅ **OpenAPI Extension** - Adds `x-event-source` and `x-event-type` to spec
- ✅ **Single Source of Truth** - Types and URLs from your C# controllers
- ✅ **Zero Magic Strings** - No hardcoded namespaces or naming conventions
- ✅ **Project Agnostic** - Works with any ASP.NET Core project

---

## Installation

### Option 1: Local Project Reference (Development)

```bash
dotnet add reference /path/to/EventSourceGen/EventSourceGen.csproj
```

### Option 2: NuGet Package (Production)

```bash
# First, pack the project
cd EventSourceGen
dotnet pack -c Release

# Then install from local
dotnet add package EventSourceGen --source ./bin/Release
```

### Option 3: Publish to NuGet.org

```bash
dotnet pack -c Release
dotnet nuget push bin/Release/EventSourceGen.1.0.0.nupkg --api-key YOUR_API_KEY --source https://api.nuget.org/v3/index.json
```

Then:
```bash
dotnet add package EventSourceGen
```

---

## Usage

### Step 1: Mark SSE Endpoints with Attribute

```csharp
using EventSourceGen;
using Microsoft.AspNetCore.Mvc;

[ApiController]
public class GameEventsController : ControllerBase
{
    [HttpGet("round-started")]
    [EventSourceEndpoint(typeof(RoundStartedEvent))]  // ← Add this!
    public async Task StreamRoundStarted([FromQuery] string gameId)
    {
        Response.Headers.Append("Content-Type", "text/event-stream");
        // ... SSE implementation
    }
}
```

### Step 2: Call Generator in Program.cs

**Add right after NSwag TypeScript generation:**

```csharp
using EventSourceGen;

var app = builder.Build();

// Your existing NSwag generation
app.GenerateApiClientsFromOpenApi("/../../client/src/generated-client.ts")
   .GetAwaiter()
   .GetResult();

// Add EventSource generation right here!
TypeScriptSseGenerator.Generate(
    openApiSpecPath: Path.Combine(Directory.GetCurrentDirectory(), "openapi-with-docs.json"),
    outputPath: Path.Combine(Directory.GetCurrentDirectory(), "../../client/src/generated-sse-client.ts")
);

await app.RunAsync();
```

### Step 3: Use in TypeScript

```typescript
import { streamRoundStarted } from './generated-sse-client';
import type { RoundStartedEvent } from './generated-client';

const es = streamRoundStarted('game-123');
es.onmessage = (e) => {
    const event: RoundStartedEvent = JSON.parse(e.data);
    console.log(event.questionText);
};
```

---

## Integration with NSwag

To add OpenAPI extensions (`x-event-source`, `x-event-type`), register the operation processor:

```csharp
// In your Swagger/NSwag configuration (usually Etc/SwaggerExtensions.cs)
using EventSourceGen;

public sealed class EventSourceEndpointOperationProcessor : IOperationProcessor
{
    public bool Process(OperationProcessorContext context)
    {
        var attribute = context.MethodInfo.GetCustomAttribute<EventSourceEndpointAttribute>();
        if (attribute == null) return true;

        context.OperationDescription.Operation.ExtensionData ??= new Dictionary<string, object>();
        context.OperationDescription.Operation.ExtensionData["x-event-source"] = true;
        context.OperationDescription.Operation.ExtensionData["x-event-type"] = attribute.EventType.Name;

        return true;
    }
}

// Register in Program.cs
services.AddOpenApiDocument(conf =>
{
    conf.OperationProcessors.Add(new EventSourceEndpointOperationProcessor());
});
```

---

## API Reference

### EventSourceEndpointAttribute

```csharp
[AttributeUsage(AttributeTargets.Method, AllowMultiple = false)]
public sealed class EventSourceEndpointAttribute : Attribute
{
    public Type EventType { get; }

    public EventSourceEndpointAttribute(Type eventType);
}
```

**Parameters:**
- `eventType` - The event DTO type this endpoint streams (e.g., `typeof(RoundStartedEvent)`)

**Example:**
```csharp
[EventSourceEndpoint(typeof(RoundStartedEvent))]
```

### TypeScriptSseGenerator

```csharp
public static class TypeScriptSseGenerator
{
    public static void Generate(
        string openApiSpecPath,
        string outputPath,
        string baseUrlImport = "./utils/BASE_URL",
        string clientImport = "./generated-client"
    );
}
```

**Parameters:**
- `openApiSpecPath` - Path to OpenAPI JSON file (e.g., `"openapi-with-docs.json"`)
- `outputPath` - Output path for generated TypeScript file
- `baseUrlImport` - Import path for BASE_URL (default: `"./utils/BASE_URL"`)
- `clientImport` - Import path for generated client types (default: `"./generated-client"`)

**Example:**
```csharp
TypeScriptSseGenerator.Generate(
    openApiSpecPath: "openapi-with-docs.json",
    outputPath: "../../client/src/generated-sse-client.ts"
);
```

---

## Generated Output Example

**Input (C#):**
```csharp
[EventSourceEndpoint(typeof(RoundStartedEvent))]
public async Task StreamRoundStarted([FromQuery] string gameId)
```

**Output (TypeScript):**
```typescript
/**
 * Subscribe to RoundStartedEvent events
 * @param gameid - gameId
 * @returns EventSource instance for RoundStartedEvent
 */
export function streamRoundStarted(gameid: string): EventSource {
    const queryParams = new URLSearchParams({ gameid });
    const url = `${BASE_URL}/RoundStartedEvent?${queryParams}`;
    return new EventSource(url);
}

export function createTypedEventStream<T>(
    url: string,
    onMessage: (event: T) => void,
    onError?: (error: Event) => void
): EventSource {
    // ... helper implementation
}
```

---

## How It Works

```
[EventSourceEndpoint(typeof(EventType))]
        ↓
NSwag Operation Processor
        ↓
x-event-source: true in OpenAPI
        ↓
TypeScriptSseGenerator.Generate()
        ↓
generated-sse-client.ts
```

1. You mark endpoints with `[EventSourceEndpoint]`
2. NSwag operation processor adds `x-event-source: true` to OpenAPI
3. `TypeScriptSseGenerator.Generate()` reads OpenAPI and finds marked endpoints
4. TypeScript EventSource functions generated

---

## Comparison: Before vs After

### Before (Node.js Script)

```bash
# Separate Node.js dependency
npm install
npm run generate:sse
```

### After (.NET NuGet Package)

```csharp
// Single .NET toolchain
TypeScriptSseGenerator.Generate(
    "openapi-with-docs.json",
    "../../client/src/generated-sse-client.ts"
);
```

**Benefits:**
- ✅ No Node.js dependency
- ✅ Single command (`dotnet run`)
- ✅ Runs inline with NSwag generation
- ✅ Same tooling as rest of project

---

## Advanced Configuration

### Custom Import Paths

```csharp
TypeScriptSseGenerator.Generate(
    openApiSpecPath: "openapi.json",
    outputPath: "output.ts",
    baseUrlImport: "@/config/api",      // Custom BASE_URL import
    clientImport: "@/api/generated"      // Custom client types import
);
```

### Multiple Event Types per Endpoint

If your endpoint streams multiple event types (not recommended), you can still use the attribute with the primary type:

```csharp
[EventSourceEndpoint(typeof(GameEvent))]  // Primary/union type
public async Task StreamAllGameEvents([FromQuery] string gameId)
```

---

## Requirements

- .NET 9.0+
- ASP.NET Core
- NSwag (for OpenAPI generation)

---

## License

MIT

---

## Contributing

1. Fork the repository
2. Create your feature branch
3. Commit your changes
4. Push to the branch
5. Create a Pull Request

---

## Support

For issues and questions, please open an issue on GitHub.
