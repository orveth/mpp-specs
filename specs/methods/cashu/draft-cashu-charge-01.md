---
title: Cashu Charge Intent for HTTP Payment Authentication
abbrev: Cashu Charge Intent
docname: draft-cashu-charge-01
version: 01
category: info
ipr: noModificationTrust200902
submissiontype: independent
consensus: false

author:
  - name: TODO
    ins: TODO
    email: todo@example.com
    org: TODO

normative:
  RFC2119:
  RFC3339:
  RFC4648:
  RFC8174:
  RFC8259:
  RFC8785:
  RFC9457:
  RFC9530:
  I-D.payment-intent-charge:
    title: "'charge' Intent for HTTP Payment Authentication"
    target: https://datatracker.ietf.org/doc/draft-payment-intent-charge/
    author:
      - name: Jake Moxey
      - name: Brendan Ryan
      - name: Tom Meagher
    date: 2026
  I-D.httpauth-payment:
    title: "The 'Payment' HTTP Authentication Scheme"
    target: https://datatracker.ietf.org/doc/draft-ryan-httpauth-payment/
    author:
      - name: Jake Moxey
    date: 2026-01
  NUT-00:
    title: "NUT-00: Notation, blinding, and tokens"
    target: https://github.com/cashubtc/nuts/blob/main/00.md
    author:
      - org: Cashu
    date: 2024
  NUT-02:
    title: "NUT-02: Keysets and fees"
    target: https://github.com/cashubtc/nuts/blob/main/02.md
    author:
      - org: Cashu
    date: 2024
  NUT-03:
    title: "NUT-03: Swap tokens"
    target: https://github.com/cashubtc/nuts/blob/main/03.md
    author:
      - org: Cashu
    date: 2024
  NUT-10:
    title: "NUT-10: Spending conditions"
    target: https://github.com/cashubtc/nuts/blob/main/10.md
    author:
      - org: Cashu
    date: 2024
  NUT-12:
    title: "NUT-12: Offline ecash signature validation (DLEQ)"
    target: https://github.com/cashubtc/nuts/blob/main/12.md
    author:
      - org: Cashu
    date: 2024
  NUT-18:
    title: "NUT-18: Payment requests"
    target: https://github.com/cashubtc/nuts/blob/main/18.md
    author:
      - org: Cashu
    date: 2024

informative:
  NUT-01:
    title: "NUT-01: Mint public key exchange"
    target: https://github.com/cashubtc/nuts/blob/main/01.md
    author:
      - org: Cashu
    date: 2024
  NUT-07:
    title: "NUT-07: Token state check"
    target: https://github.com/cashubtc/nuts/blob/main/07.md
    author:
      - org: Cashu
    date: 2024
  NUT-09:
    title: "NUT-09: Restore signatures"
    target: https://github.com/cashubtc/nuts/blob/main/09.md
    author:
      - org: Cashu
    date: 2024
  NUT-24:
    title: "NUT-24: HTTP 402 Payment Required"
    target: https://github.com/cashubtc/nuts/blob/main/24.md
    author:
      - org: Cashu
    date: 2025
  NUT-26:
    title: "NUT-26: Bech32m payment requests"
    target: https://github.com/cashubtc/nuts/blob/main/26.md
    author:
      - org: Cashu
    date: 2025
  W3C-DID:
    title: "Decentralized Identifiers (DIDs) v1.0"
    target: https://www.w3.org/TR/did-core/
    author:
      - org: W3C
    date: 2022
---

--- abstract

This document defines the "charge" intent for the "cashu" payment
method within the Payment HTTP Authentication Scheme
{{I-D.httpauth-payment}}. The server issues a Cashu payment request
{{NUT-18}} as a challenge; the client presents a Cashu token whose
value, net of the mint's swap fee, redeems to the requested amount
as a credential, which the server
verifies and redeems by swapping {{NUT-03}} it at the issuing mint. This method
relocates the challenge-and-token semantics of the existing Cashu
HTTP 402 binding {{NUT-24}} into the standard
`Authorization`/`WWW-Authenticate` framework.

--- middle

# Introduction

HTTP Payment Authentication {{I-D.httpauth-payment}} defines
a challenge-response mechanism that gates access to resources behind
micropayments. This document registers the "charge" intent for the
"cashu" payment method.

Cashu is a Chaumian ecash protocol in which a mint issues
blind-signed bearer tokens denominated in a unit. The mint's blind
signatures let a token be redeemed without the mint linking the
redemption to the token's issuance. A token carries its own value;
it is verified and redeemed by swapping {{NUT-03}} it at the mint
that signed it. The "cashu" method gates a resource
behind presentation of such a token: the server names an amount,
unit, and acceptable mint set in the challenge, and the client
returns a token the server redeems to settle.

The "cashu" method is independent of what the mint's unit
denominates. The unit is a label chosen by the service that issues
the token: it MAY be a denomination of an external asset (for
example, "sat") or a unit meaningful only within the issuing
service. This document defines the generic Cashu charge exchange and
treats the unit as an opaque identifier carried in the `currency`
field; what a unit denominates, and any backing or redemption
semantics, are defined by the issuing service and are out of scope
here.

The flow proceeds as follows:

~~~
   Client                        Server                          Mint
      |                             |                              |
      |  (1) GET /resource          |                              |
      |-------------------------->  |                              |
      |                             |                              |
      |  (2) 402 Payment Required   |                              |
      |      (request, mints)       |                              |
      |<--------------------------  |                              |
      |                             |                              |
      |  (3) Swap token to exact    |                              |
      |      amount (local)         |                              |
      |---------------------------------------------------------> |
      |  (4) Exact-amount token     |                              |
      |<--------------------------------------------------------- |
      |                             |                              |
      |  (5) GET /resource          |                              |
      |      credential: token      |                              |
      |-------------------------->  |                              |
      |                             |  (6) Swap presented token    |
      |                             |--------------------------->  |
      |                             |  (7) Fresh proofs            |
      |                             |<---------------------------  |
      |                             |                              |
      |  (8) 200 OK (resource)      |                              |
      |<--------------------------  |                              |
      |                             |                              |
