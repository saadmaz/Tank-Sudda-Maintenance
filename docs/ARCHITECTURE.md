# Architecture

<!-- nav -->

<sub><a href="../README.md">Project README</a> · <a href="README.md">Documentation index</a></sub>

> How `tank-sudda-maintenance` decides what a visitor on tanksudda.com sees. Part of the [project documentation](README.md).

## Where it sits

The Worker is attached to the zone route `*.tanksudda.com/*`. Cloudflare sends it every request for a **proxied** (orange cloud) hostname on the zone, except hostnames that are custom domains of another Worker and win over the route. DNS-only (grey cloud) records go straight to their server and never reach it.

```mermaid
flowchart TD
  R["Request to *.tanksudda.com"] --> P{"Proxied in DNS?"}
  P -- no --> S["Straight to its server<br/>(this Worker is not involved)"]
  P -- yes --> W["tank-sudda-maintenance"]
  W --> A{"Which host?"}
  A -- redirect host --> X["301 to the canonical host"]
  A -- known app --> Y["Pass through"]
  Y --> Z{"App healthy?"}
  Z -- yes --> OK["App's own response"]
  Z -- "no: throw, timeout, 502-504, 52x, 530" --> M["503 maintenance page"]
  A -- anything else --> M
```

## Hosts

| Host                        | What happens                                                                                                      | `X-Tank-Sudda-Fallback`                                             |
| --------------------------- | ----------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------- |
| `tanksudda.com`             | Served by the website Worker (custom domain). Not matched by the route.                                           | none                                                                |
| `www.tanksudda.com`         | 301 to `https://tanksudda.com`, path and query kept                                                               | `redirect`                                                          |
| `pos.tanksudda.com`         | Passed to the origin server. Maintenance page if the origin is unreachable, times out or returns a gateway error. | `origin`, `origin-<status>`, `origin-timeout`, `origin-unreachable` |
| Any other proxied subdomain | 503 maintenance page (JSON for API requests)                                                                      | `unknown-host`                                                      |

## Responses

| Case                                                 | Status | Body                                                                           |
| ---------------------------------------------------- | :----: | ------------------------------------------------------------------------------ |
| Browser (`GET`/`HEAD`, not JSON-only)                |  503   | The branded HTML page (empty for `HEAD`)                                       |
| API or asset request (wants JSON, or a write method) |  503   | `{"error":"maintenance","message":"This service is temporarily unavailable."}` |
| Redirect host                                        |  301   | None; `Location` keeps the path and query                                      |
| Healthy pass-through                                 | App's  | The app's own response, unchanged                                              |

Every maintenance response sends `Retry-After: 300`, `Cache-Control: no-store` and `X-Robots-Tag: noindex`, so search engines treat the outage as temporary and nothing is cached.

Pass-through requests are given **25 seconds** (`ORIGIN_TIMEOUT_MS`). A status in `DOWN_STATUSES` (502, 503, 504, 520 to 527, 530) is treated as "down" and replaced by the maintenance page; any other status, including 404 and 500, is passed through as the app sent it.

## Design constraints

- **No dependencies, no build step.** The page, its styles, the logo mark and the favicon are inlined in `src/index.js`. Nothing is fetched to render it, so it works when every other system is down.
- **One file of logic.** Host lists are plain constants at the top of `src/index.js`, reviewed in pull requests like any other change.
- **Universal SSL covers one level.** Only `<name>.tanksudda.com` can be proxied; `a.b.tanksudda.com` must stay DNS-only or the TLS handshake fails before the Worker runs.
