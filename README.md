# `actions/attest`

Generate signed attestations for workflow artifacts. Internally powered by the
[@actions/attest][1] package.

Attestations bind some subject (a named artifact along with its digest) to a
predicate (some assertion about that subject) using the [in-toto][2] format.
[Predicates][3] consist of a type URI and a JSON object containing
type-dependent parameters.

A verifiable signature is generated for the attestation using a short-lived
[Sigstore][4]-issued signing certificate. If the repository initiating the
GitHub Actions workflow is public, the public-good instance of Sigstore will be
used to generate the attestation signature. If the repository is
private/internal, it will use the GitHub private Sigstore instance.

Once the attestation has been created and signed, it will be uploaded to the GH
attestations API and associated with the repository from which the workflow was
initiated.

Attestations can be verified using the `attestation` command in the [GitHub
CLI][5].

## Usage

Within the GitHub Actions workflow which builds some artifact you would like to
attest:

1. Ensure that the following permissions are set:

   ```yaml
   permissions:
     id-token: write
     contents: write # TODO: Update this
   ```

   The `id-token` permission gives the action the ability to mint the OIDC token
   necessary to request a Sigstore signing certificate. The `contents`
   permission is necessary to persist the attestation.

1. Add the following to your workflow after your artifact has been built:

   ```yaml
   - uses: actions/attest@v1
     with:
       subject-path: '<PATH TO ARTIFACT>'
       predicate-type: '<PREDICATE URI>'
       predicate-path: '<PATH TO PREDICATE>'
   ```

   The `subject-path` parameter should identity the artifact for which you want
   to generate an attestation. The `predicate-type` can be any of the the
   [vetted predicate types][3] or a custom value. The `predicate-path`
   identifies a file containg the JSON-encoded predicate parameters.

### Inputs

See [action.yml](action.yml)

```yaml
- uses: actions/attest@v1
  with:
    # Path to the artifact serving as the subject of the attestation. Must
    # specify exactly one of "subject-path" or "subject-digest".
    subject-path:

    # SHA256 digest of the subject for for the attestation. Must be in the form
    # "sha256:hex_digest" (e.g. "sha256:abc123..."). Must specify exactly one
    # of "subject-path" or "subject-digest".
    subject-digest:

    # Subject name as it should appear in the attestation. Required unless
    # "subject-path" is specified, in which case it will be inferred from the
    # path.
    subject-name:

    # URI identifying the type of the predicate.
    predicate-type:

    # JSON string containing the value for the attestation predicate. Must
    # supply exactly one of "predicate-path" or "predicate".
    predicate:

    # Path to the file which contains the JSON content for the attestation
    # predicate. Must supply exactly one of "predicate-path" or "predicate".
    predicate-path:

    # Whether to push the attestation to the image registry. Requires that the
    # "subject-name" parameter specify the fully-qualified image name and that
    # the "subject-digest" parameter be specified. Defaults to false.
    push-to-registry:

    # The GitHub token used to make authenticated API requests. Default is
    # ${{ github.token }}
    github-token:
```

### Outputs

<!-- markdownlint-disable MD013 -->

| Name          | Description                                                    | Example                  |
| ------------- | -------------------------------------------------------------- | ------------------------ |
| `bundle-path` | Absolute path to the file containing the generated attestation | `/tmp/attestation.jsonl` |

<!-- markdownlint-enable MD013 -->

Attestations are saved in the JSON-serialized [Sigstore bundle][6] format.

If multiple subjects are being attested at the same time, each attestation will
be written to the output file on a separate line (using the [JSON Lines][7]
format).

## Examples

### Identify Subject by Path

For the basic use case, simply add the `attest` action to your workflow and
supply the path to the artifact for which you want to generate attestation.

```yaml
name: build-attest

on:
  workflow_dispatch:

jobs:
  build:
    permissions:
      id-token: write
      contents: write

    steps:
      - name: Checkout
        uses: actions/checkout@v4
      - name: Build artifact
        run: make my-app
      - name: Attest
        uses: actions/attest@v1
        with:
          subject-path: '${{ github.workspace }}/my-app'
          predicate-type: 'https://example.com/predicate/v1'
          predicate: '{}'
```

### Identify Subjects by Wildcard

If you are generating multiple artifacts, you can generate an attestation for
each by using a wildcard in the `subject-path` input.

```yaml
- uses: actions/attest@v1
  with:
    subject-path: 'dist/**/my-bin-*'
    predicate-type: 'https://example.com/predicate/v1'
    predicate: '{}'
```

For supported wildcards along with behavior and documentation, see
[@actions/glob][8] which is used internally to search for files.

### Container Image

When working with container images you can invoke the action with the
`subject-name` and `subject-digest` inputs.

If you want to publish the attestation to the container registry with the
`push-to-registry` option, it is important that the `subject-name` specify the
fully-qualified image name (e.g. "ghcr.io/user/app" or
"acme.azurecr.io/user/app"). Do NOT include a tag as part of the image name --
the specific image being attested is identified by the supplied digest.

> **NOTE**: When pushing to Docker Hub, please use "index.docker.io" as the
> registry portion of the image name.

```yaml
name: build-attested-image

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      packages: write
      contents: write
    env:
      REGISTRY: ghcr.io
      IMAGE_NAME: ${{ github.repository }}

    steps:
      - name: Checkout
        uses: actions/checkout@v4
      - name: Login to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - name: Build and push image
        id: push
        uses: docker/build-push-action@v5.0.0
        with:
          context: .
          push: true
          tags: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
      - name: Attest
        uses: actions/attest@v1
        id: attest
        with:
          subject-name: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          subject-digest: ${{ steps.push.outputs.digest }}
          predicate-type: 'https://in-toto.io/attestation/release/v0.1'
          predicate: '{"purl":"pkg:oci/..."}'
          push-to-registry: true
```

[1]: https://github.com/actions/toolkit/tree/main/packages/attest
[2]: https://github.com/in-toto/attestation/tree/main/spec/v1
[3]:
  https://github.com/in-toto/attestation/tree/main/spec/predicates#in-toto-attestation-predicates
[4]: https://www.sigstore.dev/
[5]: https://cli.github.com/
[6]:
  https://github.com/sigstore/protobuf-specs/blob/main/protos/sigstore_bundle.proto
[7]: https://jsonlines.org/
[8]: https://github.com/actions/toolkit/tree/main/packages/glob#patterns