~~~

## Relationship to NUT-24 {#relationship-nut24}

NUT-24 {{NUT-24}} is the existing Cashu binding of HTTP 402
"Payment Required". In NUT-24, a server returns HTTP 402 with an
`X-Cashu` header carrying a NUT-18 {{NUT-18}} `creqA` payment
request restricted to the fields `{a, u, m, nut10}` with an empty
transport; the client retries with a `cashuB` token in the
`X-Cashu` header; a bad mint, unit, or amount yields HTTP 400.

This document is the standards-aligned sibling of that binding. It
relocates the same `creqA`-challenge and `cashuB`-credential
semantics into the standard `Authorization`/`WWW-Authenticate`
authentication framework {{I-D.httpauth-payment}}, substituting a
402 response carrying a fresh `WWW-Authenticate: Payment`
re-challenge for NUT-24's flat 400. The embedded `creqA` reuses
NUT-24's challenge field subset `{a, u, m, nut10}` (amount, unit,
mints, and spending-condition kind), so a single Cashu code path
can serve both bindings: the wire envelope differs, the payment
request and token do not.

## Relationship to the Charge Intent

This document inherits the shared request semantics of the "charge"
intent from {{I-D.payment-intent-charge}}. It defines only the
Cashu-specific `methodDetails`, `payload`, verification, and
settlement procedures for the "cashu" payment method.

# Requirements Language

{::boilerplate bcp14-tagged}

# Terminology

Cashu Token
: A bearer ecash token (a `cashuB...` string, the NUT-00
  TokenV4 serialization) encoding one or more proofs issued by a
  single mint under a single unit, the mint's URL, and that unit.
  The authoritative value carried by the credential.

Payment Request
: A Cashu payment request {{NUT-18}} (a `creqA...` string, or the
  `creqb1...` Bech32m form {{NUT-26}}) encoding the amount, unit,
  acceptable mints, spending-condition kind, and single-use flag a
  payer must satisfy. It is a self-contained, mint-agnostic request
  artifact; all payment parameters are derived from it.

Proof
: A single unit of ecash: an `{amount, id, secret, C}` tuple
  {{NUT-00}} signed by a mint keyset. A token is a set of proofs.

Swap
: The mint operation {{NUT-03}} that exchanges a set of input
  proofs for a set of freshly blind-signed output proofs. A
  successful swap marks the inputs spent and is the act of
  redemption.

DLEQ Proof
: A discrete-log-equality proof {{NUT-12}} binding a blind signature
  to the mint's advertised public key, proving the mint signed with
  the key it published. In this method it lets the server detect a
  mint that returns signatures it did not validly produce (see
  {{security-dleq}}).

Swap Fee
: The NUT-03 input fee a swap deducts from the input proofs,
  `swap_fee = ceil(sum(input_fee_ppk over input proofs) / 1000)`,
  where each keyset's `input_fee_ppk` is published per {{NUT-02}}.
  Deterministic from the input proofs' keysets, so it can be
  computed offline before a token is presented.

# Intent Identifier

The intent identifier for this specification is "charge". It MUST
be lowercase.

# Intent: "charge"

The "charge" intent represents a one-time payment gating access to
a resource. The server advertises a Cashu payment request
({{NUT-18}}) naming an exact amount and unit per request. The
client presents a Cashu token whose value, after the swap fee the
mint will deduct, settles to exactly that amount as the credential.
The server verifies the token and redeems it by swapping
({{NUT-03}}) it at the issuing mint; a successful swap both proves
the token unspent and transfers its value to the server.

The "cashu" charge is exact-amount and makes no change: the server
accepts only a token that, once swapped, nets
the server the requested amount exactly, redeems the whole token,
and keeps the resulting proofs. Where the token's keyset(s) charge
a NUT-03 input fee, the holder pre-funds that fee in the presented
token (see {{fees}}); for fee-free keysets the presented value
equals the requested amount. A client holding a token larger than
that value MUST split it locally at the mint before presenting (see
{{settlement}}); the remainder is never seen by the server.

## Fees {#fees}

A NUT-03 swap deducts an input fee set by the input proofs'
keyset(s), `swap_fee = ceil(sum(input_fee_ppk) / 1000)`, where
`input_fee_ppk` is published per keyset in the mint's keyset list
({{NUT-02}}, {{NUT-03}}). The fee is deterministic from the presented
proofs, so both holder and server compute the same value before the
token is presented.

The "cashu" charge is exact-amount and the server makes no change, so
the holder pre-funds the fee: the presented token's total value MUST
equal `amount + swap_fee`. The server swaps the whole token, the mint
deducts `swap_fee`, and the server's outputs sum to exactly `amount`.
For a zero-fee keyset this reduces to `presented == amount`. The
server's exact-value check ({{verification}}, step 12) recomputes
`swap_fee` from the presented proofs and never trusts a
client-supplied value.

Selecting proofs to total `amount + swap_fee` has a mild fixpoint (a
proof added to cover the fee can raise the fee); standard fee-aware
coin selection resolves it. Where `swap_fee` is large relative to
`amount` the charge is uneconomic rather than impossible (see
{{security-fees}}).

# Encoding Conventions {#encoding}

All JSON {{RFC8259}} objects carried in auth-params or
HTTP headers in this specification MUST be serialized using the JSON
Canonicalization Scheme (JCS) {{RFC8785}} before encoding. JCS
produces a deterministic byte sequence, which is required for
any digest or signature operations defined by the base spec
{{I-D.httpauth-payment}}.

