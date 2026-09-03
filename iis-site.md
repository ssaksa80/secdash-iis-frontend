# IIS in front of SECDASH for TLS

SECDASH can terminate TLS itself (it binds `<secdash-ip>:443` with the certificate named
by `-TlsCn`). This runbook is the alternative: put **IIS in front**, so IIS holds the
corporate certificate on `:443` and reverse-proxies to SECDASH on the loopback. The point
is not that SECDASH cannot do TLS — it can — but that certificate renewal, TLS policy and
header hygiene then live where the Windows administrators already manage them.

> **This changes the `:443` topology.** On the shared co-tenant host, SECDASH currently
> *is* the `:443` listener on its own pinned address. Moving TLS to IIS means SECDASH must
> stop binding that address first, and then **IIS must bind the same specific address —
> `<secdash-ip>:443` — with the SECDASH host name (SNI), never "All Unassigned".** An IIS
> binding on the wildcard address makes HTTP.sys reserve `:443` across *every* address on
> the box and takes the other `:443` applications on the host down at the next
> reboot — the same `SO_EXCLUSIVEADDRUSE` trap the `-BindHost` install exists to avoid.
> `<secdash-ip>` must also never become the host's primary IP (`-SkipAsSource $true` stays).

## Prerequisites

- SECDASH installed and answering — see `deploy/RUNBOOK-FIRST-INSTALL.md`.
- IIS with **URL Rewrite** and **Application Request Routing (ARR)** installed. ARR is not
  part of a default IIS install; without it the `Rewrite`-to-`http(s)` proxy silently does
  nothing.
- The corporate certificate for `<secdash-dns-name>` in the machine store (`LocalMachine\My`).

## 1. Move SECDASH behind the proxy (loopback only)

SECDASH's real application is HTTPS-only; the plain-HTTP listener it can open is only an
HTTP→HTTPS redirect, which IIS will now own. Reconfigure the service so SECDASH listens on
the **loopback** and trusts the proxy's forwarded protocol. These live in the **NSSM
service environment**, which wins over `app\server\.env` at runtime (see
`deploy/RELEASING.md` §"BIND_HOST and HTTPS_PORT live in TWO places"), so set them there
and restart:

| Variable | Value | Why |
|---|---|---|
| `BIND_HOST` | `127.0.0.1` | SECDASH answers only the loopback; nothing off-box can reach it directly. |
| `HTTPS_PORT` | `8443` | The loopback backend port IIS proxies to. |
| `TRUST_PROXY` | `true` | `app.set('trust proxy', 1)` — SECDASH honours `X-Forwarded-Proto`, so its `Secure` session cookie and HSTS behave as if the original request was HTTPS. |
| `REDIRECT_HTTP` | `false` | IIS owns the HTTP→HTTPS redirect now; SECDASH must not also open port 80. |
| `COOKIE_SECURE` | `true` | Keep the session cookie `Secure` (default). |
| `TLS_SELF_SIGNED_CN` | `localhost` | The loopback backend keeps a self-signed cert; only IIS talks to it. |

Restart the service, then confirm SECDASH is loopback-only (from **another machine** this
must refuse to connect — if it answers, `BIND_HOST` is not `127.0.0.1`):

```powershell
curl.exe -k -I https://127.0.0.1:8443/healthz        # 200 on this host
```

## 2. Enable the proxy

ARR is off by default and nothing proxies until it is on.

**IIS Manager → the server node → Application Request Routing Cache → Server Proxy
Settings → tick *Enable proxy* → Apply.**

## 3. Allow the forwarded-protocol variable

**IIS Manager → the site → URL Rewrite → View Server Variables → Add →
`HTTP_X_FORWARDED_PROTO`.**

Without this the rewrite rule below cannot set the header, and SECDASH — behind a proxy it
now trusts — would see plain HTTP and refuse to set the `Secure` cookie, looping sign-in.

## 4. Create the site — pinned binding

Add an `https` binding for the SECDASH site:

- **IP address:** `<secdash-ip>` — **not** "All Unassigned" (see the warning above).
- **Port:** `443`
- **Host name:** `<secdash-dns-name>` (require SNI)
- **Certificate:** the corporate certificate for `<secdash-dns-name>`.

