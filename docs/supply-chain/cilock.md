# Supply-chain: signed build provenance with cilock

This repo's release build is signed and attested with [cilock](https://cilock.dev),
keyless, against the TestifySec platform. No long-lived secrets live in the repo.

> Status: onboarding on a fork (`manzil-infinity180/console`), pointed at the staging
> platform. Not upstreamed to `kubestellar/console` yet.

## What the workflow does (`.github/workflows/cilock-attest.yml`)

On a `v*` tag (or manual dispatch), for the goreleaser build of `console` + `kc-agent`:

1. **Attest source** (`--step source-git`): records the exact commit + CI context (git,
   github, environment attestors), signed into a DSSE envelope.
2. **Attest build** (`--step build`): runs `goreleaser build --snapshot` under `--trace`,
   so the build is hermetic and the materials/products are captured from the kernel. This
   is the SLSA build-provenance evidence.
3. **Verify gate**: checks the built binary satisfies a signed policy (offline, against a
   committed public key). Guarded until the policy is authored (step below), then blocking.
4. Uploads the attestations + the VSA (Verification Summary Attestation) as build artifacts.

Every `cilock run` signs keyless (GitHub OIDC -> platform Fulcio short-lived cert + RFC3161
TSA) and uploads the evidence to the platform's Archivista. The only non-obvious requirement
is `permissions: id-token: write`.

## One-time setup (tenant admin, once per repo)

Keyless upload needs the repo's Actions identity federated to a tenant. No secret is created.

```sh
cilock login  --platform-url "$PLATFORM_URL"
cilock trust github <org>/<repo>          # e.g. manzil-infinity180/console
# add --verify if you later want the gate to read Archivista instead of local envelopes
```

`PLATFORM_URL` defaults to staging in the workflow. Set the repo/org Actions variable
`PLATFORM_URL` to override (prod only for the real upstream release).

## Authoring the signed policy (one-time, to turn the gate on)

The verify gate is skipped until you commit a signed policy + its public key. To create them:

```sh
# 1. run the workflow once so real build evidence exists in Archivista
# 2. author a starter policy from that commit's evidence
cilock policy from-commit <commit> -o console-build.policy.json --platform-url "$PLATFORM_URL"
# 3. sign it with your release key and export the public key
cilock sign -k release.key console-build.policy.json -o console-build.policy.signed.json
# (publish the matching public key)
# 4. commit both under supply-chain/
mkdir -p supply-chain
mv console-build.policy.signed.json supply-chain/
cp release.pub supply-chain/console-policy.pub
```

Once `supply-chain/console-build.policy.signed.json` and `supply-chain/console-policy.pub`
exist, the gate enforces on every run.

## Verify a release yourself

```sh
cilock verify ./console \
  --policy    supply-chain/console-build.policy.signed.json \
  --publickey supply-chain/console-policy.pub \
  -a source-git.att.json -a build.att.json \
  --platform-url ""     # fully offline; the policy carries its own trust
```

## Why this pattern (and not the heavier one)

This uses a **key-signed** policy verified offline (`--publickey`, `--platform-url ""`) -
the simple, self-contained recipe for a single OSS repo (same as `testifysec/curl` and
`testifysec/hugo`). The platform itself uses a heavier **keyless platform-Fulcio-signed**
policy (rotating signer identity) - unnecessary here. See the deep guide for the contrast.

## What is attested vs not (v1 scope)

- Attested: source (commit + CI), build (goreleaser binaries, hermetic trace).
- Not yet: the Docker image (step is stubbed in the workflow, `--step image -a oci`), SBOM,
  and vuln-scan attestations. Follow-ups once the base flow is green.
