# Multi-Arch Images and Sigstore

This document is a walk-through of different ways to sign and attest a multi-arch image.

## Setup

Spin up a local registry:

```bash
docker run -d -p 5000:5000 --restart=always --name registry registry:2
```

Some env vars to make the commands a little bit easier to read.

```bash
ORIGINAL_IMAGE=registry.redhat.io/ubi9@sha256:c35238b35cbebe4e6c5d0f63a54e54dba41d0b425f2d101ad9f04a5e5ec100b8
IMAGE=localhost:5000/test:latest@sha256:c35238b35cbebe4e6c5d0f63a54e54dba41d0b425f2d101ad9f04a5e5ec100b8
IMAGE_LINUX_AMD64=localhost:5000/test@sha256:5a7d9caeaa9643e99110a5e5b375df963a4c7c32be2e996dcbe663a657ca9ca0
IMAGE2=localhost:5000/test-image-manifest:latest@sha256:c35238b35cbebe4e6c5d0f63a54e54dba41d0b425f2d101ad9f04a5e5ec100b8
IMAGE2_LINUX_AMD64=localhost:5000/test-image-manifest@sha256:5a7d9caeaa9643e99110a5e5b375df963a4c7c32be2e996dcbe663a657ca9ca0
IMAGE2_LINUX_ARM64=localhost:5000/test-image-manifest@sha256:021c55cd1e21793c3c97fcb30719043ba31b04301a88232b0e19ab51a6fe7087
IMAGE2_LINUX_P=localhost:5000/test-image-manifest@sha256:70dca8971dceb535745789ccb094e919db96098eb94276db7cc01103e08d26c0
IMAGE2_LINUX_Z=localhost:5000/test-image-manifest@sha256:c78b3b0b1fd46e0040ff1869e8bd025db79ca7cd7a4b5985c55218ce2662007f
IMAGE3=localhost:5000/test-recursive:latest@sha256:c35238b35cbebe4e6c5d0f63a54e54dba41d0b425f2d101ad9f04a5e5ec100b8
IMAGE3_LINUX_AMD64=localhost:5000/test-recursive@sha256:5a7d9caeaa9643e99110a5e5b375df963a4c7c32be2e996dcbe663a657ca9ca0
IMAGE3_LINUX_ARM64=localhost:5000/test-recursive@sha256:021c55cd1e21793c3c97fcb30719043ba31b04301a88232b0e19ab51a6fe7087
IMAGE3_LINUX_P=localhost:5000/test-recursive@sha256:70dca8971dceb535745789ccb094e919db96098eb94276db7cc01103e08d26c0
IMAGE3_LINUX_Z=localhost:5000/test-recursive@sha256:c78b3b0b1fd46e0040ff1869e8bd025db79ca7cd7a4b5985c55218ce2662007f
IMAGE4=localhost:5000/test-multiple-sboms:latest@sha256:c35238b35cbebe4e6c5d0f63a54e54dba41d0b425f2d101ad9f04a5e5ec100b8
IMAGE4_LINUX_AMD64=localhost:5000/test-multiple-sboms@sha256:5a7d9caeaa9643e99110a5e5b375df963a4c7c32be2e996dcbe663a657ca9ca0
IMAGE4_LINUX_ARM64=localhost:5000/test-multiple-sboms@sha256:021c55cd1e21793c3c97fcb30719043ba31b04301a88232b0e19ab51a6fe7087
IMAGE4_LINUX_P=localhost:5000/test-multiple-sboms@sha256:70dca8971dceb535745789ccb094e919db96098eb94276db7cc01103e08d26c0
IMAGE4_LINUX_Z=localhost:5000/test-multiple-sboms@sha256:c78b3b0b1fd46e0040ff1869e8bd025db79ca7cd7a4b5985c55218ce2662007f
```

Copy a test image to different repositories to isolate each experiment:

```bash
skopeo copy --all --dest-tls-verify=false docker://$ORIGINAL_IMAGE docker://localhost:5000/test:latest
skopeo copy --all --dest-tls-verify=false docker://$ORIGINAL_IMAGE docker://localhost:5000/test-image-manifest:latest
skopeo copy --all --dest-tls-verify=false docker://$ORIGINAL_IMAGE docker://localhost:5000/test-recursive:latest
skopeo copy --all --dest-tls-verify=false docker://$ORIGINAL_IMAGE docker://localhost:5000/test-multiple-sboms:latest
```

Generate the SBOM files:

