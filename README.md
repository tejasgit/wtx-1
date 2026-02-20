# WTX-1: Cross-Domain Context Preservation Protocol

**Version:** 1.0.0-draft
**Status:** Draft
**Authors:** Ravi Teja Surampudi, Nylo Contributors
**Created:** 2026-02-20
**License:** This specification is released under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/)

---

## Abstract

WTX-1 (WaiTag Transfer Protocol, version 1) defines a method for preserving pseudonymous user context across unrelated web domains without third-party cookies, browser fingerprinting, login requirements, or personal data collection. The protocol uses cryptographically generated pseudonymous identifiers (WaiTags), URL hash fragment transport, server-side verification, and DNS-based domain authorization to enable privacy-respecting cross-domain analytics.

---

## 1. Problem Statement

### 1.1 The Third-Party Cookie Sunset

Third-party cookies have been the primary mechanism for cross-domain user identification since the 1990s. With Safari ITP (2017), Firefox ETP (2019), and Chrome's Privacy Sandbox (2024+), this mechanism is being eliminated across all major browsers.

### 1.2 What Breaks

When a user navigates from `hospital-a.com` to `pharmacy-b.com`, or from `bank.com` to `investment-portal.com`, analytics systems lose the ability to understand that the same visitor made both visits. This creates blind spots in:

- Patient journey analytics across healthcare providers
- Financial customer experience across banking portals
- Government service usage across agency domains
- Multi-brand retail analytics

### 1.3 Existing Solutions and Their Limitations

| Solution | Limitation |
|----------|-----------|
| Third-party cookies | Being eliminated by all major browsers |
| Browser fingerprinting | Ethically problematic, increasingly blocked, legally risky |
| Login-based identity | Requires authentication; excludes anonymous visitors |
| First-party data sharing | Requires business partnerships and PII exchange |
| Privacy Sandbox Topics API | Coarse-grained, advertising-focused, Chrome-only |
| Adobe ECID | Same eTLD+1 only, classified as personal data under GDPR |

### 1.4 Design Goals

WTX-1 is designed to:

1. Preserve visitor context across unrelated domains (different eTLD+1)
2. Never collect, transmit, or derive personal information
3. Work without cookies, fingerprinting, or login
4. Resist tracking by unauthorized third parties
5. Degrade gracefully when consent is denied
6. Operate within GDPR, CCPA, and ePrivacy frameworks

---

## 2. Terminology

| Term | Definition |
|------|-----------|
| **WaiTag** | A pseudonymous identifier generated per-visitor using cryptographic randomness. Format: `wai_<timestamp_b36>_<random><domain_hash>`. Contains no PII. |
| **Origin Domain** | The domain where the user's session begins and the WaiTag is generated. |
| **Destination Domain** | The domain the user navigates to, which receives and verifies the WaiTag. |
| **Cross-Domain Token** | A time-limited, server-signed token encoding the WaiTag for transfer between domains. |
| **DNS Authorization** | Domain ownership verification via DNS TXT records that authorizes a domain to participate in WTX-1 identity sharing. |
| **Verification Server** | The server-side component that issues, signs, and verifies cross-domain tokens. |
| **Anonymous Mode** | A degraded operating mode where no WaiTag is generated and no identity is preserved. |

---

## 3. Protocol Overview

### 3.1 High-Level Flow

