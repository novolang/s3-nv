# s3-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`.  Installing this package works;
calling it panics with `not implemented`.

## What this is

An S3-compatible object store client, written in novo-lang: Signature
Version 4 as pure arithmetic over crypto-nv, the object and bucket
operations a program actually uses, `ListObjectsV2` with pagination the
caller drives, a multipart upload as a state machine the caller pumps,
the XML replies over xml-nv, and an endpoint value that makes AWS,
MinIO and Cloudflare R2 the same three fields.

The transport is a trait, so the whole of it runs with no socket: the
test suite signs AWS's own published worked example and gets AWS's own
signature, and drives a complete multipart upload over a recorded
exchange at `[]`.

It is the S3 REST API subset a program uses, not the whole of it.  The
section at the bottom says what is out and why for each.

## Adding it, and checking it

```bash
novo pkg add s3-nv            # into your novo.toml
novo pkg build                # type- and effect-check the package
novo test --isolate tests/s3sig_tests.nv
```

`novo test` is red today and that is the point of the release: every
assertion fails with `not implemented: s3-nv.<module>.<fn>`.  They turn
green one at a time as bodies land.

## The one example that will work

```novo
use s3cfg
use s3client

// Put one object into a MinIO bucket and read it back.
//
// The transport is built once and passed to every call; the instant is
// an argument, because nothing in the signing path reads a clock.
fn round_trip(creds: S3Credentials, data: Bytes) -> Result<Bytes, S3Fault> [io, net, time, async]
    let cfg = s3cfg.minio("http", "127.0.0.1", 9000, "us-east-1", creds)
    let c = s3client.client(cfg)
    let t = s3client.std_http(cfg, 30000)!
    let at = s3client.now_civil()

    let _put = s3client.put_object(c, t, "photos", "cat.jpg", data, at)!
    let got = s3client.get_object(c, t, "photos", "cat.jpg", at)!
    Ok(got.body)
```

## The layer, and why

`host`, and seven of the eight modules declare nothing.

| module | row | why |
| --- | --- | --- |
| `s3fault` | `[]` throughout | the error codes and what is worth retrying |
| `s3cfg` | `[]` throughout | endpoint, region, credentials, addressing style |
| `s3sig` | `[]` throughout | SigV4: HMAC over strings the caller already holds |
| `s3req` | `[]` throughout | a request and a reply as values |
| `s3op` | `[]` throughout | one builder and one reader per operation |
| `s3xml` | `[]` throughout | the reply documents, over xml-nv's tree |
| `s3mpu` | `[]` throughout | the multipart upload state machine |
| `s3client.send`, and every operation over it | `[e]` | effect-POLYMORPHIC: whatever the caller's transport costs |
| `s3client.now_civil` | `[time]` | the one function in the package that reads a clock |
| `s3client.credentials_from_env` | `[io]` | the one function that reads the environment, opt-in |
| the `S3Transport` impl for `S3StdHttp` | `[io, net, time, async]` | `std.http`'s `HttpClient.send` row, exactly |

**Why `[io, net, time, async]` and not `[net]`.**  That is
`std.http`'s own row and not a choice this package made: `HttpClient`
puts its deadline behind a clock, its socket behind the async runtime,
and the socket layer itself declares `[io]`.  llm-client-nv measured
the same four on the same surface and answered by shipping a second,
narrower transport over `std.tls`; this package ships the trait and one
transport, because the narrow path needs TLS and the next section is
about why there is not one yet.

**`layer = "host"` and not `layer = "core"` with `host_modules`**,
because the subject of the package is a request that reaches a bucket.
See "Should there be an `s3-core-nv`" below — the answer this lane
reached is no, and the reason is worth writing down.

## The load-bearing interface

**`S3PayloadHash`**, and the property it encodes is that SigV4 has to
know the body before the first byte of it is sent.

The signature covers the SHA-256 of the payload.  There are exactly
three answers to "what is that hash", and they are three different
trades:

