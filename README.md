# Web NFC for identity documents (`NFCIsoDepReader`)

**Let web pages read the chip in passports and national ID cards, safely.**

Electronic passports and most national ID cards carry a chip (ICAO Doc 9303) with the holder's
data, signed by the issuing state. Phones can read it over NFC, but the web cannot: Web NFC
only supports NDEF tags, and identity documents are ISO-DEP smart cards. This repository
proposes a narrow extension to [Web NFC](https://w3c-cg.github.io/web-nfc/) that lets a page
exchange commands with an identity document, and only with an identity document, with the
browser enforcing that restriction.

| | |
|---|---|
| **Status** | Proposal for discussion; working Chromium prototype on Android behind a flag |
| **Spec draft** | Proposed addition to the Web NFC Community Group report: TODO(link) |
| **Demo video** | TODO(link) |
| **Discussion** | [Issues](../../issues) · TPAC 2026 breakout: TODO(link) |
| **Editor** | Johan Sellström ([iProov](https://www.iproov.com/)) |

## Contents

- [The problem](#the-problem)
- [The proposal](#the-proposal)
- [Security and privacy](#security-and-privacy)
- [How this answers the concerns about Web NFC](#how-this-answers-the-concerns-about-web-nfc)
- [Relationship to the Digital Credentials API](#relationship-to-the-digital-credentials-api)
- [Implementation status](#implementation-status)
- [Try it](#try-it)
- [Considered alternatives](#considered-alternatives)
- [Open questions](#open-questions)
- [Repository layout](#repository-layout) · [Contributing](#contributing) · [Licence](#licence)

## The problem

Reading the chip in an identity document is one of the strongest ways to verify a person
remotely: the data is signed by the issuing state (Passive Authentication), and the chip can
prove it is genuine rather than a copy (Chip or Active Authentication). It is used for:

- **Onboarding to digital identity wallets**, for example issuing an EU Digital Identity Wallet
  PID, or creating an ICAO Digital Travel Credential, from the physical document.
- **Remote account opening and KYC**, where the relying party must verify the issuing state's
  signatures itself.
- **Government services**, where a citizen proves who they are once, from a link.

Today none of this works in a browser. On Android, tapping a passport while
`NDEFReader.scan()` is active fires `readingerror` ("This tag is not supported"), so every
relying party needs a native app. On iOS, App Clips with Core NFC can read these documents
without a full install; Android has no equivalent since Instant Apps were retired, and they
never allowed NFC. The web is the natural install-free option, but it cannot see the chip.

## The proposal

A page asks for identity documents; the browser hands it a session with one only when the
card really is one.

```js
const reader = new NFCIsoDepReader();

button.onclick = async () => {
  await reader.scan({ applications: ["emrtd"] }); // needs a user gesture and permission
};

reader.onreading = async ({ tag }) => {
  try {
    // SELECT the eMRTD application. Access control (BAC/PACE), secure messaging and
    // reading data groups use the same transceive() pattern, implemented by the page.
    const response = await tag.transceive(new Uint8Array([
      0x00, 0xA4, 0x04, 0x0C, 0x07, 0xA0, 0x00, 0x00, 0x02, 0x47, 0x10, 0x01,
    ]));
    console.log("status", response.slice(-2)); // 90 00 on success
  } finally {
    await tag.close();
  }
};

reader.onreadingerror = () => console.log("That card is not an identity document.");
```

```webidl
[SecureContext, Exposed=Window]
interface NFCIsoDepReader : EventTarget {
  constructor();
  attribute EventHandler onreading;
  attribute EventHandler onreadingerror;
  Promise<undefined> scan(optional NFCIsoDepScanOptions options = {});
};

dictionary NFCIsoDepScanOptions {
  AbortSignal signal;
  sequence<NFCIsoDepApplication> applications; // absent means ["emrtd"]
};

enum NFCIsoDepApplication { "emrtd" };

[SecureContext, Exposed=Window]
interface NFCIsoDepReadingEvent : Event {
  constructor(DOMString type, NFCIsoDepReadingEventInit eventInitDict);
  [SameObject] readonly attribute NFCIsoDepTag tag;
};

[SecureContext, Exposed=Window]
interface NFCIsoDepTag {
  readonly attribute Uint8Array? historicalBytes;
  readonly attribute unsigned long maxTransceiveLength;
  readonly attribute boolean extendedLengthApdu;
  Promise<Uint8Array> transceive(BufferSource commandApdu);
  Promise<undefined> close();
};
```

`applications` is an enum, not a list of raw application identifiers, so that each permitted
document type carries a browser-defined policy and the permission prompt can name it in plain
language. `"emrtd"` is the only value proposed.

**What the API is not:** a general smart card API (no payment, transit, access or SIM cards),
card emulation, or an eMRTD protocol built into the browser. The browser exchanges commands
and responses; the document protocol lives in page libraries, which keeps the browser surface
small and lets relying parties apply their own checks.

## Security and privacy

The main risk of card access from the web is that a hostile page talks to a payment card,
transit card or access badge in the user's pocket. The design answers this in layers:

1. **The browser classifies the card before the page sees it.** It selects the eMRTD
   application (`A0000002471001`) itself, or finds PACE parameters in `EF.CardAccess`. Only
   then does `reading` fire. Any other card produces `readingerror` and is never exposed.
2. **The browser filters every command.** Plaintext commands are limited to what reading an
   identity document needs: selecting the eMRTD application or its files, reading, and the
   BAC, PACE and authentication steps. Selecting any other application is refused, as are
   proprietary command classes (used by payment cards) and plaintext write commands. Once
   secure messaging starts, commands are encrypted and opaque to the browser, but by then the
   card has been classified and the page has proved it knows the document's access key. Writes
   are still not possible then: identity document chips are locked after issuance, and any
   change to their data would break the issuing state's signature.
3. **The document protects itself.** A chip reveals nothing until the reader proves knowledge
   of the access key printed on it (MRZ for BAC/PACE, or the CAN for PACE). A page without the
   key learns nothing; a page with it already has the printed data.
4. **The usual Web NFC gates:** secure context, top-level frame only, visible document,
   transient user activation for `scan()`, and a permission, ideally with a document-specific
   prompt: *"Allow example.com to read your passport or ID card chip?"*. NFC is suspended when
   the page is hidden.
5. **One session at a time**, ended by `close()`, removing the card, aborting, or the page
   losing visibility.

**Privacy.** The chip holds the holder's personal data and facial image, so the prompt must
say what is being read. Chip and Active Authentication produce signatures proving the document
was present; sites receive that evidence, which users should be told about. Identity documents
present a random identifier on each tap, and the API exposes no hardware identifiers, so it
cannot be used to track a document.

**Precedent.** iOS applies the same model to native apps: an app using Core NFC must declare
which card applications it may select (`com.apple.developer.nfc.readersession.iso7816.select-identifiers`),
and the system refuses all others. This proposal moves that control into the browser, with the
allowlist fixed by the platform rather than chosen by the site.

## How this answers the concerns about Web NFC

Mozilla ([position: negative](https://github.com/mozilla/standards-positions/issues/238)) and
WebKit ([position: oppose](https://github.com/WebKit/standards-positions/issues/584)) have
raised concerns about Web NFC in general. This proposal is deliberately narrow, and we would
like to know whether that changes the picture:

| Concern | This proposal |
|---|---|
| A permission prompt is not enough | The browser enforces a fixed document policy whatever the user grants, and the prompt names the document type |
| Users do not understand what NFC access means | "Read your passport or ID card chip" is concrete, and the user physically presents the document |
| Persistent attacks on external devices | The browser refuses plaintext writes, and identity document chips are locked after issuance; changed data would fail the issuing state's signature check |
| Device independence of the web | The target is an ICAO-standardised document interface used worldwide, not vendor-specific hardware |

## Relationship to the Digital Credentials API

The Digital Credentials API lets a page request a credential that a wallet **already holds**.
It does not cover getting the document's data in the first place, and when a wallet derives a
credential from a passport, the relying party receives the wallet issuer's signature, not the
issuing state's. The two are complementary: this proposal serves issuance, onboarding and
high-assurance proofing; Digital Credentials serves presentation.

## Implementation status

| Engine / platform | Status |
|---|---|
| Chromium on Android | Prototype behind the `WebNFCIsoDep` runtime flag. Card classification and command filter run in the browser process. Has read a real national ID card end to end (PACE, Chip Authentication, Passive Authentication) on a Galaxy Z Fold3, in about three seconds. |
| iOS | The API maps onto Core NFC (`NFCTagReaderSession`, `NFCISO7816Tag`); a WKWebView-based prototype is being tested. |
| Gecko, WebKit | No implementation; positions on Web NFC listed above. |

**Lessons from real devices**, useful for any implementation:

- Android's default reader-mode presence check dropped identity documents within milliseconds
  of discovery. Reading reliably needed `FLAG_READER_SKIP_NDEF_CHECK` and a longer presence-check
  delay while only identity-document readers are active.
- Long responses on chips without extended-length support arrive in parts (`61xx`), so the
  command filter allows `GET RESPONSE`.
- On some Samsung devices, opening the camera lets a system service take NFC reader mode and
  switch it off for all apps when it finishes. Pages should start `scan()` after any camera use.

## Try it

You need an Android phone with NFC (Android 10 or later) and a passport or ID card with a chip.

1. Install the prototype browser: [ChromePublic.apk](https://storage.googleapis.com/lab-mcp-johan-webnfc/ChromePublic.apk).
   It installs next to Chrome as "Chromium" and has `NFCIsoDepReader` enabled.
2. Open the demo page in it: TODO(link to the hosted `demo/`).
3. Tap **Scan**, allow NFC, and hold the document to the phone. The demo selects the eMRTD
   application, reads `EF.CardAccess`, and shows the browser refusing payment-card commands.

To build the prototype yourself, apply the patches in [`chromium/`](chromium/) to a Chromium
checkout (see [`chromium/README.md`](chromium/README.md)) and build `chrome_public_apk`, with
*Experimental Web Platform features* enabled in `chrome://flags`.

## Considered alternatives

- **Generic ISO-DEP access with application identifiers chosen by the page.** Simpler, but the
  browser cannot judge arbitrary identifiers and the prompt cannot explain them.
- **Document reading built into the browser** (a high-level `readPassport()`). Puts protocol
  and cryptography in the browser, freezes protocol versions, and stops relying parties
  applying their own verification and fraud checks.
- **Web Smart Card API.** Shaped around PC/SC and limited to Isolated Web Apps; not available
  to ordinary pages on mobile.
- **WebUSB with an external reader.** Smart card interface classes are blocked in WebUSB, and
  people do not carry readers.
- **Native apps / App Clips.** The status quo, with no install-free option on Android.

## Open questions

- Should `close()` be required, or should a session end with the event handler?
- Should the policy also cover national eID cards with ICAO 9303-compatible structures, or eID
  schemes that need terminal authentication?
- Is exposing `historicalBytes` needed, or is it fingerprinting surface?
- A separate permission, or the existing `nfc` permission with a document-specific prompt?

## Repository layout

| Path | Contents |
|---|---|
| `README.md` | This explainer |
| `spec/` | The proposed addition to the Web NFC CG report (also sent as a pull request) |
| `chromium/` | Prototype patches for Chromium (Android), with build notes |
| `demo/` | The demo page |

## Contributing

Feedback is welcome as [issues](../../issues). Spec contributions follow the
[Web NFC Community Group](https://www.w3.org/community/web-nfc/) process and require the W3C
Community Contributor License Agreement.

## Licence

TODO: confirm before publishing. Suggested: documents under CC BY 4.0; code and Chromium
patches under the BSD 3-Clause licence, matching Chromium.