```bash

# Generate SBOM for each image manifest
syft $IMAGE2_LINUX_AMD64 --output cyclonedx-json > sbom-linux-amd64.json
syft $IMAGE2_LINUX_ARM64 --output cyclonedx-json > sbom-linux-arm64.json
syft $IMAGE2_LINUX_P --output cyclonedx-json > sbom-linux-p.json
syft $IMAGE2_LINUX_Z --output cyclonedx-json > sbom-linux-z.json

# Generate a "unified SBOM" for the image index. Note: this is actually just the SBOM for
# the linux/amd64 image, but this is sufficient for this experiment.
syft $IMAGE --output cyclonedx-json > unified-sbom.json
```

Generate a cosign key-pair:

```bash
export COSIGN_PASSWORD=test
cosign generate-key-pair
```

## Image Index Only

In this section the signature and attestation are applied only to the Image Index (Manifest List).

```bash
# Sign the image index
cosign sign --key cosign.key $IMAGE -y
# Verify signature
cosign verify --key cosign.pub $IMAGE

# Add SBOM attestation to image index
cosign attest --key cosign.key $IMAGE --predicate unified-sbom.json --type https://cyclonedx.org/schema -y
# Verify SBOM attestation
cosign verify-attestation --key cosign.pub $IMAGE --type https://cyclonedx.org/schema > /dev/null
```

*What happens if we verify the linux/amd64 image?*

```bash
$ cosign verify --key cosign.pub $IMAGE_LINUX_AMD64
Error: no signatures found for image
main.go:69: error during command execution: no signatures found for image

$ cosign verify-attestation --key cosign.pub $IMAGE_LINUX_AMD64 --type https://cyclonedx.org/schema
Error: no matching attestations:

main.go:74: error during command execution: no matching attestations:
```

That is expected. We only signed/attested the Image Index, not the Image Manifests.

## Image Manifest Only

This experiment signs and attests each Image Manifest from the Image Index, but does not do the same
for the Image Index.

```bash
# Sign each image manifest, but not the image index
cosign sign --key cosign.key $IMAGE2_LINUX_AMD64 -y
cosign sign --key cosign.key $IMAGE2_LINUX_ARM64 -y
cosign sign --key cosign.key $IMAGE2_LINUX_P -y
cosign sign --key cosign.key $IMAGE2_LINUX_Z -y

# Verify each image manifest
cosign verify --key cosign.pub $IMAGE2_LINUX_AMD64
cosign verify --key cosign.pub $IMAGE2_LINUX_ARM64
cosign verify --key cosign.pub $IMAGE2_LINUX_P
cosign verify --key cosign.pub $IMAGE2_LINUX_Z

# Add SBOM attestation to each image manifest
cosign attest --key cosign.key $IMAGE2_LINUX_AMD64 --predicate sbom-linux-amd64.json --type https://cyclonedx.org/schema -y
cosign attest --key cosign.key $IMAGE2_LINUX_ARM64 --predicate sbom-linux-arm64.json --type https://cyclonedx.org/schema -y
cosign attest --key cosign.key $IMAGE2_LINUX_P --predicate sbom-linux-p.json --type https://cyclonedx.org/schema -y
cosign attest --key cosign.key $IMAGE2_LINUX_Z --predicate sbom-linux-z.json --type https://cyclonedx.org/schema -y

# Verify SBOM attestation of each image manifest
cosign verify-attestation --key cosign.pub $IMAGE2_LINUX_AMD64 --type https://cyclonedx.org/schema > /dev/null
cosign verify-attestation --key cosign.pub $IMAGE2_LINUX_ARM64 --type https://cyclonedx.org/schema > /dev/null
cosign verify-attestation --key cosign.pub $IMAGE2_LINUX_P --type https://cyclonedx.org/schema > /dev/null
cosign verify-attestation --key cosign.pub $IMAGE2_LINUX_Z --type https://cyclonedx.org/schema > /dev/null
```

*How to verify the image manifest for each arch given an image index?*

Crack open the image index and check each image manifest individually. There is no `--recursive` flag
for the `cosign verify*` commands.

## Recursive

In this section, we combine both of the previous approaches. At the end, verification can occur on either
the Image Index or any of the Image Manifests.

