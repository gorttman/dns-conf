# Pi-hole split-horizon DNS

Pi-hole is the internal resolver for `*.i3sec.com.au`. The goal: any LAN
client resolving one of these hostnames should get an internal IP
(Traefik VIP `192.168.2.241`, ingress-nginx VIP `192.168.2.242`, or a
direct host IP like `192.168.2.30` for qnap) and never touch the
Cloudflare Tunnel at all. Config lives in the `pihole-custom-dns`
ConfigMap (`pihole-custom-dns-cm.yml`), mounted into dnsmasq/FTL as
`02-i3sec-custom-dns.conf`.

## The recurring failure mode: record-type leaks

dnsmasq's `address=/host/ip` override only rewrites **A** records (and,
separately, `filter-AAAA` can suppress AAAA globally). It does not touch
any other DNS record type. Any hostname that is both locally overridden
*and* has a real public presence (i.e. it's also routed through the
Cloudflare Tunnel, per `cloudflare-tf`) will leak the real public answer
for every record type dnsmasq doesn't know to rewrite — letting a client
bypass split-horizon entirely for that query, reach Cloudflare's real
edge, and get blocked there (WAF/mTLS rules assume a client cert that
was never presented, since the tunnel path was never supposed to be
used from the LAN).

This has bitten twice, same root cause, different record type:

### Fix 1 (2026-07-09): AAAA leak
IPv6-preferring clients queried AAAA, got no local override, and fell
through to the real public Cloudflare AAAA answer. Fixed with a global
`filter-AAAA` — dnsmasq has no per-domain AAAA override, only this
network-wide toggle. (IPv6 is believed off at the router anyway, so
suppressing it globally has no downside here.)

### Fix 2 (2026-07-15): HTTPS/SVCB leak
Even with the AAAA leak closed, `qnap.i3sec.com.au` was still getting
blocked by Cloudflare on an iPad. Root cause: Safari (and other
HTTP/3-capable browsers) also query the newer **HTTPS record** (type
65, aka SVCB) for ALPN/QUIC and ECH connection hints. `address=` never
touched this record type, so it kept passing through untouched with
Cloudflare's real anycast IPs as `ipv4hint`. Safari preferred those
hints over the correctly-overridden A record for a follow-up navigation
(QNAP's own `redirect.html` self-check page), connected straight to
Cloudflare's edge, and was blocked by the zone-wide default-deny mTLS
WAF rule added the day before (2026-07-14, `cloudflare-tf` commit
`8e008ab`) — no client cert is ever presented on a path that was never
supposed to leave the LAN.

Confirmed via direct `dig` queries against the live Pi-hole pod: the A
record for `qnap.i3sec.com.au` correctly returned `192.168.2.30`, but
the HTTPS record returned Cloudflare's real edge IPs — identical to
querying `1.1.1.1` directly. WARP on vs. off made no difference (ruling
out client-side VPN routing); the leak was purely DNS-side.

Fixed the same way as the AAAA leak: dnsmasq has no per-domain HTTPS/SVCB
override either, only the global `filter-rr=<type>` directive. Added
`filter-rr=65` alongside `filter-AAAA` to strip HTTPS records globally.

```
filter-AAAA
filter-rr=65
```

Verified post-fix: A record still `192.168.2.30`, HTTPS record now
empty, AAAA record still suppressed.

## Gotchas that apply to any future change here

- **ConfigMap uses a `subPath` mount.** `subPath`-mounted files don't
  get Kubernetes' atomic symlink-swap update — editing the ConfigMap
  alone does nothing until the Pi-hole pod is manually restarted.
- **ArgoCD git-polling can lag.** Force a hard refresh
  (`argocd.argoproj.io/refresh: hard` annotation patch) if a pushed
  change isn't showing up as synced.
- **New record types can leak the same way again.** If a new browser
  feature or protocol introduces another DNS record type (e.g. some
  future connection-hint or discovery record) and a hostname is on both
  the internal network and the Cloudflare Tunnel, assume it needs its
  own `filter-rr=<type>` unless proven otherwise — don't assume
  `address=` + `filter-AAAA` covers everything.
- Check `dig <host> <type> @127.0.0.1` against the live pod (not a local
  machine — `dig`/`nslookup` aren't installed in this sandbox, use
  `kubectl exec` into the pihole pod) before and after any DNS-side fix.

See also `day1-foundation/apps/cloudflare-tf/README.md` and
`HISTORY.md` for the tunnel/WAF/mTLS side of this system — the two
repos are two halves of the same split-horizon design and a change to
one can surface as a bug that only manifests through the other.
