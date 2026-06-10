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
  RFC3986:
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
  NUT-13:
    title: "NUT-13: Deterministic secrets"
    target: https://github.com/cashubtc/nuts/blob/main/13.md
    author:
      - org: Cashu
    date: 2024
  NUT-24:
    title: "NUT-24: HTTP 402 Payment Required"
    target: https://github.com/cashubtc/nuts/blob/main/24.md
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
{{NUT-18}} as a challenge; the client presents a Cashu token as the
credential. The server verifies the token and redeems it by
swapping {{NUT-03}} it at the issuing mint.

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

The flow proceeds as follows:

~~~
   Client                        Server                          Mint
      |                             |                              |
      |  (1) GET /resource          |                              |
      |-------------------------->  |                              |
      |                             |                              |
      |  (2) 402 Payment Required   |                              |
      |      (paymentRequest)       |                              |
      |<--------------------------  |                              |
      |                             |                              |
      |  (3) Swap token to exact    |                              |
      |      amount (local)         |                              |
      |--------------------------------------------------------->  |
      |  (4) Exact-amount token     |                              |
      |<---------------------------------------------------------  |
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

NUT-24 {{NUT-24}} binds the same NUT-18 {{NUT-18}} payment request to
HTTP 402 over an `X-Cashu` header; this method binds it over the
standard `Authorization`/`WWW-Authenticate` framework instead. The
embedded `creqA` is byte-identical, so one Cashu code path can serve
both.

## Relationship to the Charge Intent

This document inherits the shared request semantics of the "charge"
intent from {{I-D.payment-intent-charge}}. It defines only the
Cashu-specific `methodDetails`, `payload`, verification, and
settlement procedures for the "cashu" payment method.

# Requirements Language

{::boilerplate bcp14-tagged}

# Terminology

Cashu Token
: An ecash token (a `cashuB...` string, the NUT-00
  TokenV4 serialization) encoding one or more proofs issued by a
  single mint under a single unit, the mint's URL, and that unit.
  The authoritative value carried by the credential.

Mint
: The Cashu issuer that blind-signs proofs and swaps {{NUT-03}} them
  under its keysets {{NUT-02}}. Each token names the single mint that
  issued it.

Payment Request
: A Cashu payment request {{NUT-18}} (a `creqA...` string)
  encoding the amount, unit, acceptable mints, any spending
  condition, and single-use flag a payer must satisfy. It is a
  self-contained, mint-agnostic request artifact; all payment
  parameters are derived from it.

Proof
: A single unit of ecash: an `{amount, id, secret, C}` tuple
  {{NUT-00}} signed by a mint keyset. A token is a set of proofs.

Swap
: The mint operation {{NUT-03}} that exchanges a set of input
  proofs for a set of freshly blind-signed output proofs. A
  successful swap marks the inputs spent and is the act of
  redemption.

Swap Fee
: The NUT-03 input fee a swap deducts from the input proofs (see
  {{fees}}). Deterministic from the input proofs' keysets, so it can
  be computed offline before a token is presented.

# Intent Identifier

The intent identifier for this specification is "charge"
(lowercase, per {{I-D.httpauth-payment}}).

# Intent: "charge"

The "charge" intent represents a one-time payment gating access to
a resource. The server advertises a Cashu payment request
({{NUT-18}}) naming an exact amount and unit per request. As the
credential, the client presents a Cashu token whose value, after
the swap fee the mint will deduct, covers that amount.
The server verifies the token and redeems it by swapping
({{NUT-03}}) it at the issuing mint; a successful swap both proves
the token unspent and transfers its value to the server.

The server redeems the whole token and makes no change: the holder
pre-funds the swap fee, presenting a value of at least
`amount + swap_fee`, and any excess is retained by the server (see
{{fees}}). A holder of a larger token splits it locally first (see
{{settlement}}); the remainder is never seen by the server.

## Fees {#fees}