```
┌──────────────┐         ┌──────────────┐         ┌──────────────┐
│   Domain A   │         │  Verification │         │   Domain B   │
│  (Origin)    │         │    Server     │         │ (Destination)│
└──────┬───────┘         └──────┬───────┘         └──────┬───────┘
       │                        │                        │
  1. User visits Domain A       │                        │
  2. SDK generates WaiTag       │                        │
  3. SDK registers WaiTag ──────►                        │
       │                   4. Server stores              │
       │                      WaiTag + domain            │
       │                        │                        │
  5. User clicks link           │                        │
     to Domain B                │                        │
       │                        │                        │
  6. SDK requests cross-        │                        │
     domain token ──────────────►                        │
       │                   7. Server verifies            │
       │                      DNS authorization          │
       │                      for Domain B               │
       │◄──────────────────8. Returns signed token       │
       │                        │                        │
  9. SDK appends token          │                        │
     to URL hash fragment       │                        │
       │                        │                        │
       │─ ─ ─ ─ ─ ─ ─ ─ ─ User navigates ─ ─ ─ ─ ─ ─ ─►
       │                        │                        │
       │                        │  10. SDK reads token   │
       │                        │      from hash         │
       │                        │◄──────────────────────11. SDK sends token
       │                        │                        │   for verification
       │                        │  12. Server verifies   │
       │                        │      signature, expiry │
       │                        │      domain authz      │
       │                        │──────────────────────►13. Returns identity
       │                        │                        │
       │                        │                 14. SDK restores WaiTag
       │                        │                     and session context
```

### 3.2 Step-by-Step

1. **WaiTag Generation** — When a user first visits a participating domain, the SDK generates a WaiTag using the Web Crypto API (`crypto.getRandomValues`). The WaiTag format is `wai_<timestamp_base36>_<random_id><domain_hash>`.

2. **Identity Registration** — The SDK registers the WaiTag with the verification server via `POST /api/tracking/register-waitag`. The server stores the WaiTag, session ID, originating domain, and timestamp.

3. **Cross-Domain Navigation** — When the user navigates to another participating domain, a cross-domain token must be appended to the destination URL. The mechanism for link decoration is implementation-defined (e.g., server-side link rewriting, client-side click handlers, or manual URL construction).

4. **Token Generation** — The server generates a signed, time-limited token encoding the user's WaiTag and session context. The server SHOULD verify that the destination domain is DNS-authorized before issuing the token.

5. **Hash Fragment Transport** — The token is appended to the destination URL as a hash fragment (`#nylo_token=<token>`). Hash fragments are never sent to the server in HTTP requests, providing an additional privacy layer.

6. **Token Verification** — On the destination domain, the SDK reads the token from the hash fragment and sends it to the verification server via `POST /api/tracking/verify-cross-domain-token`.

7. **Identity Restoration** — If verification succeeds, the server returns the original WaiTag and session context. The SDK restores the user's pseudonymous identity on the new domain.

---

## 4. WaiTag Format

### 4.1 Structure

```
wai_<timestamp>_<random><domain_hash>
```

| Component | Encoding | Length | Source |
|-----------|----------|--------|--------|
| Prefix | ASCII | 4 chars | Literal `wai_` |
| Timestamp | Base-36 | ~8 chars | `Date.now()` |
| Separator | ASCII | 1 char | Literal `_` |
| Random ID | Base-36 | 11 chars | `crypto.getRandomValues(new Uint8Array(8))` |
| Domain Hash | Hex | 8 chars | One-way hash of `window.location.hostname` |

### 4.2 Example

```
wai_m5x7k2a1_3f8a9c2b1d4e7a6bc1d2e3f4
```

### 4.3 Properties

- **Not reversible** — No component can be reversed to identify a person
- **Not derived from PII** — No personal information is used as input
- **Domain-scoped** — The domain hash binds the WaiTag to its origin, but the hash is one-way and cannot reveal the domain to a third party
- **Collision-resistant** — 64 bits of cryptographic randomness provides sufficient uniqueness for analytics use cases

### 4.4 What a WaiTag is NOT

- It is not a fingerprint (no hardware/software signals are used)
- It is not a cookie (it does not use the `Set-Cookie` / `Cookie` HTTP mechanism for cross-domain transfer)
- It is not PII (it cannot identify a natural person)
- It is not deterministic (the same user on the same device will get different WaiTags across sessions unless identity is restored)

---

## 5. Token Transport

### 5.1 Primary: URL Hash Fragment

The cross-domain token MUST be transported via URL hash fragment:

