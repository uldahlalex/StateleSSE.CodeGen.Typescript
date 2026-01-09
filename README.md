# StateleSSE.CodeGen.TypeScript

> **⚠️ DEPRECATED**: This package has been consolidated into `StateleSSE.AspNetCore`.
>
> **Migration:** The TypeScript code generator is now included in `StateleSSE.AspNetCore`. All types are now in the `StateleSSE.AspNetCore.CodeGen` namespace. Update your imports and use `StateleSSE.AspNetCore.CodeGen.TypeScriptSseGenerator.Generate()`.
>
> This package is no longer maintained and will not receive updates.

---

Generate TypeScript EventSource clients from OpenAPI specifications.

## Installation

```bash
dotnet add package StateleSSE.CodeGen.TypeScript
```

## Quick Start

### 1. Mark SSE Endpoints

```csharp
using StateleSSE.CodeGen.TypeScript;

[ApiController]
public class GameEventsController : ControllerBase
{
    [HttpGet("events/player-joined")]
    [EventSourceEndpoint(typeof(PlayerJoinedEvent))]
    public async Task StreamPlayerJoined([FromQuery] string gameId)
    {
        var channel = ChannelNamingExtensions.Channel<PlayerJoinedEvent>("game", gameId);
        await HttpContext.StreamSseAsync<PlayerJoinedEvent>(backplane, channel);
    }
}
```

### 2. Register OpenAPI Processor

**NSwag:**
```csharp
using StateleSSE.CodeGen.TypeScript;

services.AddOpenApiDocument(conf =>
{
    conf.OperationProcessors.Add(new NSwagEventSourceProcessor());
});
```

**Swashbuckle:**
```csharp
using StateleSSE.CodeGen.TypeScript;

services.AddSwaggerGen(options =>
{
    options.OperationFilter<SwashbuckleEventSourceFilter>();
});
```

### 3. Generate TypeScript Client

```csharp
using StateleSSE.CodeGen.TypeScript;

TypeScriptSseGenerator.Generate(
    openApiSpecPath: "openapi.json",
    outputPath: "../../client/src/generated-sse-client.ts"
);
```

### 4. Use in TypeScript

```typescript
import { streamPlayerJoined } from './generated-sse-client';
import type { PlayerJoinedEvent } from './generated-client';

const es = streamPlayerJoined('game-123');
es.onmessage = (e) => {
    const event: PlayerJoinedEvent = JSON.parse(e.data);
    console.log('Player joined:', event.playerName);
};
```

## Generated Output

**Input (C#):**
```csharp
[EventSourceEndpoint(typeof(PlayerJoinedEvent))]
public async Task StreamPlayerJoined([FromQuery] string gameId) { }
```

**Output (TypeScript):**
```typescript
export function streamPlayerJoined(gameid: string): EventSource {
    const queryParams = new URLSearchParams({ gameid });
    const url = `${BASE_URL}/events/player-joined?${queryParams}`;
    return new EventSource(url);
}
```

## Configuration

```csharp
TypeScriptSseGenerator.Generate(
    openApiSpecPath: "openapi.json",
    outputPath: "output.ts",
    baseUrlImport: "@/config/api"  // Custom BASE_URL import path
);
```

## How It Works

```
[EventSourceEndpoint(typeof(EventType))]
        ↓
OpenAPI Processor (NSwag/Swashbuckle)
        ↓
OpenAPI: x-event-source: true
         x-event-type: "EventType"
        ↓
TypeScriptSseGenerator.Generate()
        ↓
generated-sse-client.ts
```

## Requirements

- .NET 6.0+
- ASP.NET Core
- NSwag or Swashbuckle (for OpenAPI generation)

## License

MIT
