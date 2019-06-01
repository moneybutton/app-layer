---
brfc: true
title: bsvalias Pay-to-Email
authors:
  - andy (nChain)
  - Ryan X. Charles (Money Button)
version: 1
---
# Pay-to-Email

{{yfm}}

The Pay-to-Email specification extends paymail and the `bsvalias` family of protocols with a modern alternative to the long-deprecated Pay-to-IP functionality of early versions of Bitcoin.

In very early versions of Bitcoin, PAY2IP was a first-class citizen. The scheme featured an interactive message exchange between a _sender_ and a _receiver_ where the sender first requests a public key from the receiver over a direct TCP/IP connection and then constructs and submits transaction to pay to that public key.

It is stated on the [Bitcoin Wiki](https://en.bitcoin.it/wiki/IP_transaction) that IP transactions were deprecated due to their vulnerability to _man-in-the-middle_ attacks; that is to say that when a sender requested a receiver's public key, the sender had no confidence that the response really came from the intended receiver and not some adversary somewhere between the sender and the receiver.

The paymail scheme, by building on top of DNS and HTTPS, mitigates the _man-in-the-middle_ attack. [DNSSEC](https://en.wikipedia.org/wiki/Domain_Name_System_Security_Extensions) deployment is widespread, protecting the [Host Discovery](./02-01-host-discovery.md) process. HTTPS/TLS and the Root CA model protect [Capability Discovery](./02-02-capability-discovery.md) and the `bsvalias` family of protocols.

As an extension to the original PAY2IP protocol, the Pay-to-Email protocol includes the additional step of having the sender send the final, constructed, signed transaction to the receiver's paymail server, transferring responsibility for submitting the transaction to the Bitcoin network to the receiver's service. This is inspired by [BIP-70](https://github.com/bitcoin/bips/blob/master/bip-0070.mediawiki), Money Button's [Simplified Payments Protocol](https://github.com/moneybutton/bips/blob/master/bip-0270.mediawiki), and nChain's _Safe Lightweight SPV_ research.

Wallet developers have expressed significant interest in this approach, as it allows them to keep track of incoming transactions with much more ease than scanning every transaction and every block in the blockchain (or relying on a bloom filtered connection to do the same). Bloom filtering (as implemented in Bitcoin node connections) does not scale particularly well. As Bitcoin scales, Pay-to-Email becomes an invaluable tool in the scaling story of the wallet ecosystem.

## Capability Discovery

paymail services that implement Pay-to-Email advertise their support for the protocol via the `.well-known/bsvalias` document:

```json
{
  "bsvalias": "1.0",
  "capabilities": {
    "{{fm:brfc}}": "https://bsvalias.example.org/{alias}@{domain.tld}/pay-to-email"
  }
}
```

The template values `{alias}` and `{domain.tld}` refer to the components of the receiver's paymail handle `<alias>@<domain>.<tld>` and must be substituted by the client before issuing a request.

## Flow

In the diagram below, the _Payment Destination Discovery_ section represents any `bsvalias` protocol family mechanism for discovering a receiver's preferred output scripts. One such mechanism is described in [Basic Address Resolution](./04-01-basic-address-resolution.md). The specific protocol used to discover a receiver's output scripts is not important, _as long as it is one of the paymail family of protocols_.

```plantuml
boundary "Sender Client" as c
control "Receiver Paymail Service" as svc

activate c
  group Payment Destination Discovery
    c -> svc: Request output scripts
    activate svc
      svc -> c: Output Scripts
    deactivate svc
  end
  c -> svc: POST <pay-to-email URI>\napplication/json\nbitcoin transaction
  activate svc
    svc -> c: 204
  deactivate svc
deactivate c
```

## Sender Request

The `capabilities.{{fm:brfc}}` path of the `.well-known/bsvalias` document returns a URI template. Senders should replace the `{alias}` and `{domain.tld}` template parameters with the values from the receiver's paymail handle and then make an HTTP POST request against this URI.

The body of the POST request _MUST_ have a content type of `application/json` and _MUST_ conform to the following schema:

**TODO: include `txin` transactions and, where available, Merkle proofs for them**

```json
{
  "transactions": [
    {
      "version": 1,
      "locktime": 0,
      "vin": [
        {
          "txid": "aa91522c9909a1c0d2635f30cb921b11af4f1b60c13dd4b74ae930d3093e0380",
          "vout": 74,
          "scriptSig": "473044022044f24f032d218ce2eccd10ae1c9c8773be0c965ca0cff9ce3f17a19236b3474e0220294624f19f70652d82edd30f104e7787f44aee0e0a4270f70a87bca18833c1c4412102e6f260547b8bbf0254c35a2deada1d26b82262e0cb72d6bc8fe1abeca704f293",
          "sequence": 4294967295
        }
      ],
      "vout": [
        {
          "value": 4099986400,
          "n": 0,
          "scriptPubKey": "76a914ca2043b3f87971ccffe9ceaf821e3ae02f5fb0b488ac",
        },
        {
          "value": 10000012240,
          "n": 1,
          "scriptPubKey": "76a914ecc14556bd810ff874b00ac1e10cccb5cb986f1888ac"
        }
      ]
    }
  ]
}
```

* The use of the `transactions` array allows for batched submissions without multiple HTTP requests
* `txid`, `scriptSig` and `scriptPubKey` fields are hex-encoded raw Bitcoin binary.
* `value` fields are integers and represent satoshis

## Receiver Response

### 204 No Content

The transaction was accepted by the receiver's paymail service.

### 400 Bad Request

The request body was malformed. This could be for reasons including but not limited to:

* Malformed JSON
* `scriptSig` or `scriptPubKey` fields that cannot parsed
* `txid` fields that do not represent a SHA-256 hash (for example, too long/too short)
* Any reason for the transaction to be rejected by Bitcoin, for example vout exceeding vin

### 401 Unauthorized

This response status is included in the specification mainly for the discussion point. It is not intended that paymail open up a free-to-use public API for submitting transactions to the Bitcoin network.

paymail implementations may choose to reject transactions that do not pay to keys associated with the paymail handle listed in the requested URL. When rejecting requests, implementations may choose any HTTP status code they like, including `401`.

By using some other code, for example a `204` (No Content), information about the paymail user's public keys may be prevented from leaking. That is to say, a brute-force attack on an unprotected endpoint may try to identify public keys corresponding to a paymail handle simply by replaying every transaction in the blockchain to a given paymail endpoint, monitoring for success/error status codes as it goes. This would be a potential risk to privacy as it would link a user's on-chain activity together.

### 404 Not Found

The paymail handle specified in the URL is not served by this paymail service.
