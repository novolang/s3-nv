# Changelog

All notable changes to s3-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.0.1 — 2026-09-12

The **interface**: every signature and every effect row, and no bodies.
`stability = "draft"`, and the release is recorded `implemented = false`.

### Added

- `s3fault` — the error codes and what is worth sending again, `[]`
  throughout: the status is not the answer and the `<Code>` element
  inside the body is, `is_retryable` is one table rather than a class
  check, and `fix_hint` writes the sentence that names the real repair
  for the three failures whose own message does not.
- `s3cfg` — endpoint, region, credentials and addressing style, `[]`
  throughout: the style is in the value because it changes the string
  that gets signed, a dotted bucket name is forced to path-style
  VISIBLY, and `aws`, `minio` and `r2` differ in three fields.
- `s3sig` — Signature Version 4, `[]` throughout and the instant an
  argument everywhere: `S3PayloadHash` as the load-bearing type, two
  encodings of one path kept as two functions, the signing key keyed by
  the day, and `presign` with no payload parameter and a seven-day
  ceiling.
- `s3req` — a request and a reply as values, `[]` throughout: the body
  is a PLAN and not bytes, headers are a list of pairs and not a map,
  and the three range spellings that differ by one character each have
  a named constructor.
- `s3op` — one builder and one reader per operation, `[]` throughout:
  a listing ends on `is_truncated` and never on an empty page, a
  multipart ETag is not the MD5 of anything, and a 200 on COMPLETE or
  COPY can still be a failure document.
- `s3xml` — the reply documents over xml-nv, `[]` throughout: the S3
  namespace is matched rather than ignored, keys are asked for
  URL-encoded so that a key XML cannot carry survives, and a batch
  delete's per-key failures are answered rather than collapsed.
- `s3mpu` — the multipart upload as a state machine, `[]` throughout:
  `S3MpuMustAbort` because an abandoned upload is billed forever, the
  part plan computed before anything is sent because `EntityTooSmall`
  arrives after every byte is uploaded, and `resume` from an id and a
  `ListParts`.
- `s3client` — the host half: the `S3Transport[e]` trait, one
  transport over `std.http` at `[io, net, time, async]`, one `[time]`
  function and one `[io]` function.
- API tests in `tests/s3sig_tests.nv` — against AWS's own published
  worked example — and `tests/s3flow_tests.nv`, whose transport
  implements `S3Transport[]` so the whole client is driven with no
  socket at all.  `tests/s3surface_tests.nv` names every remaining
  `pub` item from outside the package, so the release's signatures are
  checked as a consumer would spell them.  Red until the bodies land.

### Named as missing

**A TLS transport.**  `std.http` routes `https` to one-shot entries
that cover GET and POST, answer no response headers and synthesise a
200; an object store needs PUT, HEAD, DELETE, the ETag and the real
status.  `s3client.std_http` refuses an `https` endpoint by name, and
the row that closes it is tls-nv's.  acme-nv reported the same gap from
the other side.
