---
title: "jwt"
description: "Fast and easy JSON Web Token library for Java"
layout: landing
github: "https://github.com/latte-java/jwt"
og_image: "/images/og/jwt.png"
---

## Install

Add to your Latte project:

```groovy
dependency(id: "org.lattejava:latte-jwt:1.0.0")
```

Or install with the CLI:

```bash
latte install org.lattejava:latte-jwt:1.0.0
```

Also available for Maven:

```xml
<dependency>
  <groupId>org.lattejava</groupId>
  <artifactId>latte-jwt</artifactId>
  <version>1.0.0</version>
</dependency>
```

And Gradle:

```groovy
implementation 'org.lattejava:latte-jwt:1.0.0'
```

## Features

- JWT signing and verification with `HS256`, `HS384`, `HS512`, `RS256`, `RS384`, `RS512`, `ES256`, `ES384`, `ES512`, `PS256`, `PS384`, `PS512`, `Ed25519`, `Ed448`
- Single dependency on Jackson -- no Bouncy Castle, Apache Commons, or Guava
- PEM decoding and encoding for private and public keys
- JSON Web Key (JWK) support including JWKS endpoint retrieval
- Key pair generation for RSA, RSA PSS, EC, and EdDSA
- OpenID Connect `at_hash` and `c_hash` support
- Clock skew tolerance for expiration and not-before validation
- Key rotation support via `kid` header

## Sign and encode a JWT using HMAC

```java
// Build an HMAC signer using a SHA-256 hash
Signer signer = HMACSigner.newSHA256Signer("too many secrets");

// Build a new JWT with an issuer(iss), issued at(iat), subject(sub) and expiration(exp)
JWT jwt = new JWT().setIssuer("www.acme.com")
                   .setIssuedAt(ZonedDateTime.now(ZoneOffset.UTC))
                   .setSubject("f1e33ab3-027f-47c5-bb07-8dd8ab37a2d3")
                   .setExpiration(ZonedDateTime.now(ZoneOffset.UTC).plusMinutes(60));
                       
// Sign and encode the JWT to a JSON string representation
String encodedJWT = JWT.getEncoder().encode(jwt, signer);
```

A higher strength hash can be used by changing the signer. The encoding and decoding steps are not affected.

```java
// Build an HMAC signer using a SHA-384 hash
Signer signer384 = HMACSigner.newSHA384Signer("too many secrets");

// Build an HMAC signer using a SHA-512 hash
Signer signer512 = HMACSigner.newSHA512Signer("too many secrets");
```

## Verify and decode a JWT using HMAC

```java
// Build an HMAC verifier using the same secret that was used to sign the JWT
Verifier verifier = HMACVerifier.newVerifier("too many secrets");

// Verify and decode the encoded string JWT to a rich object
JWT jwt = JWT.getDecoder().decode(encodedJWT, verifier);

// Assert the subject of the JWT is as expected
assertEquals(jwt.subject, "f1e33ab3-027f-47c5-bb07-8dd8ab37a2d3");
```

## Sign and encode a JWT using RSA

```java
// Build an RSA signer using a SHA-256 hash. A signer may also be built using the PrivateKey object.
Signer signer = RSASigner.newSHA256Signer(new String(Files.readAllBytes(Paths.get("private_key.pem"))));

// Build a new JWT with an issuer(iss), issued at(iat), subject(sub) and expiration(exp)
JWT jwt = new JWT().setIssuer("www.acme.com")
                   .setIssuedAt(ZonedDateTime.now(ZoneOffset.UTC))
                   .setSubject("f1e33ab3-027f-47c5-bb07-8dd8ab37a2d3")
                   .setExpiration(ZonedDateTime.now(ZoneOffset.UTC).plusMinutes(60));
        
// Sign and encode the JWT to a JSON string representation
String encodedJWT = JWT.getEncoder().encode(jwt, signer);
```

## Verify and decode a JWT using RSA

```java
// Build an RSA verifier using an RSA Public Key. A verifier may also be built using the PublicKey object.
Verifier verifier = RSAVerifier.newVerifier(Paths.get("public_key.pem"));

// Verify and decode the encoded string JWT to a rich object
JWT jwt = JWT.getDecoder().decode(encodedJWT, verifier);

// Assert the subject of the JWT is as expected
assertEquals(jwt.subject, "f1e33ab3-027f-47c5-bb07-8dd8ab37a2d3");
```

## Sign and encode a JWT using EC

```java
// Build an EC signer using a SHA-256 hash. A signer may also be built using the PrivateKey object.
Signer signer = ECSigner.newSHA256Signer(new String(Files.readAllBytes(Paths.get("private_key.pem"))));

// Build a new JWT with an issuer(iss), issued at(iat), subject(sub) and expiration(exp)
JWT jwt = new JWT().setIssuer("www.acme.com")
                   .setIssuedAt(ZonedDateTime.now(ZoneOffset.UTC))
                   .setSubject("f1e33ab3-027f-47c5-bb07-8dd8ab37a2d3")
                   .setExpiration(ZonedDateTime.now(ZoneOffset.UTC).plusMinutes(60));
        
// Sign and encode the JWT to a JSON string representation
String encodedJWT = JWT.getEncoder().encode(jwt, signer);
```

## Verify and decode a JWT using EC

