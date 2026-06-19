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

// The handler's argument is the tool call's `arguments` object.
func greet(args JsonValue) ToolResult {
    val name = args.GetString("name").GetOrElse("world")
    return OkResult(s"Hello, $name!")
}

func main() {
    NewServer("greeter", "1.0.0")
        .WithTool(Tool(
            Name = "greet",
            Description = "Greet someone by name.",
            InputSchema = JObjOf(
                JField("type", JStr("object")),
                JField("properties", JObjOf(
                    JField("name", JObjOf(JField("type", JStr("string")))),
                )),
            ),
            Handler = greet,
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
| `Tool(Name, Description, InputSchema, Handler)` | A tool; `Handler` is `func(JsonValue) ToolResult`. |
| `OkResult(text) / ErrResult(text)` | Build a tool result; `ErrResult` sets `isError`. |
| `ToolResult.Text` / `ToolResult.IsError` | The result's text payload and error flag. |

**JSON values** — `JsonValue` is a sealed type: `JNull`, `JBool`, `JNum`, `JStr`, `JArr`, `JObj`.

| Builder | Accessor |
|---------|----------|
| `JStr(s)`, `JInt(n)`, `JNum(f)`, `JBool(b)`, `JNull()` | `AsString()`, `AsInt()`, `AsNum()`, `AsBool()` → `Option[T]` |
| `JObjOf(fields...)`, `JField(key, value)` | `Get(key)`, `GetString(key)`, `GetInt(key)`, `GetObject(key)` |
| `JArrOf(items...)` | `AsArray()` → `Option[Array[JsonValue]]` |
| `ParseJson(s) Try[JsonValue]` | `RenderJson(v) string` |
