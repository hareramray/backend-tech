# Case study: video-on-demand upload and adaptive playback

## Scope and measurable promises

Design a fictional training-video service. An authorized creator uploads a recording, sees processing status, and publishes it to an allowed audience. Viewers watch using adaptive bitrate playback, captions, and seeking. This chapter covers video on demand (VOD); live ingest, low-latency broadcast, and interactive calls require different timing and failure budgets. Django owns control APIs and PostgreSQL owns metadata. Video bytes flow through object storage and a CDN, not through ordinary Django request workers. This is a design case study, not an implemented capstone extension.

Read [CDNs](../08-architecture/09-cdns-and-edge-caching.md), [storage internals](../08-architecture/13-storage-engines-lsm-trees-and-bloom-filters.md), and [capacity planning](../08-architecture/05-system-design-and-capacity-planning.md). Fictional targets are 99% playback-start success, p95 startup below three seconds on supported networks, and p95 processing completion within 20 minutes for a ten-minute supported input. Define startup from the viewer's play action to first rendered frame. Rebuffering ratio is stalled playback time divided by attempted playback time, with startup measured separately.

An upload-complete response means source bytes were verified and processing was accepted, not that a playable video exists. Publication requires a validated set of outputs and an atomic metadata transition. Existing immutable segment objects may remain cached after access revocation, so authorization expiry and edge enforcement must be explicit.

## Byte, request, and compute budgets

Assume 1,000 uploads/day, each ten minutes at 8 Mbit/s, plus four video renditions at 0.5, 1.2, 2.5, and 5 Mbit/s. All numbers use decimal units and exclude container overhead unless stated.

```text
Source/video = 600 s * 8,000,000 bit/s / 8 = 600 MB
Source/day   = 1,000 * 600 MB = 600 GB
Video ladder/video = 600 s * 9,200,000 bit/s / 8 = 690 MB
One 128 kbit/s audio track/video = 9.6 MB
Source + outputs/day = 600 + 690 + 9.6 = 1,299.6 GB
```

At 100,000 daily viewers watching 30 minutes at an average 3 Mbit/s, delivery is `100,000 * 1,800 * 3,000,000 / 8 = 67.5 TB/day`, or 6.25 Gbit/s average. A fivefold peak requires 31.25 Gbit/s of edge delivery capacity. At a fictional 95% byte hit ratio, origin transfer is still 3.375 TB/day. A request hit ratio cannot be substituted for a byte hit ratio when objects differ in size.

With six-second segments, a ten-minute video has 100 segments per track. Four video tracks and one audio track produce approximately 500 segment objects, plus initialization objects and manifests. Viewers generate about 30 million video-segment requests/day; separate audio roughly doubles this to 60 million, around 694 requests/s average before manifests and retries. Shortening segments to two seconds roughly triples segment requests for the same viewing duration.

If one measured worker transcodes the complete ladder at real time, 10,000 input minutes/day require `10,000 / 1,440 = 6.94` continuously occupied workers. At a target 70% utilization, round `6.94 / 0.70` up to ten workers before burst and failure headroom. This is an explicit benchmark assumption; codec, resolution, hardware, and content can change it substantially.

## Architecture and processing graph

```text
Creator --> Django upload API --> metadata/outbox in PostgreSQL
   |                |
   +-- signed multipart upload --> private quarantine object storage
                                          |
                                  verify/probe/scan
                                          |
                              +-----------+-----------+
                              v           v           v
                         video ladder   audio      thumbnails/captions
                              +-----------+-----------+
                                          v
                                  package + validate
                                          v
                                 publish version pointer

Viewer --> playback authorization API --> CDN --> private packaged origin
```

The job graph is a directed acyclic graph (DAG). Each node records dependencies, input digest, profile version, attempt, lease generation, and output location. A failed thumbnail can be optional; missing mandatory audio or a broken manifest blocks publication. Express that distinction in the graph rather than letting whichever worker finishes last decide readiness.