Remove the site's port 80 binding, or keep it only to redirect to HTTPS.

## 5. Add `web.config`

Put `deploy/web.config` (shipped beside this runbook) in the site root. It proxies
everything to the loopback backend, tells SECDASH the original request was HTTPS, raises
the upload limit to clear SECDASH's own 20 MB import cap plus multipart overhead, and drops
the header that advertises the stack:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
  <system.webServer>
    <rewrite>
      <rules>
        <rule name="Redirect to HTTPS" stopProcessing="true">
          <match url="(.*)" />
          <conditions>
            <add input="{HTTPS}" pattern="off" />
          </conditions>
          <action type="Redirect" url="https://{HTTP_HOST}/{R:1}" redirectType="Permanent" />
        </rule>
        <rule name="Proxy to SECDASH" stopProcessing="true">
          <match url="(.*)" />
          <action type="Rewrite" url="https://127.0.0.1:8443/{R:1}" />
          <serverVariables>
            <set name="HTTP_X_FORWARDED_PROTO" value="https" />
          </serverVariables>
        </rule>
      </rules>
    </rewrite>
    <security>
      <requestFiltering>
        <!-- SECDASH's import cap is 20 MB (server/src/routes/importRoutes.js);
             allow headroom for multipart + the 2 MB JSON body limit. -->
        <requestLimits maxAllowedContentLength="31457280" />
      </requestFiltering>
    </security>
    <httpProtocol>
      <customHeaders>
        <remove name="X-Powered-By" />
      </customHeaders>
    </httpProtocol>
  </system.webServer>
</configuration>
```

Note: SECDASH sets its own security headers, including HSTS when `NODE_ENV=production`. Do
**not** add a second set in IIS — duplicated headers are how a CSP quietly stops being
enforced.

> Unlike a generic dashboard reverse-proxy template, SECDASH has **no Server-Sent-Events endpoint
> and no `/metrics` endpoint**, so this config deliberately omits the ARR
> response-buffering step and the metrics-blocking rule. Do not copy them in from the other
> projects' `iis-site.md`; there is nothing here for them to act on.

## 6. Certificate to the loopback backend

ARR's outbound request reaches SECDASH over the loopback's self-signed certificate
(`TLS_SELF_SIGNED_CN=localhost`). ARR reverse-proxy rewrite does not validate that
certificate, so no trust step is normally needed; if a `502.3` cites a certificate error,
either trust the loopback cert on this host or regenerate it for `localhost`.

## Verify

```powershell
# through IIS, over TLS, with the real name
curl.exe -I https://<secdash-dns-name>/healthz            # 200

# the SECDASH process itself, loopback only
curl.exe -k -I https://127.0.0.1:8443/healthz             # 200

# a protected API refuses anonymous access, proving auth survives the proxy
curl.exe -i https://<secdash-dns-name>/api/health         # answers; a signed-in-only route returns 401
```

From **another machine**, this must fail to connect — if it answers, `BIND_HOST` is not
`127.0.0.1` and SECDASH is reachable around IIS:

```powershell
curl.exe -k -I https://<server>:8443/healthz              # connection refused
```

And the co-tenancy check still passes — every `:443` listener on its own address, none on
the wildcard:

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File deploy\Test-SecdashTenancy.ps1
```

## Troubleshooting

| Symptom | Cause |
| --- | --- |
| Other `:443` apps fail after a reboot | The IIS binding is on "All Unassigned", not `<secdash-ip>` — HTTP.sys reserved `:443` on every address. Re-bind to the specific IP + host name. |
| `502.3` from IIS | SECDASH is not running, or `HTTPS_PORT`/`BIND_HOST` in the **service env** do not match the rewrite target `127.0.0.1:8443`. |
| Sign-in loops back to sign-in | `TRUST_PROXY` is not `true`, or `HTTP_X_FORWARDED_PROTO` is not reaching the app, so the `Secure` cookie is dropped. |
| Uploads fail near 20 MB | `maxAllowedContentLength` was not raised, or another proxy in front applies its own limit. |
| SECDASH reachable from another machine on `:8443` | `BIND_HOST` is still an external address — set it to `127.0.0.1` in the **service env** (it wins over `.env`) and restart. |