```
https://destination.com/page#nylo_token=<signed_token>
```

**Rationale:** Hash fragments (the portion of a URL after `#`) are processed entirely client-side. They are:

- Never included in HTTP requests to the server
- Never logged in server access logs
- Never sent in the `Referer` header
- Not visible to network intermediaries (proxies, CDNs, WAFs)

This provides a stronger privacy guarantee than URL search parameters (`?key=value`), which are transmitted to the server and commonly logged.

### 5.2 Fallback: URL Search Parameters

If hash fragment transport is not feasible (e.g., the destination URL already uses hash-based routing), the token MAY be transported as a search parameter:

```
https://destination.com/page?nylo_token=<signed_token>
```

Implementations using search parameter transport SHOULD:
- Remove the token from the URL immediately after reading it
- Use `history.replaceState` to clean the URL without a page reload

### 5.3 Parameter Names

Implementations MUST accept both parameter names:
- `nylo_token` (primary)
- `wai_token` (legacy/alternative)

### 5.4 Token Cleanup

After reading the cross-domain token, the SDK MUST remove it from the URL to prevent:
- Accidental sharing of token-bearing URLs
- Token replay from browser history
- Token exposure in analytics tools that capture full URLs

---

## 6. Token Format and Verification

### 6.1 Token Contents

A cross-domain token encodes:

| Field | Description |
|-------|-------------|
| `waiTag` | The pseudonymous identifier to transfer |
| `sessionId` | The session identifier from the origin domain |
| `originDomain` | The domain that issued the token |
| `destinationDomain` | The intended recipient domain |
| `issuedAt` | UTC timestamp of token creation |
| `expiresAt` | UTC timestamp of token expiry |
| `signature` | Server-generated signature (HMAC or implementation-defined) |

### 6.2 Verification Requirements

The verification server MUST check:

1. **Signature validity** — The token has not been tampered with
2. **Expiry** — The token has not expired (configurable window, recommended default: 5 minutes)
3. **Domain authorization** — The destination domain SHOULD be DNS-authorized to receive identities
4. **Origin matching** — The token SHOULD be validated against the domain making the verification request

### 6.3 Verification Endpoint

```
POST /api/tracking/verify-cross-domain-token
Content-Type: application/json

{
  "token": "<signed_token>",
  "domain": "destination.com",
  "customerId": "<customer_id>",
  "referrer": "https://origin.com/page"
}
```

**Success Response:**

```json
{
  "success": true,
  "identity": {
    "sessionId": "<session_id>",
    "waiTag": "<wai_tag>",
    "userId": null
  }
}
```

**Failure Response:**

```json
{
  "success": false,
  "error": "TOKEN_EXPIRED"
}
```

---

## 7. DNS Domain Authorization

### 7.1 Purpose

Before a domain can receive cross-domain identities, it must prove ownership via a DNS TXT record. This prevents unauthorized domains from requesting or receiving WaiTags.

### 7.2 TXT Record Format

```
_nylo-verify.example.com  TXT  "nylo-domain-verify=<verification_code>"
```

### 7.3 Verification Flow

1. Domain owner adds the TXT record to their DNS configuration
2. Domain owner calls the verification endpoint: `POST /api/dns/verify`
3. The server performs a DNS lookup for `_nylo-verify.<domain>`
4. If the TXT record matches the expected verification code, the domain is authorized
5. Authorization is cached server-side and periodically re-verified

### 7.4 Subdomain Inheritance

If `example.com` is DNS-verified, subdomains (`blog.example.com`, `shop.example.com`) inherit authorization automatically. Implementations MAY provide configuration to exclude specific subdomains.

---

## 8. Client-Side Storage

### 8.1 Three-Layer Strategy

The SDK stores identity data using three mechanisms for resilience:

