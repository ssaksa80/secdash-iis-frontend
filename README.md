# SECDASH — IIS/TLS front-end

Operator artifacts for putting **IIS in front of SECDASH for TLS** on a shared,
co-tenant `:443` Windows host. IIS terminates TLS with the corporate certificate and
reverse-proxies (URL Rewrite + ARR) to the SECDASH loopback backend, so certificate
renewal, TLS policy and header hygiene stay where the Windows administrators already
manage them.

- **[`iis-site.md`](iis-site.md)** — the step-by-step runbook: prerequisites, the SECDASH
  service-environment changes for a loopback backend, the pinned IIS binding, the
  `web.config`, and the verify + troubleshooting steps.
- **[`web.config`](web.config)** — the IIS site-root config the runbook installs (proxy to
  the loopback backend, forward the HTTPS protocol, upload limit, strip `X-Powered-By`).

## The one rule that matters

On a shared `:443` host the IIS binding must be pinned to the app's own IP **and** host
name (SNI) — never "All Unassigned". A wildcard binding makes HTTP.sys reserve `:443` on
every address and takes the other `:443` applications down at the next reboot. The runbook
spells this out.

## Placeholders

All hostnames, IPs and certificate references are placeholders — `<secdash-dns-name>`,
`<secdash-ip>` — to be filled in for the target host. Nothing real is committed here.

## Self-contained

This repository is only the IIS/TLS front-end config. It does not vendor, fork, or depend
on the SECDASH application repository or any other project.
