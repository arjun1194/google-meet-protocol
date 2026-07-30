# Diagrams

Every diagram in this repository as reusable [Mermaid](https://mermaid.js.org)
source. GitHub renders `.mmd` fenced as ```` ```mermaid ```` blocks inline, so
these files exist for **reuse** — dropping a diagram into your own docs, a
slide, or a design review.

Render them with the [Mermaid Live Editor](https://mermaid.live), the
`@mermaid-js/mermaid-cli` package, or any Markdown renderer with Mermaid
support.

---

## Index

| File | Shows | Appears in |
| --- | --- | --- |
| [protocol-overview.mmd](protocol-overview.mmd) | the whole protocol in one picture | [README](../README.md) · [00](../docs/00-orientation.md) |
| [layer-stack.mmd](layer-stack.mmd) | three protocols stacked | [00](../docs/00-orientation.md) |
| [minimal-client.mmd](minimal-client.mmd) | shortest path to attributable audio | [00](../docs/00-orientation.md) |
| [session-lifecycle.mmd](session-lifecycle.mmd) | connected ≠ joined ≠ receiving | [00](../docs/00-orientation.md) |
| [join-sequence.mmd](join-sequence.mmd) | the full join, end to end | [03](../docs/03-join-sequence.md) |
| [token-scopes.mmd](token-scopes.mmd) | the two token caches | [02](../docs/02-authentication.md) |
| [data-channel-negotiation.mmd](data-channel-negotiation.mmd) | channels negotiated over HTTP, no DCEP | [06](../docs/06-data-channels.md) |
| [media-director-exchange.mmd](media-director-exchange.mmd) | asking to be sent anything | [07](../docs/07-media-director.md) |
| [attribution-chain.mmd](attribution-chain.mmd) | SSRC → device → person | [08](../docs/08-dcrpc.md) |
| [collections-resources.mmd](collections-resources.mmd) | the resource model | [09](../docs/09-collections-and-actions.md) |
| [action-transports.mmd](action-transports.mmd) | actions split across three transports | [09](../docs/09-collections-and-actions.md) |
| [capability-dependencies.mmd](capability-dependencies.mmd) | what unblocks what | [11](../docs/11-capabilities.md) |
| [invalid-argument-decision-tree.mmd](invalid-argument-decision-tree.mmd) | diagnosing `400 invalid argument` | [05](../docs/05-create-media-session.md) |

Diagrams that appear only inline in a chapter — the failure-mode flow in
[12](../docs/12-failure-modes.md), the capability mindmap in
[11](../docs/11-capabilities.md), the SDP section layout in
[10](../docs/10-media-plane.md), the codec pairing in
[reference/codecs.md](../reference/codecs.md) — are not duplicated here. Copy
them from the source Markdown.

---

## Colour convention

Used consistently across every diagram, so a colour means the same thing
everywhere:

| Colour | Meaning |
| --- | --- |
| 🟦 blue `#e8f4fd` | control plane — HTTPS RPCs |
| 🟩 green `#eaf6ec` | media plane, and successful end states |
| 🟪 purple `#f4ecf7` | data channels / application layer |
| 🟧 amber `#fff4e5` | **the thing that will bite you** — silent failure, no error |
| 🟥 red `#fdeaea` | an actual failure state |
| ⬜ grey `#f5f5f5` | ignorable |

Amber is load-bearing. Every amber node in this repository marks a step whose
omission produces **no error message** — a connected-but-silent session, an
unclaimed stream, a token cache that collapsed into one.

---

## Reusing these

CC BY 4.0, like everything else here. Credit is appreciated, and a link back is
more useful to your readers than to us — this is a moving target, and the
version they find will be newer than the picture you copied.
