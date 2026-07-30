# Reference · HTTP headers

Every header observed on a Meet `$rpc` call, with provenance.

Prose and reasoning: [docs/01-transport.md](../docs/01-transport.md) ·
[docs/02-authentication.md](../docs/02-authentication.md).

---

## Request headers

| Header | Required | Value | Derivable? | Evidence |
| --- | --- | --- | --- | --- |
| `authorization` | **yes** | `SAPISIDHASH <ts>_<sha1> SAPISID1PHASH <ts>_<sha1> SAPISID3PHASH <ts>_<sha1>` | ✅ from cookies | ✅ Proven |
| `cookie` | **yes** | the profile's Google cookies | from a signed-in profile | ✅ Proven |
| `origin` | **yes** | `https://meet.google.com` | constant | ✅ Proven |
| `referer` | yes | `https://meet.google.com/` | constant | 👁 Observed |
| `content-type` | **yes** | `application/x-protobuf` | constant | ✅ Proven |
| `content-encoding` | when gzipped | `gzip` | constant | ✅ Proven |
| `x-goog-api-key` | **yes** | `AIzaSy…` — see note | constant, **public** | ✅ Proven |
| `x-goog-authuser` | yes | `0` | constant | 👁 Observed |
| `x-goog-meeting-identifier` | **yes** | base64 of `{1: 3, 2: "<space id>"}` | ✅ given the space id | ✅ Proven |
| `x-goog-meeting-token` | **yes** | `<epoch ms>;<opaque>` | ❌ **server-issued** | ✅ Proven |
| `x-goog-meeting-rtcclient` | yes | `CAEQqgMYASABKAg4AQ==` | constant | 👁 Observed |
| `x-goog-meeting-debugid` | **on the media call** | `boq_hlane_<random>` | ❌ from the page | ✅ Proven required |
| `x-goog-encode-response-if-executable` | yes | `base64` | constant | 👁 Observed |
| `x-goog-meeting-bot-info` | **no — omit it** | ~3268 bytes | ❌ | ✅ Proven optional |
| `user-agent` | effectively | Chrome's product token | constant | 🧩 Inferred |

### Notes

**`origin`** — not decorative. The authorization signature is computed *over* the
origin and the server checks the two agree. Omitting it yields
`401 missing required authentication credential`, which points misleadingly at
the cookies.

**`x-goog-api-key`** — public by construction: it ships in Meet's own JavaScript
bundle and identifies the application, not the user. Not a credential.

It is still written elided here, because a literal `AIzaSy…` string in a public
repository is treated as a leaked secret by GitHub's scanner, Google's, and every
SAST tool in between, whatever it actually is. Read the current 39-character
value off any `$rpc` request in devtools.

**`x-goog-meeting-bot-info`** — present on **4 of 373** captured RPCs, all
`UpdateMeetingDevice`, and even that call omitted it 18 times out of 22. Every
critical-path request had zero. Its prefix decodes to plain protobuf, field 3,
2446 bytes — not a signature. **Omit it.**

**`x-goog-meeting-rtcclient`** — decodes to
`{1: 1, 2: 426, 3: 1, 4: 1, 5: 8, 7: 1}`. The identical bytes also appear inside
every `dcrpc` request body at field `7` of the client identity block.

**`x-goog-meeting-debugid`** — capture it from the browser's own
`CreateMediaSession`, **alongside the token that request was sent with**. The two
belong together.

---

## Response headers

| Header | Meaning |
| --- | --- |
| `X-Goog-Meeting-Token` | a **newer** meeting token. Adopt it only if its embedded epoch is **strictly greater** than the one held, and only into the **state** cache. |

---

## The two token caches

| Scope | Used by | Seeded from | Rotates | Observed lengths |
| --- | --- | --- | --- | --- |
| **State** | everything except `…MediaSessionService` | page bootstrap | **yes** | 282, 304 |
| **Media** | `CreateMediaSession` | page bootstrap | **no** | 261 |

```
scope(service) = MEDIA  if service ends with "MediaSessionService"
                 STATE  otherwise
```

Merging them is refused as `400 invalid argument`, indistinguishable from a
malformed body. **Diagnostic:** equal cached token lengths in a log mean the
scoping has collapsed.

---

## Authorization header construction

```
origin    = "https://meet.google.com"
ts        = unix seconds
sign(c)   = sha1_hex(f"{ts} {c} {origin}")

authorization =
    f"SAPISIDHASH {ts}_{sign(SAPISID)}"           + " " +
    f"SAPISID1PHASH {ts}_{sign(__Secure-1PAPISID)}" + " " +
    f"SAPISID3PHASH {ts}_{sign(__Secure-3PAPISID)}"
```

**Length is exactly 195 characters** at a 10-digit timestamp. Pin it in a test —
that length is the evidence the whole scheme was inferred from.

All three cookies must be present. A partially signed request is not a degraded
request; it is rejected.

---

## Status codes

| HTTP | Message | Cause |
| --- | --- | --- |
| `200` | — | accepted |
| `400` | *This RPC requires a meeting token but nothing was provided.* | header absent |
| `400` | *Request contains an invalid argument.* | malformed body **or wrong token scope** |
| `401` | *missing required authentication credential* | `origin` not sent |
| `403` | *The caller does not have permission* | another participant's device |
| `404` | — | wrong service prefix |

The `400` body is a `google.rpc.Status`, not text. Decode it — it can carry
`BadRequest.FieldViolation` entries naming the offending field.
→ [docs/01-transport.md](../docs/01-transport.md#reading-a-refusal)
