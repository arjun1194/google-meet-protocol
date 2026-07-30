# Reference · Field maps

Every decoded message in one place, in `field.path` notation.

Notation: `1.3.2.4` means field 4 of the message at field 2 of the message at
field 3 of the message at field 1. `repeated` means the field appears more than
once at that position.

---

## Contents

- [`CreateMediaSession` request](#createmediasession-request)
- [`CreateMediaSession` response](#createmediasession-response)
- [`SyncMeetingSpaceCollections` request](#syncmeetingspacecollections-request)
- [`UpdateMeetingDevice` request](#updatemeetingdevice-request)
- [`x-goog-meeting-identifier`](#x-goog-meeting-identifier)
- [`x-goog-meeting-rtcclient`](#x-goog-meeting-rtcclient)
- [`media-director` frames](#media-director-frames)
- [`dcrpc` frames](#dcrpc-frames)
- [`meet_messages` chat](#meet_messages-chat)
- [`google.rpc.Status`](#googlerpcstatus)

---

## `CreateMediaSession` request

~2893 bytes decompressed across twenty captures.

```
1                          message   request
1.1              string    "mediasessions/<28 chars>"     client-minted
1.3                        message   transport block
1.3.1                      message   DTLS
1.3.1.1.1        string    "sha-256"
1.3.1.1.2        string    <fingerprint>
1.3.1.2          varint    3                              setup role
1.3.1.3          varint    3                              setup role, again
1.3.2                      message   ICE
1.3.2.3          string    <ice-ufrag>                    4 chars
1.3.2.4          string    <ice-pwd>                      24 chars
1.3.3            repeated  message   codec — see below
1.3.4            repeated  message   header extension — see below
1.3.5.2          varint    6500                           bandwidth hint ❓
1.3.5.3          varint    500                            bandwidth hint ❓
1.3.7.1          varint    4                              ❓
1.3.8.1          varint    1                              ❓
1.3.8.2          varint    2                              ❓
1.3.17           repeated  message   data channel — see below
1.3.19.1         varint    2                              ❓
1.3.36           varint    2                              ❓
1.3.42           varint    1                              ❓
1.3.45.1         varint    4                              hw decode count ❓
1.3.45.2         varint    4                              ❓
1.3.45.4         varint    10                             ❓
1.3.45.6         string    <GPU renderer>                 ← replace; names the machine
1.3.46.1.1..16   varint    capability flags               5 and 6 = 2073600 (1920×1080)
1.3.50           varint    1                              ❓
```

Field `2` of the root is unused in every capture. **Nothing here names the
meeting** — the space travels in `x-goog-meeting-identifier`.

### `1.3.3` — codec

```
1   varint    payload type
2   string    name              "opus", "VP8", "rtx" — bare, no MIME prefix
3   varint    MEDIA KIND        1 = audio, 2 = video   ⚠ not a clock rate
4   varint    clock rate
5   varint    channels
6   repeated  { 1: key, 2: value }    bare fmtp params use key ""
7   varint    3                       secondary block only
8   message   { 1: mode }             video only; H265 adds { 5: "L1T1" }
9   message   { 1: layered }          video only
```

### `1.3.4` — header extension

```
1   varint    id
2   string    uri
3   varint    kind    1 = audio, 2 = video
4   varint    flag    1, except 2 on corruption-detection
```

### `1.3.17` — data channel

```
1   string    label
2   varint    0
4   varint    1
7   bytes     02 01     ⚠ opaque bytes, NOT a message. 01 for meet-p2p-signaling.
```

---

## `CreateMediaSession` response

~5222 bytes.

```
2.3    repeated  candidate  { 1: component, 2: address, 3: port, 4: priority }
2.4    repeated  codec      { 1: payload type, 2: name }
2.12   repeated  channel    { 1: label, 3: SCTP stream id }
```

One captured answer: **4 candidates, 22 codecs.**

`2.12` is the reason to parse the response at all. Stream ids observed 111–159,
odd only, and they differ per session.

⚠️ `2.4` names codecs by payload type and name only. Clock rate, channels and
fmtp must be carried over from **your** offer, matched by payload type.

---

## `SyncMeetingSpaceCollections` request

The entire body:

```
1   string   "spaces/<id>"
```

Nineteen bytes for a twelve-character space id. In the steady-state long-poll:
75 bytes in, 72 bytes out, every 20 seconds.

---

## `UpdateMeetingDevice` request

Field-masked. Two observed shapes.

### Audio state

```
1.1      string    "spaces/<id>/devices/<n>"
1.14.2   varint    <state>
2.1      string    "audio_state"
```

### Announcing the media session — load-bearing

```
1.1      string    "spaces/<space>/devices/<n>"
1.4      varint    1
1.22     string    "<session id, BARE — no mediasessions/ prefix>"
1.25     varint    1
1.38     varint    1
2.1      repeated string  "cloud_session_id"
                          "join_state"
                          "media_capture_type"
                          "participation_mode"
```

🧩 Mapping mask entries to field numbers: `22` is `cloud_session_id`; the rest
follow the mask order, unconfirmed.

**Permission probe:** `403` for another participant's device, `200` for your own.
The roster does not say which is yours — trying each is self-correcting.

---

## `x-goog-meeting-identifier`

Standard base64 of:

```
1   varint    3            ❓ constant in every capture
2   string    <space id>   ⚠ the SPACE ID, not the meeting code
```

Example: `CAMSDEVYQU1QTEVzcGFjZQ==` → `spaces/EXAMPLEspace`

⚠️ A meeting code with dashes and a space id are both twelve characters, so both
encode to a 24-character header. **Pin the bytes, not the length.**

This header is also the only place the space id appears **before** a join
completes, which makes a token-only browser visit possible.

---

## `x-goog-meeting-rtcclient`

Base64 `CAEQqgMYASABKAg4AQ==` →

```
1: 1     2: 426     3: 1     4: 1     5: 8     7: 1
```

A version and capability flags, byte-identical in every capture. **The same six
fields also appear inside every `dcrpc` request body at field `7`.**

---

## `media-director` frames

### Frame

```
1   bytes    payload             plain
2   bytes    gzip(payload)       compressed
```

### Payload

```
1   message   request
2   message   acknowledgement
```

### Client configuration — `payload.1`

```
1        varint    sequence
2.1.1    varint    1280        }  size A
2.1.2    varint    720         }
2.3.1    varint    1920        }  size B  ← field 3, not 2
2.3.2    varint    1080        }
```

### Server allocation — `payload.1`

```
1        string    name              "video"
2        varint    version
3.1      repeated  message   group
3.1.1    repeated  message   stream
```

### Stream

```
1.1.1    fixed32   SSRC          ⚠ FIXED32. The only one in the protocol.
1.2.1    varint    width
1.2.2    varint    height
1.3      varint    frames per second
1.4      varint    bits per second
1.5      string    scalability   "L1T2" / "L1T3"
1.6      varint    payload type
2        varint    2
```

### Client acknowledgement — `payload.2`

```
1   string   name
2   varint   version
3   varint   1
```

---

## `dcrpc` frames

### Frame

Identical to `media-director`: `{1: msg}` plain, `{2: gzip(msg)}` compressed.

### Message

```
1   varint    sequence
<n> message   payload — THE FIELD NUMBER IS THE METHOD
```

| Field | Direction | Method |
| --- | --- | --- |
| 2 | ? | ❓ |
| 3 | → | open session |
| 3 | ← | acknowledge open |
| 4 | → | register the client's own streams 🧩 |
| 5 | ← | announce participant streams |

### Open request — field `3`

```
1.1.1    varint    53                  client kind, constant
1.2.1    string    <build id>          e.g. boq_hlaneEXAMPLE0
1.2.4    string    <debug id>          e.g. boq_hlane_<random>
1.4      string    "en"                locale
1.7      message   { 1:1, 2:426, 3:1, 4:1, 5:8, 7:1 }   ← same as the rtcclient header
2        varint    <kind>              2, then 1 — sent twice
3        string    <session id, BARE>
5        message   {}                  empty
```

### Open reply

```
1        varint    sequence
3.1.1    varint    1
3.3.1    varint    <server time>
4        message   {}
```

### Participant stream announcement

Found at varying depths — **walk, do not index**.

```
4   string    label            "audio", "video", or a bare SSRC as text
6   string    "spaces/<space>/devices/<n>"     ⚠ must contain "/devices/"
8.1 repeated varint  ssrc                       ⚠ reject entries with none
```

### Mute request 👁

```
1.1        varint   sequence
1.2.4.1    varint
1.2.4.2    varint
1.2.4.3    string   session id
1.2.4.4    string   "audio"          ← 🧩 "video" for camera
1.2.4.5    string   opaque token     ⚠ session-scoped, no known derivation
1.2.4.10.1 varint   <state>
```

Send `08 08`, receive `08 08`. The response confirms against
`spaces/<id>/devices/<n>`.

---

## `meet_messages` chat

```
1.1.1        varint   1
1.1.3.1.2.3  varint   <epoch ms>
1.1.3.1.2.5.1 string  <text>
1.1.3.1.2.6  varint   1
```

Flattened, as the captured bytes read:

```
1 { 1 { 1: 1, 3 { 1 { 2 { 3: <epoch ms>, 5 { 1: <text> }, 6: 1 } } } } }
```

⚠️ **Note the doubled field 1 at the top.** The bytes open `0a 31 0a 2f` — two
nested length-delimited wrappers before any content. Encoding one produces a
message two bytes short that otherwise looks entirely correct.

---

## `google.rpc.Status`

The body of any failed `$rpc` call.

```
1   varint    code
2   string    message          "Request contains an invalid argument."
3   Any       details
3.1 string    "type.googleapis.com/google.rpc.BadRequest"
3.2 BadRequest
3.2.1 repeated FieldViolation
        1  string  field         "media_session.ice.ufrag"
        2  string  description   "must not be empty"
```

Walk rather than index — the details are `Any`-wrapped and their depth depends on
the detail type. Bound the recursion to ~6 levels, and require the field to look
like a path and the description to look like a sentence, or every adjacent pair
of strings in the reply reads as a violation.

---

## Resource name shapes

```
spaces/<id>                       the meeting
spaces/<id>/devices/<n>           a participant's device
spaces/<id>/messages/<id>         a chat message
spaces/<id>/handRaises/<n>        a raised hand
mediasessions/<28 chars>          a media session
```

### The same session id, three ways

| Where | Form |
| --- | --- |
| `CreateMediaSession` `1.1` | `mediasessions/<28 chars>` |
| `UpdateMeetingDevice` `1.22` | `<28 chars>` — bare |
| `dcrpc` open `3` | `<28 chars>` — bare |
