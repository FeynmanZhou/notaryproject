# Signing and Verification Workflow

This document describes workflows for signing and verifying OCI artifacts and arbitrary blobs.

## OCI artifact signing workflow

The user wants to sign an OCI artifact and push the signature to a repository.

### Signing Prerequisites

- User has access to the signing certificate and private key or a remote signing service through a notation plugin.

### Signing Steps

1. **Generate signature:** Using notation CLI or any other compliant signing tool, sign the OCI artifact. The signing tool should follow the following guideline.
    1. Verify that the signing certificate is valid and satisfies [certificate requirements](./signature-specification.md#certificate-requirements).
    1. Verify that the signing algorithm satisfies [algorithm requirements](./signature-specification.md#signature-algorithm-requirements).
    1. Generate signature.
        1. Generate signature envelope using signature formats specified in [supported signature envelopes](./signature-specification.md#supported-signature-envelopes). Also, as part of this step, the user-defined/supplied custom attributes should be added to the annotations of the signature's descriptor.
        1. If signing scheme is [`notary.x509`](./signing-scheme.md#notaryx509) and timestamp countersignature is demanded, extract the [primitive signature](./signature-specification.md#signature-envelope) (the digital signature computed on payload and signed attributes) from the signature envelope generated in the previous step. Compute hash of the signature (the hash algorithm MUST be deduced from signing certificate's public key. See [algorithm requirements](./signature-specification.md#signature-algorithm-requirements)) and send it to a [RFC 3161](https://datatracker.ietf.org/doc/html/rfc3161.html) compliant *Timestamp Authority (TSA)* for timestamping. The `certReq` field in the [timestamping request](https://datatracker.ietf.org/doc/html/rfc3161#section-2.4.1) MUST be set to true. Otherwise, continue to the next step. 
            1. Verify that the timestamp signing certificate satisfies [certificate requirements](./signature-specification.md#certificate-requirements).
            1. Verify that the timestamp signing algorithm satisfies [algorithm requirements](./signature-specification.md#signature-algorithm-requirements).
            1. On success, embed the timestamp countersignature into the *Timestamp Signature* unsigned attribute of the signature envelope. If any of the above timestamping step failed, implementations MUST fail the signing process.
1. **Push the signature envelope:** Push the signature envelope generated in the previous step to the repository.
1. **Generate signature artifact manifest:** As described in [signature specification](./signature-specification.md#storage) create the Notary Project signature manifest for the signature envelope generated in step 1.
1. **Push signature artifact manifest:** Push the Notary Project signature manifest to the repository.

The user pushes the OCI artifact to the repository before the signature generation process as the signature reference must exist for the signature push to succeed.

## OCI artifact verification workflow

The user wants to pull an OCI artifact only if they are signed by a trusted publisher and the signature is valid.

### Verification Prerequisites

- User has a fully qualified reference to an OCI artifact they want to pull.
  If the fully qualified artifact reference contains a tag then the user needs to resolve this tag to a digest.
  E.g. If a fully qualified reference is `wabbit-networks.io/software:latest` where `latest` is a tag pointing to an artifact.
  The user must resolve the `latest` tag to a digest and construct a new artifact reference using the resolved digest `wabbit-networks.io/software@sha256:${digest}`.
- User has configured [trust store and trust policy](./trust-store-trust-policy.md) required for signature verification.

### Verification Steps

1. **Should implementations of this specification verify the signature? :** Depending upon [trust-policy](./trust-store-trust-policy.md#oci-trust-policy) configuration, determine whether implementations of this specification need to verify the signature or not.
   If signature verification should be skipped for the given artifact, skip the below steps and directly jump to step 4.
1. **Get signature artifact descriptors:** Using the [OCI Distribution Referrers API](https://github.com/opencontainers/distribution-spec/blob/main/spec.md#listing-referrers) download the Notary Project signature manifest descriptors.
   The `artifactType` parameter is set to the Notary Project signature's artifact type `application/vnd.cncf.notary.signature`.
1. For each signature artifact descriptor, perform the following steps:
    1. **Get signature artifact manifest:** Download the Notary Project signature's manifest for the given artifact descriptor.
    1. **Filter signature artifact manifest:**
        1. Filter out the unsupported signature formats by comparing the signature envelope format type (`[descriptors].descriptor.mediaType`) in the signature manifest, with the supported formats defined in [signature specification](./signature-specification.md#storage).
        1. Depending upon the trust-store and trust-policy configuration, further filter out signature manifests.
            1. Using the `scopes` configured in trust policies, get the applicable trust policy.
            1. Get the list of trusted certificates from the trust stores specified in the applicable trust policy.
               If the trust policy contains multiple trust stores, create a list of trusted certificates by merging the trusted certificate list of each trust store.
                1. Calculate the SHA-256 fingerprint of all the trusted certificates and compare them against the list of SHA-256 certificate fingerprints present in  `io.cncf.notary.x509certs.fingerprint.sha256` annotation of artifact manifest.
                1. If there is at least one match, continue to the next step.
                   Otherwise, move to the next signature artifact descriptor(step 3.1).
                   If all signature artifact descriptors have already been processed, fail the signature verification and exit.
        1. If the artifact manifest is filtered out, skip the below steps and move to the next signature artifact descriptor(step 3.1).
           If all signature artifact descriptors have already been processed, fail the signature verification and exit.
    1. **Get and verify signatures:** On the filtered manifest of the Notary Project signature, perform the following steps:
        1. Download the signature envelope.
        1. Verify the signature envelope using trust-store and trust-policy as mentioned in [signature verification](./trust-store-trust-policy.md#signature-verification) section.
        1. If the signature verification fails, skip the below steps and move to the next signature artifact descriptor(step 3.1).
           If all signature artifact descriptors have already been processed, fail the signature verification and exit.
        1. If signature verification succeeds, compare the digest derived from the given OCI artifact reference with the signed digest present in the signature envelope's payload. Also, if there are any user-defined/supplied custom annotations, match them as well.
           If digests and custom annotations are equal, signature verification is considered successful.
           Otherwise, move to the next signature artifact descriptor(step 3.1).
           If all signature artifact descriptors have already been processed, fail the signature verification and exit.
1. **Get OCI artifact:** Using the verified digest, download the OCI artifact.
   This step is not in the purview of Notary Project.

## Blob signing workflow

The user wants to sign an arbitrary blob with a detached signature.

### Signing Prerequisites

- User has access to the signing certificate and private key or a remote signing service through a notation plugin.

### Signing Steps

1. **Generate signature:** Using notation CLI or any other compliant signing tool, sign the blob. The signing tool should follow the following guideline.
    1. Construct the blob payload as defined in [`signature specification`](./signature-specification.md#payload)
    1. Verify that the signing certificate is valid and satisfies [certificate requirements](./signature-specification.md#certificate-requirements).
    1. Verify that the signing algorithm satisfies [algorithm requirements](./signature-specification.md#signature-algorithm-requirements).
    1. Generate signature.
        1. Generate signature using a signature format specified in [supported signature envelopes](./signature-specification.md#supported-signature-envelopes). Also, as part of this step, the user-defined/supplied custom attributes should be added to the annotations of the signature payload.
        1. If signing scheme is [`notary.x509`](./signing-scheme.md/#notaryx509) and timestamp countersignature is demanded, extract the [primitive signature](./signature-specification.md#signature-envelope) (the digital signature computed on payload and signed attributes) from the signature envelope generated in the previous step. Compute hash of the signature (the hash algorithm SHOULD be deduced from signing certificate's public key) and send it to a [RFC 3161](https://datatracker.ietf.org/doc/html/rfc3161.html) compliant *Timestamp Authority (TSA)* for timestamping. Otherwise, continue to the next step. 
            1. Verify that the timestamp signing certificate satisfies [certificate requirements](./signature-specification.md#certificate-requirements).
            1. Verify that the timestamp signing algorithm satisfies [algorithm requirements](./signature-specification.md#signature-algorithm-requirements).
            1. On success, embed the timestamp countersignature into the *Timestamp Signature* unsigned attribute of the signature envelope. If any of the above timestamping step failed, implementations MUST fail the signing process.
1. **Save the signature envelope:** Save the signature envelope generated in the previous step to a file. File extension should be the original blob file name plus `.jws.sig` for JWS signatures and `.cose.sig` for COSE signatures.

## Arbitrary blob verification workflow

The user wants to consume an arbitrary blob only if it was signed by a trusted publisher and the signature associated with the blob is valid.

### Verification Prerequisites

- User has the blob that they want to consume, along with its detached signature.
- User has configured [trust store and trust policy](./trust-store-trust-policy.md) required for signature verification.

### Verification Steps

1. **Should implementations of this specification verify the signature? :** Depending upon [trust-policy](./trust-store-trust-policy.md#blob-trust-policy) configuration, determine whether implementations of this specification need to verify the signature or not.
   If signature verification should be skipped for the given blob, skip the below steps.
1. **Verify the detached signature:**
    1. Parse and validate the signature envelope using the detached signature's file extension as the envelope type.
    1. Verify the signature envelope using trust-store and trust-policy as mentioned in [signature evaluation](./trust-store-trust-policy.md#signature-evaluation) section.
    1. If the signature verification fails, exit.
1. Calculate the blob's size and verify that it matches the size present in `targetArtifact.size`. Fail signature verification if there is a mismatch.
1. If provided by the user, verify blob's media type to the one present in `targetArtifact.mediaType`. Fail signature verification if there is a mismatch.
1. Calculate the digest of the blob using the digest algorithm deduced from signing certificate's public key (see [Algorithm Selection](./signature-specification.md#algorithm-selection)) and match it with the digest specified at `targetArtifact.digest`.  Fail signature verification if there is a mismatch.
1. If there any user-defined/supplied custom annotations, match them against the ones present in `targetArtifact.annotations`. If they match, signature verification is considered successful.