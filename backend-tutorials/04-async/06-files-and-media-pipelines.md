# Files, uploads and media pipelines

Prerequisites: [jobs](01-jobs-and-scheduling.md), [idempotency](04-idempotency-and-deduplication.md), [data lifecycle](../02-data/04-migrations-and-data-lifecycle.md), the [course map](../README.md), and Python 3.12+ for the local validation lab.

## Objectives

Design an authorized upload workflow, separate untrusted originals from approved artifacts, process files with bounded resources, and recover orphaned uploads and partial transformations.

## Model the upload lifecycle

Files combine metadata, bytes and asynchronous processing. Keep metadata in PostgreSQL and large object bytes in storage designed for them when appropriate. An upload record should include tenant, owner, storage key, declared size/type, observed checksum, state and timestamps. Define states such as initiated, uploaded, scanning, ready, rejected and deleted.

Clients should not choose arbitrary filesystem paths or public storage keys. Generate an opaque key scoped to the tenant and upload ID. Preserve the user filename only as untrusted display metadata. A database row referring to an object does not prove the object exists, matches the expected size or is safe to serve.

Treat uploaded content as untrusted input. Filename extensions and client Content-Type are claims, not verification. Enforce size limits at the edge, application, storage policy and processing stages as relevant. Restrict allowed formats, inspect content, and isolate parsers with resource limits. OWASP documents file-upload risks including executable content, parser abuse and misleading names. [OWASP file-upload guidance](https://community.owasp.org/vulnerabilities/Unrestricted_File_Upload)

## Direct upload versus application streaming

Application-mediated upload simplifies centralized policy but consumes application bandwidth and connections. Direct-to-object-storage upload reduces that load, but requires a controlled initiation and finalization protocol.

A presigned URL is a bearer capability for a specific storage operation until it expires or its credentials cease to authorize it. Anyone possessing it can use it within those limits. Keep lifetimes short, constrain the object key and permissions, and avoid logging it. AWS documents presigned upload/download behavior and relevant expiration limits. [S3 presigned URL documentation](https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html)

Do not assume a presigned PUT is automatically single-use. If overwriting the same key is possible during the validity window, finalization must bind to a specific object version or otherwise prevent later replacement of already scanned bytes.

## Lab: a bounded local byte-validation stage

This standalone Python lab demonstrates streaming size limits, checksums, temporary staging and cleanup. It is not an antivirus scanner or complete file-format validator. Save as `upload_stage.py` and run `python upload_stage.py`:

~~~python
from hashlib import sha256
from pathlib import Path
from tempfile import TemporaryDirectory
from uuid import uuid4

MAX_BYTES = 1024 * 1024

def stage(chunks, directory: Path, expected_sha256: str) -> tuple[Path, int]:
    target = directory / (str(uuid4()) + ".pending")
    total = 0
    digest = sha256()
    created = False
    try:
        with target.open("xb") as output:
            created = True
            for chunk in chunks:
                if not isinstance(chunk, bytes):
                    raise TypeError("expected byte chunks")
                total += len(chunk)
                if total > MAX_BYTES:
                    raise ValueError("upload exceeds size limit")
                digest.update(chunk)
                output.write(chunk)
        if total == 0:
            raise ValueError("empty upload")
        if digest.hexdigest() != expected_sha256:
            raise ValueError("checksum mismatch")
        return target, total
    except BaseException:
        if created:
            target.unlink(missing_ok=True)
        raise

if __name__ == "__main__":
    payload = b"learning upload\n"
    with TemporaryDirectory(prefix="backend-upload-lab-") as folder:
        result, size = stage([payload], Path(folder), sha256(payload).hexdigest())
        print(result.name, size)
        assert result.read_bytes() == payload
        try:
            stage([b"x" * (MAX_BYTES + 1)], Path(folder), "unused")
        except ValueError as error:
            print(error)
        assert len(list(Path(folder).iterdir())) == 1
~~~

Expected output shows one staged filename and size, then an oversize error. The failed file is removed. The function does not buffer the entire input, although the demonstration's producer supplies small in-memory data.

In Django, iterate UploadedFile.chunks() instead of reading a large upload into memory. Configure request size limits and storage deliberately. The surrounding request handler must authenticate, authorize the tenant, validate expected metadata, create the upload record and schedule subsequent processing; this helper alone supplies none of that. Consult [Django file uploads](https://docs.djangoproject.com/en/5.2/topics/http/file-uploads/) when connecting this helper to a view.

## End-to-end workflow

1. Authenticate and create an initiated upload record with limits and an expiry.
2. Issue a restricted upload destination or stream into quarantine storage.
3. Finalize by checking actual storage metadata and a trustworthy checksum when available.
4. Persist the observed object version/checksum and a scan job in one local transaction.
5. Scan and decode in an isolated worker with CPU, memory, time and expansion limits.
6. Write derived artifacts under deterministic, versioned keys.
7. Atomically mark metadata ready only after required outputs exist.
8. Serve only approved versions through authorized downloads or deliberately public URLs.

Compressed files need an expanded-size and file-count budget, not only a compressed-byte limit. Archive paths must not escape the extraction root. Images can have small compressed files with huge pixel dimensions; videos can demand expensive decoding. Strip unnecessary metadata such as location information when the product's policy calls for it.

If accepting remote URLs for import, protect against server-side request forgery: restrict schemes and destinations, check redirects, resolve and reject disallowed address ranges, and prevent access to internal metadata endpoints. Network policy is a useful independent boundary.

## Idempotency, serving and cleanup

A worker can crash after writing a thumbnail but before updating PostgreSQL. Use a deterministic output identity based on upload ID, source version and transformation version. A retry should find or safely replace the same intended artifact, not create unlimited duplicates.

Signed downloads still require authorization before issuance. Choose Content-Disposition and content types deliberately, and avoid serving active user HTML from the main application origin. A CDN can retain deleted content until invalidation or expiry; deletion policy must include cached copies.

Run periodic reconciliation for initiated uploads that never finalized, blobs with no metadata, ready records with missing outputs, and failed multipart uploads. Use an age threshold and ownership evidence before cleanup so a slow active upload is not mistaken for an orphan.

## Failure tests, exercises and checklist

Test empty input, oversize streams, checksum mismatch, duplicate finalization, expired upload authorization and cross-tenant download requests. Kill the worker between output creation and metadata commit. Retry and verify exactly one logical artifact becomes ready.

Common mistakes include trusting extensions, unbounded decompression, public quarantine buckets, user-controlled paths, scanning one version but serving another, and deleting source data before required derivatives are verified.

Design a task-attachment pipeline with private originals and approved previews. Specify retention, scan failure, authorization changes, object versioning and orphan reconciliation.

- [ ] Uploads are authenticated, scoped and size-bounded.
- [ ] Quarantine and ready artifacts have distinct access policies.
- [ ] Checksums/version identities bind scanning to served bytes.
- [ ] Processing has CPU, memory, time and expansion limits.
- [ ] Retries and partial outputs are recoverable.
- [ ] Deletion and orphan cleanup cover every storage layer.