The resulting bytes MUST then be encoded using base64url
{{RFC4648}} Section 5 without padding characters
(`=`). Implementations MUST NOT append `=` padding
when encoding, and MUST accept input with or without padding when
decoding.

This encoding convention applies to: the `request`
auth-param in `WWW-Authenticate`, the credential token in
`Authorization`, and the receipt token in
`Payment-Receipt`.

The `request` object is a JCS-canonical JSON object. The Cashu
payment request (`methodDetails.request`) and the Cashu token
(`payload.cashu_token`) are each carried as an opaque string value
within that JSON. The `creqA...` (or
`creqb1...` {{NUT-26}}) and `cashuB...` strings have their own
internal encoding {{NUT-18}} {{NUT-00}}; that encoding is opaque to
the framework and is never canonicalized by it. Conformance to the
JCS requirement is a property of the enclosing JSON object, not of
these embedded strings.

# Request Schema

## Shared Fields

The `request` auth-param of the `WWW-Authenticate: Payment`
header contains a JCS-serialized, base64url-encoded JSON object
(see {{encoding}}). The following shared fields are
included in that object:

amount
: REQUIRED. The required amount in the base units of the Cashu
  unit, encoded as a canonical decimal string of ASCII digits
  (e.g., "100"). The value MUST be a positive integer with no
  leading zeros, no sign, no whitespace, and no fractional part,
  and MUST fit in an unsigned 64-bit integer. Servers MUST emit
  `amount` in this canonical form; since it is server-authored and
  echoed in the credential, an altered or malformed echoed `amount`
  is tampering, rejected as `invalid-challenge` (step 5). This value is the amount the server
  nets after the swap; it MUST equal the amount encoded in
  `methodDetails.request`. The presented token value is
  `amount + swap_fee` (see {{fees}}).

currency
: REQUIRED. The Cashu unit string {{NUT-00}} the presented token
  MUST carry (e.g., "sat"). It MUST equal the unit encoded in
  `methodDetails.request`. This is a method-defined currency
  identifier per {{I-D.payment-intent-charge}}: the Cashu unit
  string is itself the `currency` value, and `amount` is an integer
  count of that unit. This method does not decompose a unit into a
  parent asset and a sub-unit; units that differ in scale are
  distinct `currency` values (for example, "sat" and "msat" are two
  different units, never the same currency at different precisions).

description
: OPTIONAL. A human-readable memo describing the resource or
  service being paid for. If present, this value SHOULD be set as
  the description field of the Cashu payment request
  ({{NUT-18}}) and is distinct from any `description` auth-param
  that the base {{I-D.httpauth-payment}} scheme may include at
  the header level.

recipient
: OPTIONAL. Payment recipient in method-native format, per
  {{I-D.payment-intent-charge}}. Cashu implementations do not use
  this field; the recipient is the server, implied by the mint
  swap it performs. Servers SHOULD omit it.

externalId
: OPTIONAL. Merchant's reference (e.g., order ID, invoice number),
  per {{I-D.payment-intent-charge}}. May be used for
  reconciliation. If present, it is echoed in the receipt (see
  {{receipt}}).

## Method Details

The following fields are nested under `methodDetails` in the
request JSON. The Cashu payment request (`methodDetails.request`)
is the authoritative source for payment parameters. The
`methodDetails.mints` field is the server-chosen set of mints
whose tokens the server will accept for this challenge. Clients MUST
decode and verify the payment request independently before
presenting, and
MUST reject challenges where `amount` or `currency` do not match
the values encoded in the payment request.

request
: REQUIRED. The Cashu payment request string ({{NUT-18}}, a
  `creqA...` value). Servers and clients SHOULD also accept the
  equivalent Bech32m encoding ({{NUT-26}}, a `creqb1...` value);
  the two encodings are interchangeable and carry the same payment
  parameters. The request is carried in its native encoded form
  rather than as the decoded JSON it represents: the encoded string
  is the canonical, self-contained artifact existing Cashu wallets
  and libraries already produce and parse, and it is byte-identical
  to the request used by the NUT-24 {{NUT-24}} binding, so a single
  Cashu code path serves both. This field is authoritative; all payment parameters
  (amount, unit, acceptable mints, spending-condition kind,
  single-use flag, optional description) are derived from it. Its
  transport set MUST be empty, which {{NUT-18}} defines as in-band:
  the credential is returned over the same HTTP channel in the
  `Authorization` header rather than over a separate transport. Its
  spending-condition kind MUST be absent (`nut10` is `None`); see
  {{verification}}.

mints
: REQUIRED. A JSON array of mint URL strings whose tokens the
  server accepts for this challenge. The server, not the client,
  chooses these mints. The presented token's mint MUST be a member
  of this array (see {{verification}}). This array MUST be
  non-empty and MUST be a superset of, or equal to, the mint set
  encoded in `methodDetails.request`. Clients MUST reject a
  challenge whose `mints` does not include a mint they can obtain
  a token from.

# Credential Schema

The `Authorization` header carries a single base64url-encoded
JSON token (no auth-params). The decoded object contains two
top-level fields:

challenge
: REQUIRED. An echo of the challenge auth-params from the
  `WWW-Authenticate` header: `id`, `realm`, `method`, `intent`,
  `request`, and, if present in the challenge, `digest`, `opaque`,
  and `expires`. This binds the credential to the exact challenge
  that was issued. A client MUST echo each of these fields
  unchanged when the server included it (see {{verification}}).

source
: OPTIONAL. A payer identifier string, as defined by
  {{I-D.httpauth-payment}}. The RECOMMENDED format
  is a Decentralized Identifier (DID) per
  {{W3C-DID}}. Cashu tokens are bearer instruments and carry no
  payer identity; implementations MAY omit this field, and
  servers MUST NOT require it.

payload
: REQUIRED. A JSON object containing the Cashu-specific credential
  fields. The single required field is `cashu_token`: the Cashu
  token ({{NUT-00}}, a `cashuB...` string) whose value, net of the
  swap fee, settles to exactly the requested amount (see
  {{fees}}).

