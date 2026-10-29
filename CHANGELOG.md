# Changelog

All notable changes to s3-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.0 — 2026-10-29

Every body is written.  The package is an S3 client: Signature Version 4,
presigned URLs, the operations, the listing loop, the multipart state
machine and a transport over the standard library's HTTP client.

- **The signatures are AWS's.**  Five published SigV4 examples reproduce
  exactly: the `GET` with a `Range` header, the `PUT` of
  `test$file.text`, `GET ?lifecycle`, `GET ?max-keys=2&prefix=J` and the
  presigned `GET`.  Percent-encoding is spelled out in `s3sig` and not
  taken from url-nv, whose `PctSet` leaves `!'()*` alone and SigV4
  encodes them.
- `s3cfg.aws("us-east-1", …)` is host `s3.amazonaws.com`, the spelling
  AWS's own examples sign; every other region is `s3.<region>.amazonaws.com`.
- A `HEAD` that fails has no body, so its fault carries an empty
  document; `s3fault.is_no_such_key` counts an empty-bodied 404 as a
  missing key.
- The dependency ranges move to the implemented releases: xml-nv `^0.1.1`,
  url-nv `^0.1.4`, mime-nv `^0.1.1`, calendar-nv `^0.2.1`.

### Changed

- `s3sig.signing_key` takes `var out: Bytes` and `s3client.drain_into`
  takes `var into: Bytes`.  Both always wrote into the caller's buffer;
  the compiler now requires the parameter to say so (E2036, E2050).
  Callers that pass a fresh buffer write `var buf` at the call.
- `S3Mpu` gains two fields, `failure: ?S3Fault` and
  `outcome: ?S3UploadDone`.  `s3mpu.fail` answers an `S3Mpu` and
  `s3client.pump` answers an `S3Mpu` after the completion, and neither
  had anywhere to put the failure or the completion's ETag.  Nobody
  outside the package builds an `S3Mpu` by hand, so no caller breaks.
- `s3sig.authorization_headers` answers `x-amz-date`,
  `x-amz-content-sha256`, `x-amz-security-token` when there is one, and
  `Authorization`.  It does not answer `Host`: the signature covers the
  host that `s3cfg.host_header` names, and the transport sets `Host`
  from the origin `s3cfg.origin` names.

### Known

- `s3client.std_http` still refuses an `https` endpoint, as the
  published tests require.  The standard library documents `https` for
  every verb in toolchain 0.22; this release has not tried it.
- The shipped transport buffers an `s3_open` reply: its `s3_read`
  answers an empty body and the reply carries the whole of it.  A
  transport that is a copyable struct cannot keep a connection open
  between two calls.
- `s3op.conditional_writes_supported` answers `Some(true)` for AWS hosts
  and `None` for every other endpoint; no table of compatible services
  was checked.
- `s3op.delete_many`, `s3mpu` and `s3sig.presign` refuse a bad argument
  with the nearest existing fault (`S3FaultBadKey`, `S3FaultPartTooSmall`,
  `S3FaultPresignTooLong`, or `S3FaultService` carrying `EntityTooLarge`,
  `InvalidPart` or `NoSuchUpload`), because `S3Fault` has no variant for
  them.
- `s3sig.signing_key` refuses a buffer that is not 32 bytes with
  `S3FaultBadEndpoint`, for the same reason.
- The multipart reply to a `ListMultipartUploads` request has no reader.

## 0.0.4 — 2026-10-25

A file that names a sibling module's type, function or constant now has the `use` line for that module (E2033, E2003).  No signature and no behaviour changed.

## 0.0.3 — 2026-09-25

The package builds with novo 0.11.  Every body is still `todo()`.

- The lock file moves crypto-nv 0.1.3 to 0.1.6 and url-nv 0.1.1 to
  0.1.3.  crypto-nv 0.1.3 and url-nv 0.1.1 write into lists through
  names that are not declared `var`, which novo 0.11 refuses (E2038), so
  this package did not build with novo 0.11 against them.  No
  requirement in the manifest changed.

## 0.0.2 — 2026-09-15

README rewritten to the package README style guide (docs/writing-a-readme.md); no
change to this package's own interface.  The xml-nv range moves to `^0.0.2`, the
version whose `XmlFault` declares the `impl Error` that
`Result<_, xmlerror.XmlFault>` has required since SPEC § 3.4 and that the
compiler now enforces across modules.

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

### Design notes

Recorded here because the 0.0.2 README no longer carries them.

**Why `layer = "host"` and not a `core` half.** Seven of the eight
modules declare no effects, which is the ratio tls-nv used to argue for
a `tls-core-nv`.  The answer here is different, and the recommendation
is to leave the package whole.  No device talks to an object store:
Signature Version 4 needs SHA-256 and HMAC over arbitrary-length
strings, the operations need a URL parser and an XML parser, and the
object that comes back is measured in megabytes.  Nothing else
terminates S3, so there is no second consumer of a canonical request.
And a caller that wants to sign a request and send it through its own
HTTP stack can already take this package and call `s3sig` and `s3op`,
because depending on a `host` package does not make the caller's own
functions cost anything.  The modules are arranged so that a split
remains possible without moving a line: `s3client` depends on the other
seven and nothing depends on it.

**Where the effect labels come from.** `[io, net, time, async]` on the
shipped transport is `std.http`'s own `HttpClient.send` row, not a
choice this package made: the standard library's HTTP client puts its
deadline behind a clock, its socket behind the async runtime, and the
socket layer itself declares `[io]`.  Every sending function in
`s3client` is effect-polymorphic over the transport, so a recorded
transport costs nothing.  `s3client.now_civil` is the one `[time]`
function, because every signing function takes the instant it signs for
and a signer that read its own clock could not be run against AWS's
published examples.  `s3client.credentials_from_env` is the one `[io]`
function, and it is opt-in rather than a default because reading
`AWS_ACCESS_KEY_ID` decides, for every program that links the package,
that an environment variable is a credential source.

**What a registry backend would call.** The novo-lang registry backend
stores every release as a small set of files under
`<root>/packages/<name>/<version>.*`.  Moving that store to an
S3-compatible bucket is `put_object` per file, `get_object` or a
presigned URL for a download, `head_object` for the existence check
before a publish, `list_page` with a prefix for the index rebuild, and
`put_object` of an empty object for a yank marker.  Three things that
port would have to decide.  The index's atomic rename does not survive:
a `PUT` of one key is atomic, so the rename is unnecessary, but the
read-modify-write around it is now racy between two publishing
processes, which argues for keeping the write-ahead log as the authority
and the index as a cache.  A published release is permanent, so
`delete_object` has no caller at all and the bucket's lifecycle policy
becomes the only thing that could delete one.  And the tarball upload
wants `S3PayloadSha256`, because the backend already computes that hash
and handing it to the signer puts the archive's integrity inside the
signature.

**What changes from the reference implementations.** `aws-sdk-s3` for
the operation surface and `boto3` for the subset; the canonical request,
the string to sign and the signing-key derivation come from AWS's own
Signature Version 4 specification and its worked examples.  Four things
differ in the port.  Pagination is a value and the loop is the caller's,
where boto3 hides it behind a paginator and aws-sdk-s3 behind a stream.
The multipart upload is a state machine with no threads and no input or
output, where boto3's `TransferManager` owns threads and reports
progress through a callback; a resumable upload across a process restart
is therefore a matter of storing the value.  The credentials are
arguments, where both references build a credential chain and make it
the default.  And the payload hash is a type in the signature of the
function that signs, where both references bury it in a client
constructor as a flag.