| Layer | Mechanism | Persistence | ITP Impact |
|-------|-----------|-------------|------------|
| 1 | First-party cookie (`nylo_wai`) | 24 hours (JS-set under ITP) | Capped at 7 days, 24h if JS-set |
| 2 | `localStorage` | Until cleared | Partitioned by eTLD+1 under ITP |
| 3 | `sessionStorage` | Tab lifetime | Unaffected by ITP |

### 8.2 Storage Read Priority

When restoring identity, the SDK reads in order: cookie → localStorage → sessionStorage. The first valid result is used.

### 8.3 Data Stored

```json
{
  "sessionId": "<secure_id>",
  "waiTag": "<wai_tag>",
  "userId": null,
  "domain": "example.com",
  "createdAt": "2026-02-20T12:00:00.000Z",
  "integrity": "<hash>"
}
```

The `integrity` field is a one-way hash of `sessionId + domain + waiTag + customerId`, used to detect tampering.

### 8.4 Encryption at Rest

Data stored in `localStorage` is encoded with a customer-specific salt and timestamp to prevent casual inspection. This is obfuscation, not cryptographic security — the data contains no PII regardless.

---

## 9. Consent and Anonymous Mode

### 9.1 Consent API

The SDK provides a consent API that integrates with Consent Management Platforms (CMPs):

```javascript
// Deny consent — switch to anonymous mode
Nylo.setConsent({ analytics: false });

// Grant consent — restore identity tracking
Nylo.setConsent({ analytics: true });

// Query current consent state
Nylo.getConsent(); // { analytics: true|false }
```

### 9.2 Anonymous Mode Behavior

When consent is denied (or the SDK is initialized with `data-anonymous="true"`):

| Capability | Behavior |
|-----------|----------|
| Event tracking | Active (no identity attached) |
| WaiTag generation | Disabled — `waiTag` is `null` |
| `identify()` | No-op — silently ignored |
| `getSession()` | Returns `null` for `waiTag`, `userId` |
| Cross-domain identity | Disabled — no tokens generated or accepted |
| Storage | Session-scoped ID only (`anon_<timestamp>_<random>`) |
| Data written to storage | Session-scoped anonymous ID only (no persistent identity stored) |

### 9.3 Consent Restoration

When consent is granted after starting in anonymous mode:

1. SDK checks for previously stored identity data
2. If found, restores the existing WaiTag and session
3. If not found, generates a new WaiTag
4. Cross-domain token checking is activated
5. Full identity tracking resumes

### 9.4 Regulatory Position

WaiTags are designed to fall outside the GDPR definition of "personal data" because:

- They contain no information derived from a natural person
- They cannot be reversed or cross-referenced to identify someone
- They are not deterministic (same person gets different WaiTags across sessions)

However, implementers SHOULD consult local legal counsel. The anonymous mode provides a conservative fallback for jurisdictions where pseudonymous identifiers may be considered personal data.

---

## 10. Security Considerations

### 10.1 Token Security

- Tokens MUST be signed server-side (HMAC or equivalent)
- Tokens MUST have a configurable expiry (recommended: 5 minutes)
- Tokens SHOULD be single-use where feasible
- Token verification MUST check domain authorization

### 10.2 Transport Security

- All API communication MUST use HTTPS
- Hash fragment transport prevents server-side token logging
- Search parameter fallback MUST clean the URL after reading

### 10.3 Input Validation

- All client-supplied strings MUST be sanitized (HTML entity encoding)
- String fields MUST be length-limited (recommended: 1,000 characters)
- Domain values MUST be validated against DNS authorization

### 10.4 Rate Limiting

- API endpoints MUST implement rate limiting (recommended: 100 requests/IP/minute)
- Rate limit status MUST be communicated via `X-RateLimit-*` headers
- Exceeded limits MUST return HTTP 429

### 10.5 Response Headers

Servers implementing WTX-1 SHOULD set:

```
X-Content-Type-Options: nosniff
X-Frame-Options: SAMEORIGIN
X-XSS-Protection: 1; mode=block
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: camera=(), microphone=(), geolocation=()
Content-Security-Policy: default-src 'self'; script-src 'self' 'unsafe-inline'; ...
Strict-Transport-Security: max-age=31536000; includeSubDomains
```