A NUT-03 swap deducts a deterministic input fee set by the input
proofs' keyset(s):
`swap_fee = ceil(sum_over_proofs(input_fee_ppk) / 1000)`, where
`input_fee_ppk` is published per keyset in the mint's keyset list
({{NUT-02}}, {{NUT-03}}). The sum runs over proofs, not distinct
keysets, and is computable from the presented proofs alone, so
holder and server arrive at the same value.

Because the server makes no change, the holder pre-funds the fee:
the presented token's total value MUST be at least
`amount + swap_fee`. The server recomputes `swap_fee` from the
presented proofs ({{verification}}, step 7) and never trusts a
client-supplied value. Value beyond the requirement is accepted
and retained, so a client SHOULD present the exact total.

# Encoding Conventions {#encoding}

All JSON {{RFC8259}} objects carried in auth-params or
HTTP headers in this specification MUST be serialized using the JSON
Canonicalization Scheme (JCS) {{RFC8785}} before encoding. JCS
produces a deterministic byte sequence, which is required for
any digest or signature operations defined by the base spec
{{I-D.httpauth-payment}}.

The resulting bytes MUST then be encoded using base64url
{{RFC4648}} Section 5 without padding characters (`=`).
Implementations MUST NOT append `=` padding when encoding. A
padded header value is malformed under the framework's grammar
({{I-D.httpauth-payment}}); the cashu artifacts carried inside
JSON strings (`creqA...`, `cashuB...`) MUST be accepted with or
without padding, as their own encodings allow ({{NUT-18}},
{{NUT-00}}).

This encoding convention applies to: the `request`
auth-param in `WWW-Authenticate`, the credential in
`Authorization`, and the receipt in
`Payment-Receipt`.

The Cashu payment request (`methodDetails.paymentRequest`) and the Cashu
token (`payload.token`) are opaque string values within the
JCS-canonical `request` object; their own internal encoding
({{NUT-18}}, {{NUT-00}}) is never canonicalized; JCS conformance is
a property of the enclosing object only.

# Request Schema

## Shared Fields

The `request` auth-param of the `WWW-Authenticate: Payment`
header contains a JCS-serialized, base64url-encoded JSON object
(see {{encoding}}). The following shared fields are
included in that object:

amount
: REQUIRED. The amount charged, in the base units of the Cashu
  unit, as a canonical decimal string (e.g., "100"), a positive
  integer per the charge intent ({{I-D.payment-intent-charge}}).
  This is the amount the server nets after the swap; it MUST equal
  the `a` value encoded in `methodDetails.paymentRequest`,
  compared as integers. The
  presented token value is `amount + swap_fee` (see {{fees}}).

currency
: REQUIRED. The Cashu unit string {{NUT-00}} the presented token
  MUST carry (e.g., "sat"). It MUST equal the unit encoded in
  `methodDetails.paymentRequest`. This is a method-defined currency
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

The following field is nested under `methodDetails` in the
request JSON. The Cashu payment request
(`methodDetails.paymentRequest`) is the authoritative source for
all payment parameters, including the set of mints whose tokens
the server accepts for this challenge. Clients MUST decode and
verify the payment request independently before presenting, and
MUST reject challenges where `amount` or `currency` do not match
the values encoded in the payment request, or whose single-use
flag is not true.

paymentRequest
: REQUIRED. The Cashu payment request string ({{NUT-18}}, a
  `creqA...` value). This field is
  authoritative; all payment parameters
  (amount, unit, accepted mints, spending conditions, single-use
  flag, optional description) are derived from it. The
  `a` (amount), `u` (unit), and `m` (mints) fields are OPTIONAL in
  {{NUT-18}}, but this method REQUIRES all three: a server MUST
  encode `a` and `u`, whose values back the `amount` and `currency`
  checks above, and MUST populate `m` with a non-empty set of
  accepted mint URLs (compared after canonicalization, see
  {{mint-trust}}). A client MUST reject a challenge whose payment
  request omits any of them. The payment request's transport set
  MUST be empty, which {{NUT-18}} defines as in-band:
  the credential is returned over the same HTTP channel in the
  `Authorization` header rather than over a separate transport. The
  server MAY set the request's `nut10` field to a spending
  condition it can satisfy at redemption ({{NUT-10}}); the payer
  then locks the presented proofs to it, and the server supplies
  the witness at the swap ({{verification}}, step 8). The
  single-use flag MUST be true: a challenge identifies one
  payment. The payment id (`i`) SHOULD be
  omitted: the challenge `id` identifies the payment, and under
  stateless binding the `id` is computed over the `request` bytes,
  so no embedded value can equal it.

