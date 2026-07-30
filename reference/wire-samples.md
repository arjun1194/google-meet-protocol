# Reference · Wire samples

Captured frames, annotated. These are the ground truth every field map in this
repository was derived from, and the fixtures to test an encoder against.

**Test against bytes, not shapes.** A length assertion cannot distinguish a space
id from a meeting code — both are twelve characters. A test written from your own
encoder passes while the encoder is wrong.

---

## What has been substituted, and what has not

> **Every tag, wire type, length prefix, offset, and constant below is exactly as
> captured. Session-specific identifiers are not.**

Four strings were replaced with **same-length synthetic placeholders**, so that
every length prefix still agrees with its payload and every byte offset in the
annotations still lands where it says it does:

| Position | Placeholder | Length |
| --- | --- | --- |
| build id | `boq_hlaneEXAMPLE0` | 17 |
| debug id | `boq_hlane_EXAMPLE0000` | 21 |
| media session id | `EXAMPLEsession` + the real 14-char constant tail | 28 |
| space id | `EXAMPLEspace` | 12 |

Anything reading `EXAMPLE…` is synthetic. Everything else — including the
constant tail `oKAAiKYigCIAQQ`, the SSRCs, the resolutions, the scalability
modes, the payload types, and the chat body — is verbatim.

**Why.** These came from real meetings on a real account. They are expired and
grant nothing, but they are still identifiers belonging to specific sessions, and
a public repository is a poor place to keep other people's session data on the
argument that it has probably rotated by now. The teaching value is entirely in
the structure, and the structure is untouched.

**What this costs you:** nothing for writing an encoder, since the placeholders
are the right length and shape. If you want to verify a *decoder* against real
traffic, capture your own — [the methodology is
documented](../docs/13-capture-methodology.md).

Nothing here is or ever was a credential: no cookie, no authorization header, no
meeting token, no API key.

---

## 1 · `media-director` — client configuration

✅ The opening frame. **Without this, a connected session receives nothing.**

```
0a 16 0a 14 08 01 12 10 0a 06 08 80 0a 10 d0 05 1a 06 08 80 0f 10 b8 08
```

24 bytes. Decoded:

```
0a 16                          frame field 1 (plain), 22 bytes
   0a 14                       payload field 1 (request), 20 bytes
      08 01                    1: sequence = 1
      12 10                    2: sizes block, 16 bytes
         0a 06                 2.1: size A, 6 bytes
            08 80 0a           1: width  = 1280
            10 d0 05           2: height = 720
         1a 06                 2.3: size B, 6 bytes    ← field 3, not 2
            08 80 0f           1: width  = 1920
            10 b8 08           2: height = 1080
```

---

## 2 · `media-director` — server allocation

✅ Meet's second video allocation, plain (uncompressed), two streams.

```
0a 55 0a 53 0a 05 76 69 64 65 6f 10 02 1a 48 0a 46 0a 21 0a 1d 0a 05 0d
b9 5b 0c e6 12 06 08 c0 07 10 9c 04 18 1e 20 c0 84 3d 2a 04 4c 31 54 33
30 62 10 02 0a 21 0a 1d 0a 05 0d 1f 15 7b 5c 12 06 08 c0 07 10 9c 04 18
1e 20 e0 c6 5b 2a 04 4c 31 54 32 30 60 10 02
```

87 bytes. Decoded:

```
0a 55                          frame field 1 (plain), 85 bytes
   0a 53                       payload field 1 (request), 83 bytes
      0a 05 "video"            1: name = "video"
      10 02                    2: version = 2
      1a 48                    3: streams holder, 72 bytes
         0a 46                 3.1: group, 70 bytes

            0a 21              stream #1, 33 bytes
               0a 1d           description, 29 bytes
                  0a 05        ssrc holder, 5 bytes
                     0d b9 5b 0c e6      ⚠ FIXED32 (wire type 5)
                                          little-endian → 3859569593
                  12 06        size
                     08 c0 07             width  = 960
                     10 9c 04             height = 540
                  18 1e        fps = 30
                  20 c0 84 3d  bitrate = 1000000
                  2a 04 "L1T3" scalability
                  30 62        payload type = 98   (VP9)
               10 02           2: 2

            0a 21              stream #2, 33 bytes
               0a 1d
                  0a 05
                     0d 1f 15 7b 5c      → 1551570207
                  12 06 08 c0 07 10 9c 04    960 × 540
                  18 1e                       fps = 30
                  20 e0 c6 5b                 bitrate = 1500000
                  2a 04 "L1T2"
                  30 60                       payload type = 96   (VP8)
               10 02
```