## Control API and metadata contract

`POST /v1/videos/uploads` takes title, expected size, content type, and checksum intent with an idempotency key. It returns a video ID, upload-session ID, allowed part configuration, expiration, and narrowly scoped upload credentials. `POST /v1/videos/{id}/complete` identifies completed parts; the server validates ownership and storage state before queuing work. `GET /v1/videos/{id}` returns `uploading`, `processing`, `ready`, `failed`, or `deleted` with safe error codes.

```json
{
  "video_id": "v100",
  "state": "processing",
  "source_version": 1,
  "profile_version": "vod-ladder-3",
  "progress": {"completed_tasks": 3, "required_tasks": 7}
}
```

`POST /v1/videos/{id}/playback-sessions` verifies current entitlement and returns a short-lived playback URL/token, allowed manifest version, and expiration. It does not reveal a permanent private origin URL. Token refresh repeats authorization.

```text
video(tenant_id, id, creator_id, state, source_key, source_digest,
      source_version, profile_version, published_version, policy_version)
upload_session(id, tenant_id, video_id, provider_upload_id, expected_size,
               checksum_algorithm, expires_at, state)
processing_task(video_id, source_version, profile_version, task_name,
                state, attempt, lease_generation, output_prefix, error_code)
  UNIQUE (video_id, source_version, profile_version, task_name)
rendition(video_id, published_version, track_id, codec, bitrate,
          width, height, manifest_key, verified_bytes)
outbox(event_id, video_id, kind, version, payload, published_at)
```

Every storage key is generated by the service and bound to the tenant/video/session. User-provided filenames are display metadata. Do not allow a completion request to name another customer's object or upload session.

## Upload and publication flows

1. Authorize creation and reserve a bounded tenant storage quota. Create the upload session and server-chosen quarantine key.
2. Issue expiring permissions for specific multipart operations. The browser sends bytes directly to storage and retries individual failed parts.
3. On completion, verify the expected part list, size, checksum metadata, ownership, and session state. Abort abandoned sessions through a lifecycle process so incomplete parts do not accumulate indefinitely.
4. Probe the actual media in an isolated worker with CPU, memory, duration, and decoder limits. Treat content type and extension as hints, not validation. Mark verified source availability and enqueue the DAG through a durable event.
5. Workers create versioned outputs in a staging prefix. Validate duration, track alignment, codec compatibility, and every referenced output before declaring packaging complete.
6. Atomically update the published-version pointer if the source/profile/policy generation still matches and the video is not deleted. Duplicate or stale workers cannot overwrite a newer publication.

Multipart upload combines parts through an explicit completion operation. An object ETag is not universally the MD5 of a multipart object's contents; select the provider's documented checksum mode and validate it deliberately. [S3 multipart upload and checksums](https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html).

Presigned URLs are bearer capabilities whose effective lifetime is also limited by the signing credentials and policy. Keep them out of logs, scope them narrowly, and design renewal and expiration handling. [S3 presigned URL behavior](https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html).

## Packaging, manifests, and adaptive bitrate