# Credential Schema

The `Authorization` header carries a single base64url-encoded
JSON token (no auth-params). The decoded object contains two
top-level fields:

challenge
: REQUIRED. An echo of the challenge auth-params from the
  `WWW-Authenticate` header: `id`, `realm`, `method`, `intent`,
  `request`, and, if present in the challenge, `digest`, `opaque`,
  `description`, and `expires`. This binds the credential to the
  exact challenge that was issued. A client MUST echo each of
  these fields unchanged when the server included it (see
  {{verification}}).

source
: OPTIONAL. A payer identifier string, as defined by
  {{I-D.httpauth-payment}}. The RECOMMENDED format
  is a Decentralized Identifier (DID) per
  {{W3C-DID}}. Cashu tokens carry no payer identity;
  implementations MAY omit this field, and servers MUST NOT
  require it.

payload
: REQUIRED. A JSON object containing the Cashu-specific credential
  fields. The single required field is `token`: the Cashu
  token ({{NUT-00}}, a `cashuB...` string) whose value, net of the
  swap fee, covers the requested amount (see
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
    "token": "cashuBpGF0gaJhaUgA..."
  }
}
~~~

# Verification Procedure {#verification}

Upon receiving a request with a credential, the server MUST:

1. Decode the base64url credential and parse the JSON.
2. Verify that `payload.token` is present, is a string, and
   decodes as a Cashu token ({{NUT-00}}). The token MUST be a
   `cashuB...` (TokenV4) serialization carrying at least one
   proof; a `cashuA...` (TokenV3) string MUST be rejected, as MUST
   a token that does not parse. A valid TokenV4 structurally
   carries exactly one mint and one unit. Servers SHOULD reject a
   token carrying more than a configured maximum number of proofs
   (see {{security-dos}}).
3. Authenticate and validate the echoed challenge per
   {{I-D.httpauth-payment}}: recover the challenge (under
   stateless operation, RECOMMENDED, recompute the `id`-HMAC over
   the echoed `credential.challenge` with the server key; under
   stored operation, look it up by `credential.challenge.id`) and
   verify its field consistency, body `digest` ({{RFC9530}}), and
   `expires` freshness. A
   tampered or inconsistent challenge MUST be rejected as
   `invalid-challenge`; a `credential.challenge.expires` in the past
   is a `payment-expired` condition (see {{errors}}). Because a
   Cashu `creqA` carries no expiry of its own, the `expires`
   auth-param is the sole challenge-expiry signal.
4. Verify the token's unit equals `currency`.
5. Verify the token's mint is a member of the payment request's
   mint set (`m`), comparing by canonicalized mint URL (see
   {{mint-trust}}). A token whose mint is not a member is rejected
   as `verification-failed`.
6. Resolve every proof's keyset id against the mint's published
   keysets {{NUT-02}}. When a proof uses a short keyset id, the
   server MUST resolve it to the full keyset via the mint's fetched
   keyset list and MUST reject a short id that is ambiguous or does
   not resolve (see {{short-keyset}}). Verify each resolved
   keyset's unit equals `currency`: the token's declared unit is
   client-supplied data, the keyset is the authority; a proof whose
   keyset belongs to a different unit is rejected as
   `verification-failed`.
7. Compute `expected_swap_fee` over the resolved keyset(s) of the
   presented proofs ({{fees}}) and verify the token's total value
   is at least `amount + expected_swap_fee`. A token worth less
   MUST be rejected as `payment-insufficient` ({{errors}}); value
   above the requirement is accepted and retained. The server
   makes no change either way.
