# Public ADNS

A free, public authoritative DNS service. Your zones are served from dual-stack name servers with DNSSEC signing (ECDSAP256SHA256) on by default. No web panel, no API key — zones are submitted by request and managed as zone files.

| | |
|---|---|
| **DNSSEC** | ECDSAP256SHA256 (algorithm 13) |
| **Network** | IPv4 and IPv6 |
| **Logging** | None |
| **Name servers** | `name1`, `name2` |
| **Cost** | $0 |

**Website:** [public-adns.com](https://public-adns.com/)

---

## Table of contents

- [Quick Start](#quick-start)
- [Name Servers](#name-servers)
- [Privacy](#privacy)
- [How to Use](#how-to-use)
- [Zone File Format](#zone-file-format)
- [DNSSEC](#dnssec)
- [Delegating Your Domain](#delegating-your-domain)
- [Verifying Your Zone](#verifying-your-zone)
- [Features](#features)
- [Infrastructure](#infrastructure)
- [Troubleshooting](#troubleshooting)
- [FAQ](#faq)
- [Acceptable Use](#acceptable-use)
- [Other Projects](#other-projects)
- [Contact and Support](#contact-and-support)

---

## Quick Start

1. Email [root@public-common.com](mailto:root@public-common.com) with the domain you want hosted and the zone contents (or a link to a zone file).
2. You will receive back a DS record for your domain.
3. At your registrar, set the name servers to `name1.public-adns.com` and `name2.public-adns.com`, and publish the DS record.
4. Once the parent zone updates, your domain is live and DNSSEC-validated.

---

## Name Servers

| | |
|---|---|
| **Primary** | `name1.public-adns.com` |
| **Secondary** | `name2.public-adns.com` |

### IP addresses

| | |
|---|---|
| **name1 IPv4** | `135.181.224.220` |
| **name1 IPv6** | `2a01:4f9:3071:2d1a::220` |
| **name2 IPv4** | `135.181.224.231` |
| **name2 IPv6** | `2a01:4f9:3071:2d1a::231` |

Both name servers answer on UDP and TCP port 53, plus DNS-over-TLS on port 853. IPv4 and IPv6 are supported on every address.

---

## Privacy

Authoritative DNS by its nature exposes the zones it serves to anyone who queries them — that is the whole point. What we do not do is profile or track who is querying.

- **No client query logging** — individual queries are not written to disk. Only operational logs (process state, errors) are kept.
- **Encrypted storage** — zone files, signing keys, and operational data at rest are protected with ZFS native encryption.
- **No shell history** — operator sessions leave no command history on the server.
- **Headless server** — eliminating physical access vectors.
- **Hidden version and identity** — `version.bind` and `id.server` probes are refused.
- **ANY queries refused** — to mitigate amplification and reduce zone-walking surface.
- **No analytics or trackers** — this site is static HTML and sets no cookies.

---

## How to Use

There is no self-service portal. To host a zone:

1. Decide on the zone contents — at minimum SOA / NS / A / AAAA records for your apex.
2. Email the request to [root@public-common.com](mailto:root@public-common.com). Include either the desired records inline, or attach a zone file in standard BIND format.
3. The zone is created on the server. New KSK and ZSK ECDSA P-256 keys are generated, the zone is signed with a 90-day signature lifetime, and NSD reloads.
4. You receive the DS record (the parent-side fingerprint of your KSK). Submit it at your registrar along with the NS change to `name1.public-adns.com` / `name2.public-adns.com`.
5. To update records later, send the new zone contents and the zone is re-signed and reloaded. Keys are kept; the DS does not need to change.

The server-side workflow is the two short scripts in the `script/` directory of the repo: `domain-add.sh` creates and signs a new zone, `domain-update.sh` re-signs an existing one. Both call `ldns-keygen` / `ldns-signzone` and HUP NSD to pick up changes.

---

## Zone File Format

Zones use the standard RFC 1035 master-file format that BIND, NSD, Knot, and PowerDNS all accept. A minimal apex looks like this:

```
$ORIGIN example.com.
$TTL    3600

@   IN  SOA name1.public-adns.com. hostmaster.example.com. (
            2026060901 ; serial (YYYYMMDDnn)
            7200       ; refresh
            3600       ; retry
            1209600    ; expire
            3600 )     ; minimum / negative TTL

@   IN  NS  name1.public-adns.com.
@   IN  NS  name2.public-adns.com.

@   IN  A     203.0.113.10
@   IN  AAAA  2001:db8::10
www IN  A     203.0.113.10
www IN  AAAA  2001:db8::10
@   IN  MX    10 mail.example.com.
@   IN  TXT   "v=spf1 -all"
```

Increment the SOA serial every time you submit changes — convention is `YYYYMMDDnn` (date plus a two-digit revision). The signing tool will reject zones whose serial has not advanced.

---

## DNSSEC

Every hosted zone is signed automatically. There is no opt-out — DNSSEC is the default, and the service exists in part to make it painless.

- **Algorithm:** ECDSAP256SHA256 (algorithm 13). Small signatures, fast validation, supported by every modern resolver.
- **Keys:** separate KSK (key-signing key) and ZSK (zone-signing key) per domain.
- **Signing tool:** `ldns-signzone`.
- **Signature lifetime:** 90 days. The zone is re-signed at every update; long-lived zones must be refreshed before signatures expire.
- **DS record:** generated by `ldns-signzone` as a `.ds` file alongside the keyset, and provided to you for upload at your registrar.

If your registrar accepts only DS digests of a particular type, ask and an alternate digest can be supplied.

---

## Delegating Your Domain

At your registrar, two changes are needed:

1. **Name servers:** set to `name1.public-adns.com` and `name2.public-adns.com`.
2. **DS record:** publish the DS record we provide. Most registrars expose a "DNSSEC" or "DS records" panel that takes the four fields directly:
   - Key tag
   - Algorithm: 13 (ECDSAP256SHA256)
   - Digest type: 2 (SHA-256)
   - Digest hex string

Do not skip the DS step. Without it the zone is signed but not *validated* — resolvers treat it as insecure.

---

## Verifying Your Zone

After delegation, check from any recursive resolver:

```bash
# Records present
dig example.com A AAAA NS SOA

# DNSSEC chain valid
dig +dnssec +multiline example.com SOA
dig +dnssec example.com DS

# Signed answer (look for "ad" flag)
dig @1.1.1.1 example.com +dnssec +multiline | grep -E "flags|RRSIG"
```

The `ad` (Authenticated Data) flag in the response header confirms that a validating resolver has successfully walked the chain from the root through your DS down to the answer. Any DNSSEC-validating resolver works for this check, including [public-rdns.com](https://public-rdns.com/).

Online validators such as [DNSViz](https://dnsviz.net/) and [Verisign DNSSEC Analyzer](https://dnssec-analyzer.verisignlabs.com/) give a visual breakdown if anything is off.

---

## Features

- DNSSEC signing on by default (ECDSAP256SHA256, separate KSK + ZSK)
- IPv4 and IPv6, two addresses in each family
- Two name server hostnames (`name1`, `name2`)
- NSD authoritative server — small, fast, audited
- ANY refused, identity and version hidden
- UDP and TCP on port 53, DNS-over-TLS on port 853
- EDNS0 / large-response support
- Standard zone file format — portable in and out
- No charge, no signup, no rate-tiered "premium"

---

## Infrastructure

- **OS:** FreeBSD
- **Daemon:** NSD
- **Filesystem:** ZFS with native encryption
- **Signing:** `ldns-keygen` / `ldns-signzone`, ECDSAP256SHA256, 90-day signature window
- **Reload:** SIGHUP to NSD on every zone change
- **Connections:** high TCP backlog tuned for many simultaneous transfers and queries

Operator does not retain shell history, and the system has no remote console exposed to the public internet beyond the services listed above.

---

## Troubleshooting

### Domain resolves but DNSSEC says insecure

The DS record is missing or wrong at the parent. Check with `dig DS yourdomain.tld` against a public resolver and compare to the DS we provided. Registrars sometimes silently strip DS records when nameservers are also changed in the same operation — re-submit the DS as a separate change if needed.

### SERVFAIL after a recent update

Most likely the SOA serial was not incremented, so caching resolvers still hold the old version while the new keys / signatures are live. Wait for the negative-cache TTL (set by your zone's SOA minimum) or flush at a test resolver.

### Signatures expired

Signature lifetime is 90 days. If a zone has not been refreshed in that time, validators will return `RRSIG-EXPIRED`. Send any update (even a serial bump) and the zone is re-signed. If validation fails on a freshly signed zone, check that the validating client's clock is correct — DNSSEC signatures, like TLS, depend on it. See [public-utc.com](https://public-utc.com/).

### Registrar rejects the DS

Some registrars only accept DS digest type 2 (SHA-256). Algorithm must be 13 (ECDSAP256SHA256). If your registrar's UI lacks algorithm 13, switch registrars or ask for help — but do not fall back to a weaker algorithm.

### EDNS / large response problems

DNSSEC answers are bigger than plain DNS. If a network path drops large UDP, the resolver should retry over TCP. If TCP is also blocked, fix the network — there is no way to deliver signed answers reliably without it.

---

## FAQ

**Is this really free?**  
Yes. There is no charge, no signup, no API key. Donations via Bitcoin are appreciated but not required — see [Contact and Support](#contact-and-support).

**Why no web panel?**  
Web panels are the largest source of authoritative DNS compromises. Zones here are managed by zone-file submission via email, signed automatically on the server, then served by NSD. That keeps the public attack surface to NSD itself.

**Can I host a zone for any TLD?**  
Anything routable in the public DNS, yes — `.com`, ccTLDs, new gTLDs, all fine. We won't host private / internal-only zones for redirected domains used in phishing or abuse.

**Do you offer secondary / slave service for an existing primary?**  
Not by default — this server is a hidden-master-style authoritative for its own zones. If you have a specific need for AXFR/IXFR-based secondary service, email and ask; it can be configured per zone.

**Can I run my own primary and use you as secondary?**  
Same answer as above — possible, ask first. We will need TSIG keys and ACLs for AXFR.

**What about dynamic updates / RFC 2136?**  
Not supported. Updates go through the email submission flow, and the zone is re-signed each time.

**Does it support DNS over TLS / HTTPS?**  
DNS-over-TLS is available on port 853. Authoritative DoH is not yet widely deployed. Plain DNS on UDP/TCP 53 remains what every recursive resolver actually queries.

**What's the SLA?**  
Best-effort. The service is operated as a public good, not a paid product. Two name server hostnames on independent IPv4/IPv6 addresses provide basic redundancy; for stricter availability requirements, also configure a secondary at another operator.

---

## Acceptable Use

- No malware command-and-control, phishing, or other abuse zones. These will be removed without notice.
- No fast-flux or DGA hosting.
- Keep zones reasonable in size — this is not a bulk wildcard CDN backend.
- Increment SOA serials on every change.
- Keep your registrar contact details accurate so abuse complaints reach you, not us.

---

## Other Projects

| Site | Service |
|------|---------|
| [public-consortium.com](https://public-consortium.com/) | Project home and operations |
| [public-adns.com](https://public-adns.com/) | Public authoritative DNS service (this site) |
| [public-rdns.com](https://public-rdns.com/) | Public recursive DNS service |
| [public-blank.com](https://public-blank.com/) | Public static / parking service |
| [public-repo.com](https://public-repo.com/) | Public mirror service |
| [public-utc.com](https://public-utc.com/) | Public NTP / NTS time service |

---

## Contact and Support

For questions, abuse reports, zone hosting requests, or operational issues, email [root@public-common.com](mailto:root@public-common.com).

Discord: [Public Consortium](https://discord.gg/Hjgz5EWb)

Support via BTC: `bc1qxfcfkvgyjs6pq6cdmv6syx5s6pc9dn3wsuewjq`