Example (decoded):

~~~json
{
  "challenge": {
    "id": "kM9xPqWvT2nJrHsY4aDfEb",
    "realm": "api.example.com",
    "method": "cashu",
    "intent": "charge",
    "request": "eyJ...",
    "expires": "2026-03-15T12:05:00Z"
  },
  "source": "did:key:z6MkhaXgBZDvotDkL5257faiztiGiC2QtKLGpbnnEGta2doK",
  "payload": {
    "cashu_token": "cashuBpGF0gaJhaUgA..."
  }
}
~~~

When the challenge carried `digest` and `opaque`, the credential
echoes them too:

~~~json
{
  "challenge": {
    "id": "kM9xPqWvT2nJrHsY4aDfEb",
    "realm": "api.example.com",
    "method": "cashu",
    "intent": "charge",
    "request": "eyJ...",
    "digest": "sha-256=:X48E9qOokqqrvdts8nOJRJN3OWDUoyWxBf7kbu9DBPE=:",
    "opaque": "eyJpbnRlbnQiOiJwaV8xMjMifQ",
    "expires": "2026-03-15T12:05:00Z"
  },
  "payload": {
    "cashu_token": "cashuBpGF0gaJhaUgA..."
  }
}
~~~

# Verification Procedure {#verification}

Upon receiving a request with a credential, the server MUST:

1. Decode the base64url credential and parse the JSON.
2. Verify that `payload.cashu_token` is present, is a string, and
   decodes as a Cashu token ({{NUT-00}}). The token MUST be a
   `cashuB...` (TokenV4) serialization; a `cashuA...` (TokenV3)
   string MUST be rejected. Reject a token that does not parse or
   carries zero proofs. Servers SHOULD reject a token carrying more
   than a configured maximum number of proofs (see {{security-dos}}).
3. Verify that all proofs in the token reference a single mint and a
   single unit. Reject any token whose proofs name more than one
   mint or more than one unit.
4. Recover and authenticate the challenge parameters. Under
   stateless operation (RECOMMENDED), recompute the HMAC-SHA256
   `id` binding of {{I-D.httpauth-payment}} over the echoed
   `credential.challenge` parameters with the server key, and
   reject the request unless it equals `credential.challenge.id`;
   the echoed `request` parameters (amount, unit, accepted mints)
   are thereby authenticated. Under stored operation, look up the
   challenge by `credential.challenge.id`, reject if none is found,
   and take the stored `request` parameters as authoritative.
5. Verify the echoed `credential.challenge` fields (`id`, `realm`,
   `method`, `intent`, `request`, and, when issued, `digest`,
   `opaque`, `expires`) are consistent with the authenticated
   challenge from step 4 — an exact match against the stored
   auth-params under stored operation, or covered by the `id`-HMAC
   over those same fields under stateless operation. A mismatch is
   tampering and MUST be rejected as `invalid-challenge`.
6. If the challenge carried a `digest` auth-param, the server MUST
   compute the content digest of the current request body per
   {{RFC9530}} and reject the credential if it does not match the
   echoed `digest`.
7. If `credential.challenge.expires` is present, the server MUST
   reject the credential when that timestamp is in the past. A
   Cashu `creqA` carries no expiry of its own, so the `expires`
   auth-param is the sole challenge-expiry signal; rejection on a
   past `expires` is a `payment-expired` condition (see {{errors}}).
8. Verify the token's unit equals `currency`.
9. Verify the token's mint is a member of `methodDetails.mints`.
   Mint membership SHOULD be compared by mint identity key
   {{NUT-01}} rather than by URL string.
10. Verify that no proof carries a NUT-10 {{NUT-10}} well-known
    (P2PK or HTLC) secret. This intent accepts plain-secret BEARER
    proofs only; a proof bound to a spending condition is rejected
    as `verification-failed` (see {{spending-conditions}}).
11. Resolve every proof's keyset id against the mint's published
    keysets {{NUT-02}}. When a proof uses a short keyset id, the
    server MUST resolve it to the full keyset via the mint's fetched
    keyset list and MUST reject a short id that is ambiguous or does
    not resolve (see {{short-keyset}}).
12. Compute `expected_swap_fee` over the resolved keyset(s) of the
    presented proofs ({{fees}}) and verify the token's total value
    equals `amount + expected_swap_fee` EXACTLY. A token worth more
    OR less MUST be rejected; the server makes no change.
13. Swap ({{NUT-03}}) the whole token at its mint, following the
    durability and idempotency requirements of {{settlement}}. A
    successful swap is the redemption step and is the authoritative
    check that the input proofs were genuine, validly signed, and
    unspent. The server MUST verify the DLEQ proofs {{NUT-12}} on
    the blind signatures the swap returns and SHOULD reject a mint
    that omits them (see {{security-dleq}}); it does not verify DLEQ
    on the presented input proofs, since the swap already proves
    their validity. A swap rejected because a proof is already
    spent, or because a DLEQ check on the returned signatures fails,
    is a `verification-failed` condition; a swap rejected because
    the keyset has retired or its `final_expiry` has passed is a
    `payment-expired` condition (see {{settlement}}, {{errors}}).

Steps 8 through 12 are structural and MUST be performed before the
network swap in step 13, so a structurally invalid token never
produces a mint round trip. The keyset resolution of step 11 MAY
require fetching the mint's keysets {{NUT-02}} before the swap.

These steps discharge the verification responsibilities the "charge"
intent ({{I-D.payment-intent-charge}}) places on the server:
challenge-match and freshness are steps 4–7; payment-proof
verification is steps 8–13, where the swap itself is the proof of
payment; the amount-match responsibility is step 12, read as the
NET settled amount — the server nets exactly `amount` while the
holder pre-funds the swap fee, so the presented total is
`amount + expected_swap_fee` (see {{fees}}); and the recipient-match
responsibility is satisfied implicitly, because redemption is the
server swapping the presented token to itself at the mint, so there
is no distinct recipient to compare — the `recipient` field is
correspondingly omitted.