8. Swap ({{NUT-03}}) the whole token at the configured URL of the
   mint-set entry matched in step 5 (see {{settlement}}). When the
   presented proofs carry the spending condition the payment
   request advertised, the server supplies the corresponding
   witness data in the swap inputs ({{NUT-10}}). A successful swap
   is the redemption step and is the authoritative check that the
   input proofs were genuine, validly signed, unspent, and that
   any spending condition was satisfied; a rejected swap consumes
   nothing ({{NUT-03}}), an atomicity this procedure relies on.
   The server SHOULD verify the DLEQ proofs {{NUT-12}} on the
   blind signatures the swap returns; a failure indicates a
   misbehaving mint, not a payment failure, and the payment stands
   ({{settlement}}). A rejected swap, whether for a spent proof,
   an unsatisfied spending condition, a retired keyset or passed
   `final_expiry` {{NUT-02}}, or any other cause, is a
   `verification-failed` condition: the token was not redeemed
   (see {{settlement}}, {{errors}}).

Steps 4 through 7 are structural and MUST be performed before the
network swap in step 8, so a structurally invalid token never
reaches the swap. The keyset resolution of step 6 MAY require
fetching the mint's keysets {{NUT-02}} first. The expected values
these steps check (`currency`, the mint set, `amount`) are
derived from the authenticated challenge's embedded payment
request, the authoritative artifact (see Method Details); the
top-level auth-params were already required to match it at
challenge construction.

These steps satisfy the "charge" intent's verification
responsibilities ({{I-D.payment-intent-charge}}): challenge-match and
freshness in step 3, payment-proof verification (the swap itself)
in steps 4-8, and amount-match in step 7 (read as the NET settled
amount, per {{fees}}). Recipient-match is implicit: redemption swaps
the token to the server itself, so there is no distinct recipient and
`recipient` is omitted.

## Challenge Binding

To prevent token replay across different resources or challenges,
the server MUST bind the issued `request` parameters to the
challenge `id` and verify, when a credential is presented, that
`credential.challenge` is an exact echo of an issued challenge.
The server SHOULD compute `id` as the HMAC-SHA256 binding defined
by {{I-D.httpauth-payment}} so that binding is stateless;
alternatively the server MAY store issued challenges and verify by
lookup. Under stateless operation a presented credential MUST echo
each challenge auth-param byte-for-byte as issued, in particular
the `request` string, used byte-as-issued (a decode/re-encode
cycle changes the bytes and the recomputation fails). The
challenge carries the framework-REQUIRED auth-params (`id`,
`realm`, `method`, `intent`, `request`). A server operating
statelessly MUST include `expires`: a stateless challenge has no
server-side state to expire it, so one issued without `expires`
never lapses, stays presentable indefinitely, and pins stale
pricing. Under stored operation `expires` remains RECOMMENDED.
When the charge gates a request with a body the server SHOULD
include
`digest` and SHOULD include any `opaque` correlation data it needs
echoed.

The single-use flag inside the embedded `creqA` ({{NUT-18}}) is
always true for this method (see Method Details), but replay
protection does not depend on it: it comes from challenge binding
(above) together with the proof-level single-use property of the
redeemed token (see {{security-replay}}).

## Short Keyset Identifiers {#short-keyset}

When a presented proof uses a short (version-`01`) keyset id, the
server MUST resolve it to the full keyset against the mint's
published keyset list ({{NUT-02}}); resolution is required to
compute the swap fee ({{fees}}) and construct correct swap outputs.
A short id that resolves to no keyset, or is ambiguous across the
mint's keysets, MUST be rejected as `verification-failed`. A
failure to fetch the keyset list at all (a network error reaching
the mint) is distinct: the server answers HTTP 503 with the token
NOT consumed (see {{errors}}), not `verification-failed`.

# Settlement Procedure {#settlement}

