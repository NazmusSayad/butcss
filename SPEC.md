# CSS Server Specification

This document consolidates the architecture, syntax, and requirements for css-server.

## Overview

css-server is a transpiler that converts CSS syntax into a running Express.js server. The system parses CSS files into an AST, compiles them into Express route handlers, and runs them with a runtime that also manages SQLite access.

```
CSS File -> Parser -> AST -> Compiler -> Express Routes -> Runtime
```

## Architecture

### CLI (src/index.ts)

- Parses command-line arguments using Commander
- Reads the CSS file from disk
- Invokes the parser and runtime

### Parser (src/parser.ts)

Uses PostCSS to parse CSS into a ParsedCSS object:

```ts
interface ParsedCSS {
  config: ServerConfig
  routes: RouteRule[]
}
```

Parsing steps:

1. Server config extraction from `@server`
2. Route extraction for `[path="..."]:METHOD`
3. Declaration parsing for variables, status, and `@return`
4. Expression parsing: sql, param, query, body, header, var, if

### Evaluator (src/evaluator.ts)

Executes expressions during request handling.

Key functions:

- `evaluateExpression()`
- `evaluateSql()`
- `evaluateIf()`
- `evaluateCondition()`

Request context:

```ts
interface RequestContext {
  params: Record<string, string>
  query: Record<string, string>
  body: Record<string, any>
  headers: Record<string, string>
  variables: Record<string, any>
}
```

### Compiler (src/compiler.ts)

Transforms parsed routes into Express handlers.

Handler logic:

1. Build RequestContext from req
2. Evaluate variable assignments in order
3. Evaluate status code (if present)
4. Evaluate return value
5. Send response (json or html)

### Runtime (src/runtime.ts)

- Initializes Express middleware (JSON, URL-encoded)
- Initializes SQLite connection
- Registers compiled routes
- Handles 404 fallback
- Starts HTTP server

### File Structure

```
src/
  types.ts
  parser.ts
  evaluator.ts
  compiler.ts
  runtime.ts
  index.ts

tests/
  parser.test.ts
  evaluator.test.ts
  integration.test.ts
```

### Dependencies

| Package          | Purpose               |
| ---------------- | --------------------- |
| postcss          | CSS parsing           |
| express          | HTTP server framework |
| better-sqlite3   | SQLite database       |
| commander        | CLI argument parsing  |

### Extension Points

Adding new functions:

1. Add type to `Expression` in `src/types.ts`
2. Add parsing logic in `src/parser.ts:parseExpression()`
3. Add evaluation logic in `src/evaluator.ts:evaluateExpression()`

Adding new conditions:

1. Add type to `Condition` in `src/types.ts`
2. Add parsing logic in `src/parser.ts:parseCondition()`
3. Add evaluation logic in `src/evaluator.ts:evaluateCondition()`

Adding new HTTP methods:

1. Add to `HttpMethod` type in `src/types.ts`
2. Add case in `src/runtime.ts:registerRoute()`

## Requirements

### Functional Requirements

#### FR-01: CSS-Based Routing

Define HTTP routes using CSS selector syntax.

```css
[path="<route-path>"]:http_method {
}
```

Supported methods: GET, POST, PUT, PATCH, DELETE, HEAD, OPTIONS.

Examples:

```css
[path="/users"]:get {}
[path="/users/:id"]:get {}
[path="/users"]:post {}
[path="*"]:get {}
```

#### FR-02: Server Configuration

Support `@server` at-rule.

| Property | Type   | Description                             |
| -------- | ------ | --------------------------------------- |
| port     | number | Server port (default: 3000)             |
| database | string | SQLite database path                    |
| host     | string | Server host binding (default: localhost) |

Environment variables:

```css
@server {
  port: env(PORT, 3000);
  database: env(DATABASE_PATH, ./app.db);
}
```

#### FR-03: Request Data Extraction

| Function       | Description            | Example                     |
| -------------- | ---------------------- | --------------------------- |
| param(:name)   | Route parameter        | --id: param(:id)            |
| query(name)    | Query string parameter | --q: query(q)               |
| body(name)     | Request body field     | --name: body(name)          |
| header(name)   | HTTP header            | --role: header(x-user-role) |

#### FR-04: Variable System

```css
--variable-name: expression;
```