```java
// Build an EC verifier using an EC Public Key. A verifier may also be built using the PublicKey object.
Verifier verifier = ECVerifier.newVerifier(Paths.get("public_key.pem"));

// Verify and decode the encoded string JWT to a rich object
JWT jwt = JWT.getDecoder().decode(encodedJWT, verifier);

// Assert the subject of the JWT is as expected
assertEquals(jwt.subject, "f1e33ab3-027f-47c5-bb07-8dd8ab37a2d3");
```

## Sign and verify a JWT using EdDSA

```java
// Build an EdDSA signer. The algorithm (Ed25519 or Ed448) is determined from the key.
Signer signer = EdDSASigner.newSigner(new String(Files.readAllBytes(Paths.get("private_key.pem"))));

JWT jwt = new JWT().setIssuer("www.acme.com")
                   .setSubject("f1e33ab3-027f-47c5-bb07-8dd8ab37a2d3")
                   .setExpiration(ZonedDateTime.now(ZoneOffset.UTC).plusMinutes(60));

String encodedJWT = JWT.getEncoder().encode(jwt, signer);

// Verify using the public key
Verifier verifier = EdDSAVerifier.newVerifier(Paths.get("public_key.pem"));
JWT decoded = JWT.getDecoder().decode(encodedJWT, verifier);
```

## Clock skew

When verifying JWTs across distributed systems, clocks may not be perfectly synchronized. You can allow a tolerance window for the `exp` and `nbf` claims:

```java
// Allow up to 60 seconds of clock skew
JWT jwt = JWT.getDecoder().withClockSkew(60).decode(encodedJWT, verifier);
```

## Key rotation

When using multiple signing keys, include a `kid` (key ID) in the JWT header and provide a map of verifiers:

```java
// Sign with a kid
Signer signer = HMACSigner.newSHA256Signer("secret", "key-1");
String encodedJWT = JWT.getEncoder().encode(jwt, signer);

// Verify with multiple keys -- the decoder selects the correct verifier using the kid header
Map<String, Verifier> verifiers = Map.of(
    "key-1", HMACVerifier.newVerifier("secret"),
    "key-2", HMACVerifier.newVerifier("other-secret")
);
JWT decoded = JWT.getDecoder().decode(encodedJWT, verifiers);
```

## Custom claims

```java
// Add custom claims when building a JWT
JWT jwt = new JWT().setIssuer("www.acme.com")
                   .addClaim("email", "user@example.com")
                   .addClaim("roles", List.of("admin", "user"));
String encodedJWT = JWT.getEncoder().encode(jwt, signer);

// Retrieve custom claims after decoding
JWT decoded = JWT.getDecoder().decode(encodedJWT, verifier);
String email = decoded.getString("email");
List<Object> roles = decoded.getList("roles");
```

## Time machine decoder

For testing with hard-coded JWTs that may be expired, you can adjust the decoder's concept of "now":

```java
// Decode a JWT as if the current time were January 1, 2019
ZonedDateTime thePast = ZonedDateTime.of(2019, 1, 1, 0, 0, 0, 0, ZoneOffset.UTC);
JWT jwt = JWT.getTimeMachineDecoder(thePast).decode(encodedJWT, verifier);
```

## JSON Web Keys

### Retrieve keys from a JWKS endpoint

```java
// From a known JWKS endpoint
List<JSONWebKey> keys = JSONWebKeySetHelper.retrieveKeysFromJWKS("https://www.googleapis.com/oauth2/v3/certs");

// From an OpenID Connect well-known configuration endpoint
List<JSONWebKey> keys = JSONWebKeySetHelper.retrieveKeysFromWellKnownConfiguration("https://accounts.google.com/.well-known/openid-configuration");

// From an OpenID Connect issuer
List<JSONWebKey> keys = JSONWebKeySetHelper.retrieveKeysFromIssuer("https://accounts.google.com");
```

### Convert keys to and from JWK

```java
// Build a JWK from a public key
JSONWebKey jwk = JSONWebKey.build(publicKey);
String json = jwk.toJSON();

// Parse a public key from a JWK
JSONWebKey jwk = Mapper.deserialize(jsonBytes, JSONWebKey.class);
PublicKey publicKey = JSONWebKey.parse(jwk);

// Add custom properties
JSONWebKey jwk = JSONWebKey.build(privateKey)
                           .add("boom", "goes the dynamite")
                           .add("more", "cowbell");
```

## Key generation helpers

```java
// Generate RSA key pairs (2048, 3072, or 4096 bits)
KeyPair keyPair = JWTUtils.generate2048_RSAKeyPair();

// Generate EC key pairs (P-256, P-384, or P-521)
KeyPair keyPair = JWTUtils.generate256_ECKeyPair();

// Generate EdDSA key pairs
KeyPair keyPair = JWTUtils.generate_ed25519_EdDSAKeyPair();

// Generate HMAC secrets with ideal lengths
String secret = JWTUtils.generateSHA256_HMACSecret();
```

## PEM encoding and decoding

```java
// Decode a PEM file to get the private and public keys
PEM pem = PEM.decode(Files.readAllBytes(Paths.get("private_key.pem")));
PrivateKey privateKey = pem.privateKey;
PublicKey publicKey = pem.publicKey;

// Encode a key back to PEM format
String encoded = PEM.encode(privateKey);
```

## Third party JCE providers

Once you have enabled an additional provider such as Bouncy Castle, no change should be necessary to your code:

```java
// Insert the provider ahead of the JCA
Security.insertProviderAt(new BouncyCastleFipsProvider(), 1);
```

## License

Latte JWT is MIT licensed. Code derived from [fusionauth-jwt](https://github.com/FusionAuth/fusionauth-jwt) retains its Apache 2.0 license.
