# langchain4j-custom-mcp-server-course

A standalone Java **MCP (Model Context Protocol) server** that exposes three tools for querying course information — **list courses**, **get course price**, and **get course duration** — over the MCP **stdio transport**. It is designed to be launched as a subprocess by an MCP client/host application such as the companion **[langchain4j-mcp-client-course](../langchain4j-mcp-client-course)** project, which uses an LLM to decide when and how to call these tools.

This repository contains the **MCP server** side only. The client/host application that talks to it lives in a separate repository: **[langchain4j-mcp-client-course](../langchain4j-mcp-client-course)**.

## What this project demonstrates

A minimal MCP server built directly on the official `io.modelcontextprotocol.sdk:mcp` Java SDK, with no web framework and no network listener — it communicates purely over **standard input/output**, which is the MCP "stdio transport" pattern used for locally-spawned tool servers.

```
MCP Client (subprocess parent) ──stdin/stdout, JSON-RPC──► MCP Server (this project)
                                                                   │
                                                          CourseServices (business logic)
```

## Project structure

```
langchain4j-custom-mcp-server-course/
├── pom.xml                                       # Maven build, MCP SDK dependency, shade plugin
└── src/main/java/com/org/
    ├── Main.java                                 # Server bootstrap: transport, tool registration, schemas
    └── services/
        └── CourseServices.java                   # Plain Java business logic backing each tool
```

### `pom.xml`
- Java 21, Maven.
- Dependency: `io.modelcontextprotocol.sdk:mcp` (v2.0.1) — the official MCP Java SDK providing `McpServer`, `McpSyncServer`, `McpSchema`, and the stdio transport provider.
- **`maven-shade-plugin`** packages the app and all dependencies into a single **fat/shaded jar** (`custom-mcp-server-1.0-SNAPSHOT-shaded.jar`) with `com.org.Main` set as the manifest's main class. This shaded jar is what the MCP client launches directly via `java -jar ...` — it must be self-contained since the client has no classpath knowledge of this project.

### `CourseServices.java` — the actual business logic
Plain, framework-free Java with no MCP awareness at all:

```java
public class CourseServices {
    public String listCourses() { ... }                      // "Available courses: Java, Python, DevOps"
    public String getCoursePrice(String courseName) { ... }  // switch on course name → price string
    public String getCourseDuration(String courseName) { ... } // switch on course name → duration string
}
```

Supported courses (hardcoded demo data): **Java**, **Python**, **DevOps**. This class is intentionally decoupled from MCP — it's the kind of service layer that could just as easily sit behind a REST controller; the MCP layer in `Main.java` is purely an adapter on top of it.

### `Main.java` — the MCP server bootstrap
This is where the three MCP tools are defined and wired to `CourseServices`:

1. **Transport** — `StdioServerTransportProvider` configures the server to read/write MCP JSON-RPC messages over `stdin`/`stdout`.
2. **Server instance** — `McpServer.sync(transport)` builds a synchronous MCP server (`McpSyncServer`), declaring `serverInfo("course-mcp-server", "1.0.0")` and advertising tool-calling capability (`.tools(true)`).
3. **Tool registration** — each tool is defined as a `McpServerFeatures.SyncToolSpecification` with:
   - a **name** (`listCourses`, `getCoursePrice`, `getCourseDuration`),
   - a **description** (used by the LLM on the client side to decide when to call it),
   - an **input JSON schema**, and
   - a **call handler** — a lambda that extracts arguments from the incoming request, delegates to the matching `CourseServices` method, and wraps the plain-string result into an MCP `CallToolResult` via the `result(...)` helper (which wraps text in `McpSchema.TextContent`).
4. **Registered with the server** via `server.addTool(...)`.
5. A `"Course MCP Server ready"` message is printed to **stderr** (not stdout, since stdout is reserved for the MCP protocol stream) once startup completes.

## Tool catalogue