| case | what it costs | what it buys |
| --- | --- | --- |
| `S3PayloadSha256(hex)` | a pass over the object before the upload starts | integrity end to end |
| `S3PayloadUnsigned` | nothing | nothing — the body is outside the signature |
| `S3PayloadStreaming(n)` | a chunk framing the transport has to write | integrity with no pre-pass |

A library that took `Bytes` for a body chose the first for everybody,
and an object store whose uploads have to fit in memory is not an
object store: a five-gibibyte PUT would have been a five-gibibyte
allocation, quietly, and the only sign of it is a program that stops
working on large objects.  A library that defaulted to the second
removed integrity from every upload and never said so.

So the caller names one, and three rules follow from the type existing:

- **`s3sig.payload_allowed` refuses `S3PayloadUnsigned` over an
  `http://` endpoint.**  Over TLS an unsigned payload is a trade;
  over plaintext it means anything on the path can replace the object
  and the signature still verifies.  A refusal, not a warning.
- **`s3req.S3Body` has four cases and not one `Bytes` field**, because
  three of them cannot produce a hash without reading the whole body
  first.  The type that forced that is this one.
- **`s3sig.presign` has no payload parameter at all.**  A presigned URL
  is signed before its body exists — the whole point is that somebody
  else supplies one — so it is always `UNSIGNED-PAYLOAD`.  The
  consequence is worth saying out loud: a presigned PUT authorises the
  upload of *any* body to that key until it expires, so the lifetime is
  the whole of the control, and `presign` refuses one past the seven-day
  ceiling.

The second decision, and the one that costs the most support time when
it is got wrong, is **`S3Addressing` living in the configuration**.
Virtual-host addressing signs a canonical URI of `/key` and puts the
bucket in `Host`; path-style signs `/bucket/key`.  Sign one, send the
other, and the service answers `SignatureDoesNotMatch` — a 403 that
reads exactly like a wrong secret key and sends everybody to check
their credentials first.  `s3cfg.addressing_for` also answers where the
caller's choice cannot be honoured: a bucket name containing a dot
cannot be virtual-host addressed over TLS, because the wildcard
certificate covers one label, and this package forces path-style rather
than producing a certificate error the caller reads as a network
problem.

## What the registry backend would call

`orbit/orbit-registry-backend` stores every release as a small set of
files under `<root>/packages/<name>/<version>.*` — the tarball, the
hash, the manifest snapshot, the provenance, the facets, the publish
time, the publisher, the generated API page and a yank marker.  Moving
that store to an S3-compatible bucket is five calls:

| what the backend does today | what it would call |
| --- | --- |
| `fs.write` the tarball and its eight sidecars | `s3client.put_object` per file, keys `packages/<name>/<version><suffix>` |
| serve a download | `s3client.get_object`, or `s3sig.presign` and a redirect |
| the existence check before a publish | `s3client.head_object` — the 404 that means "this version is free" |
| rebuild `index.toml` | `s3client.list_page` with `prefix = "packages/"` and no delimiter, looped on `has_more` |
| the yank marker | `s3client.put_object` of an empty object |

Three things the port would have to decide, and they are the reasons
this is a design note rather than a patch:

- **The index's atomic replacement does not survive the move.**  The
  backend writes a new `index.toml` and renames it over the old one,
  which is atomic on a filesystem.  A `PUT` of one key is atomic in S3
  too, so the rename is unnecessary — but the read-modify-write around
  it is now racy between two publishing processes in a way a local
  rename was not, which is an argument for keeping `wal.log` as the
  authority and treating the index as a cache.
- **A published release is permanent**, so `delete_object` has no
  caller at all in this port.  A yank adds a marker; it removes
  nothing.  That makes the bucket's lifecycle policy the only thing
  that could ever delete a release, which is worth an explicit "no
  expiry" rule on it.
- **The tarball upload wants `S3PayloadSha256`** and not the unsigned
  form, because the backend already computes that hash — it stores it
  in `<version>.sha256` — and handing it to the signer costs nothing
  and puts the archive's integrity inside the signature.

