# tank-sudda-maintenance

Branded "Sorry, we're under maintenance" fallback for **tanksudda.com**, deployed as a Cloudflare Worker on the
route `*.tanksudda.com/*`.

- Unknown or mistyped subdomains get a 503 maintenance page (JSON for API requests) instead of a Cloudflare error.
- `www.tanksudda.com` → `https://tanksudda.com` (301, path and query kept).
- Origin apps (`pos.tanksudda.com`) pass through, with the maintenance page if the origin is down. They only reach this Worker while proxied.

Every response carries `X-Tank-Sudda-Fallback` saying which branch answered.

```sh
node test/index.test.mjs        # local tests, no dependencies
npx wrangler deploy             # deploy (uses the existing wrangler login)
```

When a new app goes live on a subdomain of tanksudda.com, add it here and redeploy, or the route may answer it with
the maintenance page:

- a Worker (custom domain): add it to `SERVICE_HOSTS` and a matching `services` binding in `wrangler.jsonc`;
- a proxied record pointing at an origin server: add it to `ORIGIN_HOSTS`.

DNS-only (grey cloud) records never reach this Worker. Check with `curl -sI https://<host>/`: a `X-Tank-Sudda-Fallback`
header means this Worker answered.
