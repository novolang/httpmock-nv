# Changelog

All notable changes to httpmock-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.0.2 — 2026-09-16

README rewritten to the package README style guide
(docs/writing-a-readme.md).

### Fixed

- The README's example escapes the reply template.  It wrote
  `"{\"id\":${id}}"`, which novo-lang interpolates at the call site —
  `id` is not a variable there, so the block failed to compile with
  `E2003` and `novo pkg publish` refused the release over it.  `\${id}`
  is the spelling the `mockreply` doc comments already use: it passes a
  literal `${id}` through to `mockreply.render`, which is what
  substitutes the path binding.

## 0.0.1 — 2026-09-11

The **interface**: every signature and every effect row, and no bodies.
`stability = "draft"`, and the release is recorded `implemented = false`.

### Added

- `mockexpect` — `HmMethod`, `HmTimes`, `HmParamRule` and
  `HmExpectation`; `expect` over a router-nv pattern, `expect_exact`,
  the header, query and body builders over matchers-nv matchers, the
  four `HmTimes` constructors, and `describe`, the one line every
  failure is built from. Every function `[]`.
- `mockmatch` — `HmField`, `HmMiss`, `HmDecision`, `HmBinding` and
  `HmRequest`; `decide`, `matches`, `miss_of`, `bindings`; the request
  builders a test writes in three lines; `header` with RFC 7230 § 3.2's
  case folding; `json_equal`; and `unmatched_report`, which is both the
  404's body and the verification failure. Every function `[]`.
- `mockreply` — `HmHeader` and `HmReply`; `status`, `text`, `json`,
  `body`, `redirect`; `with_header`, `with_delay`, `body_literal`;
  `render` and `template_names` for `${name}` substitution from the
  path bindings; and the two replies the mock generates itself,
  `unmatched` (404) and `exhausted` (409).
- `mockserver` — `HmRule`, `HmMock`, `HmError` and
  `impl Error for HmError`; `start` and `start_on` binding port 0 on
  loopback, `base_url`, `url_for`, `stop`, `stop_within`, `with_mock`,
  `is_running`; and the three functions that name the seam with
  `std.http` — `dispatch`, `to_request`, `to_response`.
- `mockverify` — `HmRecorded`, `HmUnsatisfied` and `HmVerdict`;
  `recorded`, `request_count`, `last_request`, `served_by`,
  `call_counts`, `unexpected`, `clear` at `[mutate]`; `verify_counts`
  at `[]`; `verify`; `assert_verified` and `assert_requests` at
  `[mutate, io]`; and `report` / `transcript`.

### Known

- **`mockmatch.decide` is `[]`.** The request-to-expectation decision
  is a pure function of the expectations, the call counts and the
  request, so the half of this package where the bugs live is tested
  without a port, a process or a teardown.
- **A miss names the field.** `HmUnmatched` carries one `HmMiss` per
  expectation, with matchers-nv's `describe` and `describe_mismatch`
  either side of it — which is the difference between a usable mock and
  a bare 404.
- **Port 0, loopback only.** The kernel picks and `HmMock.port`
  reports; a hard-coded port is flaky and a scan for a free one is
  racy.
- **Teardown is explicit**, because the language has no destructors:
  `stop` for a fixture that outlives a function, `with_mock` for the
  shape that cannot be forgotten.
- **One mock per process**, because `std.http`'s async server keeps its
  queue, its flag, its worker count and its handler in module-level
  state and says so. `start` refuses a second rather than letting the
  first's tasks serve the second's expectations. **The row to widen is
  `std.http`'s**: that state belongs on the `HttpServer` value.
- **`start`'s row is `[io, fs, net, time, mutate, async]`**, not the
  plan's `[net]`, because that is exactly what `serve_async` and its
  handler are declared to cost.
- **The handler cannot capture.** `http.server`'s app is a plain `fn`,
  so `dispatch` is one static function and the expectations live with
  the running server.
- **Every request is recorded, matched or not**, and a request nobody
  expected is its own part of the verdict.
- **Two `core` dependencies**, router-nv for the path grammar and
  matchers-nv for the value matchers and the failure words.
