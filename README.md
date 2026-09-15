# s3-nv

Amazon S3 is an object store: a program puts an object into a bucket
under a key and reads it back by that key, over HTTP. The requests and
replies are the
[S3 REST API](https://docs.aws.amazon.com/AmazonS3/latest/API/Welcome.html),
and every request is authenticated with
[Signature Version 4](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_sigv4-signing.html).
This package is a client for that API, and for the services that copy
it, such as MinIO and Cloudflare R2. It is built on
[crypto-nv](https://novo-lang.org/packages/crypto-nv) for SHA-256 and
HMAC, [xml-nv](https://novo-lang.org/packages/xml-nv) for the reply
documents, [url-nv](https://novo-lang.org/packages/url-nv) for
percent-encoding, [mime-nv](https://novo-lang.org/packages/mime-nv) for
content types, and
[calendar-nv](https://novo-lang.org/packages/calendar-nv) for the two
timestamp formats a signature carries.

**Status: NOT IMPLEMENTED — interface only.** Every function is declared
with its full signature, but every body is a `todo()` that panics when
called. The package is published so its design can be reviewed and
depended on before it is implemented. Version 0.1.0 will be the first
working release.

## What an S3 object store is

A **bucket** is a named container. A **key** is the name of one object
inside it, up to 1024 bytes of UTF-8, and any byte is legal in one: a
key may contain a newline, a space or a `#`. An **object** is the bytes
plus the headers stored with them, such as `Content-Type` and the
`x-amz-meta-` user metadata. An **ETag** is the identity the service
returns for an object's contents.

A bucket reaches the wire in one of two **addressing styles**.
Virtual-host addressing puts the bucket in the host name,
`https://bucket.s3.region.amazonaws.com/key`. Path-style addressing puts
it in the path, `https://endpoint/bucket/key`. The two styles sign
different strings, so the choice is part of the request and not a
routing detail.

Every request carries a signature. Signature Version 4 builds a
**canonical request** from the method, the encoded path, the sorted
query string, the sorted headers and the hash of the body; hashes it;
prefixes the algorithm, the timestamp and a **credential scope** of day,
region and service; and signs the result with a key derived from the
secret by four chained HMAC-SHA256 operations. The signature therefore
covers the body's SHA-256, which means the body has to be known before
the first byte of it is sent.

A **presigned URL** moves that signature from a header into the query
string, so a URL can be handed to a browser or to a `curl` that holds no
credentials.

An object larger than one request can carry is uploaded as a
**multipart upload**: the client initiates it and receives an upload id,
sends the object in numbered parts, and completes the upload with the
list of part numbers and their ETags. An upload that is never completed
and never aborted leaves its parts in the bucket, reachable by no key
and billed.

| Limit | Value |
| --- | --- |
| Key length | 1024 bytes |
| Bucket name | 3 to 63 characters, lowercase letters, digits, dots and hyphens |
| Single `PUT` | 5 GiB |
| Multipart part, every part but the last | at least 5 MiB |
| Multipart part, largest | 5 GiB |
| Parts in one upload | 10000 |
| Largest object a multipart upload can produce | 5 TiB |
| Keys in one `ListObjectsV2` page | 1000 |
| Keys in one batch delete | 1000 |
| User metadata, names and values together | 2 KiB |
| Presigned URL lifetime | 7 days |
| Clock skew a signature tolerates | 15 minutes |

## Install

```
novo pkg add s3-nv
```

## Example

```novo
use std.bytes
use s3cfg
use s3client

fn main() [io, net, time, async]
    // The credentials are three strings the caller supplies. This
    // package reads no environment variable unless it is asked to.
    let creds = s3cfg.credentials("AKIAIOSFODNN7EXAMPLE",
                                  "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY")

    // A MinIO deployment on this machine: path-style, plaintext.
    let cfg = s3cfg.minio("http", "127.0.0.1", 9000, "us-east-1", creds)
    let c = s3client.client(cfg)

    // The transport every operation is sent over, with a 30 second
    // timeout. An https endpoint is refused here; see "What is not
    // included".
    match s3client.std_http(cfg, 30000)
        Err(e) => println("no transport: ${e.message()}")
        Ok(t) =>
            // The instant the request is signed for. This is the one
            // function in the package that reads a clock.
            let at = s3client.now_civil()

            // Store one object, then read back what the service said.
            match s3client.put_object(c, t, "photos", "cat.jpg",
                                      bytes.from_str("not really a jpeg"), at)
                Err(e)  => println("the upload failed: ${e.message()}")
                Ok(put) => println("stored, and its etag is ${put.etag}")
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: s3-nv.<module>.<fn>` panic. The tests are the
specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `s3fault` | The error codes an `<Error>` document carries, which of them are worth retrying, and the fault a caller receives. |
| `s3cfg` | The endpoint, the region, the credentials and the addressing style, with ready-made configurations for AWS, MinIO and R2. |
| `s3sig` | Signature Version 4: the canonical request, the string to sign, the signing key, the `Authorization` header and presigned URLs. |
| `s3req` | A request and a reply as values: headers, query parameters, the four kinds of body, ranges and metadata. |
| `s3op` | One request builder and one reply reader per operation: get, head, put, delete, batch delete, copy, list, create bucket. |
| `s3xml` | The reply documents, read over xml-nv's tree, and the three request documents that are XML. |
| `s3mpu` | The multipart upload as a state machine: the part plan, the step to send next, resuming, and the abort. |
| `s3client` | The transport trait, one transport over the standard library's HTTP client, and one call per operation. |

## How to choose an entry point

**`s3client` is the whole client.** It signs a request, sends it over a
transport and reads the reply. Use it unless you already own an HTTP
stack.

**`s3op` with `s3sig` is the client without the socket.** `s3op` builds
the request and reads the reply, `s3sig.authorize` produces the headers
that authenticate it, and the sending is yours. Every function in both
modules performs no input or output.

**`s3sig.presign` alone signs a URL.** Use it to give somebody else a
single upload or download that needs no credentials at their end.

**`s3mpu` drives an upload larger than one request.** It holds part
numbers, ETags and a plan, and it sends nothing. An upload therefore
survives a process restart if the caller stores the value, and
`s3client.pump` is the loop that sends what the state machine asks for.

**A transport of your own implements `S3Transport[e]`.** The four
methods are send-and-buffer, send-and-leave-the-body, read and close.
A transport that performs nothing costs nothing: the package's own test
suite drives a complete exchange over a recorded reply.

## The rules a user needs

1. **The status code is not the answer; the `<Code>` element in the body
   is.** S3 answers 403 for a wrong key, a wrong signature, a clock more
   than fifteen minutes out and a policy refusal. `S3FaultService`
   carries the parsed document, and `s3fault.fix_hint` writes the
   sentence that names the repair for the three whose own message does
   not.
2. **The addressing style changes the string that gets signed.**
   Virtual-host addressing signs a path of `/key` and puts the bucket in
   `Host`; path-style signs `/bucket/key`. Sign one and send the other
   and the service answers `SignatureDoesNotMatch`, which reads exactly
   like a wrong secret key. `s3cfg.addressing_for` answers the style
   that will actually be used, and forces path-style for a bucket name
   containing a dot: a wildcard certificate covers one label, so the
   virtual-host name would not match it.
3. **The caller names the payload hash.** `S3PayloadSha256` costs a pass
   over the object before the upload starts and covers the body end to
   end. `S3PayloadUnsigned` costs nothing and leaves the body outside
   the signature. `S3PayloadStreaming` signs each chunk, at the cost of
   a framing the transport must write. `s3sig.payload_allowed` refuses
   the unsigned form over a plaintext endpoint, where nothing else would
   be protecting the body.
4. **A presigned URL is always unsigned-payload, and `presign` takes no
   payload argument.** The body does not exist when the URL is signed.
   A presigned `PUT` therefore authorises the upload of any body to that
   key until it expires, so the lifetime is the whole of the control.
   Seven days is the ceiling, and `presign` refuses more.
5. **`x-amz-date` is the basic ISO 8601 form.** `20130524T000000Z`: no
   dashes and no colons. The credential scope carries the same day as
   eight digits. An extended-form timestamp produces a signature the
   service computes differently and a 403 with nothing in it to read.
6. **The canonical URI and the canonical query string encode
   differently.** The canonical URI percent-encodes every byte outside
   `A-Za-z0-9-._~` except `/`, and for S3 it encodes once, where every
   other AWS service encodes the path twice. The canonical query string
   encodes `/` as well, and writes `name=` for a parameter with an empty
   value.
7. **A listing ends on `is_truncated` and never on an empty page.** A
   page can hold no objects and still be truncated, when every key in
   its range rolled up into a common prefix. `s3op.has_more` is the loop
   condition, and `S3ListPage.next_continuation` is what the next
   request sends. This package always asks for `encoding-type=url`,
   because it is the only way a key containing a character XML cannot
   carry survives the reply.
8. **A multipart part below 5 MiB is refused at completion, after every
   byte has been uploaded.** The service answers `EntityTooSmall` at the
   end. `s3mpu.plan_parts` computes the sizes before anything is sent,
   and raises the part size rather than exceeding ten thousand parts.
9. **An upload that fails still owes an abort.** `S3MpuMustAbort`
   carries the abort request, and the parts stay in the bucket until it
   is sent: they appear in no listing, are reachable by no key, and are
   billed. `s3mpu.upload_id_of` is public so the id can be stored and
   the abort sent after a crash, and `s3mpu.abort_request` builds one
   from the three strings alone.
10. **A 200 on a completion or a copy can still be a failure.** The
    service answers 200 as soon as it starts the operation, streams
    whitespace while it works, and puts the error document inside that
    body. `s3op.body_is_error` is the check, and `read_copy` and
    `supply_complete` make it for you.
11. **A multipart ETag is not the MD5 of anything the caller can
    compute.** It is the MD5 of the concatenated part MD5s, with
    `-<count>` after it. A program that verified downloads by comparing
    the ETag to its own MD5 works until the first large object and then
    reports corruption on every one. `s3op.etag_is_multipart` says which
    kind an ETag is.
12. **A ranged `GET` reads its total from `Content-Range`, not
    `Content-Length`.** On that reply `Content-Length` is the length of
    the range. `s3req.content_range_of` answers the first byte, the last
    byte and the total. `bytes=-n` is the last n bytes and `bytes=n-` is
    everything from offset n: the two differ by one character.
13. **User metadata round-trips in lowercase.** S3 lowercases metadata
    names, so `metadata_header` lowercases on the way in. The names and
    values together must fit 2 KiB, and exceeding it is a 400 after the
    body has been sent.
14. **A `CreateBucket` in `us-east-1` must not name its region.** The
    region that predates the `CreateBucketConfiguration` document is the
    one the document may not mention, so `s3xml.write_bucket_config`
    answers an empty body there.
15. **`delete_object` cannot tell you whether anything was removed.**
    S3 answers 204 for a key that never existed, because delete is
    idempotent. `delete_if_match` is what a caller that needs to know
    uses. A batch delete answers 200 with the per-key failures inside
    it, and `read_delete_many` answers every outcome.
16. **Retry is a property of the code, not of the status class.**
    `SlowDown` and `InternalError` want a backoff; `AccessDenied` wants
    none, ever. `s3fault.is_retryable` is the table, and
    `fault_is_retryable` extends it to a transport failure, which is
    retryable only for a request the caller says is idempotent. The
    backoff, the jitter and the ceiling are the caller's.

## What is not included

- **A TLS transport.** The standard library's HTTP client routes an
  `https` URL through one-shot entries that cover `GET` and `POST`,
  answer no response headers and report a status of 200. An object store
  needs `PUT`, `HEAD` and `DELETE`, the `ETag` and `Content-Range`
  headers, and the difference between a 404, a 403 and a 412.
  `s3client.std_http` therefore refuses an `https` endpoint by name
  rather than appearing to work against one. What it does serve is every
  plaintext endpoint: MinIO in a test rig, a gateway inside a private
  network, and this package's own tests. A transport over
  [tls-nv](https://novo-lang.org/packages/tls-nv) closes the gap, and a
  caller may supply one today by implementing `S3Transport`.
- **Bucket policies, ACLs, CORS, lifecycle, versioning configuration,
  replication, encryption configuration, tagging, inventory, analytics
  and logging.** Each is a separate XML document with its own schema,
  and none of them is on the path a program takes to read and write
  objects.
- **STS, instance metadata, profile files and a credential chain.** The
  credentials are three strings the caller supplies.
  `s3client.credentials_from_env` reads `AWS_ACCESS_KEY_ID`,
  `AWS_SECRET_ACCESS_KEY` and `AWS_SESSION_TOKEN`, and it is one opt-in
  function rather than a default, because where secrets come from is the
  program's decision.
- **A function that lists a whole bucket.** A bucket with four million
  objects is four thousand round trips. A single call could not be
  stopped, checkpointed or rate-limited, and would answer an array
  nobody sized. The loop is a few lines in the caller.
- **A retry loop.** `s3fault.is_retryable` answers the question; the
  schedule is the caller's.
- **Signature Version 2.** It is deprecated everywhere and accepted
  nowhere new.
- **MD5 as a general dependency.** One operation still requires it: a
  batch `DeleteObjects` carries a `Content-MD5` header. crypto-nv
  publishes `md5`, so nothing else is needed, but this is the one place
  in a package built on SHA-256 where MD5 is not optional.
- **Printing.** Every failure is a value with a `message()`.

## Related packages

- [crypto-nv](https://novo-lang.org/packages/crypto-nv) is SHA-256,
  HMAC-SHA256 and the constant-time comparison the whole of Signature
  Version 4 is built from, and the `md5` a batch delete needs.
- [xml-nv](https://novo-lang.org/packages/xml-nv) parses the documents.
  `s3xml` is a reader over its tree and parses no XML itself, so a reply
  carrying an element this package has never seen still parses.
- [url-nv](https://novo-lang.org/packages/url-nv) supplies the two
  percent-encodings rule 6 keeps apart.
- [mime-nv](https://novo-lang.org/packages/mime-nv) answers the
  `Content-Type` a `put` sends when the caller names none. S3 serves
  back whatever it was given, so a bucket uploaded without one serves a
  website that downloads its own pages.
- [calendar-nv](https://novo-lang.org/packages/calendar-nv) is the civil
  date and time every signing function takes as an argument.
- [tls-nv](https://novo-lang.org/packages/tls-nv) is the transport
  security this package's shipped transport does not have. Take it, with
  [http-codec-nv](https://novo-lang.org/packages/http-codec-nv), to
  write an `S3Transport` that reaches an `https` endpoint.
- [acme-nv](https://novo-lang.org/packages/acme-nv) is the other host
  package that signs every request it sends, with JSON Web Signature
  rather than Signature Version 4, against a certificate authority
  rather than an object store.
- `std.http` in the standard library is the HTTP client the shipped
  transport uses. A program that wants only plaintext needs nothing
  else.

## Tests

```bash
novo test tests/s3sig_tests.nv       # 14 tests: the signature and the configuration
novo test tests/s3flow_tests.nv      # 17 tests: the operations, the listing, the upload
novo test tests/s3surface_tests.nv   # 9 tests: every public name, spelled as a consumer would
```

The signature the suite asserts is AWS's own, from the Signature Version
4 worked example: a `GET` of `test.txt` from `examplebucket` in
`us-east-1` at `20130524T000000Z`, with the canonical request, the
string to sign and the signature each step produces. Nothing in the
signing path reads a clock, so that example reproduces exactly. The
operation tests assert the request each builder produces and the reply
each reader accepts, over a transport that implements `S3Transport[]`:
the compiler checks, before any assertion runs, that seven of the eight
modules reach no socket.

The tests compile today and fail at run, each on the
`not implemented: s3-nv.<module>.<fn>` panic that is its body. They turn
green one at a time as bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| The `S3_*` constants in `s3sig`, `s3op` and `s3mpu` | yes (they are constants) |
| `s3fault.code_of_text`, `.code_text`, `.is_retryable`, `.fault_is_retryable`, `.fix_hint`, `.empty_doc`, `.is_no_such_key`, `.S3Fault.message` | no |
| `s3cfg.aws`, `.minio`, `.r2`, `.endpoint_of_url`, `.credentials`, `.temporary_credentials`, `.with_addressing`, `.with_region` | no |
| `s3cfg.host_header`, `.canonical_path`, `.origin`, `.addressing_for`, `.bucket_name_ok`, `.key_ok`, `.is_plaintext` | no |
| `s3sig.amz_date`, `.amz_day`, `.scope`, `.scope_text`, `.payload_of_body`, `.payload_text`, `.payload_allowed` | no |
| `s3sig.canonical_uri`, `.canonical_query`, `.canonical_headers`, `.signed_headers`, `.canonical_request`, `.canonical_text` | no |
| `s3sig.string_to_sign`, `.signing_key`, `.key_day_of`, `.sign`, `.authorize`, `.authorization_headers` | no |
| `s3sig.presign`, `.presign_expiry`, `.skew_ok`, `.service_time_of` | no |
| `s3req.request`, `.with_query`, `.with_header`, `.with_body`, `.with_bytes`, `.body_length` | no |
| `s3req.header_of`, `.request_header_of`, `.range_header`, `.range_from`, `.range_suffix`, `.content_range_of` | no |
| `s3req.content_type_for`, `.metadata_header`, `.metadata_of`, `.metadata_fits`, `.if_match_header`, `.if_none_match_any` | no |
| `s3op.get`, `.get_range`, `.get_if_none_match`, `.read_get`, `.head`, `.read_head` | no |
| `s3op.put`, `.put_streamed`, `.read_put`, `.delete`, `.delete_if_match`, `.read_delete` | no |
| `s3op.delete_many`, `.read_delete_many`, `.list`, `.read_list`, `.has_more` | no |
| `s3op.copy`, `.read_copy`, `.copy_source_header`, `.create_bucket`, `.head_bucket` | no |
| `s3op.etag_is_multipart`, `.etag_part_count`, `.etag_unquoted`, `.body_is_error`, `.fault_of_reply`, `.conditional_writes_supported` | no |
| `s3xml.is_s3_document`, `.error_doc`, `.list_result`, `.upload_start`, `.upload_done`, `.list_parts`, `.parts_truncated` | no |
| `s3xml.delete_result`, `.copy_result`, `.write_completion`, `.write_delete`, `.write_bucket_config` | no |
| `s3mpu.part_size_ok`, `.plan_parts`, `.planned_part_size`, `.needs_multipart`, `.upload`, `.resume` | no |
| `s3mpu.step`, `.supply_start`, `.supply_part`, `.supply_complete`, `.fail`, `.abort`, `.abort_request` | no |
| `s3mpu.list_parts_request`, `.list_uploads_request`, `.upload_id_of`, `.state_of`, `.parts_of`, `.progress_of`, `.owes_abort`, `.completion` | no |
| `s3client.client`, `.std_http`, `.send`, `.open`, and the `S3Transport` implementation for `S3StdHttp` | no |
| `s3client.get_object`, `.get_range`, `.head_object`, `.put_object`, `.delete_object`, `.copy_object`, `.list_page` | no |
| `s3client.pump`, `.abort_upload`, `.drain_into`, `.now_civil`, `.credentials_from_env` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
