# Contributing

Corrections are the most valuable thing you can send. Several conclusions in the
history of this document were confidently wrong for weeks.

---

## The one rule

**Every claim carries its evidence marker, and you may not promote an inference
to a fact without saying what you ran.**

| Marker | Means | You may use it when |
| --- | --- | --- |
| ✅ **Proven** | a from-scratch client sent or parsed this and the server behaved as described | you rebuilt the message and it worked |
| 👁 **Observed** | seen in a capture of the real client | you saw it on the wire |
| 🧩 **Inferred** | consistent with the evidence | you reasoned it out and did not test it |
| ❓ **Unknown** | named so the gap is visible | you know it exists and nothing more |

The gap between 👁 and ✅ is the whole point of this scheme. *"The browser sends
these bytes"* and *"I sent these bytes and the server accepted them"* are
different claims, and the second is worth roughly ten of the first.

---

## What a good change looks like

### Correcting a fact

Say **what you ran** and **what happened**. A diff that changes a field number
with no accompanying evidence cannot be evaluated, and a wrong correction is
worse than the original error because it looks freshly verified.

> `1.3.42` is `1`, not `2`. Twelve captures on 2026-08-14, Chrome 152, two
> accounts. A request sending `2` returns `400 invalid argument`; sending `1`
> returns `200`.

### Adding a finding

Include:

1. **What you observed**, in terms of bytes or status codes
2. **How many times**, and over how long — one capture is an anecdote
3. **What you tried that did not work**, if anything. This is often the most
   valuable part and it is the part people delete.
4. **Which marker it earns**, and why

### Answering an open question

[docs/14-open-questions.md](docs/14-open-questions.md) lists what is unknown,
with a suggested experiment for each. Answering one is the single highest-value
contribution available.

Move the entry out of that file and into the relevant chapter with its new
marker. Do not leave a resolved question sitting in the open list — the list is
only useful if it is honest.

### Reporting drift

If Google changed something, say **when you checked** and **what version of
Chrome**. Add a dated note rather than silently overwriting: knowing that a field
meant one thing in July 2026 and another in November is more useful than knowing
only the current value.

---

## What not to do

**Do not add code that joins meetings.** This repository is information only.
That boundary is deliberate: it keeps the material clearly documentary, and it
keeps the ethical question — *should I automate this?* — with the reader rather
than being answered for them by a working client sitting in the repo.

**Do not commit captures.** They contain live session cookies, meeting tokens,
and the names, avatars, and speech of people who did not consent to being
recorded. Reduce them to structure and delete the raw material.

**Do not commit real credentials of any kind.** Cookies, `authorization`
headers, meeting tokens. The `x-goog-api-key` value is fine — it ships in Meet's
own bundle and identifies the application, not a user.

**Do not paste Google source.** Not minified bundles, not decompiled WASM, not
protobuf descriptors extracted from a bundle. Describe behaviour in your own
words. Reading Meet's compiled JavaScript to *find* a field-number constant is
reasonable; reproducing the code here is not.

**Do not remove a failure-mode entry because it seems obvious.** Every entry in
[docs/12-failure-modes.md](docs/12-failure-modes.md) cost someone real time, and
each one looks obvious once you know it.

---

## Style

The existing documents have a voice. Roughly:

- **Say what is true, then what it costs you if you get it wrong.** A field map
  without its trap is half a document.
- **Prefer the concrete.** "18 of 19 channels were invisible" beats "the
  instrumentation was incomplete."
- **Mark the amber.** Anything that fails *silently* — no error, no log, no
  exception — gets called out explicitly. Those are the expensive ones.
- **Do not bury a correction.** If something here is wrong, the correction goes
  where the wrong thing was, not in a footnote.

Diagrams are [Mermaid](https://mermaid.js.org), which GitHub renders natively.
The colour convention is in [diagrams/README.md](diagrams/README.md) — amber
means *this fails without telling you*, and it means that consistently.

---

## Ethics, briefly

Capture only meetings you are a participant of, with an account you own.
Everyone else in a capture is a person whose data you are now holding. Automating
Meet is against Google's Terms of Service, and accounts running automation get
flagged.

Full position: [README](README.md#legal-and-ethical-position) ·
[docs/13-capture-methodology.md](docs/13-capture-methodology.md#ethics-of-capturing)

---

## Licence

Contributions are accepted under [CC BY 4.0](LICENSE), the same terms as the rest
of the repository.
