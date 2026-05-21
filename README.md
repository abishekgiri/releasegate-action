# ReleaseGate GitHub Action

Sign and verify your GitHub deploys against a ReleaseGate tenant.
Outputs a DSSE-attested governance decision that an auditor can
verify offline, without access to your infrastructure.

This Action is the customer-installable companion to
[ReleaseGate](https://github.com/abishekgiri/change-risk-predictor-).
See [the trust page](https://releasegate.io/trust) for our security
posture and [pricing](https://releasegate.io/pricing) for tiers.

## What this Action does

On every pull request (or workflow_dispatch), this Action:

1. Runs ReleaseGate's deterministic decision gate against the PR
   using your tenant's policies.
2. Signs the decision with your tenant's Ed25519 key, producing a
   DSSE-wrapped attestation.
3. Uploads the attestation as a workflow artifact for offline
   verification.
4. Optionally fails the workflow if the decision indicates a
   blocked or denied change.

## Quick start

1. Email **hello@releasegate.io** to provision a tenant. You will
   receive a tenant identifier and an Ed25519 signing key. The
   design-partner phase is free; see
   [pricing](https://releasegate.io/pricing).
2. Add the tenant identifier and signing key as repository
   secrets: `RELEASEGATE_TENANT_ID` and `RELEASEGATE_SIGNING_KEY`.
3. Add `.github/workflows/releasegate.yml`:

```yaml
name: ReleaseGate

on:
  pull_request:
    branches: [main]

jobs:
  releasegate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: releasegate/releasegate-action@v0.1.2
        with:
          tenant_id: ${{ secrets.RELEASEGATE_TENANT_ID }}
          signing_key: ${{ secrets.RELEASEGATE_SIGNING_KEY }}
```

## Inputs

| Input | Required | Description |
|---|---|---|
| `tenant_id` | yes | Your ReleaseGate tenant identifier. |
| `signing_key` | yes | Ed25519 PEM private key. Pass via a secret. |
| `pr_number` | no | PR to analyze. Defaults to the triggering PR. |
| `fail_on` | no | Comma-separated decision values that fail the Action. Default: `BLOCK,FAIL,DENIED`. |
| `releasegate_ref` | no | Git ref of the main releasegate repo to install. Pinned per Action release. |

## Outputs

| Output | Description |
|---|---|
| `decision` | The verdict (`ALLOW`, `BLOCK`, etc). |
| `decision_id` | Deterministic `analysis-<sha256[:24]>`. Same PR + same commit + same policy produces the same id. |
| `attestation_id` | Per-run unique id derived from the DSSE payload. |

## Verifying an attestation offline

Download the `releasegate-attestation` artifact from any workflow
run. Verify it with:

```bash
pip install "git+https://github.com/abishekgiri/change-risk-predictor-"
releasegate verify-dsse \
  --dsse attestation.dsse.json \
  --key-file your-public-key.pem
```

No ReleaseGate server access is required to verify.

## What this Action does NOT do (yet)

- It does not yet support per-tenant custom policy directories. The
  Action uses the policy bundle compiled into the pinned engine
  (currently the standards bundle — SOC 2 CC8.1, ISO 27001 baseline
  policies). Custom policy authoring is on the roadmap for tenants
  who request it.
- It does not yet enforce signed-commit requirements on the PR.
- It does not yet integrate with Sigstore / Rekor public
  transparency logs (planned 2026 Q3).
- It does not yet support self-serve tenant signup; tenants are
  provisioned manually during the design-partner phase. Email
  hello@releasegate.io.
- It does not yet support Kubernetes admission control. See the
  main repo for the roadmap.

## Security

- Signing keys are passed via GitHub Actions secrets and never
  written to disk outside the runner's ephemeral workspace.
- The Action itself is open source and pinned to a specific commit
  of the main releasegate repo per release. Audit either repo at
  will.
- Security reports: **security@releasegate.io**. See
  [trust](https://releasegate.io/trust).

## License

MIT.

## Versioning

This Action follows semver. Pin to a specific tag (e.g.
`releasegate/releasegate-action@v0.1.2`) in production. The
`releasegate_ref` input pins the main engine SHA at the same time.