Settlement is the mint swap ({{NUT-03}}) of the presented token.
The server swaps the whole token for fresh proofs it controls;
holding those proofs is settlement. The mint deducts the swap fee
({{fees}}) from the inputs, so the server's output proofs sum to
at least `amount`. The server's outputs are blinded against the
mint's currently ACTIVE keyset for the unit ({{NUT-02}}), which MAY
differ from the keyset(s) that signed the input proofs. Cashu
settlement is final once the swap succeeds: the input proofs are
spent and cannot be restored. The server makes no change and
returns no proofs to the client.

A successful swap is destructive: the input proofs are consumed
whether or not the server retains the result. The server MUST be
able to reconstruct its swap outputs if it crashes between sending
the swap and durably storing the returned signatures: either by
persisting the blinded output secrets before sending the swap, or
by deriving them deterministically ({{NUT-13}}) and recovering the
mint's response via restore ({{NUT-09}}, which covers interrupted
swaps). A server that cannot do so destroys the redeemed value on
a crash: the mint has recorded the inputs as spent, and no party
can recover the outputs.

A swap whose request was transmitted but whose result was not
received leaves consumption unknown. The server MUST resolve such
an outcome before honoring any further presentation of the same
challenge: restore its own swap outputs ({{NUT-09}}), which the
reconstruction requirement above makes always possible (if the
mint returns signatures, the original swap succeeded and the
payment is complete), or check the input proofs' spent state
({{NUT-07}}). Until resolved, the server answers
HTTP 503 ({{errors}}).

The client presents a token worth `amount + swap_fee` (value
beyond that is not returned, so there is no reason to present
more). If it does not already hold matching denominations, it
first swaps
({{NUT-03}}) at the mint to produce them; the remainder of that
local split stays with the client, and neither the mint nor the
server learns the remainder's secrets. The split happens before
the `Authorization` request; the server never performs it.

## Consume-Once and Resource Delivery

The server MUST treat the redemption (the swap) and the decision to
return HTTP 200 as a single operation: a challenge whose token has
been redeemed MUST NOT be accepted again, even if resource delivery
subsequently fails. A server using stored challenges records
consumption only upon, and atomically with, swap success; a
challenge whose swap never succeeded remains presentable
(HTTP 503, {{errors}}). Once the swap succeeds the
payment is complete:
the server MUST NOT respond with a payment-failure status (402) or
issue a fresh challenge for a condition detected after a successful
swap. If resource delivery fails after the
token is redeemed, the server MUST return an appropriate HTTP error
(e.g., 500) and MUST NOT reissue the same challenge. The client
MUST treat such a response as a payment loss and MAY retry with a
new token. Cashu settlement is final once the swap succeeds; the
redeemed token cannot be refunded by the server.

The same loss mode applies when a success response is lost in
transit: a later re-presentation of the same credential is a new
request against an already-consumed challenge and fails: under
stateless operation at the swap, as a spent token
(`verification-failed`); under stored operation at the challenge
lookup (`invalid-challenge`, {{errors}}). The protocol provides
no replay. A client can confirm what happened with a proof-state
check ({{NUT-07}}).

A server that implements the framework's optional
`Idempotency-Key` ({{I-D.httpauth-payment}}) MUST perform the
idempotency lookup before the verification procedure: a redeemed
token cannot be re-verified, so a verify-then-replay
implementation never replays.

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
  exact `token` credential string received from the client
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

A malformed REQUEST, as opposed to a failed payment, follows the
framework's status handling rather than returning 402: a credential
naming an unsupported method yields `method-unsupported` (HTTP 400),
and a request bearing more than one `Authorization: Payment`
credential is rejected with HTTP 400 per {{I-D.httpauth-payment}}.
The 402 problem types below are scoped to payment-verification
failures.

Payment-verification failures surface as the framework's
registered problem types, with the cashu-specific causes that map
to each:

https://paymentauth.org/problems/malformed-credential
: HTTP 402. The credential could not be decoded, the JSON
  could not be parsed, required fields (`challenge`, `payload`,
  `payload.token`) are absent or have the wrong type,
  `token` does not decode as a Cashu token, the token is a
  `cashuA...` (TokenV3) serialization, the token carries zero
  proofs, or it exceeds the server's proof-count bound
  ({{security-dos}}). A fresh challenge
  MUST be included in `WWW-Authenticate`.

