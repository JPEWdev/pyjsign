# Introducing pyjsign: signing SPDX 3.0 JSON-LD documents with X.590 (JSS)

SPDX 3.0 recently implemented the ability to sign documents serialized as
[JSON-LD][jsonld] using [ITU-T X.590][x590] a.k.a. the **JSON Signature Scheme
(JSS)**.

Until now, there hasn't been a simple, standalone Python tool to actually
produce or verify JSS signatures, so we have released
[`pyjsign`](https://github.com/spdx/pyjsign).

`pyjsign` is a Python library and command-line tool implementing two related
JSON signing formats:

- **[JSON Signature Format (JSF)](https://cyberphone.github.io/doc/security/jsf.html)**
- **ITU-T X.590 JSON Signing Schema (JSS)** - the format SPDX 3.0 uses

It's available now on PyPI:

```bash
python3 -m pip install pyjsign
```

This post focuses on JSS, since that's what what SPDX 3 mandates for signing a
document.

## How JSS signatures work

A JSS signature is a self-describing entry appended to a top-level `signatures`
array in the JSON document. "Self-describing" means each signature entry embeds
what's needed to verify it: the algorithm, a hash of the canonicalized
document, and either the public key, an X.509 certificate chain, a certificate
URL, or a certificate thumbprint. To produce the signature, the document is
first canonicalized with [JCS][jcs] (JSON Canonicalization Scheme), then
hashed, then signed.

One important nuance worth calling out for SPDX/JSON-LD users: JCS
canonicalizes the *JSON text* (consistent key ordering, number/string
formatting), not the *RDF graph* that the JSON-LD represents. Two JSON-LD
documents that are semantically equivalent as RDF (e.g. with array elements
reordered) are **not** guaranteed to canonicalize to the same bytes, and
therefore won't produce or verify against the same signature. In practice this
means you cannot roundtrip a JSON-LD document through RDF, and expect the
signature to match. Signature verification must be done on the JSON-LD document
before converting it to RDF, and when converting from RDF back to JSON-LD, a
new signature must be made for the new document.

## Signing an SPDX 3.0 document

Here's a minimal (illustrative) SPDX 3.0 JSON-LD document, `sbom.json`:

```json
{
  "@context": "https://spdx.org/rdf/3.0.1/spdx-context.jsonld",
  "@graph": [
    {
      "type": "SpdxDocument",
      "spdxId": "https://example.com/spdx/doc1",
      "name": "Example SBOM",
      "creationInfo": "_:creationinfo",
      "rootElement": ["https://example.com/spdx/package1"]
    },
    {
      "type": "software_Package",
      "spdxId": "https://example.com/spdx/package1",
      "name": "example-package",
      "software_packageVersion": "1.0.0",
      "creationInfo": "_:creationinfo"
    },
    {
      "type": "CreationInfo",
      "spdxId": "_:creationinfo",
      "created": "2026-01-15T12:00:00Z",
      "specVersion": "3.0.1",
      "createdBy": ["https://example.com/agents/example-org"]
    }
  ]
}
```

Generate a signing key (any `cryptography`-supported private key works; this
example uses ECDSA P-256) and sign the document, embedding the public key so
the signature is self-describing:

```bash
pyjsign jss sign sbom.json --key signing_key.pem ES256 --embed-public-key -o sbom.signed.json
```

The output has a `signatures` array appended:

```json
{
  "@context": "https://spdx.org/rdf/3.0.1/spdx-context.jsonld",
  "@graph": [ ... ],
  "signatures": [
    {
      "hash_algorithm": "sha-256",
      "algorithm": "ES256",
      "public_key": "MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAE...",
      "value": "xebEWRR5DFxxEDQd2-XSZZYhS60_HT1VBvYhttJFsGBujM385N6o..."
    }
  ]
}
```

Verify it:

```bash
pyjsign jss verify sbom.signed.json
```

```
✓ Signature is valid
```

## Signing with an X.509 certificate chain

For supply-chain use cases, you'll usually want to embed a certificate
chain instead of a bare public key, so a verifier can validate it against a
trust root:

```bash
pyjsign jss sign sbom.json --key signing_key.pem ES256 --cert cert_chain.pem -o sbom.signed.json
```

`cert_chain.pem` is a PEM file holding the leaf certificate followed by any
intermediates (leaf-to-root order). `pyjsign` embeds the chain as
`public_cert_chain` in the signature entry, and:

```bash
pyjsign jss certs sbom.signed.json -o chains.pem
```

extracts and prints it back out as PEM, so you can feed it to a tool like
[`cryptography.x509.verification`](https://cryptography.io/en/latest/x509/verification/)
or plain `openssl` to validate the chain of trust:

```bash
openssl verify -CAfile root_ca.pem chains.pem
```

(`-CAfile` must point to your trusted root(s), not the extracted chain
itself; add `-partial_chain` if the chain's root isn't in your trust store
as a CA.) **`pyjsign` only extracts and verifies the signature itself - it
does not perform chain-of-trust, expiry, or revocation validation.** That's
a deliberate scope boundary: trust policy belongs to the consumer, not the
signing tool.

## Multiple signatures

JSS supports multiple independent signers on the same document (e.g. a
build system and a separate security review signing off independently):

```bash
pyjsign jss sign sbom.json \
  --key build_key.pem ES256 --embed-public-key \
  --key review_key.pem Ed25519 --embed-public-key \
  -o sbom.signed.json
```

Each signer gets its own entry in the `signatures` array, and
`pyjsign jss verify` checks all of them.

## Signing with HashiCorp Vault

If you'd rather not have the signing private key touch disk on your CI
runner at all, `pyjsign` can delegate the actual signing operation to a
[HashiCorp Vault](https://www.vaultproject.io/) Transit secrets engine
(`pip install pyjsign[vault]`):

```bash
export VAULT_ADDR=https://vault.example.com:8200
export VAULT_TOKEN=s.xxxxxxxx
pyjsign jss sign sbom.json --vault-key my-signing-key ES256 --embed-public-key -o sbom.signed.json
```

Only the public key ever leaves Vault; the private key never does.

## Learn More

Full documentation, including the JSF command set, signature chains, and the
Vault integration, is in the
[README](https://github.com/spdx/pyjsign#readme). Issues and contributions
are welcome at [github.com/spdx/pyjsign](https://github.com/spdx/pyjsign).

[jsonld]: https://json-ld.org/
[x590]: https://www.itu.int/epublications/en/publication/itu-t-x-590-2023-10-json-signature-scheme-jss/en
[jcs]: https://datatracker.ietf.org/doc/html/rfc8785