| Tool name | Description | Input schema | Backing method |
|---|---|---|---|
| `listCourses` | List all available courses | `{}` (no parameters) | `CourseServices.listCourses()` |
| `getCoursePrice` | Get the price of a course | `{ "courseName": string, required }` | `CourseServices.getCoursePrice(name)` |
| `getCourseDuration` | Get the duration of a course | `{ "courseName": string, required }` | `CourseServices.getCourseDuration(name)` |

These name/description/schema triples are exactly what an MCP client's tool provider (e.g., LangChain4j's `McpToolProvider`) fetches during the MCP handshake and surfaces to the LLM as callable functions.

## End-to-end flow (this server's role)

This server is a **passive responder** — it never initiates anything. From its point of view:

1. It is **spawned as a child process** by an MCP client (e.g., via `java -jar custom-mcp-server-1.0-SNAPSHOT-shaded.jar`), with its `stdin`/`stdout` piped to the parent process.
2. The client performs the MCP **initialize handshake**, after which it may request the **tool list** — the server responds with the three tool definitions above (name, description, JSON schema).
3. When the client/LLM decides a tool is needed (e.g., the LLM on the client side decided it must call `getCoursePrice` for "Java"), the client sends a `tools/call` JSON-RPC request over stdin with the tool name and arguments.
4. The matching `SyncToolSpecification`'s call handler runs: it reads `courseName` from `request.arguments()`, calls the real `CourseServices` method, and builds a `CallToolResult` containing the text response.
5. The server writes the JSON-RPC response back over **stdout**, which the client reads and passes back up the chain (MCP client → LangChain4j → LLM → final answer).
6. The server keeps running, waiting for further tool calls, until the client/host process terminates it (since it's a child process of the client).

```
                 spawn (java -jar ...shaded.jar)
Client process ───────────────────────────────────► Server process (this project)
                                                             │
                 tools/list  (JSON-RPC, stdin) ─────────────►
                 ◄──────────── tool definitions (stdout) ────┘

                 tools/call: getCoursePrice{courseName:"Java"}  (stdin)
                 ─────────────────────────────────────────────►
                                                             │
                                               CourseServices.getCoursePrice("Java")
                                                             │
                 ◄──────── CallToolResult: "Java course costs 3000 INR" (stdout)
```

## How this talks to the MCP client

There is **no network layer** here — no HTTP port, no socket server. The entire integration surface is:

- **Process spawning**: the client launches this server with a plain `java -jar` command (it must be built into a shaded jar first — see below).
- **Transport**: `StdioServerTransportProvider` reads requests from `System.in` and writes responses to `System.out`, following the MCP JSON-RPC wire format. Any diagnostic logging must go to `System.err` (as `Main.java` does) so it never corrupts the protocol stream on stdout.
- **Contract**: the tool names, descriptions, and JSON input schemas registered in `Main.java` are the entire API surface. Any MCP-compliant client — not just the companion LangChain4j client — can discover and call these tools without knowing anything about the Java implementation behind them.

Because of this tight coupling to a local process launch, both projects are expected to be checked out **on the same machine**, with the client's launch command pointing at this server's built jar path.

## Prerequisites

- Java 21+
- Maven 3.8+

## Build

```bash
mvn clean package
```

This produces `target/custom-mcp-server-1.0-SNAPSHOT-shaded.jar` — the self-contained jar that an MCP client launches as a subprocess.

## Run standalone (for manual/debug testing)

```bash
java -jar target/custom-mcp-server-1.0-SNAPSHOT-shaded.jar
```

The process will print `Course MCP Server ready` to stderr and then block, waiting for MCP JSON-RPC messages on stdin. It is not meant to be driven by hand from a terminal — it's designed to be spawned and talked to programmatically by an MCP client (see the companion `custom-mcp-client` repo for a working example).

## Related repository

- **MCP Client / Host App**: [`langchain4j-mcp-client-course`](../langchain4j-mcp-client-course) — a LangChain4j-based Java application that spawns this server, wires its tools to an OpenAI LLM via `McpToolProvider`, and lets the LLM answer natural-language questions about courses using these tools. See that repository's README for the client-side flow.