> ⚠️ **`0d` is the tag for field 1, wire type 5 — `fixed32`.** The only one in
> this protocol. A generic varint walker reads `b9 5b 0c e6` as a varint and
> produces a plausible number that matches no arriving packet. The session then
> looks silently empty while RTP flows perfectly.

---

## 3 · `media-director` — client acknowledgement

✅ Acknowledging allocation 2. Meet re-sends an allocation until it gets this.

```
0a 0d 12 0b 0a 05 76 69 64 65 6f 10 02 18 01
```

15 bytes. Decoded:

```
0a 0d                          frame field 1 (plain), 13 bytes
   12 0b                       payload field 2 (acknowledgement), 11 bytes
      0a 05 "video"            1: name
      10 02                    2: version = 2
      18 01                    3: 1
```

---

## 4 · `media-director` — Meet's acknowledgement of our configuration

✅ Received live by a from-scratch client, byte-identical to the browser's.

```
0a 08 12 06 08 01 12 00 1a 00
```

```
0a 08                          frame field 1, 8 bytes
   12 06                       payload field 2 (acknowledgement), 6 bytes
      08 01                    1: 1
      12 00                    2: {}
      1a 00                    3: {}
```

Worth having as a fixture for a reason beyond correctness: **an acknowledgement
carries no streams**, so a logger that only prints allocations makes a replying
channel look identical to a silent one. This frame went unseen for several runs.

---

## 5 · `dcrpc` — client open request

✅ The first frame of a real join. Sequence 1, kind 2.

```
0a 6b 08 01 1a 67 0a 43 0a 02 08 35 12 2a 0a 11 62 6f 71 5f 68 6c 61 6e
65 45 58 41 4d 50 4c 45 30 22 15 62 6f 71 5f 68 6c 61 6e 65 5f 45 58 41
4d 50 4c 45 30 30 30 30 22 02 65 6e 3a 0d 08 01 10 aa 03 18 01 20 01 28
08 38 01 10 02 1a 1c 45 58 41 4d 50 4c 45 73 65 73 73 69 6f 6e 6f 4b 41
41 69 4b 59 69 67 43 49 41 51 51 2a 00
```

109 bytes. Decoded:

```
0a 6b                                frame field 1 (plain), 107 bytes
   08 01                             1: sequence = 1
   1a 67                             3: open request, 103 bytes    ← field number IS the method

      0a 43                          1: client identity, 67 bytes
         0a 02 08 35                 1.1: { 1: 53 }        client kind, constant
         12 2a                       1.2: names, 42 bytes
            0a 11 "boq_hlaneEXAMPLE0"      1: build id
            22 15 "boq_hlane_EXAMPLE0000"  4: debug id
         22 02 "en"                  1.4: locale
         3a 0d                       1.7: client descriptor, 13 bytes
            08 01                       1: 1
            10 aa 03                    2: 426
            18 01                       3: 1
            20 01                       4: 1
            28 08                       5: 8
            38 01                       7: 1

      10 02                          2: kind = 2      (sent again with kind = 1)
      1a 1c "EXAMPLEsessionoKAAiKYigCIAQQ"
                                     3: session id — ⚠ NO mediasessions/ prefix
      2a 00                          5: {}            empty
```

Two traps visible in this one frame:

- **`3a 0d 08 01 10 aa 03 …` is byte-for-byte the decoding of the
  `x-goog-meeting-rtcclient` header** (`CAEQqgMYASABKAg4AQ==`). The header and
  the body carry identical bytes; getting one right and the other wrong produces
  no distinct symptom.
- **The session id here is bare.** `CreateMediaSession` names the same id with a
  `mediasessions/` prefix. Sending the prefixed form here names a session Meet
  does not recognise.

Note the session id ends `oKAAiKYigCIAQQ` — the base64 of the 11-byte constant
tail every session id shares.

---

## 6 · `meet_messages` — a chat message

✅ 51 bytes for a 26-character body, reproduced byte for byte by a from-scratch
encoder.

