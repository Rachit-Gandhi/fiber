# app.Benchmark Issue Discussion

Status: local discussion record, not yet approved for issue tracker publication.

Date: 2026-06-14h77hhhh

## Thread Discussion Log

This section records the full discussion thread in chronological order.

### Where `app.Test()` Is Defined

Question: where is `app.Test()` defined?

Answer:

- `app.Test()` is defined in `app.go`.
- Signature:

```go
func (app *App) Test(req *http.Request, config ...TestConfig) (*http.Response, error)
```

- The method is documented in `docs/api/app.md`.

### What `app.Test()` Is For

Question: what is the scope of this method, and why would people need it?

Discussion:
1
- `app.Test()` is a public method on `*fiber.App`, not a Go interface.
- Its scope is to run one synthetic HTTP request through a Fiber app and return an HTTP response without opening a real network listener.
- It is intended for `_test.go` files and route/middleware debugging.
- It is useful for checking route matching, middleware behavior, headers, cookies, response bodies, status codes, error handlers, mounted apps, groups, and request parsing behavior.
- It is not meant for TCP behavior, TLS, listener lifecycle, load/performance, client behavior, or production networking edge cases.
- Internally, it bridges from standard `net/http` request/response types into Fiber's `fasthttp` server machinery.

### Benchmarking With Go's `testing.B`

Question: does Go have a benchmark equivalent to `httptest`?

Discussion:

- Go benchmarking is provided by the standard `testing` package through `testing.B`.
- There is no separate `net/http/benchtest` package.
- `httptest` can still be used to build synthetic HTTP requests.
- Benchmarks are functions named `BenchmarkXxx(b *testing.B)`.
- They are run with:

```bash
go test -bench=.
```

Example Fiber shape discussed:

```go
func BenchmarkRoute(b *testing.B) {
	app := fiber.New()

	app.Get("/", func(c fiber.Ctx) error {
		return c.SendString("ok")
	})

	req := httptest.NewRequest(http.MethodGet, "/", http.NoBody)

	b.ReportAllocs()
	b.ResetTimer()

	for b.Loop() {
		resp, err := app.Test(req, fiber.TestConfig{Timeout: 0})
		if err != nil {
			b.Fatal(err)
		}
		_ = resp.Body.Close()
	}
}
```

Clarification:

- `testing.B` does not return a value.
- Like `testing.T`, Go passes it into the function.
- Failures are reported through `b.Fatal`, `b.Errorf`, and related methods.
- The benchmark result is printed by the Go test runner, for example `ns/op`, `B/op`, and `allocs/op`.

### Why The Assigned Issue Exists

Question: if `app.Test()` is stable enough to benchmark with, why was the `app.Benchmark()` issue assigned?

Discussion:

- The benchmark shape using `app.Test()` works mechanically.
- The concern is measurement quality, not usability alone.
- `app.Test()` measures Fiber plus the testing harness.
- That harness includes standard-library request dumping, fake connection I/O, goroutine scheduling, channel synchronization, timeout logic, `ServeConn`, and response parsing.
- For correctness tests, that is acceptable and useful.
- For performance benchmarks, it can obscure what users are trying to measure: Fiber routing, middleware, handler execution, and response generation.

Refined view:

- `app.Test()` answers correctness questions.
- `app.Benchmark()` should answer hot-path performance questions.
- The value of a benchmark helper is not primarily output formatting.
- Go already owns benchmark formatting through `testing.B` and `go test -bench`.
- The value is preparing and invoking the Fiber request path with less harness overhead.

### Initial API Intuition

Initial intuition:

- `app.Benchmark()` might encapsulate looping.
- It might call `b.ReportAllocs()`.
- It might call `b.ResetTimer()`.
- It might format results.

Refinement:

- Formatting should probably be left to Go's benchmark runner.
- It may or may not own timer/allocation calls, depending on API shape.
- The core abstraction should probably encapsulate request preparation and lower-overhead request execution.

### Express Research Request

Question: search the internet and find what Express does for benchmark abstractions, since Fiber often emulates Express.

Findings:

- Express does not expose an `app.benchmark()` method.
- Express apps are callable Node HTTP handlers.
- Express testing commonly uses SuperTest.
- SuperTest accepts an Express app function or `http.Server`; if the server is not already listening, it binds to an ephemeral port.
- Express's `app.listen()` is a convenience around `http.createServer(app).listen(...)`.
- Benchmarking in the Express ecosystem is typically done by starting the app as a real HTTP server and driving it with external tools such as `autocannon`, `wrk`, or benchmark suites.
- `autocannon` reports latency, requests per second, bytes per second, duration, errors, timeouts, non-2xx responses, and related histogram data.
- Fastify's public benchmarks include Express by running real servers and benchmarking them with `autocannon`.

Conclusion:

- If Fiber wants to emulate Express, the relevant precedent is the separation between correctness testing and real server benchmarking.
- A Fiber `app.Benchmark()` would be a Go/Fiber-specific helper, not a direct Express API clone.
- The helper should be documented carefully so users do not confuse in-process handler benchmarking with full network/load testing.

### Local Recording Request

Request: record the issue discussion locally in `.scratch` using the `to-issues` skill.

Action:

- Created `.scratch/app-benchmark-issue-discussion.md`.
- Recorded the proposal, current Fiber context, suboptimal aspects of `app.Test()`, Express research, design direction, open questions, and a draft issue breakdown.
- Kept the issue breakdown local and unapproved rather than publishing it to an issue tracker.

## Source Prompt

Proposal:

Add a new `app.Benchmark()` method to make it easier for users to run benchmarks of their Fiber applications. The current `app.Test()` method is only intended for testing purposes and is suboptimal for running benchmarks.

## Current Fiber Context

`app.Test()` is defined in `app.go`:

```go
func (app *App) Test(req *http.Request, config ...TestConfig) (*http.Response, error)
```

Its purpose is correctness testing and route debugging. It accepts a standard `*http.Request`, runs it through Fiber, and returns a standard `*http.Response`.

The test path currently does the following:

- Adds `Content-Length` when needed.
- Ensures a `Host` is present.
- Clears `RequestURI` so `httputil.DumpRequest` emits an origin-form request.
- Serializes the `*http.Request` into raw HTTP bytes with `httputil.DumpRequest`.
- Writes those bytes into a private in-memory `testConn`.
- Calls `app.startupProcess()`.
- Runs `app.server.ServeConn(conn)` in a goroutine.
- Waits through timeout/channel machinery.
- Reads raw response bytes from the fake connection.
- Parses the response back into `*http.Response`.
- Skips interim 1xx responses when needed.

The private `testConn` is defined in `helpers.go`.

## Why `app.Test()` Is Suboptimal For Benchmarks

`app.Test()` measures more than Fiber request handling. A benchmark based on `app.Test()` includes:

- `net/http` request serialization.
- In-memory fake connection I/O.
- Goroutine scheduling.
- Channel synchronization.
- Timeout/select overhead.
- `fasthttp.Server.ServeConn` connection handling.
- Raw response parsing back into `*http.Response`.
- Potential per-call app startup checks.

That is acceptable for correctness tests because it exercises a realistic HTTP request/response boundary. It is less suitable for microbenchmarks where the user wants to measure routing, middleware, handler execution, response generation, or app hot-path behavior.

Working distinction:

- `app.Test()` answers: does this request produce the correct response?
- `app.Benchmark()` should answer: how fast does this Fiber app handle this prepared request path with minimal harness overhead?

## Go Benchmark Context

Go benchmarks use `testing.B`, not `testing.T`.

`testing.B` does not return benchmark results to the benchmark function. Instead, the Go test runner controls iteration and reports results from `go test -bench`.

Example benchmark shape today:

```go
func BenchmarkRoute(b *testing.B) {
	app := fiber.New()

	app.Get("/", func(c fiber.Ctx) error {
		return c.SendString("ok")
	})

	req := httptest.NewRequest(http.MethodGet, "/", http.NoBody)

	b.ReportAllocs()
	b.ResetTimer()

	for b.Loop() {
		resp, err := app.Test(req, fiber.TestConfig{Timeout: 0})
		if err != nil {
			b.Fatal(err)
		}
		_ = resp.Body.Close()
	}
}
```

This shape is stable and usable, but it reports the cost of the `app.Test()` harness plus Fiber handling, not a cleaner Fiber hot-path benchmark.