## Should there be an `s3-core-nv`

**No, and this is the recommendation.**

Seven of eight modules are `[]`, which is the same ratio tls-nv used to
argue for `tls-core-nv` — and the answer is different here, because the
three consumers that made the TLS case do not exist for this one.

- **A device does not talk to an object store.**  The `core` half of
  tls-nv has a real embedded consumer: a microcontroller speaking HTTPS
  over its own radio.  SigV4 needs SHA-256 and HMAC over arbitrary-length
  strings, the operations need a URL and an XML parser, and the object
  that comes back is measured in megabytes.  There is no firmware that
  wants this and cannot have it.
- **Nothing else terminates S3.**  A TLS state machine serves a
  terminator, a QUIC stack and a fuzzer; there is no second consumer of
  a canonical request.
- **The pure half is already usable from a `host` package.**  A caller
  that wants to sign a request and send it through its own HTTP stack
  takes this package and calls `s3sig` and `s3op` — every one of them
  is `[]`, and depending on a `host` package does not make the caller's
  own functions cost anything.  What a `core` package buys over that is
  a `core` consumer, and there is not one.

What is worth doing instead, and this lane is not doing it: the `[]`
modules are arranged so that a split remains possible without moving a
line.  `s3client` depends on the other seven and nothing depends on it.

## What is missing, by name

**A TLS transport, which is the honest limit of this release.**
`std.http` routes an `https` URL to a pair of one-shot registry entries
rather than through its own connection surface, and those entries cover
GET and POST, answer no response headers, and synthesise a status of
200.  An object store needs `PUT`, `HEAD` and `DELETE`; it needs the
`ETag`, `Content-Range`, `x-amz-request-id` and `x-amz-version-id` out
of the headers; and it needs to tell a 404 from a 403 from a 412.  None
of that survives that path.

So `s3client.std_http` **refuses an `https` endpoint by name** rather
than appearing to work against one, and what it serves is every
deployment whose endpoint is plaintext: MinIO in a test rig, a gateway
inside a VPC, and the whole of this package's own test story.  The row
that closes it is **tls-nv's**: `std.http`'s own seam is one `if` that
disappears the day a `TlsStream` implements `Read`/`Write`, and until
then an `S3TlsHttp` over tls-nv plus http-codec-nv is the transport to
write.  acme-nv reported the same gap from the other side, for the same
reason — the one-shots discard the headers a protocol is carried in.

**MD5, for one operation.**  A batch `DeleteObjects` still requires a
`Content-MD5` header; crypto-nv publishes `md5`, so this is not a
missing row, but it is the one place in a package built on SHA-256
where MD5 is not optional and it is worth knowing before somebody
removes it.

**Nothing else.**  Every other primitive this package needs —
SHA-256, HMAC-SHA256, percent-encoding, XML, civil dates, media
types — is published on the grid today.

## Where a row wanted to widen

**`[io]` for `credentials_from_env`.**  The package's own row is
`[io, net, time, async]` through the transport, so the label was
already there; what is worth recording is that the function exists at
all.  Reading `AWS_ACCESS_KEY_ID` is what every other S3 client does by
default, and doing it by default would decide, for every program that
links this, that an environment variable is a credential source — which
is wrong for a server with an instance metadata endpoint and wrong for
a CLI with a profile file.  So it is one opt-in function with its own
label, and the default is that the caller supplies the three strings.

**`[time]` for one clock read.**  Every function in `s3sig` takes the
instant it signs for, because AWS publishes worked examples with a
fixed timestamp and a signer that read its own clock could not be run
against them.  `s3client.now_civil` is the single function that
produces one.  Same shape as tls-nv's `now_civil`, smtp-nv's `now_ms`
and postgres-nv's — the cohort now has four instances of this pattern
and it is worth naming as one.