https://paymentauth.org/problems/invalid-challenge
: HTTP 402. The value of `credential.challenge.id` does not match
  any challenge issued by this server (stored operation), or
  `credential.challenge` is not an exact echo of an issued
  challenge (including a `digest` that does not match the request
  body) or fails its `id`-HMAC (stateless operation). Under stored
  operation this also covers a challenge already consumed; under
  stateless operation a replayed token is instead caught at the
  swap as a spent token (`verification-failed`). A fresh challenge
  MUST be included in `WWW-Authenticate`.

https://paymentauth.org/problems/payment-expired
: HTTP 402. The challenge `expires` auth-param echoed in the
  credential is in the past. The challenge is stale, not the
  token: the client SHOULD re-present the same token against the
  fresh challenge, which MUST be included in `WWW-Authenticate`.

https://paymentauth.org/problems/payment-insufficient
: HTTP 402. The token's total value is less than
  `amount + swap_fee` (see {{fees}}); the server makes no change
  and the token is not redeemed. A fresh challenge MUST be
  included in `WWW-Authenticate`. There is no over-payment
  counterpart: value above the requirement is accepted and
  retained ({{fees}}).

https://paymentauth.org/problems/verification-failed
: HTTP 402. The token failed a non-amount verification check: its
  unit does not equal `currency`, its mint is not in the payment
  request's mint set, a proof uses an unresolvable or ambiguous
  short keyset id, a resolved keyset's unit differs from
  `currency`, or the mint rejected the swap, whether for a spent
  proof, an unsatisfied spending condition, a retired keyset or
  passed `final_expiry` {{NUT-02}}, or any other cause
  (verification step 8). The server SHOULD name the cause in the
  problem `detail`. A client can distinguish an expired keyset
  locally, since a wallet knows its proofs' keysets and their
  `final_expiry`; such proofs can no longer be swapped, and
  payment requires proofs from an active keyset. A fresh
  challenge MUST be included in `WWW-Authenticate`.

Mint unreachability is an infrastructure failure, not a
payment-verification outcome, and carries no problem type: when
the mint cannot be reached, or a swap was sent but its outcome is
unknown, the server MUST answer HTTP 503 (Service Unavailable)
and SHOULD include a `Retry-After` header. If the swap request
was never transmitted (DNS, connect, or TLS failure), the token
is NOT consumed and the client MAY retry the same token. If the
swap was transmitted but its result was not received, consumption
is unknown: the server MUST NOT claim the token unconsumed and
MUST resolve the outcome before acting further on the same
challenge (see {{settlement}}).

A token whose mint is reachable but is not in the payment
request's mint set, or whose unit is otherwise disallowed by
server policy, is a `verification-failed` condition (HTTP 402),
not a policy denial of an otherwise-valid payment. Servers that
distinguish a successfully-redeemed payment from a subsequent
policy denial of access MUST use HTTP 403 with no challenge, per
{{I-D.httpauth-payment}}.

Example error response body:

~~~json
{
  "type": "https://paymentauth.org/problems/payment-insufficient",
  "title": "Payment Insufficient",
  "status": 402,
  "detail": "Presented token value is less than amount plus swap fee"
}
~~~

# Security Considerations

## Client-Side Verification {#security-client}

A client that skips the independent checks required by the
`paymentRequest` definition (Method Details), decoding the
payment request and confirming its amount, unit, and mint set,
can be induced to pay the wrong amount or unit, or to pay toward
an attacker-substituted mint.

After an outcome that never resolved (a timeout, connection
loss, or 5xx after a credential was sent), a client can settle
its token's fate with a proof-state check at the mint
({{NUT-07}}): proofs SPENT mean the server redeemed the token
(the payment happened; whether the resource arrived is the loss
mode of {{settlement}}), proofs UNSPENT mean nothing was redeemed
and the same token remains safe to re-present. Clients SHOULD
perform this check before reusing or writing off a token whose
presentation produced no definite answer.

