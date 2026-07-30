# Reference · Data channels

Thirteen declared in the request. Nineteen present on the connection.

Behaviour and rules: [docs/06-data-channels.md](../docs/06-data-channels.md).

---

## Declared in `CreateMediaSession` — 13, in order

Repeated at `1.3.17`:

```
1: "<label>"
2: 0
4: 1
7: <opaque bytes>
```

| # | Label | Field 7 | Documented |
| --- | --- | --- | --- |
| 1 | `reactions` | `02 01` | ❓ |
| 2 | `copresent` | `02 01` | ❓ |
| 3 | **`media-director`** | `02 01` | [docs/07](../docs/07-media-director.md) |
| 4 | **`dcrpc`** | `02 01` | [docs/08](../docs/08-dcrpc.md) |
| 5 | `audio-mesh` | `02 01` | ❓ |
| 6 | `video-processing` | `02 01` | ❓ |
| 7 | `media-session` | `02 01` | ❓ |
| 8 | `meeting-agent` | `02 01` | ❓ |
| 9 | `agenda` | `02 01` | ❓ |
| 10 | `cohort` | `02 01` | ❓ |
| 11 | **`meet_messages`** | `02 01` | [docs/09](../docs/09-collections-and-actions.md) |
| 12 | `coannotations` | `02 01` | ❓ |
| 13 | `meet-p2p-signaling` | **`01`** | ❓ |

⚠️ **Twelve carry `02 01`; `meet-p2p-signaling` carries `01` alone.** A catalogue
that writes one value everywhere loses that difference without failing anything.

⚠️ **Field 7 is opaque bytes, not a nested message.** `02 01` read as a message
describes field number **zero**, which protobuf does not permit — so a decoder
produces nothing and a re-encoder silently writes `{2: 1}` instead. Both encode
cleanly; only the server can tell them apart. Carry the bytes verbatim.

⚠️ **Losing `media-director` costs the session its stream control**, and the
symptom is a connection that carries nothing rather than an error.

---

## Present on the connection — 19

Traffic counts from one 90-second capture of a real client with panels open.

| Label | Send / Recv | Declared? | Role |
| --- | --- | --- | --- |
| `dcrpc` | 13 / 13 | ✔ | generic sequenced RPC; **the busiest**; SSRC → device |
| `media-session` | 4 / 4, ~5 KB | ✔ | media negotiation |
| `media-director` | 3 / 3 | ✔ | **stream allocation control** |
| `audioprocessor` | 0 / 7 | — | audio state, tiny frames 🧩 |
| `meet_messages` | 1 / 1 | ✔ | **outbound chat** |
| `collections` | 0 / 2 | — **remote** | roster, chat, hand raises, meeting state |
| `captions` | 0 / 1 | — | present, dormant; language list observed |
| `reactions` | 0 / 0 | ✔ | present, unused in capture |
| `copresent` | 0 / 0 | ✔ | presenting |
| `coannotations` | 0 / 0 | ✔ | annotation |
| `meeting-agent` | low | ✔ | unclassified |
| `agenda` | low | ✔ | unclassified |
| `cohort` | low | ✔ | unclassified |
| `audio-mesh` | low | ✔ | unclassified |
| `video-processing` | low | ✔ | unclassified |
| `p2p` | low | — | unclassified |
| `meet-p2p-signaling` | low | ✔ | unclassified |
| `s11y-sync` | low | — | "s11y" = stability 🧩 |
| `ignored` | low | — | literally named `ignored` |

**Eighteen are created by the client; only `collections` arrives remotely** —
which is why, when instrumentation patched instances instead of prototypes, it
was the *only* channel that showed up.

The six not in the declared list (`audioprocessor`, `collections`, `captions`,
`p2p`, `s11y-sync`, `ignored`) appear in the response's stream table or arrive
through `ondatachannel`.

---

## Stream assignment — response field `2.12`

```
2.12  repeated  { 1: label, 3: SCTP stream id }
```

👁 One captured session:

| Label | Stream id |
| --- | --- |
| `dcrpc` | 113 |
| `reactions` | 115 |
| `media-director` | 131 |
| `meet_messages` | 137 |

Observed range: **111–159, odd numbers only** — consistent with Meet holding the
DTLS server role.

> **The ids are the server's to choose and differ per session. They cannot be
> hardcoded.**

---

## The claiming rules

Every one of these produces silence or a misleading error when broken.
→ [docs/06](../docs/06-data-channels.md)

1. **Claim every assigned id up front** as a pre-negotiated channel
   (`negotiated: true`, `id: <stream id>`). Meet writes binary onto them with no
   DCEP open message. An unclaimed stream produces
   `Unknown PayloadProtocolIdentifier 53`.
2. **Poll every claimed channel**, not just the ones you act on. An unread
   channel has its receiver closed underneath it, and the stack then reports
   `Disconnected` for deliveries on **every** channel.
3. **Keep every handle alive.** Dropping one does the same thing.
4. **Still create one channel the ordinary way**, so the description carries an
   `m=application` section and the SCTP association comes up at all. Meet also
   replies on that channel rather than on the pre-negotiated stream it assigned.
5. **Send on ready *and* on open.** A channel Meet hands over is already open —
   its open event has been and gone. A locally-created one is not ready
   immediately, and sending early fails with `data channel not existed`, a
   *warning* the caller never sees.

---

## Framing, shared by `media-director` and `dcrpc`

```
frame = { 1: payload }              plain
      | { 2: gzip(payload) }        compressed — Meet's choice, no way to ask
```

Both must be handled on receive. Sending plain is accepted.

`collections` messages are gzipped protobuf, 30–800 bytes.

---

## Subscription

**Meet pushes only what a client subscribes to.**

| Client state | `collections` traffic in 90s |
| --- | --- |
| no panels open | **1** keepalive |
| People + Chat + Captions open | **17** messages |

Same bot, same meeting, same actions. ❓ The subscription message itself has not
been isolated. → [docs/14](../docs/14-open-questions.md)