**No effect parameter wanted a second one.**  `s3client`'s generic
functions bind exactly one, over the transport, and nothing else in the
package wanted to be polymorphic at the same time — which is the
difference between this package and tls-nv, where the signature checker
and the byte pipe both did.

## What this does not do, on purpose

- **No bucket policies, ACLs, CORS, lifecycle, versioning
  configuration, replication, encryption configuration, tagging,
  inventory, analytics or logging.**  Every one of them is a separate
  XML document with its own schema, and none of them is on the path
  a program takes to read and write objects.  They are `s3-admin-nv`
  if anybody wants them.
- **No STS, no instance metadata, no profile files and no credential
  chain.**  The credentials are three strings the caller supplies.  A
  chain is a policy about where secrets come from, and a library that
  chose one decides it for every program that links it.
- **No `list_all`.**  A bucket with four million objects is four
  thousand round trips; a function that hid them behind one call could
  not be stopped, checkpointed or rate-limited, and would return an
  array nobody sized.  The loop is five lines in the caller.
- **No retry loop.**  `s3fault.is_retryable` and
  `s3fault.fault_is_retryable` answer the question; the backoff, the
  jitter and the ceiling are the caller's, and `S3SlowDown` on a
  thousand-key upload is a decision about throughput rather than a
  library default.
- **No signature version 2.**  It is deprecated everywhere and accepted
  nowhere new.
- **No device claim.**  The package is `host`, and the section above
  says why a `core` split would have no consumer.
- **It does not print.**  Every failure is a value with a `message()`.

## The reference implementation

`aws-sdk-s3` for the operation surface and `boto3` for the subset —
`put_object`, `get_object`, `head_object`, `delete_object`,
`delete_objects`, `copy_object`, `list_objects_v2` and the multipart
family are boto3's own names for the same eight things, and the
canonical request, string to sign and signing-key derivation come
straight from AWS's Signature Version 4 specification along with its
worked examples, which `tests/s3sig_tests.nv` asserts against.

Four things change in the port.

`boto3` hides pagination behind a paginator and `aws-sdk-s3` behind a
stream; here a page is a value and the loop is the caller's, for the
reason the previous section gives.

`boto3`'s `TransferManager` runs a multipart upload for you, in threads
it owns, and reports progress through a callback; here `s3mpu` is a
state machine with no threads and no I/O, so a resumable upload across
a process restart is a matter of storing the value and a parallel one
is the caller's scheduler rather than this package's.

Both SDKs build a credential chain — environment, profile, container,
instance metadata — and both make it the default.  Here the credentials
are arguments and `credentials_from_env` is one opt-in function, for
the reason the widening section gives.

And both expose the payload hash as a configuration flag buried in a
client constructor.  Here it is a type in the signature of the function
that signs, because it is the decision that determines whether an
upload is covered by anything at all.

## Status