---

## 11. Privacy Guarantees

WTX-1 makes the following privacy commitments:

1. **No PII collection** — The protocol never collects, transmits, or derives personal information
2. **No fingerprinting** — No hardware, software, or behavioral signals are used to generate identifiers
3. **No third-party cookies** — Cross-domain identity does not use the cookie mechanism
4. **No login required** — Identity preservation works for anonymous visitors
5. **Consent-respecting** — The protocol degrades to fully anonymous tracking when consent is denied
6. **Domain-authorized** — Only DNS-verified domains can participate in identity sharing
7. **Time-limited tokens** — Cross-domain tokens expire, preventing indefinite tracking
8. **Client-side token reading** — Hash fragment transport ensures tokens are never sent to servers in HTTP requests

---

## 12. Implementation Requirements

### 12.1 Client SDK Requirements

An implementation MUST:
- Generate WaiTags using `crypto.getRandomValues` (with `Math.random` fallback)
- Support hash fragment token transport
- Support search parameter token fallback
- Implement the consent API (`setConsent`, `getConsent`)
- Implement anonymous mode as specified in Section 9.2
- Clean tokens from URLs after reading
- Sanitize all user-supplied input

An implementation SHOULD:
- Use the three-layer storage strategy
- Monitor performance with the Performance API
- Batch events before sending to reduce network requests

### 12.2 Server Requirements

An implementation MUST:
- Verify cross-domain tokens (signature, expiry, domain authorization)
- Implement DNS domain authorization
- Enforce rate limiting on all API endpoints
- Set security response headers
- Never log cross-domain tokens in plain text

An implementation SHOULD:
- Support configurable token expiry windows
- Provide token revocation capabilities
- Monitor and alert on verification failures

---

## 13. Reference Implementation

The reference implementation is the [Nylo SDK](https://github.com/tejasgit/nylo):

- **Client SDK:** `src/nylo.js` (~1,000 lines, zero dependencies)
- **Server integration:** `server/` (Express.js)
- **Demo:** `examples/demo-server.js` + `examples/demo.html`
- **License:** MIT (core tracking), Commercial (cross-domain features)

---

## 14. Future Work

- **W3C Community Group** — Propose WTX-1 as a web platform standard
- **postMessage transport** — Support iframe-based cross-domain communication
- **Navigation API integration** — Use the Navigation API for cleaner token transport
- **Multi-party verification** — Decentralized token verification without a single server
- **Formal privacy analysis** — Commission independent privacy review

---

## 15. Acknowledgments

WTX-1 was developed as part of the Nylo project to address the analytics gap created by the deprecation of third-party cookies, with a focus on privacy-respecting solutions for regulated industries.

---

## Appendix A: Comparison with Existing Approaches

| Property | Third-Party Cookies | Fingerprinting | Login-Based | Adobe ECID | WTX-1 |
|----------|-------------------|----------------|-------------|------------|-------|
| Cross eTLD+1 | Yes (deprecated) | Yes | Yes | No | Yes |
| Requires login | No | No | Yes | No | No |
| Collects PII | Varies | Yes | Yes | Debated | No |
| Browser support | Declining | Declining | Universal | Declining | Universal |
| Consent required | Yes (ePrivacy) | Yes | Varies | Yes (GDPR) | Conservative: Yes |
| Works on Safari | No (ITP) | Partially | Yes | Limited | Yes (degraded) |

## Appendix B: Glossary

- **eTLD+1** — Effective top-level domain plus one label (e.g., `example.com` for `blog.example.com`)
- **ITP** — Intelligent Tracking Prevention (Safari/WebKit)
- **ETP** — Enhanced Tracking Protection (Firefox)
- **CMP** — Consent Management Platform
- **HMAC** — Hash-based Message Authentication Code
- **PII** — Personally Identifiable Information
