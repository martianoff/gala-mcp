# gala-mcp

A small, dependency-free [Model Context Protocol](https://modelcontextprotocol.io) server
runtime written in [GALA](https://github.com/martianoff/gala). It speaks JSON-RPC 2.0 over
newline-delimited **stdio** — the transport Claude Desktop and Claude Code use to launch local
MCP servers — and implements the `initialize` / `tools/list` / `tools/call` / `ping` handshake.

The JSON layer is a pure-GALA `JsonValue` model (sealed type) with its own parser and
serializer, so the library has no third-party dependencies beyond the GALA standard library.

```
import path:  github.com/martianoff/gala-mcp
package:      mcp
protocol:     MCP 2025-06-18, JSON-RPC 2.0 over stdio
```

## Install

```bash
gala mod add github.com/martianoff/gala-mcp
```

## Usage

A server is built by registering tools and calling `Run()`, which serves over stdio until EOF:

```gala
package main

import . "github.com/martianoff/gala-mcp"

// Handlers are named functions with an explicit signature (see "cross-module
// notes" below). The argument is the tool call's `arguments` object.
func greet(args JsonValue) ToolResult {
    val name = args.GetString("name").GetOrElse("world")
    return OkResult(s"Hello, $name!")
}

func main() {
    NewServer("greeter", "1.0.0")
        .WithTool(NewTool(
            "greet",
            "Greet someone by name.",
            JObjOf(
                Field("type", JString("object")),
                Field("properties", JObjOf(
                    Field("name", JObjOf(Field("type", JString("string")))),
                )),
            ),
            greet,
        ))
        .Run()
}
```

## API

**Server**

| Symbol | Description |
|--------|-------------|
| `NewServer(name, version) Server` | Create a server with no tools. |
| `(Server) WithTool(t Tool) Server` | Return a copy with one more tool (immutable). |
| `(Server) Run()` | Serve JSON-RPC over stdin/stdout until EOF. |
| `(Server) HandleLine(line) Option[string]` | Pure core: one request line → optional response line. Ideal for tests. |
| `NewTool(name, description, inputSchema, handler) Tool` | Build a tool; `handler` is `func(JsonValue) ToolResult`. |
| `OkResult(text) / ErrResult(text)` | Build a tool result; `ErrResult` sets `isError`. |
| `(ToolResult) Failed() bool` / `(ToolResult) Message() string` | Cross-module-safe accessors. |

**JSON values** — `JsonValue` is a sealed type: `JNull`, `JBool`, `JNum`, `JStr`, `JArr`, `JObj`.

| Builder | Accessor |
|---------|----------|
| `JString(s)`, `JInt(n)`, `JNumber(f)`, `JBoolOf(b)`, `JNullValue()` | `AsString()`, `AsInt()`, `AsNum()`, `AsBool()` → `Option[T]` |
| `JObjOf(fields...)`, `Field(key, value)` | `Get(key)`, `GetString(key)`, `GetInt(key)`, `GetObject(key)` |
| `JArrOf(items...)` | `AsArray()` → `Option[Array[JsonValue]]` |
| `ParseJson(s) Try[JsonValue]` | `RenderJson(v) string` |

## Cross-module notes

When this package is imported from **another module**, prefer the plain constructor functions
over the underlying sealed-type / struct constructors, and write tool handlers as **named
functions** rather than lambdas:

- Build values with `NewTool`, `JObjOf`, `JArrOf`, `Field`, `JString`, … (plain functions that
  resolve cleanly across modules) — not `Tool{...}`, `JStr(...)`, `JField(...)` directly.
- Write handlers as named `func(JsonValue) ToolResult`, not inline lambdas.
- Read `ToolResult` via `Failed()` / `Message()`, not the `IsError` / `Text` fields.

These conventions exist because of a few cross-module type-inference gaps in the current GALA
transpiler; the in-module API (sealed constructors, lambdas, field reads) is unaffected.