| item | implemented |
| --- | --- |
| `s3fault` — `S3ErrorCode`, `S3ErrorDoc`, `S3Fault` | types only |
| `s3fault.code_of_text`, `.code_text`, `.is_retryable`, `.fault_is_retryable`, `.fix_hint`, `.empty_doc`, `.is_no_such_key`, the `message` impl | no |
| `s3cfg` — `S3Addressing`, `S3Endpoint`, `S3Credentials`, `S3Config` | types only |
| `s3cfg.aws`, `.minio`, `.r2`, `.endpoint_of_url`, `.credentials`, `.temporary_credentials` | no |
| `s3cfg.with_addressing`, `.with_region`, `.host_header`, `.canonical_path`, `.origin`, `.addressing_for` | no |
| `s3cfg.bucket_name_ok`, `.key_ok`, `.is_plaintext` | no |
| `s3sig` — `S3PayloadHash`, `S3CanonicalRequest`, `S3Scope`, `S3Signature` | types only |
| `s3sig.S3_SIGV4_ALGORITHM`, `.S3_UNSIGNED_PAYLOAD`, `.S3_STREAMING_PAYLOAD`, `.S3_EMPTY_SHA256`, `.S3_PRESIGN_MAX_SECONDS`, `.S3_CLOCK_SKEW_SECONDS` | yes — they are constants |
| `s3sig.amz_date`, `.amz_day`, `.scope`, `.scope_text`, `.payload_of_body`, `.payload_text`, `.payload_allowed` | no |
| `s3sig.canonical_uri`, `.canonical_query`, `.canonical_headers`, `.signed_headers`, `.canonical_request`, `.canonical_text` | no |
| `s3sig.string_to_sign`, `.signing_key`, `.key_day_of`, `.sign`, `.authorize`, `.authorization_headers` | no |
| `s3sig.presign`, `.presign_expiry`, `.skew_ok`, `.service_time_of` | no |
| `s3req` — `S3Header`, `S3QueryPair`, `S3Body`, `S3Request`, `S3Reply` | types only |
| `s3req.request`, `.with_query`, `.with_header`, `.with_body`, `.with_bytes`, `.body_length` | no |
| `s3req.header_of`, `.request_header_of`, `.range_header`, `.range_from`, `.range_suffix`, `.content_range_of` | no |
| `s3req.content_type_for`, `.metadata_header`, `.metadata_of`, `.metadata_fits`, `.if_match_header`, `.if_none_match_any` | no |
| `s3op` — `S3Object`, `S3Head`, `S3Put` | types only |
| `s3op.S3_PART_MIN_BYTES`, `.S3_PUT_MAX_BYTES`, `.S3_LIST_MAX_KEYS`, `.S3_DELETE_MAX_KEYS` | yes — they are constants |
| `s3op.get`, `.get_range`, `.get_if_none_match`, `.read_get`, `.head`, `.read_head` | no |
| `s3op.put`, `.put_streamed`, `.read_put`, `.delete`, `.delete_if_match`, `.read_delete` | no |
| `s3op.delete_many`, `.read_delete_many`, `.list`, `.read_list`, `.has_more` | no |
| `s3op.copy`, `.read_copy`, `.copy_source_header`, `.create_bucket`, `.head_bucket` | no |
| `s3op.etag_is_multipart`, `.etag_part_count`, `.etag_unquoted`, `.body_is_error`, `.fault_of_reply`, `.conditional_writes_supported` | no |
| `s3xml` — `S3ListEntry`, `S3ListPage`, `S3UploadStart`, `S3PartInfo`, `S3UploadDone`, `S3DeleteOutcome` | types only |
| `s3xml.is_s3_document`, `.error_doc`, `.list_result`, `.upload_start`, `.upload_done`, `.list_parts`, `.parts_truncated` | no |
| `s3xml.delete_result`, `.copy_result`, `.write_completion`, `.write_delete`, `.write_bucket_config` | no |
| `s3mpu` — `S3MpuState`, `S3MpuStep`, `S3MpuPart`, `S3Mpu` | types only |
| `s3mpu.S3_MAX_PARTS`, `.S3_PART_MAX_BYTES`, `.S3_OBJECT_MAX_BYTES` | yes — they are constants |
| `s3mpu.part_size_ok`, `.plan_parts`, `.planned_part_size`, `.needs_multipart`, `.upload`, `.resume` | no |
| `s3mpu.step`, `.supply_start`, `.supply_part`, `.supply_complete`, `.fail`, `.abort`, `.abort_request` | no |
| `s3mpu.list_parts_request`, `.list_uploads_request`, `.upload_id_of`, `.state_of`, `.parts_of`, `.progress_of`, `.owes_abort`, `.completion` | no |
| `s3client` — `S3Transport[e]`, `S3StdHttp`, `S3Client` | types only |
| `s3client.client`, `.std_http`, `.send`, `.open` | no |
| `s3client.get_object`, `.get_range`, `.head_object`, `.put_object`, `.delete_object`, `.copy_object`, `.list_page` | no |
| `s3client.pump`, `.abort_upload`, `.drain_into`, `.now_civil`, `.credentials_from_env`, the `S3Transport` impl | no |
