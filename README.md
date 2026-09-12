# httpmock-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`. Installing this package works;
calling it panics with `not implemented`.

## What this is

A stub HTTP server for tests —
[httpmock](https://docs.rs/httpmock)'s and
[responses](https://github.com/getsentry/responses)' idea: say what
requests you expect and what to answer, point the client under test at
it, and afterwards ask what actually arrived.

- `mockexpect` — an expectation, as a value: method, a router-nv path
  pattern, query and header rules over matchers-nv matchers, a body
  matcher, and a times-called constraint;
- `mockmatch` — `decide`, the whole request-to-expectation decision, at
  `[]`;
- `mockreply` — what an expectation answers with, also a value;
- `mockserver` — the socket: port 0, the base URL, the explicit stop;
- `mockverify` — what the mock saw, and whether that is what was asked
  for.

```
novo pkg add httpmock-nv
novo pkg build
novo test
```

## The one example that will work

```novo ignore
use std.test
use std.http
use mockexpect
use matchtext
use mockreply
use mockserver
use mockverify

fn the_client_sends_its_token(m: HmMock) -> Bool [io, net, async]
    match http.client.get_with_headers(mockserver.url_for(m, "/users/7"),
                                       "Authorization: Bearer t0ken")
        Some(body) => body == "{\"id\":7}"
        None       => false

@test
fn test_the_client_authenticates() [io, fs, net, time, mutate, async]
    let rs = [mockserver.rule(
                mockexpect.with_header(mockexpect.expect(HmGet, "/users/:id"),
                                       "Authorization",
                                       matchtext.starts_with("Bearer ")),
                mockreply.json(200, "{\"id\":${id}}"))]
    match mockserver.with_mock(rs, the_client_sends_its_token)
        Err(e) => test.fail(e.message())
        Ok(ok) => test.assert(ok)
```

No port is written anywhere: `start` binds port 0, the kernel picks,
and `url_for` is what the client gets.

## The load-bearing interface: `HmDecision`, and `decide` is `[]`

```novo ignore
pub fn decide(es: [HmExpectation], calls: [Int], r: HmRequest) -> HmDecision []

pub enum HmDecision
    HmServe(index: Int, params: [HmBinding])
    HmUnmatched(misses: [HmMiss])
    HmExhausted(index: Int, label: Str, limit: Int)
```

Given the expectations, how many times each has already been served,
and one request, the decision is a **pure function**. Two things fall
out of that, and both are the reason for the shape.

**The matching is testable without a socket.** A mock server's bugs are
almost never in the socket; they are in *why did my expectation not
match* — a header compared case-sensitively, a path that needed its
trailing slash, a body whose JSON members came out in another order.
Every one of those is a value in and a value out in this package's own
suite: no port, no process, no teardown, no flake.

**The failure can name the field.** Every other mock answers a bare 404
and leaves the reader guessing. `HmUnmatched` carries one `HmMiss` per
expectation — the index, the field, what was wanted
(matchers-nv's `describe`) and what arrived (`describe_mismatch`):

```
no expectation matched POST /users
  #0 the token endpoint — method: wanted GET, was POST
  #1 create a user      — header Authorization: wanted a string
                          starting with "Bearer ", was "Basic dXNlcg=="
```

That report is the 404's body as well as the verification failure, so
the two cannot drift.

Fields are checked in a fixed order — method, path, query, headers,
body — and only the **first** miss per expectation is reported: cheapest
first, the order a person reads a request in, and one line per
expectation rather than four at the moment a report most needs to be
readable.

## A random port, and an explicit stop

`start` binds **port 0**, which asks the kernel for an unused port, and
`HmMock.port` is what it gave. The alternatives are both broken and both
common: a hard-coded port is flaky under a parallel run and under
`TIME_WAIT`, and scanning for a free one is racy because the scan and
the bind are two steps. `net.local_port` exists for exactly this, and
`HttpServer.port()` surfaces it.

It binds **loopback only**. A test's stub server has no reason to be
reachable from the network, and a mock that bound `0.0.0.0` would be a
listening service on every machine that ran the suite.

Teardown is explicit, because the language has no destructors — the same
fact tempdir-nv is built around. `stop` drains, closes the listener and
answers; `with_mock(rules, named_fn)` is the shape that cannot be
forgotten and stops the server whether the body returned or panicked.
Both are published, because a suite-wide mock outlives one function and
a per-test mock should not need a teardown hook.

## One mock per process, and that is a row to widen

`std.http`'s async server keeps its connection queue, its shutdown flag,
its live-worker count and its handler in **module-level state**, and its
own documentation says so: *"a second `serve_async` overwrites the
first's shared state, and the first's tasks then serve the second's
app."*

So `start` refuses a second mock with `HmAlreadyRunning` rather than
producing one that silently answers the other's expectations, and
`mockverify.clear` exists so a suite can share one mock across tests
without a rebind.

A suite that needs two services at once — a client that talks to an
authentication server and an API — needs `std.http` to carry that state
on the `HttpServer` value instead. That is the row to widen, and it is
named here rather than worked around.

## Why `start`'s row is six labels and not `[net]`

`http.server.serve_async` is declared `[io, fs, net, time, mutate,
async]`, and so is the handler it takes. Anything that raises a server
inherits all six: `[net]` for the bind and the sockets, `[async]`
because the accept loop and the workers are tasks, `[mutate]` for the
shared queue and the live-worker count, `[time]` for the read budget,
and `[io]`/`[fs]` from the stdlib's own handler row.

The half that matters is narrow, and that is the point of the split:

| what | row |
| --- | --- |
| every expectation, every reply, `decide`, every report | `[]` |
| reading the recording, `verify`, `clear` | `[mutate]` |
| `start`, `dispatch`, `with_mock` | `[io, fs, net, time, mutate, async]` |
| `stop` | `[io, net, time, mutate, async]` |

Also from `http.server`: the app is a plain `fn` with **no capture**, so
the expectations cannot live in a closure. `mockserver.dispatch` is
published as that one static function, which is also how a caller mounts
the mock's behaviour inside a server of their own.

## What the mock saw

The mock records **every** request, matched or not, with the misses
attached. A request nobody expected is as interesting as an expectation
nobody called — a client that fires a telemetry request the test never
intended is a real defect — and most mock libraries report only the
second. Here it is `HmVerdict.unexpected`.

`verify_counts` is the whole rule at `[]`: the rules, the counts and the
recording in, a verdict out. `verify` is that over a running mock's
state, and `assert_verified` is the wrapper that reports through
`std.test` — the same split snapshot-nv draws between `check` and
`assert_snapshot`, for the same reason.

## Two dependencies, both `core`

**router-nv** for the path patterns. An expectation matches
`/users/:id/posts` rather than a literal, and router-nv already owns
that grammar, the trailing-slash and case policies, and the span-based
parameter capture — so this package writes no path matching of its own.

**matchers-nv** for the value matchers and the failure text. A header or
body expectation is a `Matcher<Str>`, so `contains_text`,
`matches_regex` and every combinator mean here what they mean elsewhere
in the suite; and `describe` / `describe_mismatch` are what turn
"expectation 1 did not match" into a sentence.

## Reference implementation

[httpmock](https://docs.rs/httpmock) (Rust) for the expectation builder,
the times-called constraints and the verification;
[responses](https://github.com/getsentry/responses) (Python) for the
recorded-calls list. The departures are the pure decision, the
per-expectation miss report, and the explicit one-per-process
constraint.

## Status

Interface only. Every body is `todo()`; `novo pkg build` type-checks and
effect-checks the whole surface, and `novo test tests` is red until the
bodies land.
