# CDNs and edge caching

[Curriculum](../README.md) · [System-design map](README.md) · Prerequisites: HTTP, [application caching](../02-data/06-caching-and-redis.md), [load balancing](08-load-balancing-and-traffic-management.md)

## Design from the representation

A content delivery network places serving/cache capacity near clients and forwards misses to an origin. It can reduce latency, origin load, and long-distance transfer. It also creates copies that must respect identity, freshness, deletion, and publication rules.

Decide separately for each representation: who may read it, which request dimensions change its bytes, how long it may be stale, and what must happen after deletion or permission revocation. Public versioned JavaScript and a private task export have different answers even when both are files in an object store.

```text
Browser cache -> nearby edge cache -> optional regional shield -> origin
                                                     origin = object store or API
```

A shield aggregates misses from multiple edges and can reduce origin fanout. It adds another hop/cache boundary and does not eliminate origin outages, cold starts, or inconsistent configuration.

## Cache keys are a correctness contract

At minimum, think about authority/host and the resource path. Query parameters, normalized language, representation format, content encoding, tenant scope, and selected headers can also change the response. Include all dimensions that actually determine bytes, and exclude irrelevant dimensions only through an explicit policy.

If `lang=en` and `lang=fr` produce different content but share a cache key, a hit can return the wrong language. If a tenant-private response shares a public key, the problem becomes data exposure. Adding every cookie/header to the key avoids some collisions but can fragment the cache until nearly every request is a miss.

CloudFront's documentation illustrates how cache policies choose headers, cookies, and query strings for cache keys. Different CDNs have different defaults and normalization rules; inspect the policy actually deployed. [CloudFront cache keys](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/controlling-the-cache-key.html).

Edge and origin must agree about URL parsing. Encoded separators, duplicate parameters, case, path normalization, and alternate hostnames can create cache-poisoning or cache-deception paths. Do not let an unkeyed attacker-controlled header alter the origin response. Restrict forwarded headers as well as the cache key.

## Freshness, revalidation, and stale responses