Text `"probe message from the bot"`, timestamp `1785104796458`.

```
0a 31 0a 2f 08 01 1a 2b 0a 29 12 27 18 aa 96 a6 84 fa 33 2a 1c 0a 1a 70
72 6f 62 65 20 6d 65 73 73 61 67 65 20 66 72 6f 6d 20 74 68 65 20 62 6f
74 30 01
```

Decoded:

```
0a 31                          field 1, 49 bytes      ⚠ wrapper 1
   0a 2f                       field 1, 47 bytes      ⚠ wrapper 2
      08 01                    1: 1
      1a 2b                    3: 43 bytes
         0a 29                 3.1: 41 bytes
            12 27              3.1.2: 39 bytes
               18 aa 96 a6 84 fa 33        3: 1785104796458   (epoch ms)
               2a 1c                       5: 28 bytes
                  0a 1a "probe message from the bot"
                                           5.1: the text
               30 01                       6: 1
```

> ⚠️ **The doubled field 1 at the top.** The bytes open `0a 31 0a 2f` — two
> nested length-delimited wrappers before any content. Encoding one of them
> produces a message **two bytes short** that parses cleanly and is otherwise
> entirely correct.

---

## 7 · `x-goog-meeting-identifier`

Header value, standard base64:

```
CAMSDEVYQU1QTEVzcGFjZQ==
```

Decoded:

```
08 03                    1: 3               ❓ constant in every capture
12 0c "EXAMPLEspace"     2: space id
```

→ `spaces/EXAMPLEspace`

> ⚠️ Twelve characters. A meeting code with dashes (`dhi-repi-xsa`) is also
> twelve characters, and also produces a 24-character header value. **The length
> cannot tell them apart.** Assert the prefix.

---

## 8 · `x-goog-meeting-rtcclient`

```
CAEQqgMYASABKAg4AQ==
```

Decoded:

```
08 01           1: 1
10 aa 03        2: 426
18 01           3: 1
20 01           4: 1
28 08           5: 8
38 01           7: 1
```

Byte-identical in every captured request, and repeated inside every `dcrpc`
request body at field `7`.

---

## 9 · Media session id — the constant tail

Eighteen captures agree on the **last eleven of twenty-one bytes**:

```
0a 0a 00 08 8a 62 28 02 20 04 10
```

Full structure:

```
mediasessions/ EXAMPLEsession oKAAiKYigCIAQQ
               └─10 random───┘ └─11 constant─┘
                 base64url of 21 bytes = 28 chars, no padding
```

❓ **Not decoded.** It does not parse as protobuf — and it never varies. A fully
random id of the correct length and alphabet is refused exactly as loudly as a
malformed one.

Generation:

```
bytes = random(10) ++ [0a 0a 00 08 8a 62 28 02 20 04 10]
name  = "mediasessions/" + base64url_nopad(bytes)
```

**URL-safe base64.** Standard base64 eventually emits a `/` into the middle,
turning a two-segment resource name into three — roughly one session in sixty,
and it looks like a server fault.

A correctly generated name always ends `oKAAiKYigCIAQQ`. Assert that.

---

## Assertions worth writing

Derived from what actually broke:

| Assert | Because |
| --- | --- |
| the `dcrpc` open frame, byte for byte | the client descriptor and the bare session id are both silently wrong-able |
| the `media-director` configuration, byte for byte | sizes at fields 1 and **3**, not 1 and 2 |
| the `media-director` acknowledgement, byte for byte | trivial message, load-bearing |
| a 51-byte chat message, byte for byte | the doubled wrapper |
| the identifier header's **prefix**, not its length | a meeting code passes a length check |
| the authorization header length is **195** | it is the evidence the scheme was inferred from |
| a generated session id ends `oKAAiKYigCIAQQ` | random ids of the right shape are refused |
| a generated session id contains exactly one `/` | standard base64 breaks the resource name |
| exactly 13 channel declarations, one carrying `01` | a uniform catalogue loses the odd one |
| exactly 20 extensions: 5 audio, 15 video, last flag = 2 | off-by-one puts a video extension on audio |
| 22 codecs, unique payload types, `rtx` never crossing blocks | a stack's defaults are not Meet's list |
| SSRC decodes with `u32::from_le_bytes` | it is a `fixed32` |
