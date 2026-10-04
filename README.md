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

Report an expected tool failure with `ErrResult`, so the model can read it. A request whose
handling panics — in a tool handler or in the server — is answered with a JSON-RPC internal
error (`-32603`, carrying the request's id; no response for a notification), the panic is
logged to stderr, and the server goes on serving the next request.

**JSON values** — `JsonValue` is a sealed type: `JNull`, `JBool`, `JNum`, `JStr`, `JArr`, `JObj`.

| Builder | Accessor |
|---------|----------|
| `JStr(s)`, `JInt(n)`, `JNum(f)`, `JBool(b)`, `JNull()` | `AsString()`, `AsInt()`, `AsNum()`, `AsBool()` → `Option[T]` |
| `JObjOf(fields...)`, `JField(key, value)` | `Get(key)`, `GetString(key)`, `GetInt(key)`, `GetObject(key)` |
| `JArrOf(items...)` | `AsArray()` → `Option[Array[JsonValue]]` |
| `ParseJson(s) Try[JsonValue]` | `RenderJson(v) string` |

`ParseJson` follows RFC 8259 strictly. Malformed input is a `Failure` holding a
`JsonSyntaxError(Msg, Offset)`, where `Offset` is the byte offset of the offending byte (the
input length when the input ends early, reported as `unexpected end of input`). Nesting
deeper than `MaxJsonDepth` (512) arrays/objects is rejected. A number outside `float64`'s range
(`1e400`) is rejected, and one too small to represent (`1e-400`) parses as `0`. A UTF-16
surrogate escape without its partner (`"\ud83d"`) decodes as U+FFFD, as Go's `encoding/json`
does. Duplicate object keys are kept in input order, and `Get` returns the first.

`RenderJson` always produces valid JSON: NaN and ±Inf render as `null` (as `JSON.stringify`
does), invalid UTF-8 in a string renders as U+FFFD, and numbers use the shortest round-trip
form, switching to exponent notation outside `1e-6 <= |n| < 1e21` (`1e+21`, `1.5e-7`).