```css
--id: param(:id);
--user: sql("SELECT * FROM users WHERE id = ?", var(--id));
@return json(var(--user));
```

#### FR-05: SQL Query Execution

```css
sql("query", arg1, arg2, ...)
```

Return values:

| Query Type | Return                                   |
| ---------- | ---------------------------------------- |
| SELECT     | Array of rows or single row              |
| INSERT     | { id: lastInsertRowid, changes: number } |
| UPDATE     | { changes: number }                      |
| DELETE     | { changes: number }                      |

#### FR-06: Conditional Logic

```css
if(
  condition: value;
  else: default_value;
)
```

Condition types: truthy, equals, not equals, greater than, less than, greater or equal, less or equal, AND, OR, NOT.

#### FR-07: Response Types

JSON:

```css
@return json({ "key": "value" })
@return json(var(--data))
```

HTML:

```css
@return html("<h1>Hello</h1>")
@return html(var(--htmlContent))
```

#### FR-08: Status Codes

```css
status: 404;
status: if(--authorized: 200; else: 403);
```

#### FR-09: CLI Interface

```bash
css-server <file> [options]
```

Options:

| Option              | Description           |
| ------------------- | --------------------- |
| -p, --port <number> | Override server port  |
| -h, --host <string> | Override server host  |
| -v, --version       | Display version       |
| --help              | Display help          |

### Non-Functional Requirements

#### NFR-01: Performance

- Parse CSS files in under 100ms for typical API definitions
- Execute SQL queries using prepared statements
- Reuse database connection across requests

#### NFR-02: Error Handling

- Report parse errors with line numbers
- Return 404 for unmatched routes
- Handle missing database gracefully
- Return JSON error objects for SQL failures

#### NFR-03: Security

- Use parameterized queries to prevent SQL injection
- Do not expose internal error details to clients
- Validate route paths

### Limitations

Current limitations:

1. No middleware support
2. No authentication
3. SQLite only
4. No file uploads
5. No CORS
6. No rate limiting

Future considerations:

1. Middleware hooks
2. Multiple database support
3. WebSocket support
4. Server-side events
5. Request validation
6. Response caching

## Syntax Reference

### Server Configuration

```css
@server {
  port: <number>;
  database: <string>;
  host: <string>;
}
```

Environment variables:

```css
@server {
  port: env(PORT, 3000);
  database: env(DATABASE_PATH, ./app.db);
  host: env(HOST, localhost);
}
```

### Database Schema

```css
@database {
  CREATE TABLE users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT,
    email TEXT
  );
  CREATE INDEX users_email ON users(email);
}
```

The `@database` block is optional. If omitted, database features are unavailable unless a database is configured another way.

### Routes

```css
[path="<route-path>"]:http_method {
}
```

HTTP methods: GET, POST, PUT, PATCH, DELETE, HEAD, OPTIONS.

Path patterns:

| Pattern                      | Description          | Example                                    |
| --------------------------- | -------------------- | ------------------------------------------ |
| /                           | Root path            | [path="/"]:GET                           |
| /users                      | Static path          | [path="/users"]:GET                      |
| /users/:id                  | Path with parameter  | [path="/users/:id"]:GET                  |
| /posts/:postId/comments/:id | Multiple parameters  | [path="/posts/:postId/comments/:id"]:GET |
| *                           | Catch-all / fallback | [path="*"]:GET                           |

Examples:

```css
[path="/"]:GET {
  @return html("<h1>Home</h1>");
}

[path="/users"]:GET {
  @return json(sql("SELECT * FROM users"));
}

[path="/users"]:POST {
  --name: body(name);
  @return json(sql("INSERT INTO users (name) VALUES (?)", var(--name)));
}

[path="/users/:id"]:GET {
  --id: param(:id);
  @return json(sql("SELECT * FROM users WHERE id = ?", var(--id)));
}

[path="*"]:GET {
  status: 404;
  @return json({ "error": "Not found" });
}
```

### Variables

```css
--variable-name: <expression>;
```

```css
--id: param(:id);
--user: sql("SELECT * FROM users WHERE id = ?", var(--id));
@return json(var(--user));
```

### Functions

param():

```css
param(:parameter-name)
```

query():

```css
query(parameter-name)
```

body():

```css
body(field-name)
```

header():

