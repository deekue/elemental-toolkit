
---
title: "Cosign"
linkTitle: "Cosign"
weight: 2
date: 2020-11-02
description: >
  Secure supply chain
---

[Cosign](https://github.com/sigstore/cosign) is a project that signs and verifies containers and stores the signatures on OCI registries.

You can check the cosign [github repo](https://github.com/sigstore/cosign) for more information.

In elemental-toolkit we sign every OCI artifact that we generate as part of our publish process so the signature can be verified during package installation with luet or during deploy/upgrades from a deployed system to verify that the containers have not been altered in any way since their build.

Currently cosign provides 2 methods for signing and verifying.

 - private/public key
 - keyless

We use keyless signatures based on OIDC Identity tokens provided by github, so nobody has access to any private keys and can use them. (For more info about keyless signing/verification check [here](https://github.com/sigstore/cosign/blob/main/KEYLESS.md))

This signature generation is provided by [luet-cosign](https://github.com/rancher-sandbox/luet-cosign) which is a luet plugin that generates the signatures on image push when building, and verifies them on package unpack when installing/upgrading/deploying.

The process is completely transparent to the end user when upgrading/deploying a running system and using our published artifacts.

When using luet-cosign as part of `luet install` you need to set `COSIGN_EXPERIMENTAL=1` so it can use keyless verification


## Derivatives

## Signing a container with cosign

When building a derivative which is a standard container image, it is enough to run `cosign sign`. Refer to the cosign documentation on how to achieve that in detail.

With github actions, it is possible to have the keyless signing process running in the workflow, for example:

```yaml
name: build
jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      id-token: write  # Note: Required for OIDC support
    steps:
      - name: Install Cosign
        uses: sigstore/cosign-installer@main
      # ... build your image here ...
      - name: Sign the images with GitHub OIDC Token
        run: cosign sign image:tag
        env:
          COSIGN_EXPERIMENTAL: 1 # Note: Required for keyless
```

The signing process should be returning more or less:

```
Generating ephemeral keys...
Retrieving signed certificate...
        Note that there may be personally identifiable information associated with this signed artifact.
        This may include the email address associated with the account with which you authenticate.
        This information will be used for signing this artifact and will be stored in public transparency logs and cannot be removed later.
Successfully verified SCT...
tlog entry created with index: 2611430
Pushing signature to: ....
```

The cosign verification now can be enabled with the `elemental-cli` with the ```cosign``` flag or by enabling it in the `elemental` config file.

## Signing packages with luet
If building a derivative, you can also sign and verify you final artifacts with the use of [luet-cosign](https://github.com/rancher-sandbox/luet-cosign).

As keyless is only possible to do in an CI environment (as it needs an OIDC token) you would need to set up private/public signature and verification.

{{% alert title="Note" %}}
If you are building and publishing your derivatives with luet on github, you can see an example on how we generate and push the keyless signatures ourselves on [this workflow](https://github.com/rancher/elemental-toolkit/blob/master/.github/workflows/build-master-teal-x86_64.yaml#L445)
{{% /alert %}}


### Verify elemental-toolkit artifacts as part of derivative building

If you consume elemental-toolkit artifacts in your Dockerfile as part of building a derivative you can verify the signatures of the artifacts by setting:

```dockerfile
ENV COSIGN_EXPERIMENTAL=1
RUN luet install -y meta/cos-verify # install dependencies for signature checking
```

{{% alert title="Note" %}}
The {{<package package="meta/cos-verify" >}} is a meta package that will pull {{<package package="toolchain/cosign" >}} and {{<package package="toolchain/luet-cosign" >}} .
{{% /alert %}}


And then making sure you call luet with `--plugin luet-cosign`. You can see an example of this in our [standard Dockerfile example](https://github.com/rancher/elemental-toolkit/tree/master/examples/standard) 

That would verify the artifacts coming from our repository.


For signing resulting containers with a private/public key, please refer to the [cosign](https://github.com/sigstore/cosign) documents.

For verifying with a private/public key, the only thing you need is to set the env var `COSIGN_PUBLIC_KEY_LOCATION` to point to the public key that signed and enable the luet-cosign plugin.

{{% alert title="Note" %}}
Currently there is an issue in which if there is more than one repo and one of those repos is not signed the whole install will fail due to cosign failing to verify the unsigned repo.

If you are using luet with one or more unsigned repos, it's not possible to use cosign to verify the chain.

Please follow up in https://github.com/rancher-sandbox/luet-cosign/issues/6 for more info.
{{% /alert %}}