## Token Replay {#security-replay}

Replay protection for the "cashu" method lives at the proof level.
A presented token is single-use: redeeming it swaps ({{NUT-03}})
its proofs, after which the mint marks them spent and refuses any
further swap. A second presentation of the same token therefore
fails verification at the swap step. Servers MUST treat swap
success as consume-once: the swap and the decision to return HTTP
200 MUST be atomic, so that concurrent requests presenting the
same token result in exactly one success and one rejection, with
no window in which both are accepted.

## Challenge Binding

The token's single-use property protects the token, but not the
challenge: absent binding, a token valid for one challenge could
be presented against a different one, and a server that neither
binds nor stores its challenges cannot detect the redirection.
The normative binding and echo rules live in {{verification}}
(Challenge Binding).

## Keyset Rotation and Expiry

A `final_expiry` boundary {{NUT-02}} can fall between the last
structural check (verification step 7) and the swap (step 8): a
token that passes verification is not guaranteed to swap, because
the mint enforces keyset retirement and `final_expiry` at swap
time; the rejected swap surfaces as `verification-failed`
({{errors}}). A keyset that is merely inactive (no longer the
mint's current signing keyset but not yet retired or past
`final_expiry`) remains valid for redemption; the server MUST NOT
reject a proof for keyset inactivity alone, only for actual
retirement or expiry. Output proofs are blinded against the unit's
ACTIVE keyset (see {{settlement}}), so the server holds spendable
proofs even when the input keyset is on the verge of retiring.

## Mint Trust {#mint-trust}

The server trusts the mints it lists in the payment request: a
listed mint custodies the value the server redeems and could, in
principle, refuse to honor a swap or rotate its keyset early.
Membership is decided by canonicalized mint URL (verification
step 5). Two mint URLs are equal when these transformations
yield identical strings (the list is exhaustive; no further
{{RFC3986}} normalization, such as percent-encoding case changes,
is applied): lowercase the scheme and host (comparing an
internationalized host in its punycode A-label form), drop a
default port (443 for `https`, 80 for `http`), and strip all
trailing slashes (as token serialization does, {{NUT-00}}); the
path and query are otherwise case-sensitive and preserved
verbatim. Clients likewise rely on the listed mints to honor the
tokens they hold.

## Privacy

Cashu tokens carry no payer identity, and the mint's blind
signatures {{NUT-00}} unlink a token's redemption from its
issuance. The local-split model adds to this:
because the holder splits its token locally and presents only what
the charge requires, neither the server nor the
mint observes the remainder or its secrets, and the server learns
that one charge was paid without learning the size of the holder's
remaining balance. The server still observes that a redemption
occurred: the blind signatures hide the link to issuance, not the
redemption itself. The mint, however, can still correlate the
holder's pre-payment split with the redemption moments later by
amount and timing; clients that need to avoid that SHOULD hold
pre-made exact-value tokens.
Implementations MUST NOT log token secrets, and
MUST use the token hash, not the token, as a receipt reference (see
{{receipt}}). The stateless `id`-HMAC key is a server secret and
MUST NOT be logged or shared; its compromise lets an attacker forge
challenges and defeat challenge binding.

## Denial of Service {#security-dos}

A token carrying a very large number of proofs inflates both
verification cost and the swap fee. Servers SHOULD bound the number
of proofs they accept in a single token; this bound is a
server-internal limit and is not advertised in the challenge (which
carries no proof-count field), so a client learns of it only when
an over-large token is rejected as `malformed-credential`.
Servers SHOULD also rate-limit challenge issuance and
credential-verification attempts per {{I-D.httpauth-payment}}.

## Transport Security {#security-transport}

A Cashu token without a spending condition is a bearer
credential: any party that observes it in transit before it is
redeemed can redeem it itself, and challenge binding does not
prevent this (it binds the credential to a challenge, not the
token to a holder). The framework's TLS requirement
({{I-D.httpauth-payment}}) therefore carries the token's full
value, including across TLS-terminating hops such as reverse
proxies. Servers SHOULD redeem a presented token promptly; TLS
and prompt redemption shrink the exposure window rather than
close it.