```bash
# Sign recursively!
cosign sign --key cosign.key $IMAGE3 -y --recursive

# Verify all the things
cosign verify --key cosign.pub $IMAGE3
cosign verify --key cosign.pub $IMAGE3_LINUX_AMD64
cosign verify --key cosign.pub $IMAGE3_LINUX_ARM64
cosign verify --key cosign.pub $IMAGE3_LINUX_P
cosign verify --key cosign.pub $IMAGE3_LINUX_Z

# Attest recursively? Not today
cosign attest --key cosign.key $IMAGE3 --predicate unified-sbom.json --type https://cyclonedx.org/schema --recursive -y
# NOTE: This does not appear to work. In the command above, only the Image Index is attested. 
# Even if it did work, applying a unified SBOM to each arch-specific Image Manifest feels odd.
# Fill the gap by attesting each image manifest directly.
cosign attest --key cosign.key $IMAGE3_LINUX_AMD64 --predicate sbom-linux-amd64.json --type https://cyclonedx.org/schema -y
cosign attest --key cosign.key $IMAGE3_LINUX_ARM64 --predicate sbom-linux-arm64.json --type https://cyclonedx.org/schema -y
cosign attest --key cosign.key $IMAGE3_LINUX_P --predicate sbom-linux-p.json --type https://cyclonedx.org/schema -y
cosign attest --key cosign.key $IMAGE3_LINUX_Z --predicate sbom-linux-z.json --type https://cyclonedx.org/schema -y

# Verify all the SBOMs
cosign verify-attestation --key cosign.pub $IMAGE3 --type https://cyclonedx.org/schema > /dev/null
cosign verify-attestation --key cosign.pub $IMAGE3_LINUX_AMD64 --type https://cyclonedx.org/schema > /dev/null
cosign verify-attestation --key cosign.pub $IMAGE3_LINUX_ARM64 --type https://cyclonedx.org/schema > /dev/null
cosign verify-attestation --key cosign.pub $IMAGE3_LINUX_P --type https://cyclonedx.org/schema > /dev/null
cosign verify-attestation --key cosign.pub $IMAGE3_LINUX_Z --type https://cyclonedx.org/schema > /dev/null
```

## Multiple SBOMs

This is a variant of the previous experiment. Here we do away with the idea of a unified SBOM. Instead, each
arch-specific SBOM is associated with the Index Image.

```
# Sign recursively
cosign sign --key cosign.key $IMAGE4 -y --recursive

# Verify all the things
cosign verify --key cosign.pub $IMAGE4
cosign verify --key cosign.pub $IMAGE4_LINUX_AMD64
cosign verify --key cosign.pub $IMAGE4_LINUX_ARM64
cosign verify --key cosign.pub $IMAGE4_LINUX_P
cosign verify --key cosign.pub $IMAGE4_LINUX_Z

# Add SBOM attestation to each image manifest
cosign attest --key cosign.key $IMAGE4_LINUX_AMD64 --predicate sbom-linux-amd64.json --type https://cyclonedx.org/schema -y
cosign attest --key cosign.key $IMAGE4_LINUX_ARM64 --predicate sbom-linux-arm64.json --type https://cyclonedx.org/schema -y
cosign attest --key cosign.key $IMAGE4_LINUX_P --predicate sbom-linux-p.json --type https://cyclonedx.org/schema -y
cosign attest --key cosign.key $IMAGE4_LINUX_Z --predicate sbom-linux-z.json --type https://cyclonedx.org/schema -y

# Verify SBOM attestation on each image manifest
cosign verify-attestation --key cosign.pub $IMAGE4_LINUX_AMD64 --type https://cyclonedx.org/schema > /dev/null
cosign verify-attestation --key cosign.pub $IMAGE4_LINUX_ARM64 --type https://cyclonedx.org/schema > /dev/null
cosign verify-attestation --key cosign.pub $IMAGE4_LINUX_P --type https://cyclonedx.org/schema > /dev/null
cosign verify-attestation --key cosign.pub $IMAGE4_LINUX_Z --type https://cyclonedx.org/schema > /dev/null

# Add each SBOM attestation to the image index
cosign attest --key cosign.key $IMAGE4 --predicate sbom-linux-amd64.json --type https://cyclonedx.org/schema -y
cosign attest --key cosign.key $IMAGE4 --predicate sbom-linux-arm64.json --type https://cyclonedx.org/schema -y
cosign attest --key cosign.key $IMAGE4 --predicate sbom-linux-p.json --type https://cyclonedx.org/schema -y
cosign attest --key cosign.key $IMAGE4 --predicate sbom-linux-z.json --type https://cyclonedx.org/schema -y

# Verify SBOM attestation on the image index. Output should display four entries
cosign verify-attestation --key cosign.pub $IMAGE4 --type https://cyclonedx.org/schema | \
  jq '.payload | @base64d | fromjson | .predicate.metadata.component.name' -r
```