## Spending-Condition-Locked Tokens {#spending-conditions}

The "cashu" charge accepts plain-secret BEARER proofs only. A
proof whose secret is a NUT-10 {{NUT-10}} well-known secret (for
example a P2PK or HTLC lock) requires a witness the server cannot
produce, so its swap would fail with no diagnosable reason. The
server therefore rejects any locked proof before the swap (step 10)
as `verification-failed`. Consistently, the challenge's embedded
`creqA` MUST set `nut10` to `None` (absent); a `creqA` requesting a
spending-condition kind is not used by this intent. Support for
spending-condition-locked tokens is out of scope for this intent.

## Challenge Binding

To prevent token replay across different resources or challenges,
the server MUST bind the issued `request` parameters to the
challenge `id` and verify, when a credential is presented, that
`credential.challenge` is an exact echo of an issued challenge.
The server SHOULD compute `id` as the HMAC-SHA256 binding defined
by {{I-D.httpauth-payment}} so that binding is stateless;
alternatively the server MAY store issued challenges and verify by
lookup. The challenge MUST be constructed with all framework-
REQUIRED auth-params — `id`, `realm`, `method`, `intent`, and
`request` — and the server SHOULD include `expires`; when the
charge gates a request with a body the server SHOULD include
`digest` and SHOULD include any `opaque` correlation data it needs
echoed.

The `single_use` flag carried inside the embedded `creqA`
({{NUT-18}}) is informational for this method. Replay protection
does not depend on it: it comes from challenge binding (above)
together with the proof-level single-use property of the redeemed
token (see {{security-replay}}).

Replay of the underlying token is independently prevented at the
proof level: a token can be swapped at most once, after which its
proofs are spent and the mint refuses any further swap (see
{{security-replay}}). Challenge binding additionally prevents a
token valid for one challenge from being presented against a
different challenge.

## Short Keyset Identifiers {#short-keyset}

A proof carries a keyset id identifying the signing key. A
version-`00` keyset id is a short 8-byte (16 hex character)
identifier; a version-`01` keyset id is the full 33-byte (66 hex
character) identifier ({{NUT-02}}). When a presented proof
uses a short keyset id, the server MUST resolve it to a full keyset
by fetching the mint's keyset list {{NUT-02}} and matching, and
MUST derive the keyset per {{NUT-02}}. A short id that matches no
published keyset, or that is ambiguous across the mint's keysets,
MUST be rejected as `verification-failed`. A failure to FETCH the
keyset list at all (a network error reaching the mint), as distinct
from a short id that resolves but matches no or several keysets, is
a `mint-unavailable` (HTTP 503) condition with the token not
consumed (see {{errors}}), not a `verification-failed`. Resolution
is required both to compute the swap fee ({{fees}}) and to construct
correct swap outputs.

# Settlement Procedure {#settlement}

Settlement is the mint swap ({{NUT-03}}) of the presented token.
The server swaps the whole token for fresh proofs it controls;
holding those proofs is settlement. The mint deducts the swap fee
({{fees}}) from the inputs, so the server's output proofs sum to
exactly `amount`. The server's outputs are blinded against the
mint's currently ACTIVE keyset for the unit ({{NUT-02}}), which MAY
differ from the keyset(s) that signed the input proofs. Cashu
settlement is final once the swap succeeds: the input proofs are
spent and cannot be restored. The server makes no change and
returns no proofs to the client.

Because the "cashu" charge is exact-amount, a client holding a
token larger than the value it must present MUST split it locally
before presenting. The client swaps ({{NUT-03}}) its token at the
mint into (a) a token worth exactly `amount + swap_fee`, which it
presents, and (b) a remainder it keeps, generating the blinded
outputs for both halves itself. This local split is itself a
fee-bearing swap: to end up holding a presentable token worth
`amount + swap_fee` AND keep a remainder, the holder must spend
inputs worth `amount + swap_fee + split_fee`, where `split_fee` is
the fee of the local split swap computed over its own input proofs
({{fees}}). Neither the mint nor the server learns the remainder's
secrets. This local split is the client's responsibility and
happens before the `Authorization` request; the server never
performs it.

## Durability, Idempotency, and Crash Recovery {#durability}

The swap is a money-moving operation; a crash or timeout around it
can lose the server's value or double-charge the holder. Servers
MUST therefore:

- Persist the swap's output secrets (the blinding factors and
  blinded messages) BEFORE sending the swap to the mint, so that a
  crash after the mint has spent the inputs does not lose the
  ability to reconstruct or restore the resulting proofs.
- Serialize redemption per presented token and per `challenge.id`,
  so concurrent requests presenting the same token or hitting the
  same challenge cannot issue two swaps. The redemption and the
  decision to return HTTP 200 MUST be a single atomic operation
  (see {{security-replay}}).
- Accept an `Idempotency-Key` request header per
  {{I-D.httpauth-payment}} for non-idempotent target methods and,
  on a retry bearing the same key, return the original response
  without re-swapping.

If a swap request times out or returns 5xx with an indeterminate
outcome, the server MUST NOT blindly re-swap. It MUST first query
the proofs' state with NUT-07 `/checkstate` {{NUT-07}}: if the
inputs are already spent, the first swap succeeded and the server
recovers its outputs via NUT-09 `/restore` {{NUT-09}} using the
persisted output secrets; only if the inputs are unspent may the
server re-swap. A swap that cannot be resolved this way MUST be
surfaced as `mint-unavailable` (see {{errors}}) with the token
treated as not consumed.

## Consume-Once and Resource Delivery