HTTP distinguishes storing a response, reusing a fresh response, and validating a stored response. `no-store` prohibits storage by conforming caches; `no-cache` permits storage but requires validation before reuse. `private` restricts shared caching. `max-age` sets freshness, while `s-maxage` targets shared caches. `Vary` identifies request headers affecting representation selection. A conditional revalidation can return 304 without retransmitting the body. [HTTP caching specification](https://www.rfc-editor.org/rfc/rfc9111.html).

Example policy choices, **not automatic headers for every Django view**:

| Representation | Example starting policy | Reason |
| --- | --- | --- |
| Public asset with content hash in URL | `public, max-age=31536000, immutable` | A changed asset gets a new URL |
| Public catalogue JSON | Short shared freshness plus validators | Origin load reduction with defined staleness |
| Personal account response | `private, no-store` | Avoid reuse by a shared intermediary |
| Signed private media segment | Product-specific edge authorization and bounded reuse | Cached bytes must still require access |

The one-year lifetime is an illustrative policy, not a requirement. An immutable URL must truly identify immutable content. Stale-while-revalidate or stale-if-error can improve availability, but explicitly forbid stale reuse where revoked access or changed security state must be respected. CDN-specific minimum TTL settings can override intuitive application expectations; test effective behavior.

## Lab: test cache-key isolation

Save as `cache_key_lab.py` and run with Python. This is a local representation-cache model; it does not implement HTTP cache semantics or a CDN.

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class Request:
    tenant: str
    path: str
    language: str
    tracking: str = ""


def origin(request):
    return f"{request.tenant}:{request.language}:{request.path}"


def get(cache, request, key_function):
    key = key_function(request)
    if key not in cache:
        cache[key] = origin(request)
    return cache[key]


a = Request("tenant-a", "/report", "en", "campaign-one")
b = Request("tenant-b", "/report", "en", "campaign-two")
bad_cache = {}
bad_key = lambda r: r.path
assert get(bad_cache, a, bad_key) == get(bad_cache, b, bad_key)
assert get(bad_cache, b, bad_key) != origin(b)  # a cross-tenant leak

good_cache = {}
good_key = lambda r: (r.tenant, r.path, r.language)
assert get(good_cache, a, good_key) == origin(a)
assert get(good_cache, b, good_key) == origin(b)
same_content = Request("tenant-a", "/report", "en", "different-campaign")
assert get(good_cache, same_content, good_key) == origin(a)
assert len(good_cache) == 2
print("bad key leaks; scoped key isolates; irrelevant tracking does not fragment")
```

The model assumes authorization has already established tenant identity. A safe-looking key containing an unchecked tenant header is still unsafe. For real private content, evaluate whether shared caching is justified at all, then test authorization on hits as well as misses.

## Publishing, invalidation, and revocation

For immutable assets, upload the new object first, verify it, then publish a manifest referencing its versioned URL. Retain previous versions long enough for old pages and deployments. Reusing an existing hash-like URL with new bytes breaks the promise.

For mutable resources, use bounded freshness and validators, plus invalidation when required. An invalidation request is not evidence that every edge and browser has already discarded the object. Track completion where supported and define the residual browser/third-party cache behavior. Application responses and CDN purge identifiers should come from a reliable publication path so a crash does not leave a permanent stale entry.

For urgent removal, block serving at an authorization/routing layer, invalidate cached copies, remove or quarantine the origin, and verify from representative paths. Copies outside the service's control may remain; retention and user-facing deletion promises must reflect this reality.

Signed URLs and cookies usually grant bounded access. Validate expiry, scope, key rotation, and permitted methods according to the provider's mechanism. Do not simply strip signature parameters from the cache key while forwarding unauthenticated hits: authorization must be enforced before a cached response is served. AWS distinguishes signed content access from making the origin publicly reachable. [Private content with CloudFront](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/PrivateContent.html).

## Capacity calculation and origin protection

Assume a fictional workload of 20,000 image requests/s with average response size 100 KiB. Viewer payload is approximately 2,048,000,000 bytes/s, or 1.91 GiB/s. At a 95% request hit rate, origin sees about 1,000 requests/s. If object sizes are uniform, miss payload is about 97.7 MiB/s; real designs should measure a separate byte hit ratio because a few large misses can dominate transfer.

A cold-cache event can send far more than steady-state misses to the origin. Plan request coalescing, bounded origin concurrency, gradual warming, shield behavior, and failover. Do not benchmark only a fully warm cache. Origin access controls should prevent clients from bypassing edge policy through a public storage URL.

Compression reduces transfer for suitable content but consumes compute and changes representations. Do not recompress already compressed media without a reason. Range requests, conditional requests, negative responses, and redirects need explicit cache policies; test them rather than assuming every successful GET is equivalent.

## Verification lab with a real configured cache

After installing/configuring your chosen cache in an owned test environment:

1. Fetch a public versioned asset twice; compare cache diagnostics, Age, ETag, bytes, and origin request count.
2. Change only an irrelevant tracking parameter and verify the intended key policy.
3. Change a meaningful language/format dimension and verify correct representation separation.
4. Repeat private reads as two identities and verify authorization on a warm hit.
5. Remove an object, revoke access, or invalidate it; measure when each serving path stops returning it.
6. Stop the origin and test the declared stale/error policy.
7. Simulate a cold cache with a bounded workload and measure origin saturation.

No CDN or cloud distribution is created by this tutorial. The standalone model above can be run without infrastructure; these integration steps require the specified setup.

## Completion checklist

- [ ] Cache keys include every meaningful representation/security dimension.
- [ ] Origin and edge parse requests consistently.
- [ ] Hit authorization and revocation are tested separately from origin authorization.
- [ ] Freshness and invalidation satisfy the product's actual staleness contract.
- [ ] Request hit ratio, byte hit ratio, and origin load are measured separately.
- [ ] Cold-cache and origin-outage behavior have bounded failure paths.

Apply these decisions to the [video-streaming design](../11-system-design-case-studies/04-video-streaming.md) and [URL shortener](../11-system-design-case-studies/01-url-shortener.md).
