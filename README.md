# **Trust Token API Explainer**

This document is an explainer for a potential future web platform API that allows propagating trust across sites, using the [Privacy Pass](https://privacypass.github.io) protocol as an underlying primitive.


## Motivation

The web ecosystem relies heavily on building trust signals to detect fraudulent or spammy actors. One common way this is done is via tracking an individual browser’s activity across the web, usually via associating stable identifiers across sites.

Preventing fraud is a legitimate use case that the web should support, but it shouldn’t require an API as powerful as a stable, global, per-user identifier. In third party contexts, merely segmenting users into trusted and untrusted sets seems like a useful primitive that also preserves privacy. This kind of fraud protection is important both for CDNs, as well as for the ad industry which receives a large amount of invalid, fraudulent traffic.

Segmenting users into very coarse sets satisfies other use-cases as well. For instance, sites could use this as a set inclusion primitive in order to ask questions like, “do I have identity at all for this user?” or even do non-personalized cross-site authentication ("Is this user a subscriber?").


## Overview

This API proposes a new per-origin storage area for “Privacy Pass” style cryptographic tokens, which are accessible in third party contexts. These tokens are non-personalized and cannot be used to track users, but are cryptographically signed so they cannot be forged.

When an origin is in a context where they trust the user, they can issue the browser a batch of tokens, which can be “spent” at a later time in a context where the user would otherwise be unknown or less trusted. Crucially, the tokens are indistinguishable from one another, preventing websites from tracking users through them.

We further propose an extension mechanism for the browser to sign outgoing requests with keys bound to a particular token redemption.


## Potential API


### Trust Token Issuance

When an issuer.com context wants to provide tokens to a user (i.e. when the user is trusted), they can use a new Javascript API that invokes Fetch:


```
fetchTrustTokens('/request-tokens').then(...)
```


This API will invoke the [Privacy Pass](https://privacypass.github.io) Issuance protocol:



*   Generate a set of nonces.
*   Blind them and attach them to the HTTP request
*   Send a POST to the provided endpoint

When a response comes back with blind signatures, they will be unblinded, stored, and associated with the unblinded nonces internally in the browser. The pairs of nonces and signatures are trust tokens that can be redeemed later. Raw tokens are never accessible to Javascript. The issuer can store a limited amount of metadata in the signature of a nonce by choosing one of a set of keys to use to sign the nonce and providing a zero-knowledge proof that it signed the nonce using a particular key or set of keys. The browser will verify the proof and may choose to keep or drop the token based on other metadata constraints and limits from the UA.


### Trust Token Redemption

When the user is browsing another site (publisher.com), that site (or issuer.com embedded on that site) can optionally redeem issuer.com tokens to learn something about the trust of a user. One way to do this would be via a new Javascript API:


```
getTrustAttestation([issuer], {refresh-policy: {use-cached, refresh}).then(...)
```


Calling this invokes the Privacy Pass redemption protocol for an issuer, and sends to the issuer the token. The result of this protocol is an object called a Signed Redemption Record (SRR):


```
{
  Redemption timestamp,
  top level origin (publisher.com),
  signature of the above verifiable by well-known public key of the issuer
}
```


The SRR is HTTP-only and Javascript is only able to access/send the SRR via these APIs. It is also cached in new first-party storage accessible only by these APIs for subsequent visits to that first-party.

To mitigate [token exhaustion](#trust-token-exhaustion), a site can only redeem tokens for a particular issuer if they have no cached SRRs from that issuer. An iframe with the issuer origin can refresh the SRR by setting a `refresh-policy` to get a new record and refresh the trust attestation.


### Forwarding Redemption Attestation

Signed Redemption Records are only accessible via a new option to the Fetch API:


```
fetch(<resource-url>, {
  ...
  signedRedemptionRecord: [issuer],
  ...
});
```


The SRR will be added as a new request header `Sec-Signed-Redemption-Record`. This option to Fetch is only usable in the top-level document.


### Extension: Trust-Bound Keypair and Request Signing

An additional extension to the Trust Attestation allows us to ensure the integrity of the SRR, fetch data, and other headers by associating the SRR with a public/private keypair on the browser. This integrity allows the SRR and signed data to be passed around via third parties while preventing manipulation of the data. This is particularly useful in cases like ads and CDNs where the intermediary parties may not be fully trusted. This keypair is bound to the SRR (and the original token redemption) so that the browser can sign arbitrary request data with the private key and transfer trust to the request from the token.

In order to achieve this, the browser generates the public/private keypair at redemption time, and includes the hash of the public key in the redemption request to the token issuer, which is also included in the returned SRR. The keypair is then stored in new first-party storage only accessible via these APIs.

An additional parameter to the Fetch API then allows the browser to include a signature over the SRR, request data, and additional request headers (specified using a new opt-in header), using the browser's private key associated with the SRR:


```
fetch(<resource-url>, {
  ...
  signedRedemptionRecord: [issuer],
  signRequestData: <include, omit, headers-only>,
  headers: new Headers('Signed-Headers', '"sec-signed-redemption-record", "referer"')
  ... 
});
```


If `signRequestData` is `include`, then the browser will sign over the request data with the private key associated with the Signed Redemption Record. It will sign over the entire URL, POST body, public key, and any headers included in the `Signed-Headers` request header. If `signRequestData` is `headers-only`, we will only sign over those in `Signed-Headers`. Additionally we will limit the size of POST bodies that can be sent via this API when signed, to prevent having to buffer the entire body in memory. To do this, we can (aligning as close as possible to the model in the [Signed Exchanges](https://wicg.github.io/webpackage/draft-yasskin-http-origin-signed-responses.html#rfc.section.3.2) spec) create a canonical [CBOR](https://cbor.io) representation of the resource URL, public key, and headers:


```
{
  'url': 'https://example.test/subresource',
  'sec-signed-redemption-record':  <SRR>,
  'referer': 'https://example.test/',
  'public-key': <pk>
}
```


The browser will add a new request header with the resulting signature over a context string and CBOR data (`"Trust-Token-v1"||CBOR data)`, along with the public key. Something like:


```
Sec-Signature:
  public-key=<pk>
  sig=<signature>
  sign-request-data=<include, headers-only>
  timestamp=<high resolution client timestamp>
```


The canonical CBOR data (verifiable by the signature) should be computable from a request, and so does not need to be sent over the wire from the browser. The `Signed-Headers` header, and the value of `sign-request-data` should be enough to re-construct it server side, robust to things like header re-ordering, etc.


### Extension: Private Metadata

In addition to attesting trust in a user, an issuer may want to provide a limited amount of private metadata in the token (and forward it as part of the SRR) to provide limited information about the token to themselves and partners without revealing metadata to the client. This could be used as a negative indicator of trust or other limited information that the client shouldn't know about.

This can be managed by having a set of keys that sign the token at issuance, with one key being used to indicate the bit of metadata is true, while a different key is used to indicate the bit of metadata is false. The zero-knowledge proof returned during the token issuance then proves that one of two keys was used to sign the token, without revealing which key was actually used. At redemption time, the issuer can then check which of the two keys was used to retrieve the value of the private metadata.

Then by packaging the private metadata in the SRR as a signature and another zero-knowledge proof of signing by one of a small set of keys, the client is able to verify exactly how many bits of private information are contained in the SRR.

This small change opens up a new application for Privacy Passes: embedding small amounts (e.g. 1 bit) of information that is hidden from the client. This increases the rate of cross-site information transfer somewhat, but introduces new use-cases for passes like marking bad clients, so _distrust_ can propagate across sites as well as trust. Private metadata makes it possible to mask a decision about whether traffic is fraudulent, and increase the time it takes to reverse-engineer detection algorithms. This is because distrusted clients would still be issued tokens, but with the private distrusted bit set.


## Privacy Considerations


### Privacy Guarantee: Issuer Blinding

The privacy of the protocol relies on the issuer being unable to correlate its issuances on one site with redemptions occurred on another site. That way, the issuer gives out N tokens each to M users, and later receives up to N*M requests for redemption on various sites. The issuer can't correlate those redemption requests to any user identity (unless M = 1); it learns only aggregate information about which sites users visit.


#### **Key Consistency**

If the server uses different values for their private keys for different clients, they can de-anonymize clients at redemption time and break the blinding guarantee. To mitigate this, the [Privacy Pass](https://privacypass.github.io) protocol should ensure that issuers publish a public key commitment list, verifies it is small (e.g. max 3 keys), and verifies consistency between issuance and redemption.


#### **Potential Attack: Side Channel Fingerprinting**

If the issuer is able to use network-level fingerprinting or other side-channels to associate a browser at redemption time with the same browser at token issuance time, privacy is lost. Importantly, the API itself has not revealed any new information, since sites could use the same technique by issuing GET requests to the issuer in the two separate contexts anyway.


### Cross site Information Transfer

Trust tokens transfer information about one first-party cookie to another, and we have cryptographic guarantees that each token only contains a small amount of information. Still, if we allow many token redemptions on a single page, the first-party cookie for user U on domain A can be encoded in the trust token information channel and decoded on domain B, allowing domain B to learn the user's domain A cookie until either 1p cookie is cleared.


#### **Mitigation: Dynamic Issuance / Redemption Limits**

To mitigate this attack, we place limits on both issuance and redemption. At issuance, we require [user activation](https://html.spec.whatwg.org/multipage/interaction.html#activation) with the issuing site. At redemption, we can slow down the rate of redemption by returning cached Signed Redemption Records when an issuer attempts too many refreshes (see also the [token exhaustion](#trust-token-exhaustion) problem). These mitigations should make the attack take a longer time and require many user visits to recover a full user ID.


#### **Mitigation: Allowed/Blocked Issuer Lists**

To prevent abuse of the API, browsers could maintain a list of allowed or disallowed issuers. Inclusion in the list should be easy, but should mean agreeing to certain policies about what an issuer is allowed to do. An analog to this is the [baseline requirements](https://cabforum.org/baseline-requirements/) in the CA ecosystem.


#### **Mitigation: Per-Site Issuer Limits**

The rate of identity leakage from one site to another increases with the number of tokens redeemed on a page. To avoid abuse, there should be strict limits on the number of token issuers contacted per origin (e.g. 2), and those contacted should be persisted in browser storage to avoid excessively rotating issuers on subsequent visits.


### First Party Tracking Potential

Cached SRRs and their associated browser public keys have a similar tracking potential to first party cookies. Therefore these should be clearable by browser’s existing Clear Site Data functionality.

The SRR and its public key are untamperable first-party tracking vectors. They allow sites to share their first-party user identity with third parties on the page in a verifiable way. To mitigate this potentially undesirable situation, user agents can request multiple SRRs in a single token redemption, each bound to different keypairs, and use different SRRs and keypairs when performing requests based on the third-party or over time.

In order to prevent the issuer from binding together multiple simultaneous redemptions, the UA can blind the keypairs before sending them to the issuer. Additionally, the client may need to produce signed timestamps to prevent the issuer from using the timestamp as another matching method.


## Security Considerations


### Trust Token Exhaustion

The goal of a token exhaustion attack is to deplete a legitimate user's supply of tokens for a given issuer, so that user is less valuable to sites who depend on the issuer’s trust tokens.

We have a number of mitigations against this attack:



*   Issuers issue many tokens at once, so users have a large supply of tokens.
*   Browsers will only ever redeem one token per top-level page view, so it will take many page views to deplete the full supply.
*   The browser will cache SRRs per-origin and only refresh them when an issuer iframe opts-in, so malicious origins won't deplete many tokens. The “freshness” of the SRR becomes an additional trust signal.
*   Browsers may choose to limit redemptions on a time-based schedule, and either return cached SRRs if available, or require consumers to cache the SRR.
*   Issuers will be able to see the Referer, subject to the page's referrer policy, for any token redemption, so they'll be able to detect if any one site is redeeming suspiciously many tokens.

When the issuer detects a site is attacking its trust token supply, it can fail redemption (before the token is revealed) based on the referring origin, and prevent browsers from spending tokens there.


### Double-Spend Prevention

Issuers can verify that each token is seen only once, because every redemption is sent to the same token issuer. This means that even if a malicious piece of malware exfiltrates all of a users' trust tokens, the tokens will run out over time. Issuers can sign fewer tokens at a time to mitigate the risk.


## Future Extensions


### Publicly Verifiable Tokens

The tokens used in the above design require private key verification, necessitating a roundtrip to the issuer at redemption time for token verification and SRR generation. Here, the unlinkability of a client’s tokens relies on the assumption that the client and issuer can communicate anonymously (i.e. the issuer cannot link issuance and redemption requests from the same user via a side channel like network fingerprint).

An alternative design that avoids this assumption is to instead use _publicly verifiable_ tokens (i.e. tokens that can be verified by any party). It is possible to instantiate these tokens using blind signatures as well, achieving the [same unlinkability properties](#privacy-guarantee-issuer-blinding) of the existing design. These tokens can be spent without a round trip to the issuer, but it requires either decentralized double spend protection, or round trips to a centralized double-spend aggregator.


### Request mechanism not based on `fetch()`

A possible enhancement would be to allow for sending Signed Redemption Records (and signing requests using the trust keypair) for requests sent outside of `fetch()`, e.g. on top-level and iframe navigation requests. This would allow for the use of the API by entities that aren't running Javascript directly on the page, or that want some level of trust before returning a main response.


### Optimizing redemption RTT

If the publisher can configure issuers in response headers (or otherwise early in the page load), then they could invoke a redemption in parallel with the page loading, before the relevant call to `getTrustAttestation`.


## Appendix


### Sample API Usage


```
areyouahuman.com - Trust Token Issuer
coolwebsite.com - Publisher Top-Level Site
foo.com - Site requiring a Trust Token to prove the user is trusted.

```



1.  User visits `areyouahuman.com`.
1.  `areyouahuman.com` verifies the user is a human, and calls `fetchTrustTokens('/request-tokens')`.
    1.  The browser stores the trust tokens associated with `areyouahuman.com`.
1.  Sometime later, the user visits `coolwebsite.com`.
1.  `coolwebsite.com` wants to know if the user is a human, by asking `areyouahuman.com` that question, by calling` getTrustAttestation('areyouahuman.com')`.
    1.  The browser requests a redemption.
    1.  The issuer returns an SRR (this indicates that `areyouahuman.com` at some point issued a valid token to this browser).
    1.  When the promise returned by the method resolves, the SRR can be used in subsequent resource requests.
1.  Script running code in the top level `coolwebsite.com` document can call `fetch("foo.com/get-content", {signedRedemptionRecord: 'areyouahuman.com'})`
    1.  The third-party receives the SRR, and now has some indication that `areyouahuman.com` thought this user was a human.
    1.  The third-party responds to this fetch request based on that fact.
