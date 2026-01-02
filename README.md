# StateleSSE.CodeGen.TypeScript

Generate TypeScript EventSource clients from OpenAPI specifications with C# attributes. Part of the StateleSSE ecosystem for type-safe, stateless SSE architectures.

## Installation

```bash
dotnet add package StateleSSE.CodeGen.TypeScript
```

## Features

- ✅ **[EventSourceEndpoint] Attribute** - Explicitly mark SSE endpoints in C#
- ✅ **TypeScript Code Generation** - Auto-generate typed EventSource functions
- ✅ **OpenAPI Extension** - Adds `x-event-source` and `x-event-type` metadata
- ✅ **Single Source of Truth** - Types and URLs derived from C# controllers
- ✅ **Zero Magic Strings** - No hardcoded namespaces or conventions
- ✅ **Framework Agnostic** - Works with any ASP.NET Core project

## Quick Start

### 1. Mark SSE Endpoints

```csharp
using StateleSSE.CodeGen.TypeScript;
using Microsoft.AspNetCore.Mvc;

[ApiController]
public class GameEventsController : ControllerBase
{
    [HttpGet("events/player-joined")]
    [EventSourceEndpoint(typeof(PlayerJoinedEvent))]  // ← Mark SSE endpoint
    public async Task StreamPlayerJoined([FromQuery] string gameId)
    {
        // ... SSE implementation
    }
}
```

### 2. Generate TypeScript Client

```csharp
using StateleSSE.CodeGen.TypeScript;

// In Program.cs, after NSwag generation
var app = builder.Build();

// Generate OpenAPI spec
app.GenerateApiClientsFromOpenApi("/../../client/src/generated-client.ts")
   .GetAwaiter()
   .GetResult();

// Generate EventSource clients
TypeScriptSseGenerator.Generate(
    openApiSpecPath: Path.Combine(Directory.GetCurrentDirectory(), "openapi-with-docs.json"),
    outputPath: Path.Combine(Directory.GetCurrentDirectory(), "../../client/src/generated-sse-client.ts")
);

await app.RunAsync();
```

### 3. Use in TypeScript

```typescript
import { streamPlayerJoined } from './generated-sse-client';
import type { PlayerJoinedEvent } from './generated-client';

const es = streamPlayerJoined('game-123');
es.onmessage = (e) => {
    const event: PlayerJoinedEvent = JSON.parse(e.data);
    console.log('Player joined:', event.playerName);
};
```

## OpenAPI Integration

To add `x-event-source` and `x-event-type` extensions to OpenAPI spec, register the operation processor:

```csharp
using StateleSSE.CodeGen.TypeScript;
using NSwag.Generation.Processors;
using NSwag.Generation.Processors.Contexts;

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
- `eventType` - The event DTO type this endpoint streams

**Example:**
```csharp
[EventSourceEndpoint(typeof(RoundStartedEvent))]
public async Task StreamRoundStarted([FromQuery] string gameId) { }
```

### TypeScriptSseGenerator.Generate()

```csharp
public static void Generate(
    string openApiSpecPath,
    string outputPath,
    string baseUrlImport = "./utils/BASE_URL",
    string clientImport = "./generated-client"
)
```

**Parameters:**
- `openApiSpecPath` - Path to OpenAPI JSON file
- `outputPath` - Output path for generated TypeScript file
- `baseUrlImport` - Import path for BASE_URL constant (default: `"./utils/BASE_URL"`)
- `clientImport` - Import path for generated client types (default: `"./generated-client"`)

**Example:**
```csharp
TypeScriptSseGenerator.Generate(
    openApiSpecPath: "openapi-with-docs.json",
    outputPath: "../../client/src/generated-sse-client.ts",
    baseUrlImport: "@/config/api",
    clientImport: "@/api/client"
);
```

## Generated Output

**Input (C#):**
```csharp
[EventSourceEndpoint(typeof(RoundStartedEvent))]
public async Task StreamRoundStarted([FromQuery] string gameId) { }
```

**Output (TypeScript):**
```typescript
import { BASE_URL } from './utils/BASE_URL';
import type { RoundStartedEvent } from './generated-client';

/**
 * Subscribe to RoundStartedEvent events
 * @param gameid - gameId
 * @returns EventSource instance for RoundStartedEvent
 */
export function streamRoundStarted(gameid: string): EventSource {
    const queryParams = new URLSearchParams({ gameid });
    const url = `${BASE_URL}/events/round-started?${queryParams}`;
    return new EventSource(url);
}

/**
 * Helper for creating typed EventSource streams with callbacks
 */
export function createTypedEventStream<T>(
    url: string,
    onMessage: (event: T) => void,
    onError?: (error: Event) => void
): EventSource {
    const es = new EventSource(url);
    es.onmessage = (e) => onMessage(JSON.parse(e.data));
    if (onError) es.onerror = onError;
    return es;
}
```

## Complete Example

**C# Controller:**
```csharp
using StateleSSE.CodeGen.TypeScript;
using StateleSSE.AspNetCore;
using StateleSSE.Abstractions;

