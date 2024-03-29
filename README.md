# Transaction-Binding-Proof
## Goal
The goal of this specification is to define how a JSON Web Token (JWT) can be bound to a transaction in such a way that the JWT cannot be used with another transaction without being detected by the recipient of the JWT. A specific application of this binding mechanims is to bind a SPIFFE Verifiable Identity Document JWT (SVID JWT) to a transaction.

## Introduction
SPIFFE Verifiable Identity Document JWT (SVID JWTs) are bearer tokens defined by the Secure Production Identifier Framework For Everyone (SPIFFE). These tokens does not contain any key material by definition. JWT SVIDs are issued to individual workloads. They are short lived, but if an atacker gets hold of them, the attacker may use them to impersonate the workload that was issued with the JWT SVID.

One way to avoid this is by:

1. Adding a reference to a public key to a JWT SVID by including a key ID (kid) claim in the JWT payload.
2. Define a transaction binding proof by profiling the JSON Web Signature (JWS) specification to contain specific claims that can be bound to the transaction.

## SVID JWT kid claim
To support transaction binding, JWT SVIDs MUST include a kid claim as defined in https://datatracker.ietf.org/doc/html/rfc7515#section-4.1.4 (open question, can this be the X5u or X5t parameter representing a X.509 certificate, such as a X.509 SVID).

## Transaction Binding Proof
The transaction binding proof is a a JWT [RFC7519] that is signed (using JSON Web Signature (JWS) [RFC7515]) with a private key chosen by the workload. The JOSE Header of a Transaction Binding Proof MUST contain at least the following parameters:

* typ: A field with the value dpop+jwt, which explicitly types the DPoP proof JWT as recommended in Section 3.11 of [RFC8725].
* alg: An identifier for a JWS asymmetric digital signature algorithm from [IANA.JOSE.ALGS]. It MUST NOT be none or an identifier for a symmetric algorithm (Message Authentication Code (MAC)).
* kid: Claim indicating which key was used to secure the JWS. It must match the kid claim in the JWT SVID (open question, can this be the X5u or X5t parameter representing the X.509 SVID).

The JWS payload contains the following claims:

* iat (Issued At): The timestamp at which the Transaction Binding proof was created (REQUIRED)
* jti (JWT ID): A unique identifier for the JWT to mitigate replay attacks (REQUIRED).
* tth: Hash of the Transaction Token (see transaction token draft - https://datatracker.ietf.org/doc/draft-tulshibagwale-oauth-transaction-tokens/). The value MUST be the result of a base64url encoding (as defined in Section 2 of [RFC7515]) the SHA-256 [SHS] hash of the ASCII encoding of the associated access token's value. (OPTIONAL)
* rqd: A claim, whose value is a JSON object the describes the request details bound to the workload identity. The contents of the rqd claim changes, depending on the typeof request.

### Request Details Claim
The JSON value of the rqd claim MAY include the following values, depending on the request type:

* htm: The value of the HTTP method (Section 9.1 of [RFC9110]) of the request to which the JWT is attached. This value MUST be used if the request is an HTTP request.
* htu: The HTTP target URI (Section 7.1 of [RFC9110]) of the request to which the JWT is attached, without query and fragment parts. This value MUST be used if the request is an HTTP request.
* krt: The Kafka Request Type. This value MUST be used if this is a Kafka request.
* ktn: The Kafka Topic name to which the request is being directed. This value MUST be used if this is a Kafka request.
* ktp: The Kafka Partition within a topic. This value MUST be used if this is a Kafka request.
* gsm: The gRPC service method. This value MUST be used if this is a gRPC request.

## Processing Rules

### Proof generation
The proof is generated by using the private key corresponding to the kid claim in the JWT SVID to sign over the JWT transaction binding proof. (mor details to be added).

### Proof verification
The proof is verified by using the public key corresponding to the kid claim in the JWT SVID to verify the Transaction Binding Proof. In addition to the cryptographic verification, the verifier MUST verify that:

* The kid claim in the JWT SVID equals the kid claim in the Transaction Bindng Proof
* The iat claim is in the accepted boundaries (less than 5 minutes old).
* The tth claim matches the hash of the Transaction Token hash, if one is used. 
* The rqd claim contents matches the actual request.