HLS uses playlists that identify media segments and variants; a completed VOD media playlist can include an end marker. DASH describes presentations and representations using an MPD. A compatible fragmented-MP4/CMAF pipeline can sometimes share media between HLS and DASH, but compatibility depends on codecs, timing, encryption, and target players. Validate actual supported clients. [HLS specification](https://www.rfc-editor.org/rfc/rfc8216.html), [DASH-IF timing model](https://dashif.org/Guidelines-TimingModel/).

The player estimates throughput and buffer health, then selects an appropriate representation. The server publishes honest bandwidth and codec information; it does not force one global quality setting for every network. Align switching points across renditions and package independently decodable segment boundaries. A manifest pointing to an absent segment is a publication defect even if most renditions work.

Six seconds is a fictional starting segment duration here. Longer segments reduce request overhead but can increase initial download and switching granularity. Shorter segments improve responsiveness but increase object/request volume and packaging overhead. For a six-second segment at 3 Mbit/s, the nominal payload is 2.25 MB. A client sustaining only 1.5 Mbit/s needs about 12 seconds to fetch it, so remaining at that representation drains its buffer.

Use immutable versioned segment URLs and long cache lifetimes for published bytes. Keep the authorization decision separate from byte reuse: an edge may cache one encrypted or protected object while still validating each viewer's permission before serving it. Do not place authorization only on the manifest if segment URLs remain anonymously accessible. Prevent direct origin bypass.

DRM is optional and separate from ordinary signed URLs or TLS. It adds content encryption, key management, license services, player/CDM support, and entitlement rules. The browser Encrypted Media Extensions API provides interaction with content protection systems; it is not itself a DRM system. [W3C Encrypted Media Extensions](https://www.w3.org/TR/encrypted-media/).

## Failures, security, and operations

Idempotency keys for processing include source digest/version, profile version, and task identity. A worker writes attempt-specific output and wins publication through a conditional metadata update. If it times out after upload, a retry can verify existing output or produce another staging attempt; it must not create conflicting published manifests. Lease generations fence a paused worker that resumes after reassignment.

Deletion first blocks new playback authorization and increments the relevant generation. It then invalidates permitted edge paths and schedules source/output cleanup. Running workers must check generation before publication. Define the remaining lifetime of existing playback tokens and cached segments; zero-second revocation is not achieved by deleting a metadata row alone.

Isolate decoders from the control database and credentials, limit upload bytes and duration, scan untrusted files, and separate tenant prefixes plus access policies. Captions and titles need output escaping. Metrics should not contain bearer URLs. Content moderation and copyright workflows require product policy, but do not change the technical requirement to audit who published or removed a video.

Monitor oldest processing age, per-stage retries, worker saturation, output-validation failures, storage growth, abandoned parts, CDN byte hit ratio, origin egress, startup failures, and rebuffering by player/network. Server HTTP success does not prove the viewer saw a frame; collect privacy-conscious player telemetry. Restore metadata and source objects first, then regenerate missing derived outputs using recorded profiles. Test whether retained source versions and keys actually permit reconstruction.

## Tabletop lab: a broken processing attempt

Use video V, source version 1, profile P, four required video renditions, one audio track, and publication version initially null. Draw the DAG and track each node as pending/running/verified.

| Injected failure | Expected outcome |
|---|---|
| Part 3 upload fails | Retry that part; no processing begins before verified completion |
| Complete request repeats after a timeout | Same source/session result; one logical processing graph |
| Worker uploads output then loses its lease | Output is staging only; generation check prevents stale publication |
| Three renditions finish, fourth fails | Video remains processing or failed according to retry policy; required manifest is not published |
| Manifest references a missing segment | Validation blocks publication and reports exact missing object |
| Delete occurs while packaging runs | New playback denied; old worker cannot republish |
| CDN loses all cached segments | Origin load rises; rate/headroom policy is tested against byte demand |
| Viewer permission expires mid-session | Renewal reauthorizes; expired capability does not silently gain indefinite access |

For each row, record the database transition, orphan-output cleanup, retry identity, and user-visible state. Then halve segment duration and recalculate request volume. Compare preserving all original sources for a year with a shorter source-retention policy, including the loss of future re-encoding capability.

## Completion checklist

- [ ] Upload completion, processing completion, and publication are separate states.
- [ ] Checksums, quotas, ownership, and abandoned uploads are addressed.
- [ ] DAG retries and stale workers cannot corrupt the published version.
- [ ] Playback budgets include bytes, segment requests, and viewer experience.
- [ ] Entitlement checks cover manifests, segments, origin access, and renewal.
- [ ] Recovery can reproduce outputs from retained sources and profiles.

Continue with [inventory and reservations](05-inventory-and-reservations.md) or return to the [case-study index](README.md).