The server MUST treat the redemption (the swap) and the decision to
return HTTP 200 as a single operation: a challenge whose token has
been redeemed MUST NOT be accepted again, even if resource delivery
subsequently fails. If resource delivery fails after the token is
redeemed, the server MUST return an appropriate HTTP error
(e.g., 500) and MUST NOT reissue the same challenge. The client
MUST treat such a response as a payment loss and MAY retry with a
new token. Cashu settlement is final once the swap succeeds; the
redeemed token cannot be refunded by the server.

Servers MUST include `Cache-Control: no-store` on all HTTP
402 responses. The challenge contains a single-use payment request;
caching it could cause clients to present a token against a stale
challenge.

## Receipt Generation {#receipt}

The server MUST include a `Payment-Receipt` header on the 200
response per {{I-D.httpauth-payment}}, and MUST NOT include it on
error responses. It carries the following fields:

method
: REQUIRED. The string "cashu".

challengeId
: REQUIRED. The challenge identifier (the `id` from the
  WWW-Authenticate challenge) for audit and traceability
  correlation.

reference
: REQUIRED. A SHA-256 hash, as a lowercase hex string, of the
  exact `cashu_token` credential string received from the client
  (the `cashuB...` string as presented, not a re-encoding). Serves
  as a stable, shareable settlement identifier. The token string
  itself MUST NOT be used here: although a redeemed token is spent
  and cannot be replayed, the proof secrets remain sensitive and
  MUST NOT be exposed in logs, analytics, or shared receipts.

status
: REQUIRED. The string "success".

timestamp
: REQUIRED. The settlement time in {{RFC3339}} format.

externalId
: OPTIONAL. Echo of the `externalId` from the request, if one was
  provided.

The response carrying a `Payment-Receipt` header MUST include
`Cache-Control: private` per {{I-D.httpauth-payment}}, so that no
shared cache stores the receipt.

Example (decoded):

~~~json
{
  "method": "cashu",
  "challengeId": "kM9xPqWvT2nJrHsY4aDfEb",
  "reference": "9b71d224bd62f3785d96d46ad3ea3d73319bfbc2890caadae2dff72519673ca7",
  "status": "success",
  "timestamp": "2026-03-10T21:00:00Z",
  "externalId": "order_12345"
}
~~~

# Error Responses {#errors}

When rejecting a credential for a payment-verification failure, the
server MUST return HTTP 402 (Payment Required) with a fresh
`WWW-Authenticate: Payment` challenge per {{I-D.httpauth-payment}}.
The server SHOULD include a response body conforming to RFC 9457
{{RFC9457}} Problem Details, with
`Content-Type: application/problem+json`.

A malformed REQUEST — as opposed to a failed payment — follows the
framework's status handling rather than returning 402: a credential
naming an unsupported method yields `method-unsupported` (HTTP 400),
and a request bearing more than one `Authorization: Payment`
credential is rejected with HTTP 400 per {{I-D.httpauth-payment}}.
The 402 problem types below are scoped to payment-verification
failures.

The following problem types are defined for this intent:

https://paymentauth.org/problems/cashu/malformed-credential
: HTTP 402. The credential token could not be decoded, the JSON
  could not be parsed, required fields (`challenge`, `payload`,
  `payload.cashu_token`) are absent or have the wrong type,
  `cashu_token` does not decode as a Cashu token, or the token is a
  `cashuA...` (TokenV3) serialization. A fresh challenge
  MUST be included in `WWW-Authenticate`.

https://paymentauth.org/problems/cashu/invalid-challenge
: HTTP 402. The value of `credential.challenge.id` does not match
  any challenge issued by this server (stored operation), or
  `credential.challenge` is not an exact echo of an issued
  challenge (including a `digest` that does not match the request
  body) or fails its `id`-HMAC (stateless operation). Under stored
  operation this also covers a challenge already consumed; under
  stateless operation a replayed token is instead caught at the
  swap as a spent token (`verification-failed`). A fresh challenge
  MUST be included in `WWW-Authenticate`.

https://paymentauth.org/problems/cashu/payment-expired
: HTTP 402. The challenge `expires` auth-param echoed in the
  credential is in the past, or the mint rejected the swap because
  the token's keyset has retired or its `final_expiry` {{NUT-02}}
  has passed. A fresh challenge MUST be included in
  `WWW-Authenticate`.

https://paymentauth.org/problems/cashu/amount-mismatch
: HTTP 402. The token's total value does not equal
  `amount + swap_fee` (see {{fees}}) — over- or under-funded; the
  server makes no change. A fresh challenge MUST be included in
  `WWW-Authenticate`.

https://paymentauth.org/problems/cashu/verification-failed
: HTTP 402. The token failed a non-amount, non-expiry verification
  check: its unit does not equal `currency`, its proofs reference
  more than one mint or unit, its mint is not a member of
  `methodDetails.mints`, a proof carries a NUT-10 {{NUT-10}}
  spending condition, a proof uses an unresolvable or ambiguous
  short keyset id, the mint omitted DLEQ proofs on the swap-returned
  signatures, or the
  mint rejected the swap because a proof was already spent. A fresh
  challenge MUST be included in `WWW-Authenticate`.

https://paymentauth.org/problems/cashu/mint-unavailable
: HTTP 503. The mint could not be reached (DNS, TCP, TLS, or
  timeout), or a swap outcome could not be resolved (see
  {{durability}}), so the token could neither be verified nor
  redeemed — an infrastructure failure, not a payment-verification
  outcome. The token is NOT consumed and the client MAY retry the
  same token. The server SHOULD include a `Retry-After` header and
  MUST NOT treat the token as consumed.

A token whose mint is reachable but is not in
`methodDetails.mints`, or whose unit is otherwise disallowed by
server policy, is a `verification-failed` condition (HTTP 402),
not a policy denial of an otherwise-valid payment. Servers that
distinguish a successfully-redeemed payment from a subsequent
policy denial of access MUST use HTTP 403 with no challenge, per
{{I-D.httpauth-payment}}.