## Express Research

Express does not expose an `app.benchmark()` API.

Relevant Express patterns:

- Express apps are callable Node HTTP handlers.
- Express tests commonly use SuperTest: `request(app).get(...).expect(...)`.
- SuperTest accepts an `http.Server` or a function; if the server is not already listening, it binds to an ephemeral port.
- Express benchmarking is typically done by starting a real server with `app.listen()` / `http.createServer(app)` and driving it with external tools such as `autocannon`, `wrk`, or benchmark suites.
- Fastify's public benchmarks include Express by running real servers and using `autocannon` with concurrency, duration, pipelining, warm-up, and measured rounds.

Sources:

- Express app API: https://expressjs.com/en/5x/api/application/#app.listen
- Express application implementation: https://github.com/expressjs/express/blob/master/lib/application.js
- SuperTest README: https://github.com/ladjs/supertest#readme
- autocannon README: https://github.com/mcollina/autocannon#readme
- Fastify benchmarks: https://github.com/fastify/benchmarks

Conclusion: if Fiber wants to emulate Express, the precedent is not a direct `app.Benchmark()` API. Express separates correctness testing ergonomics from throughput benchmarking. A Fiber `app.Benchmark()` would be a Go/Fiber-specific ergonomic improvement, not a direct Express clone.

## Design Direction Discussed

The initial intuition was that `app.Benchmark()` might encapsulate:

- benchmark looping,
- allocation reporting,
- timer reset,
- output formatting.

Refinement:

- It should probably not own formatting. `testing.B` and `go test -bench` own formatting.
- It may reasonably own `b.ReportAllocs()` and timer boundaries, but this is a public API choice.
- Its core value should be avoiding `app.Test()` overhead by preparing the request once and invoking Fiber's request path more directly.

Potential responsibilities:

- one-time app startup,
- one-time request preparation/conversion,
- direct invocation of Fiber's `fasthttp` handler path,
- per-iteration context/request/response reset,
- optional status/body validation hooks,
- optional allocation reporting and timer reset,
- clear docs explaining this is not a network/load test.

Important API tension:

- Accepting `*http.Request` is convenient and symmetrical with `app.Test()`, but requires conversion to `fasthttp`.
- Accepting `*fasthttp.Request` is lower overhead and more honest, but less approachable for users coming from `httptest.NewRequest`.
- Accepting `*testing.B` makes it easy to encapsulate benchmark loop/timer/allocation behavior, but imports `testing` into the public Fiber package API.
- Returning a prepared runner avoids coupling the app method to `testing.B`, but may not match the proposal wording.

Candidate shapes to evaluate:

```go
func (app *App) Benchmark(b *testing.B, req *http.Request, config ...BenchmarkConfig)
```

```go
func (app *App) Benchmark(b *testing.B, req *fasthttp.Request, config ...BenchmarkConfig)
```

```go
runner, err := app.NewBenchmark(req, config...)
for b.Loop() {
	resp, err := runner.Run()
}
```

## Open Questions

- Should `app.Benchmark()` accept `*testing.B`, or should Fiber expose a prepared runner and leave `testing.B` usage to the caller?
- Should the public request type be `*http.Request`, `*fasthttp.Request`, or both?
- Should the benchmark helper expose response data, or only fail on non-success/error conditions?
- Should it support validation callbacks to prevent benchmarking broken handlers?
- Should it call `b.ReportAllocs()` automatically?
- Should it call `b.ResetTimer()` and own the loop?
- Should it support `b.RunParallel()` or leave parallel benchmarks to users?
- Should it document that external load tests are still recommended for end-to-end network throughput?

## Draft Issue Breakdown

These are local draft issues following the `to-issues` skill. They are not published and should be reviewed before conversion into tracker issues.

### 1. Define The Benchmark API Contract

Type: HITL

Blocked by: None - can start immediately

User stories covered:

- As a Fiber maintainer, I need agreement on whether `app.Benchmark()` is a `testing.B` helper, a prepared runner, or both.
- As a Fiber user, I need the API to be understandable from existing `app.Test()` experience without misleading benchmark results.

What to build:

Define the public contract for benchmarking Fiber applications. Decide whether the API accepts `*testing.B`, `*http.Request`, `*fasthttp.Request`, or a prepared runner type. Document the intended measurement boundary: Fiber handler path, not full TCP/load-test behavior.

Acceptance criteria:

- [ ] The accepted API shape is documented in a local design note or issue comment.
- [ ] The decision explicitly explains how `app.Benchmark()` differs from `app.Test()`.
- [ ] The decision explicitly states whether `testing` belongs in Fiber's public API.
- [ ] The decision identifies whether external benchmark tools remain recommended for network/load benchmarks.

### 2. Add A Minimal Prepared Benchmark Path

Type: AFK after issue 1

Blocked by: Define The Benchmark API Contract

User stories covered:

- As a Fiber user, I can benchmark a single prepared request without paying the `app.Test()` serialization/fake connection/response parsing cost.
- As a maintainer, I can verify the helper exercises the same route/middleware path expected from a normal request.

What to build:

Implement the smallest benchmark helper that prepares the app and request once, then runs the Fiber request handler path repeatedly with per-iteration reset. Keep the implementation scoped and avoid formatting benchmark results.

Acceptance criteria:

- [ ] The helper avoids `httputil.DumpRequest`.
- [ ] The helper avoids fake `net.Conn` request/response transport.
- [ ] The helper invokes Fiber's real request handling path.
- [ ] The helper has tests or benchmarks proving it can repeatedly execute the same request.
- [ ] The implementation does not change `app.Test()` behavior.

### 3. Add Benchmark Validation And Failure Semantics

Type: AFK after issue 2

Blocked by: Add A Minimal Prepared Benchmark Path

User stories covered:

- As a Fiber user, I can detect when my benchmarked route is failing instead of accidentally measuring error paths.
- As a Fiber maintainer, I can keep benchmark helper behavior predictable and testable.

What to build:

Define and implement how the benchmark helper reports handler errors, unexpected empty responses, status validation, and optional response checks. Keep validation light enough to avoid dominating the measured path unless explicitly enabled.

Acceptance criteria:

- [ ] Handler/request errors are surfaced through `testing.B` failure or returned errors, depending on the accepted API shape.
- [ ] Users can verify status code or response expectations when needed.
- [ ] Default validation overhead is documented.
- [ ] Tests cover failure behavior.

### 4. Document Benchmarking Guidance

Type: AFK after issue 2

Blocked by: Add A Minimal Prepared Benchmark Path

User stories covered:

- As a Fiber user, I understand when to use `app.Test()`, `app.Benchmark()`, and external tools.
- As a Fiber maintainer, I reduce misuse of `app.Test()` benchmark numbers.

What to build:

Add documentation explaining benchmark use cases, examples, measurement boundaries, and caveats. Include a comparison with `app.Test()` and a note that `app.Listen()` plus external tools is still the right path for end-to-end throughput/load testing.

Acceptance criteria:

- [ ] Docs include a minimal route benchmark example.
- [ ] Docs explain why `app.Test()` is suboptimal for benchmarks.
- [ ] Docs explain what `app.Benchmark()` does and does not measure.
- [ ] Docs mention external load testing for real network/server behavior.

### 5. Add Internal Benchmarks Comparing Harness Overhead

Type: AFK after issue 2

Blocked by: Add A Minimal Prepared Benchmark Path

User stories covered:

- As a maintainer, I can quantify the overhead difference between `app.Test()` and the new benchmark helper.
- As a reviewer, I can validate that the new API solves the problem it claims to solve.

What to build:

Add internal benchmarks that compare a simple route through `app.Test()` versus the new benchmark path. Use them as evidence and guardrails, not as public performance guarantees.

Acceptance criteria:

- [ ] Benchmarks include a simple static route.
- [ ] Benchmarks include at least one middleware path.
- [ ] Benchmarks report allocations.
- [ ] Benchmark names make the measured path clear.
- [ ] Results are not asserted as fixed thresholds.

## Proposed Review Questions

- Does this granularity feel right: too coarse, too fine, or about right?
- Are the dependency relationships correct?
- Should the API contract issue be split into separate `testing.B` and request-type decisions?
- Should validation be part of the first implementation slice, or kept separate?
- Are the correct slices marked HITL vs AFK?
