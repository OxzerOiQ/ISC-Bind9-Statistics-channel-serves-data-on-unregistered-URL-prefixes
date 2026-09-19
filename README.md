# STATS-1 — Statistics channel serves data on unregistered URL prefixes

**Affected product:** ISC BIND 9 (`named`) — `statistics-channels` embedded HTTP server  
**Affected code:** `lib/isc/httpd.c:767-773`  
**Severity:** Low (Medium where path-based filtering is relied upon)  
**Status:** Reported to ISC (confidential)

> **Every claim below was independently
> verified by me against a live `named` instance built from the current source
> tree; all command outputs shown are from my own test runs.

---

## Table of contents

- [Summary](#summary)
- [Affected versions](#affected-versions)
- [Preconditions](#preconditions)
- [Impact](#impact)
- [Screenshots / proof](#screenshots--proof)
- [Steps to reproduce](#steps-to-reproduce)
- [Current vs expected behavior](#current-vs-expected-behavior)
- [Suggested fix](#suggested-fix)
- [Disclosure timeline](#disclosure-timeline)

---

## Summary

The embedded HTTP server backing `statistics-channels` dispatches requests by
comparing the request path against registered URLs with
`strncmp(path, u->url, path_len)`, where `path_len` is the length of the
**requester's** path, not of the registered URL. Any request path that is a
strict byte-prefix of a registered endpoint is routed to that endpoint's
handler instead of being rejected with 404 — e.g. `GET /x` returns the full
`/xml` statistics dataset, `GET /j` returns the full `/json` dataset. This
silently defeats path-based access control enforced in front of BIND (reverse
proxy / WAF rules such as "allow `/xml/*`, deny `/json/*`").

## Affected versions

- Tested: BIND 9.21.26-dev (development branch, built from source).
- Likely affected: all supported branches — the affected code in
  `lib/isc/httpd.c` is long-standing. I did not re-run the test on stable
  releases.

```
BIND 9.21.26-dev (Development Release) <id:9.21.26-dev-test>
running on Linux x86_64 6.6.87.2-microsoft-standard-WSL2 (Ubuntu 24.04.1 LTS)
built by meson with -Dgeoip=enabled
compiled by GCC 13.3.0
compiled with OpenSSL version: OpenSSL 3.0.13 30 Jan 2024
compiled with libuv version: 1.48.0
```

## Preconditions

- `statistics-channels` must be configured (disabled by default), e.g.:
  `statistics-channels { inet 127.0.0.1 port 8080 allow { localhost; }; };`
- The security impact materializes when an operator relies on **path-based
  filtering in front of BIND** (reverse proxy / WAF allowing some statistics
  paths and blocking others) — a standard deployment pattern for monitoring
  endpoints. Within `named` itself all statistics URLs currently share one
  ACL, so no internal authorization boundary is crossed today; the defect
  breaks the routing-exactness assumption that external filters depend on.

## Impact

- Live server operational data (query rates, cache counters, zone lists,
  resolver performance data, the `bind9.xsl` stylesheet) is served on paths
  the operator never registered and may believe are rejected or filtered out.
- Quantified: 100% of the dataset is served on unregistered paths —
  byte-identical responses to the registered parents in my tests (e.g. `/x`
  returns the same 22,681-byte XML document as `/xml/v3`).
- No memory-safety or availability impact.

## Screenshots / proof

<img width="1900" height="892" alt="image" src="https://github.com/user-attachments/assets/d46aa962-3da1-4dc6-8df8-9ff502885cc9" />
<img width="1132" height="712" alt="image" src="https://github.com/user-attachments/assets/8dafcf16-512f-4b4f-a47a-a127717d7e50" />

## Steps to reproduce

1. Build `named` from current source (any build with `HAVE_LIBXML2` /
   `HAVE_JSON_C`; both present in default builds).
2. Use this minimal `named.conf`:

   ```
   options {
       directory "/tmp/nstest";
       listen-on port 5353 { 127.0.0.1; };
       listen-on-v6 { none; };
       recursion no;
   };
   zone "example" { type primary; file "example.db"; };
   statistics-channels {
       inet 127.0.0.1 port 8080 allow { localhost; };
   };
   ```

   (any trivial primary zone file for `example` works)
3. Start the server: `named -c named.conf`
4. Request registered and unregistered paths and compare:

   ```
   curl -s -o /dev/null -w "%{http_code} %{content_type}\n" http://127.0.0.1:8080/xml/v3
   curl -s -o /dev/null -w "%{http_code} %{content_type}\n" http://127.0.0.1:8080/x
   curl -s -o /dev/null -w "%{http_code} %{content_type}\n" http://127.0.0.1:8080/j
   curl -s -o /dev/null -w "%{http_code} %{content_type}\n" http://127.0.0.1:8080/json/v
   curl -s -o /dev/null -w "%{http_code} %{content_type}\n" http://127.0.0.1:8080/bind
   curl -s -o /dev/null -w "%{http_code} %{content_type}\n" http://127.0.0.1:8080/zzz
   ```

## Current vs expected behavior

### Current (buggy) behavior

Unregistered strict prefixes of registered URLs return the full live datasets
(byte-identical to the registered parents):

```
GET /xml/v3   -> 200 text/xml         (22681 bytes)   registered endpoint - control
GET /x        -> 200 text/xml         (22681 bytes)   NOT registered (prefix of /xml)
GET /j        -> 200 application/json (8601 bytes)    NOT registered (prefix of /json)
GET /json/v   -> 200 application/json (8601 bytes)    NOT registered (prefix of /json/v1)
GET /bind     -> 200 text/xslt+xml    (40928 bytes)   NOT registered (prefix of /bind9.xsl)
GET /zzz      -> 404 text/plain       (14 bytes)      only true non-prefixes are rejected
```

(`/`, `/xml`, `/json` themselves ARE registered at
`bin/named/statschannel.c:3867-3922`; the paths above appear nowhere in the
registration table.)

### Expected (correct) behavior

Requests for unregistered paths must return 404. The dispatch should require
an exact match (`path_len == strlen(u->url)`), preserving the special case for
the root `/` and the empty-path default. Suggested regression checks: `/x`,
`/j`, `/json/v`, `/bind`, `/xml/v3/EXTRA` → 404; all registered URLs → 200
unchanged.

## Suggested fix

In `lib/isc/httpd.c`, the dispatch condition currently reads:

```c
if ((strncmp(path, u->url, path_len) == 0) ...)
```

Change it to require a full match on the **registered** URL length, e.g.:

```c
if ((strlen(u->url) == path_len) && (memcmp(path, u->url, path_len) == 0) ...)
```

After this check, the existing handler also compares against the empty string /
root separately, so `/` must keep its special case.

## Disclosure timeline

- **2026-09:** Discovered and verified against a live build
- **2026-09:** Reported to ISC (confidential GitLab issue, `/label ~Bug ~Security`)
- **Public:** At ISC's discretion

--- 

**Discovered by:** MZOUGHI Mohamed Amine(zerOiQ), KyubiSec