Example error response body:

~~~json
{
  "type": "https://paymentauth.org/problems/cashu/amount-mismatch",
  "title": "Amount Mismatch",
  "status": 402,
  "detail": "Presented token value does not equal amount plus swap fee"
}
~~~

# Security Considerations

## Client-Side Verification {#security-client}

Before presenting a token, a client MUST verify the challenge
independently rather than trusting the server's `amount` and
`currency` auth-params: it MUST decode `methodDetails.request`
({{NUT-18}}) and confirm the amount and unit it encodes match the
`amount` and `currency` fields, and MUST confirm
`methodDetails.mints` contains a mint it trusts and can obtain a
token from. A client that skips these checks can be induced to pay
a different amount, in a different unit, or against an
attacker-substituted mint set. These checks are stated normatively
in the Request Schema and restated here as a security requirement,
as the sibling method specifications do.

## Token Replay {#security-replay}

Replay protection for the "cashu" method lives at the proof level.
A presented token is single-use: redeeming it swaps ({{NUT-03}})
its proofs, after which the mint marks them spent and refuses any
further swap. A second presentation of the same token therefore
fails verification at the swap step. Servers MUST treat swap
success as consume-once: the swap and the decision to return HTTP
200 MUST be atomic, so that concurrent requests presenting the
same token result in exactly one success and one rejection, with
no window in which both are accepted (see {{durability}}).

## Challenge Binding

The token's single-use property protects the token, but not the
challenge: absent binding, a token valid for one challenge could
be presented against a different one. Servers MUST bind the
`request` parameters to the challenge `id` (see {{verification}})
and SHOULD use the HMAC-SHA256 binding of {{I-D.httpauth-payment}}
for stateless verification. A server that neither binds nor stores
its challenges cannot detect a token redirected from another
challenge instance and MUST NOT be considered conformant.

## DLEQ Verification {#security-dleq}

The security-relevant DLEQ check {{NUT-12}} is on the blind
signatures the mint RETURNS from the swap, not on the input proofs
the client presents. Servers MUST verify the DLEQ proofs on the
swap-returned signatures and SHOULD reject a mint that omits them:
without that check a malicious mint could make the server report a
successful charge for output proofs it never validly signed, which
the server then cannot spend.

The server does not verify DLEQ on the presented input proofs:
input-proof DLEQ exists to let an offline party verify ecash
without contacting the mint, but here the server swaps the token
immediately and the swap itself proves the inputs were validly
signed and unspent, so an input-proof DLEQ check would add nothing.
It is also frequently stripped from `cashuB` tokens, so requiring it
would needlessly reject valid tokens.

## Amount and Fee Determinism {#security-fees}

The "cashu" charge is exact-amount. The server MUST verify that the
token's total value equals `amount + expected_swap_fee` exactly
(see {{fees}}), rejecting both over- and under-funded tokens, and
MUST perform this check before the swap. The swap fee is
deterministic: it is a pure function of the presented proofs'
keysets and the per-keyset `input_fee_ppk` published by the mint
{{NUT-02}}, so the holder computes the same `amount + swap_fee` the
server will check, and the server recomputes the fee from the
proofs it actually received rather than trusting any client-
supplied value. Pre-funding the fee on the holder side keeps the
whole-token redemption and the exact net `amount` mutually
satisfiable, which a naive "present exactly `amount`" rule would
not be at any keyset with `input_fee_ppk > 0`. Where the swap fee
is large relative to `amount` the charge remains satisfiable but
uneconomic; servers SHOULD price `amount` well above the swap fee
of the mints they accept (see {{fees}}).

## Keyset Rotation and Expiry

A `final_expiry` boundary {{NUT-02}} can fall between the
last structural check (step 12) and the swap (step 13): a token that
passes verification is not guaranteed to swap, because the mint
enforces keyset retirement and `final_expiry` at swap time. Servers
MUST treat a swap rejected for keyset retirement or passed
`final_expiry` as `payment-expired`, distinct from the
double-spend, disallowed-mint, and bad-DLEQ cases that are
`verification-failed`. Output proofs are blinded against the unit's
ACTIVE keyset (see {{settlement}}), so the server holds spendable
proofs even when the input keyset is on the verge of retiring.

## Mint Trust

The server trusts the mints it lists in `methodDetails.mints`: a
listed mint custodies the value the server redeems and could,
in principle, refuse to honor a swap or rotate its keyset early.
Servers MUST choose the mint set and SHOULD identify mints by
mint identity key {{NUT-01}} rather than by URL, so that a
DNS or URL takeover cannot substitute an untrusted mint. Clients
likewise rely on the listed mints to honor the tokens they hold.

## Privacy

Cashu tokens are bearer instruments carrying no payer identity,
and the mint's blind signatures {{NUT-00}} unlink a token's
redemption from its issuance. The exact-amount model adds to this:
because the holder splits its token locally before presenting and
presents only an exact-amount token, neither the server nor the
mint observes the remainder or its secrets, and the server learns
that one charge was paid without learning the size of the holder's
remaining balance. The server still observes that a redemption
occurred — the blind signatures hide the link to issuance, not the
redemption itself. Implementations MUST NOT log token secrets, and
MUST use the token hash, not the token, as a receipt reference (see
{{receipt}}).

## Denial of Service {#security-dos}

A token carrying a very large number of proofs inflates both
verification cost and the swap fee. Servers SHOULD bound the number
of proofs they accept in a single token; this bound is a
server-internal limit and is not advertised in the challenge (which
carries no proof-count field), so a client learns of it only when an
over-large token is rejected. Servers SHOULD also rate-limit
challenge issuance and credential-verification attempts per
{{I-D.httpauth-payment}}.

## Transport Security

