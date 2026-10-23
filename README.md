# example-web-service

A small todo REST service in [Zell](https://github.com/zell-lang/zell),
written in clean architecture on the standard `web` framework and the `data`
ORM, configured with [zconf](https://github.com/zell-lang/zconf).

```
src/domain/   entities, invariants, domain errors, the repository interface   (no framework)
src/app/      use cases (TodoService), written against the interface
src/infra/    SQLite repository (data ORM + migration), in-memory repository
src/api/      controller with routing attributes, DTOs + @validate, error mapping,
              middleware, metrics, OpenAPI; NewApp/Configure wire it all
server/       the executable: zconf settings, SQLite, DI registrations, graceful shutdown
```

What it shows:

- **Routing attributes**: `@route("/api/todos")` + `@get/@post/@delete`,
  path/query/body binding, `@validate(required: true)` on the DTO, `Result<T>`
  (`Ok`, `CreatedAt`, `NoContent`, `Problem`, `ValidationProblem`).
- **DI with constructor injection**: `AddSingleton<TodoRepository, SqliteRepository>()`
  binds the interface to SQLite in production and to `MemoryRepository` in
  tests; `TodoService.New(repo TodoRepository)` receives it; the controller gets
  `*TodoService` in a field. The compiler checks the graph.
- **Error mapping** (`src/api/errors.zl`): domain `ValidationError` -> 400
  problem details with `errors`, `ErrNotFound` -> 404, `ErrConflict` -> 409,
  anything else -> 500 whose detail only carries the request id (the message
  is logged). A middleware (`hideServerErrors`) also strips the detail of any
  other 5xx.
- **Middleware**: `UseRequestId`, `UseRecover`, `UseTracing` (diag spans; an
  incoming W3C `traceparent` becomes the parent) + our `traceparent` middleware
  that returns the span's context, `UseLogging` (one `log` event per request),
  a Prometheus counter middleware.
- **Operations**: `/metrics` (`diag.PrometheusText`), `/healthz`, `/health`
  (health checks incl. `db.Ping`), `/openapi.json` (OpenAPI 3.1 generated from
  the compiled route table), `/swagger` (Swagger UI page), SIGINT/SIGTERM
  graceful shutdown (`os.signal.WaitAny` + `app.Shutdown()`).
- **Configuration** with zconf: `appsettings.yaml` (optional) < `.env`
  (optional) < `TODO_*` environment variables < command line; `--help` is
  generated from the `Settings` struct.

## Install Zell

```sh
curl -fsSL https://raw.githubusercontent.com/zell-lang/zell/master/scripts/install.sh | sh
```

## Run

```sh
cd server
zell build --release
./target/release/todo-server --help
TODO_ADDR=127.0.0.1:18080 ./target/release/todo-server --database todos.db
```

```text
$ ./target/release/todo-server --help
Options:
  --Addr         string  (default "127.0.0.1:8080")  [env TODO_ADDR]  listen address
  --Database     string  (default "todos.db")  [env TODO_DATABASE]  SQLite database file (:memory: for none)
  --Environment  string  {development|production}  (default "production")  [env TODO_ENVIRONMENT]  environment
  --LogLevel     string  {debug|info|warn|error}  (default "info")  [env TODO_LOGLEVEL]  minimum log level
```

OpenAPI: <http://127.0.0.1:18080/openapi.json>, Swagger UI:
<http://127.0.0.1:18080/swagger>, metrics: <http://127.0.0.1:18080/metrics>.

## curl transcript

Recorded against the release build (`Date`/`Content-Length` headers removed):

```text
$ curl -X POST http://127.0.0.1:18080/api/todos -d {"Title":"buy milk"}
HTTP/1.1 201 Created
X-Request-Id: d44347a9-8499-4218-af72-e7961747b1ee
Traceparent: 00-003fb5d938f90d77e4464a47be5c912f-387ab876529f4171-01
Location: /api/todos/1
Content-Type: application/json

{"id":1,"title":"buy milk","done":false}


$ curl -X POST http://127.0.0.1:18080/api/todos -d {"Title":"write the README"}
HTTP/1.1 201 Created
X-Request-Id: d9b0b05f-a877-401b-96b7-314100da7d72
Traceparent: 00-0ae40d0556db02e49b91533004f8ac60-01836aaf9f5c596b-01
Location: /api/todos/2
Content-Type: application/json

{"id":2,"title":"write the README","done":false}


$ curl http://127.0.0.1:18080/api/todos
HTTP/1.1 200 OK
X-Request-Id: d96ec3da-f4fb-47ab-896a-16e76f40f3b4
Traceparent: 00-5f388c382f64799878ee54ce58b47488-30d758c4b78945a0-01
Content-Type: application/json

[{"id":1,"title":"buy milk","done":false},{"id":2,"title":"write the README","done":false}]


$ curl -X POST http://127.0.0.1:18080/api/todos/1/complete
HTTP/1.1 200 OK
X-Request-Id: 0885ca67-c5f5-4c3f-a9a5-9e06fd4bcd35
Traceparent: 00-172e4fcf574a658e07e981958e4eae55-1ea815556e845363-01
Content-Type: application/json

{"id":1,"title":"buy milk","done":true}


$ curl http://127.0.0.1:18080/api/todos?open=true
HTTP/1.1 200 OK
X-Request-Id: 392792a4-3ca0-4f62-a607-26241baba23d
Traceparent: 00-7280482feeb4da04b9f072a7bb5bf50c-d191df70a908ac52-01
Content-Type: application/json

[{"id":2,"title":"write the README","done":false}]


$ curl http://127.0.0.1:18080/api/todos/42
HTTP/1.1 404 Not Found
X-Request-Id: 98b15369-1938-439b-a6b6-851eca87dfcf
Traceparent: 00-3b952653a909878a0134ec78b6a019bc-9e4dd478f9afcd58-01
Content-Type: application/problem+json

{"type":"about:blank","title":"Not Found","status":404,"detail":"todo 42: todo not found","instance":"/api/todos/42"}


$ curl -X POST http://127.0.0.1:18080/api/todos -d {}
HTTP/1.1 400 Bad Request
X-Request-Id: 75460270-2f3c-4dd2-861c-64ac943535b4
Traceparent: 00-013a9621ad2c8f57813a79615e160c14-e264776e44bda72c-01
Content-Type: application/problem+json

{"type":"https://tools.ietf.org/html/rfc9110#section-15.5.1","title":"One or more validation errors occurred.","status":400,"instance":"/api/todos","errors":{"Title":["The Title field is required."]}}


$ curl -X POST http://127.0.0.1:18080/api/todos/1/complete
HTTP/1.1 409 Conflict
X-Request-Id: 0d57b5ce-a8a7-4b1e-a379-06dbe42ae13c
Traceparent: 00-607640b35cb8db52ec64e0b7d224ae27-f3ee82dd5e6e0d38-01
Content-Type: application/problem+json

{"type":"about:blank","title":"Conflict","status":409,"detail":"todo 1 is already done: conflict","instance":"/api/todos/1/complete"}


$ curl http://127.0.0.1:18080/api/todos/abc
HTTP/1.1 400 Bad Request
X-Request-Id: a496f883-3a51-4e5f-9d27-5ea808c8bc6b
Traceparent: 00-9a94c84fdcf96b3b028792a777c7ea50-dd3684ad454cc825-01
Content-Type: application/problem+json

{"type":"https://tools.ietf.org/html/rfc9110#section-15.5.1","title":"One or more validation errors occurred.","status":400,"instance":"/api/todos/abc","errors":{"id":["The value 'abc' is not valid for id: expected an integer."]}}


$ curl -X DELETE http://127.0.0.1:18080/api/todos/2
HTTP/1.1 204 No Content
X-Request-Id: c995100c-1519-4dda-ae94-a6dc8f7fc6de
Traceparent: 00-3fb4c0567f2c473e9662439a5cd58157-68486b43946adba7-01



$ curl -H traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01 http://127.0.0.1:18080/healthz
HTTP/1.1 200 OK
X-Request-Id: ee355934-64b6-46b2-a491-043912d755c4
Traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-bb6966c748c017d2-01
Content-Type: text/plain; charset=utf-8

ok


$ curl http://127.0.0.1:18080/health
HTTP/1.1 200 OK
X-Request-Id: 9dcf8fea-12ae-40b2-a796-d5055e509f85
Traceparent: 00-0647cf1c189b73fb38f94e48d94dd6d2-ba48f651bc4d9a44-01
Content-Type: application/json

{"checks":{"sqlite":{"status":"Healthy"}},"status":"Healthy"}
```

`/metrics` after that session:

```text
http_requests_total{method="POST",status="201"} 2
http_requests_total{method="GET",status="200"} 4
http_requests_total{method="POST",status="200"} 1
http_requests_total{method="GET",status="404"} 1
http_requests_total{method="POST",status="400"} 1
http_requests_total{method="POST",status="409"} 1
http_requests_total{method="GET",status="400"} 1
http_requests_total{method="DELETE",status="204"} 1
todos_created_total 2
```

Server log, ending with `kill -INT`:

```text
INFO listening on http://127.0.0.1:18080
INFO web: listening on http://127.0.0.1:18080 (production)
INFO HTTP POST /api/todos responded 201 in 0.3240 ms
...
INFO HTTP GET /swagger responded 200 in 0.0150 ms
INFO SIGINT: shutting down
INFO stopped
```

## Test

```sh
zell test
```

`src/app/todos_test.zl` tests the use cases over the in-memory repository;
`src/api/api_test.zl` tests the HTTP surface (CRUD, error mapping, request id,
traceparent, metrics, the OpenAPI paths) through `web.TestServer`, in memory.
They are async *examples* (`// Output:` blocks) because `zell test` does not
run `async @test` functions yet.

## Notes and current gaps

- An interface cannot declare `async func` methods yet, so `TodoRepository`
  methods return `Task<T!>` and implementations delegate to private async
  methods.
- The framework turns a handler error into Problem Details at the endpoint, so
  a middleware cannot map domain errors itself; mapping is a helper the
  controller calls (`Fail<T>`), and the middleware only sanitizes 5xx bodies.
- `zell test` builds only the package's own sources (no dependencies), which
  is why the zconf-using executable lives in `server/`.

## License

MIT
