# httpmock-nv

A **stub HTTP server** stands in for a real service while a test runs.
The test says what requests it expects and what to answer, points the
client under test at the stub instead of the service, and afterwards
asks what actually arrived. The idea and the vocabulary are
[httpmock](https://docs.rs/httpmock)'s in Rust and
[responses](https://github.com/getsentry/responses)' in Python. This
package brings them to novo-lang, over
[matchers-nv](https://novo-lang.org/packages/matchers-nv) for the value
matchers and the words a failure is written in, and
[router-nv](https://novo-lang.org/packages/router-nv) for the path
patterns.

**Status: NOT IMPLEMENTED — interface only.** Every function is declared
with its full signature, but every body is a `todo()` that panics when
called. The package is published so its design can be reviewed and
depended on before it is implemented. Version 0.1.0 will be the first
working release.

## What it is

An **expectation** is a value that says what a request has to look like:
a method, a path pattern, query parameters, headers, a body, and how
many times it may be served. A **reply** is a value that says what to
answer with: a status, headers, a body and an optional delay. A **rule**
is one of each. A **mock** is a server started from a list of rules.

The path is a pattern rather than a literal. `/users/:id` matches
`/users/7` and **binds** `id` to `7`, which is router-nv's grammar. A
reply's body may refer to a binding by writing `${id}`, and the
substitution happens when the reply is rendered.

A header or body expectation is a `Matcher<Str>`, which is matchers-nv's
type: a value that both judges a string and says, in words, what it was
looking for. `matchtext.starts_with("Bearer ")` is one. Because the
matcher can describe itself, the mock can say what it wanted as well as
what it got.

The heart of the package is one function that declares no effects.
`mockmatch.decide` takes the expectations, how many times each has
already been served, and one request, and answers what happens to it.
The answer is one of three things: an expectation serves it, nothing
matched and here is why each expectation did not, or an expectation
matched but has already been served as many times as it allows. Because
that is a value in and a value out, the part of a mock server where the
mistakes actually live — a header compared case-sensitively, a path that
needed its trailing slash, a body whose JSON members came out in another
order — is tested with no port, no process and no teardown.

The server half is the socket around it. It binds a port the kernel
chooses, on the loopback interface, runs four worker tasks, records
every request it saw whether or not anything matched, and is stopped
explicitly.

## Install

```
novo pkg add httpmock-nv
```

## Example

```novo
use std.test
use std.http
use matchtext
use mockexpect
use mockreply
use mockserver

// The code under test. It is handed the mock and asks it for a user.
fn the_client_sends_its_token(m: HmMock) -> Bool [io, net, async]
    // `url_for` is where the port comes from; nothing hard-codes one.
    match http.client.get_with_headers(mockserver.url_for(m, "/users/7"),
                                       "Authorization: Bearer t0ken")
        Some(body) => body == "{\"id\":7}"
        None       => false

@test
fn test_the_client_authenticates() [io, fs, net, time, mutate, async]
    let rules = [mockserver.rule(
                   // A GET whose path binds `id`, and whose
                   // Authorization header starts with "Bearer ".
                   mockexpect.with_header(mockexpect.expect(HmGet, "/users/:id"),
                                          "Authorization",
                                          matchtext.starts_with("Bearer ")),
                   // `${id}` is replaced by what the path bound.
                   mockreply.json(200, "{\"id\":${id}}"))]

    // The server is started, the body runs, and the server is stopped
    // whether the body returned or panicked.
    match mockserver.with_mock(rules, the_client_sends_its_token)
        Err(e) => test.fail(e.message())
        Ok(ok) => test.assert(ok)
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: httpmock-nv.<module>.<fn>` panic. The tests are the
specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `mockexpect` | The expectation: the methods, the times-called constraint, the builders that add a header, a query parameter or a body rule, and the one-line description every failure is written from. |
| `mockmatch` | The decision: the request as this package sees it, the three answers, the miss that names a field, the builders a test writes a request with in three lines, and the report the two failure paths share. |
| `mockreply` | The reply: the five constructors, the header and delay builders, the `${name}` substitution and the names a template uses, and the two replies the mock generates itself. |
| `mockserver` | The socket: a rule, a running mock, the refusals, starting and stopping, the scoped form, and the three functions that name the seam with `std.http`. |
| `mockverify` | What the mock saw: one record per request with the rule that served it, the readers over the recording, the verdict and the rule that produces it, the two assertions, and the transcript. |

## How to choose an entry point

**`mockexpect.expect` takes a path pattern and `expect_exact` takes a
literal.** Use the second when the path contains characters the pattern
grammar would read, and `mockexpect.path_ok` to check a pattern before
building an expectation from it.

**`mockserver.with_mock` starts and stops around one function.** It is
the shape that cannot be forgotten. `mockserver.start` and
`mockserver.stop` are the pair to use when the mock outlives a single
function, such as a fixture shared by a whole file.

**`mockmatch.decide` is the whole decision and takes no server.** Write
a request with `mockmatch.request` and its builders, pass the
expectations and the call counts, and assert the answer. This is how a
test of matching behaviour should be written.

**`mockmatch.matches` and `.miss_of` are the same for one expectation.**
`matches` answers yes or no; `miss_of` answers the field that failed.

**`mockverify.verify` reads a running mock and `verify_counts` does not.**
`verify_counts` takes the rules, the counts and the recording and
answers a verdict, so the verification rule itself is testable without a
server. `mockverify.assert_verified` is the wrapper that reports the
verdict through `std.test`.

**`mockserver.dispatch` is the handler itself.** A caller who wants the
mock's behaviour inside a server of their own mounts this rather than
reimplementing it.

## The rules a user needs

1. **The decision is a pure function, and it takes the call counts as an
   argument.** `mockmatch.decide(expectations, calls, request)` answers
   `HmServe(index, bindings)`, `HmUnmatched(misses)` or
   `HmExhausted(index, label, limit)`. Passing an empty list for `calls`
   means nothing has been served yet.
2. **First match wins, in list order.** Put the more specific
   expectation first.
3. **A miss names one field, and the fields are checked in a fixed
   order**: method, path, query, headers, body. Only the first miss per
   expectation is reported, so a report has one line per expectation
   rather than four. `HmField` is a type, so a test can assert the field
   rather than read a sentence.
4. **"Never matched" and "called too often" are different answers.**
   `HmExhausted` is not a miss. A mock that reported the second as the
   first would send a reader looking at their matchers for a problem
   that is in their call count.
5. **The unmatched report is both the 404 body and the verification
   failure.** `mockmatch.unmatched_report` writes it, from the same
   misses, so the two cannot drift. It reads like this:

   ```
   no expectation matched POST /users
     #0 the token endpoint — method: wanted GET, was POST
     #1 create a user      — header Authorization: wanted a string
                             starting with "Bearer ", was "Basic dXNlcg=="
   ```

6. **Header names are compared case-insensitively.** That is
   [RFC 7230](https://www.rfc-editor.org/rfc/rfc7230) section 3.2 and
   not a convenience. `mockmatch.header` folds case too.
7. **A query parameter the expectation does not name is ignored.** A
   mock that required an exact query string would break the first time a
   client added a trace parameter. Name the parameters that matter.
8. **A header rule can be conditional.**
   `mockexpect.with_optional_header` means "if this header is present it
   must look like this", which is how optional authentication is
   expressed. Everything else a caller writes is required.
9. **An expectation with no body rule ignores the body.**
   `mockexpect.with_body` takes a matcher, `.with_body_text` a literal
   string, and `.with_body_json` a JSON document compared by value, so
   member order does not matter.
10. **The times-called constraint is enforced at the call, not at
    verification.** `HmTimes` has a floor and a ceiling.
    `mockexpect.any_times`, `.exactly`, `.at_least` and `.at_most` build
    them; `at_least: 0` makes an expectation optional and a ceiling of
    `-1` means no ceiling. The request past the ceiling is answered
    `HmExhausted`, which is what makes "called twice when it should have
    been cached" a failure at the second call.
11. **A pattern that does not parse becomes an expectation that matches
    nothing and says so in its label.** `mockexpect.expect` is not
    fallible, so a caller does not write a `match` at every line;
    `mockexpect.path_ok` is where a caller is told, and
    `mockserver.start` refuses a rule whose pattern is bad with
    `HmBadPattern`.
12. **`${name}` in a reply is substituted from the path bindings, and an
    unbound name is left as written.** A response carrying `${id}` where
    the pattern never bound `id` is a mistake in the test, and replacing
    it with an empty string would hide it.
    `mockreply.template_names` lists the names a reply uses, so a test
    can assert they are the ones the pattern binds.
    `mockreply.body_literal` turns substitution off for a body that
    contains `${` on purpose.
13. **The port is the kernel's, and the mock listens on loopback only.**
    `start` binds port 0, which asks for any unused port, and
    `HmMock.port` reports what it got. A hard-coded port is flaky under
    a parallel run and under `TIME_WAIT`, and scanning for a free one is
    racy because the scan and the bind are two steps. Use
    `mockserver.url_for` and write no port anywhere.
14. **Teardown is explicit, because the language has no destructors.**
    `mockserver.stop` drains what is in flight, closes the listener and
    answers. `mockserver.stop_within` takes the grace period;
    `mockserver.default_grace_ms` is 500. `mockserver.with_mock` stops
    the server whether the body returned or panicked.
15. **One mock runs per process.** `std.http`'s asynchronous server
    keeps its connection queue, its shutdown flag, its live-worker count
    and its handler in module-level state, and its own documentation
    says a second `serve_async` overwrites the first's. So `start`
    refuses a second with `HmAlreadyRunning` rather than producing one
    that silently answers another's expectations.
    `mockserver.is_running` is the question a shared fixture asks, and
    `mockverify.clear` lets a suite reuse one mock across tests without
    rebinding.
16. **The handler cannot capture anything.** `std.http`'s app is a plain
    function, so the expectations live with the running server rather
    than in a closure, and `mockserver.dispatch` is that one static
    function.
17. **Every request is recorded, matched or not.** A request nobody
    expected is as interesting as an expectation nobody called: a client
    that fires a telemetry request the test never intended is a real
    defect. `HmVerdict` has all three — the rules that were not called
    enough, the requests nothing matched, and the requests that matched
    a rule already out of calls.
18. **Starting a server costs six effects, and almost nothing else
    does.** `http.server.serve_async` and the handler it takes are
    declared `[io, fs, net, time, mutate, async]`, so anything that
    raises a server inherits all six.

    | What | Effects |
    | --- | --- |
    | every expectation, every reply, `decide`, every report | none |
    | reading the recording, `verify`, `clear` | `[mutate]` |
    | `start`, `start_on`, `dispatch`, `with_mock` | `[io, fs, net, time, mutate, async]` |
    | `stop`, `stop_within` | `[io, net, time, mutate, async]` |
    | `assert_verified`, `assert_requests` | `[mutate, io]` |

19. **A mock runs four worker tasks.** `mockserver.workers` answers 4:
    enough that a test of a client's own concurrency sees more than one
    request in flight, and few enough that a suite starting a mock per
    test is not building a pool each time.

## What is not included

- **A second mock in the same process.** See rule 15. The fix belongs in
  `std.http`, which should carry that state on its `HttpServer` value
  rather than at module level.
- **An HTTP client.** `std.http`'s client is what a test points at the
  mock.
- **Path matching of its own.** router-nv owns the pattern grammar, the
  trailing-slash and case policies, and the parameter capture.
- **Value matchers of its own.** matchers-nv owns those, so a header
  rule behaves the way every other assertion in the suite behaves.
- **A binary request body.** `HmRequest.body` is text. A client's
  request bodies are text in every case someone writes a matcher for,
  and a binary body matches only a matcher that ignores it.
- **Proxying or recording a real service.** The rules are written by
  hand.
- **Running on a microcontroller.** No such claim is made. The package
  binds a socket.

## Related packages

- [matchers-nv](https://novo-lang.org/packages/matchers-nv) is where a
  header, query or body rule comes from. `matchers.describe` and
  `.describe_mismatch` are what turn "expectation 1 did not match" into
  a sentence naming the field, so a failure here reads like every other
  assertion in the suite.
- [router-nv](https://novo-lang.org/packages/router-nv) is the path
  grammar. An expectation holds one of its patterns and one of its
  policies.
- [snapshot-nv](https://novo-lang.org/packages/snapshot-nv) compares a
  value against a stored copy of itself. It draws the same split this
  package does between a check that answers and an assertion that
  reports, for the same reason.
- [tempdir-nv](https://novo-lang.org/packages/tempdir-nv) is the other
  test fixture that must be torn down by hand, and it has the same
  scoped form for the same reason.
- `std.http` in the standard library is the server underneath and the
  client a test uses. `mockserver.to_request` and `.to_response` are the
  two conversions across that seam.

## Tests

```bash
novo test --isolate tests/mockmatch_tests.nv   # 12 tests: the decision
novo test --isolate tests/mockreply_tests.nv   # 12 tests: the replies and the substitution
novo test --isolate tests/mockserver_tests.nv  # 12 tests: the socket, the refusals and the seam
novo test --isolate tests/mockexpect_tests.nv  # 10 tests: the expectation and the times constraint
novo test --isolate tests/mockverify_tests.nv  # 11 tests: the recording and the verdict
```

The behaviour asserted is httpmock's, for the expectation builder, the
times-called constraints and the verification, and responses', for the
list of recorded calls. RFC 7230 section 3.2 is the authority for the
case folding of header names.

Most of the suite needs no server. `mockmatch_tests.nv` builds requests
with `mockmatch.request` and asserts the decision: which expectation
wins when two could, that a miss names the field and the first field
only, that a header name matches in any case, that an unnamed query
parameter is ignored, and that being out of calls is a different answer
from not matching. `mockverify_tests.nv` does the same for
`verify_counts`, which takes the recording as an argument.
`mockserver_tests.nv` holds the part that genuinely needs a socket: that
the port is the kernel's and is reported, that a second `start` is
refused, that stopping twice is not a failure, that a rule whose pattern
does not parse is refused at `start`, and that the two conversions across
the `std.http` seam declare no effects.

The tests compile today and fail at run, each on the
`not implemented: httpmock-nv.<module>.<fn>` panic that is its body.
That is the expected state of an interface release. They turn green one
at a time as bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| `mockexpect.expect`, `.expect_exact`, `.path_ok`, `.labelled`, `.with_policy` | no |
| `mockexpect.with_header`, `.with_optional_header`, `.with_header_value` | no |
| `mockexpect.with_query`, `.with_query_value` | no |
| `mockexpect.with_body`, `.with_body_text`, `.with_body_json` | no |
| `mockexpect.any_times`, `.exactly`, `.at_least`, `.at_most`, `.with_times` | no |
| `mockexpect.times_satisfied`, `.times_exhausted` | no |
| `mockexpect.method_name`, `.method_named`, `.describe` | no |
| `mockmatch.request`, `.request_target`, `.with_query`, `.with_header`, `.with_body` | no |
| `mockmatch.decide`, `.matches`, `.miss_of`, `.bindings`, `.binding`, `.header` | no |
| `mockmatch.decision_name`, `.field_name`, `.miss_line`, `.request_line` | no |
| `mockmatch.unmatched_report`, `.describe_wanted`, `.describe_got`, `.json_equal` | no |
| `mockreply.status`, `.text`, `.json`, `.body`, `.redirect` | no |
| `mockreply.with_header`, `.with_delay`, `.body_literal`, `.header` | no |
| `mockreply.render`, `.template_names`, `.reply_line` | no |
| `mockreply.unmatched`, `.exhausted` | no |
| `mockserver.rule`, `.start`, `.start_on`, `.with_mock`, `.is_running` | no |
| `mockserver.base_url`, `.url_for`, `.stop`, `.stop_within` | no |
| `mockserver.dispatch`, `.to_request`, `.to_response` | no |
| `mockserver.workers`, `.default_grace_ms`, `mockserver.HmError.message` | no |
| `mockverify.recorded`, `.request_count`, `.last_request`, `.served_by` | no |
| `mockverify.call_counts`, `.unexpected`, `.clear` | no |
| `mockverify.verify`, `.verify_counts`, `.satisfied` | no |
| `mockverify.report`, `.unsatisfied_line`, `.transcript` | no |
| `mockverify.assert_verified`, `.assert_requests` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