[ApiController]
public class GameEventsController(ISseBackplane backplane) : ControllerBase
{
    [HttpGet("events/player-joined")]
    [EventSourceEndpoint(typeof(PlayerJoinedEvent))]
    public async Task StreamPlayerJoined([FromQuery] string gameId)
    {
        var channel = ChannelNamingExtensions.Channel<PlayerJoinedEvent>("game", gameId);
        await HttpContext.StreamSseAsync<PlayerJoinedEvent>(backplane, channel);
    }

    [HttpGet("events/round-started")]
    [EventSourceEndpoint(typeof(RoundStartedEvent))]
    public async Task StreamRoundStarted([FromQuery] string gameId)
    {
        var channel = ChannelNamingExtensions.Channel<RoundStartedEvent>("game", gameId);
        await HttpContext.StreamSseAsync<RoundStartedEvent>(backplane, channel);
    }
}

public record PlayerJoinedEvent(string GameId, string PlayerName, DateTime JoinedAt);
public record RoundStartedEvent(string GameId, int RoundNumber, string Question);
```

**TypeScript Client:**
```typescript
import { streamPlayerJoined, streamRoundStarted } from './generated-sse-client';
import type { PlayerJoinedEvent, RoundStartedEvent } from './generated-client';

// Subscribe to player joined events
const playerEs = streamPlayerJoined('game-123');
playerEs.onmessage = (e) => {
    const event: PlayerJoinedEvent = JSON.parse(e.data);
    console.log(`${event.playerName} joined at ${event.joinedAt}`);
};

// Subscribe to round started events
const roundEs = streamRoundStarted('game-123');
roundEs.onmessage = (e) => {
    const event: RoundStartedEvent = JSON.parse(e.data);
    console.log(`Round ${event.roundNumber}: ${event.question}`);
};
```

## How It Works

```
[EventSourceEndpoint(typeof(EventType))]
        ↓
NSwag Operation Processor
        ↓
OpenAPI: x-event-source: true
         x-event-type: "EventType"
        ↓
TypeScriptSseGenerator.Generate()
        ↓
generated-sse-client.ts
```

1. Mark endpoints with `[EventSourceEndpoint]`
2. NSwag operation processor adds `x-event-source` metadata to OpenAPI
3. `TypeScriptSseGenerator.Generate()` reads OpenAPI and generates TypeScript functions
4. Client imports and uses typed EventSource functions

## Benefits Over Manual Approach

| Before (Manual) | After (StateleSSE.CodeGen.TypeScript) |
|-----------------|--------------------------------------|
| Hardcoded URLs in TypeScript | Generated from C# routes |
| Duplicate type definitions | Shared from NSwag-generated types |
| Manual sync between C# and TS | Single source of truth |
| No compile-time safety | Full TypeScript typing |
| Separate Node.js build step | Inline with .NET build |

## Integration with StateleSSE Ecosystem

```bash
# Full stack setup
dotnet add package StateleSSE.Abstractions
dotnet add package StateleSSE.Backplane.Redis
dotnet add package StateleSSE.AspNetCore
dotnet add package StateleSSE.CodeGen.TypeScript
```

**Workflow:**
1. Define event DTOs in C#
2. Mark SSE endpoints with `[EventSourceEndpoint]`
3. Use `StateleSSE.AspNetCore` extension methods for zero-boilerplate endpoints
4. Generate TypeScript clients with `TypeScriptSseGenerator.Generate()`
5. Consume strongly-typed EventSource functions in frontend

## Custom Configuration

### Custom BASE_URL Import

```csharp
TypeScriptSseGenerator.Generate(
    openApiSpecPath: "openapi.json",
    outputPath: "output.ts",
    baseUrlImport: "@/config/api"  // Custom import path
);
```

Generated imports:
```typescript
import { BASE_URL } from '@/config/api';
```

### Custom Client Types Import

```csharp
TypeScriptSseGenerator.Generate(
    openApiSpecPath: "openapi.json",
    outputPath: "output.ts",
    clientImport: "@/api/types"  // Custom import path
);
```

Generated imports:
```typescript
import type { PlayerJoinedEvent } from '@/api/types';
```

## Requirements

- .NET 6.0+
- ASP.NET Core
- NSwag (for OpenAPI generation)

## Related Packages

| Package | Purpose |
|---------|---------|
| `StateleSSE.Abstractions` | Core `ISseBackplane` interface |
| `StateleSSE.Backplane.Redis` | Redis backplane for horizontal scaling |
| `StateleSSE.AspNetCore` | Extension methods for SSE endpoints |

## License

MIT