All communication MUST use TLS per {{I-D.httpauth-payment}}.
A Cashu token is a bearer credential: any party that observes it
in transit before it is redeemed can redeem it themselves.
Credentials MUST only be transmitted over HTTPS, and servers MUST
redeem a presented token promptly to minimize the window in which
an intercepted token could be spent by an attacker.

# IANA Considerations

## Payment Method Registration

This document requests registration of the following entry in
the "HTTP Payment Methods" registry established by
{{I-D.httpauth-payment}}:

| Method Identifier | Description | Reference |
|-------------------|-------------|-----------|
| `cashu` | Cashu (Chaumian ecash) bearer token payment | This document |

Contact: TODO (<todo@example.com>)

## Payment Intent Registration

This document requests registration of the following entry in
the "HTTP Payment Intents" registry established by
{{I-D.httpauth-payment}}:

| Intent | Applicable Methods | Description | Reference |
|--------|-------------------|-------------|-----------|
| `charge` | `cashu` | One-time exact-amount Cashu token payment gating access to a resource | This document |

--- back

# Examples

## Initial Request and 402 Challenge

~~~http
GET /weather HTTP/1.1
Host: api.example.com

HTTP/1.1 402 Payment Required
WWW-Authenticate: Payment id="kM9xPqWvT2nJrHsY4aDfEb",
  realm="api.example.com",
  method="cashu",
  intent="charge",
  request="eyJhbW91bnQiOiIxMDAiLCJjdXJyZW5jeSI6InNhdCIsImRlc2NyaXB0aW9uIjoiV2VhdGhlciByZXBvcnQgZm9yIDk0MTA3IiwibWV0aG9kRGV0YWlscyI6eyJtaW50cyI6WyJodHRwczovL21pbnQuZXhhbXBsZS5jb20iXSwicmVxdWVzdCI6ImNyZXFBLi4uIn19",
  expires="2026-03-15T12:05:00Z"
Cache-Control: no-store
~~~

Decoded `request`:

~~~json
{
  "amount": "100",
  "currency": "sat",
  "description": "Weather report for 94107",
  "methodDetails": {
    "mints": ["https://mint.example.com"],
    "request": "creqA..."
  }
}
~~~

## Retry with Credential

~~~http
GET /weather HTTP/1.1
Host: api.example.com
Authorization: Payment eyJjaGFsbGVuZ2UiOnsiaWQiOiJrTTl4UHFXdlQybkpySHNZNGFEZkViIiwicmVhbG0iOiJhcGkuZXhhbXBsZS5jb20iLCJtZXRob2QiOiJjYXNodSIsImludGVudCI6ImNoYXJnZSIsInJlcXVlc3QiOiJleUouLi4iLCJleHBpcmVzIjoiMjAyNi0wMy0xNVQxMjowNTowMFoifSwic291cmNlIjoiZGlkOmtleTp6Nk1raGFYZ0JaRHZvdERrTDUyNTdmYWl6dGlHaUMyUXRLTEdwYm5uRUd0YTJkb0siLCJwYXlsb2FkIjp7ImNhc2h1X3Rva2VuIjoiY2FzaHVCcEdGMGdhSmhhVWdBLi4uIn19

HTTP/1.1 200 OK
Payment-Receipt: eyJjaGFsbGVuZ2VJZCI6ImtNOXhQcVd2VDJuSnJIc1k0YURmRWIiLCJleHRlcm5hbElkIjoib3JkZXJfMTIzNDUiLCJtZXRob2QiOiJjYXNodSIsInJlZmVyZW5jZSI6IjliNzFkMjI0YmQ2MmYzNzg1ZDk2ZDQ2YWQzZWEzZDczMzE5YmZiYzI4OTBjYWFkYWUyZGZmNzI1MTk2NzNjYTciLCJzdGF0dXMiOiJzdWNjZXNzIiwidGltZXN0YW1wIjoiMjAyNi0wMy0xMFQyMTowMDowMFoifQ
Cache-Control: private
Content-Type: application/json

{"temperature": 72, "condition": "sunny"}
~~~

Decoded credential:

~~~json
{
  "challenge": {
    "id": "kM9xPqWvT2nJrHsY4aDfEb",
    "realm": "api.example.com",
    "method": "cashu",
    "intent": "charge",
    "request": "eyJ...",
    "expires": "2026-03-15T12:05:00Z"
  },
  "source": "did:key:z6MkhaXgBZDvotDkL5257faiztiGiC2QtKLGpbnnEGta2doK",
  "payload": {
    "cashu_token": "cashuBpGF0gaJhaUgA..."
  }
}
~~~

Decoded receipt:

~~~json
{
  "challengeId": "kM9xPqWvT2nJrHsY4aDfEb",
  "externalId": "order_12345",
  "method": "cashu",
  "reference": "9b71d224bd62f3785d96d46ad3ea3d73319bfbc2890caadae2dff72519673ca7",
  "status": "success",
  "timestamp": "2026-03-10T21:00:00Z"
}
~~~

## Service-Defined Unit

The same exchange with a service-defined unit: only the `currency`
and the unit encoded in the payment request differ. Here the mint's
keyset for the unit is fee-free (`input_fee_ppk = 0`, the operator's
choice), so the presented token is worth exactly the requested
`amount`; against a fee-bearing keyset it would be `amount + swap_fee`
(see {{fees}}). The token is verified and redeemed identically; any
backing or expiry the unit carries is enforced by the mint at swap
time and is transparent to this method.

Decoded `request`:

~~~json
{
  "amount": "1",
  "currency": "credits",
  "methodDetails": {
    "mints": ["https://mint.example.com"],
    "request": "creqA..."
  }
}
~~~

# Acknowledgements

The authors thank the Cashu developer community for the NUT
specifications this method builds on, and Brendan Ryan and the
Tempo Labs team for the Payment HTTP Authentication framework.