```css
header(header-name)
```

var():

```css
var(--variable-name)
```

sql():

```css
sql("query", arg1, arg2, ...)
```

Return values:

| Query Type         | Return Value                    |
| ------------------ | ------------------------------- |
| SELECT (no args)   | Array of row objects            |
| SELECT (with args) | Single row object (.get())      |
| INSERT             | { id: number, changes: number } |
| UPDATE             | { changes: number }             |
| DELETE             | { changes: number }             |
| Error              | { error: string }               |

### Conditionals

```css
if(
  condition: value;
  condition: value;
  else: default-value;
)
```

Condition types:

| Type             | Syntax          | Description                                     |
| ---------------- | --------------- | ----------------------------------------------- |
| Truthy           | --var           | Variable is truthy (not null, not empty, not 0) |
| Equals           | --var = value   | Variable equals value                           |
| Not Equals       | --var != value  | Variable does not equal value                   |
| Greater Than     | --var > number  | Variable is greater than number                 |
| Less Than        | --var < number  | Variable is less than number                    |
| Greater or Equal | --var >= number | Variable is greater or equal                    |
| Less or Equal    | --var <= number | Variable is less or equal                       |
| AND              | --a and --b     | Both conditions must be true                    |
| OR               | --a or --b      | Either condition must be true                   |
| NOT              | not --var       | Negate a condition                              |

### Responses

```css
@return json(<expression>);
@return html(<expression>);
```

### Status Codes

```css
status: <number>;
status: if(<condition>: <number>; else: <number>);
```

### Complete Example

```css
@server {
  port: env(PORT, 3000);
  database: env(DATABASE, ./app.db);
}

@database {
  CREATE TABLE users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT,
    email TEXT
  );
}

[path="/"]:GET {
  @return html("<h1>Welcome to CSS Server</h1>");
}

[path="/users"]:GET {
  @return json(sql("SELECT * FROM users"));
}

[path="/users/:id"]:GET {
  --id: param(:id);
  --user: sql("SELECT * FROM users WHERE id = ?", var(--id));
  @return json(if(--user: var(--user); else: { "error": "User not found" }));
}

[path="/users"]:POST {
  --name: body(name);
  --email: body(email);
  @return json(sql("INSERT INTO users (name, email) VALUES (?, ?)", var(--name), var(--email)));
}

[path="/users/:id"]:PUT {
  --id: param(:id);
  --name: body(name);
  --email: body(email);
  @return json(sql("UPDATE users SET name = ?, email = ? WHERE id = ?", var(--name), var(--email), var(--id)));
}

[path="/users/:id"]:DELETE {
  --id: param(:id);
  @return json(sql("DELETE FROM users WHERE id = ?", var(--id)));
}

[path="/search"]:GET {
  --q: query(q);
  --results: sql("SELECT * FROM users WHERE name LIKE ?", var(--q));
  @return json(if(--q: var(--results); else: []));
}

[path="/admin"]:GET {
  --role: header(x-user-role);
  status: if(--role = admin: 200; else: 403);
  @return json(if(
    --role = admin: { "message": "Welcome, admin!" };
    else: { "error": "Access denied" };
  ));
}

[path="/check-age"]:GET {
  --age: query(age);
  @return json(if(
    --age >= 18: { "status": "adult" };
    --age >= 13: { "status": "teen" };
    else: { "status": "child" };
  ));
}

[path="*"]:GET {
  status: 404;
  @return json({ "error": "Not found" });
}
```

### Quick Reference

| Feature       | Syntax                      |
| ------------- | --------------------------- |
| Server config | @server { ... }             |
| Route         | [path="/path"]:GET { ... } |
| Variable      | --name: value;              |
| Param         | param(:name)                |
| Query         | query(name)                 |
| Body          | body(name)                  |
| Header        | header(name)                |
| Variable ref  | var(--name)                 |
| SQL           | sql("query", args...)       |
| If            | if(cond: val; else: val)    |
| Return JSON   | @return json(...)           |
| Return HTML   | @return html(...)           |
| Status        | status: 404;                |
| Equals        | --var = value               |
| Not equals    | --var != value              |
| Greater than  | --var > number              |
| Less than     | --var < number              |
| AND           | --a and --b                 |
| OR            | --a or --b                  |
| NOT           | not --var                   |