# IANA Considerations

## Payment Method Registration

This document requests registration of the following entry in
the "HTTP Payment Methods" registry established by
{{I-D.httpauth-payment}}:

| Method Identifier | Description | Reference |
|-------------------|-------------|-----------|
| `cashu` | Cashu (Chaumian ecash) token payment | This document |

Contact: TODO (<todo@example.com>)

## Payment Intent Registration

This document requests registration of the following entry in
the "HTTP Payment Intents" registry established by
{{I-D.httpauth-payment}}:

| Intent | Applicable Methods | Description | Reference |
|--------|-------------------|-------------|-----------|
| `charge` | `cashu` | One-time Cashu token payment gating access to a resource | This document |

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
  request="eyJhbW91bnQiOiIxMDAiLCJjdXJyZW5jeSI6InNhdCIsImRlc2NyaXB0aW9uIjoiV2VhdGhlciByZXBvcnQgZm9yIDk0MTA3IiwibWV0aG9kRGV0YWlscyI6eyJwYXltZW50UmVxdWVzdCI6ImNyZXFBLi4uIn19",
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
    "paymentRequest": "creqA..."
  }
}
~~~

## Retry with Credential

~~~http
GET /weather HTTP/1.1
Host: api.example.com
Authorization: Payment eyJjaGFsbGVuZ2UiOnsiZXhwaXJlcyI6IjIwMjYtMDMtMTVUMTI6MDU6MDBaIiwiaWQiOiJrTTl4UHFXdlQybkpySHNZNGFEZkViIiwiaW50ZW50IjoiY2hhcmdlIiwibWV0aG9kIjoiY2FzaHUiLCJyZWFsbSI6ImFwaS5leGFtcGxlLmNvbSIsInJlcXVlc3QiOiJleUouLi4ifSwicGF5bG9hZCI6eyJ0b2tlbiI6ImNhc2h1QnBHRjBnYUpoYVVnQS4uLiJ9LCJzb3VyY2UiOiJkaWQ6a2V5Ono2TWtoYVhnQlpEdm90RGtMNTI1N2ZhaXp0aUdpQzJRdEtMR3Bibm5FR3RhMmRvSyJ9

HTTP/1.1 200 OK
Payment-Receipt: eyJjaGFsbGVuZ2VJZCI6ImtNOXhQcVd2VDJuSnJIc1k0YURmRWIiLCJleHRlcm5hbElkIjoib3JkZXJfMTIzNDUiLCJtZXRob2QiOiJjYXNodSIsInJlZmVyZW5jZSI6IjliNzFkMjI0YmQ2MmYzNzg1ZDk2ZDQ2YWQzZWEzZDczMzE5YmZiYzI4OTBjYWFkYWUyZGZmNzI1MTk2NzNjYTciLCJzdGF0dXMiOiJzdWNjZXNzIiwidGltZXN0YW1wIjoiMjAyNi0wMy0xMFQyMTowMDowMFoifQ
Cache-Control: private
Content-Type: application/json

{"temperature": 72, "condition": "sunny"}
~~~

Decoded credential (keys in JCS order, as the wire bytes carry
them; see {{encoding}}):

~~~json
{
  "challenge": {
    "expires": "2026-03-15T12:05:00Z",
    "id": "kM9xPqWvT2nJrHsY4aDfEb",
    "intent": "charge",
    "method": "cashu",
    "realm": "api.example.com",
    "request": "eyJ..."
  },
  "payload": {
    "token": "cashuBpGF0gaJhaUgA..."
  },
  "source": "did:key:z6MkhaXgBZDvotDkL5257faiztiGiC2QtKLGpbnnEGta2doK"
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

# Acknowledgements

The authors thank the Cashu developer community for the NUT
specifications this method builds on, and Brendan Ryan and the
Tempo Labs team for the Payment HTTP Authentication framework.
