# team-alpha-gitops

Owned by **team-alpha**. Declares which services alpha runs and their config per
environment, cloud and region. The golden chart version is set by team-alpha itself, per
cluster leaf (`goldenChart` below) — upgrade it whenever the team is ready, independently of
other teams and other clusters. CI in each service repo bumps the sbx pin here; the
`promote` workflow moves it up the rings one cluster at a time.

```
apps/orders-api/
├── base/values.yaml                       # common to everything (chart schema)
└── envs/
    ├── sbx/
    │   ├── values.yaml                    # env-wide: replicas, autoscaling       (optional layer)
    │   ├── azure/centralus/values.yaml    # CLUSTER LEAF: image tag+digest pin, goldenChart, route host
    │   └── aws/us-east-2/values.yaml      # CLUSTER LEAF on the EKS cluster (ECR image)
    ├── dev/ stg/                          # same shape (convention; added by promotion)
    └── prd/
        ├── values.yaml                    # env-wide prd sizing
        ├── azure/
        │   ├── values.yaml                # cloud-wide: prod ACR repository
        │   ├── <region-a>/values.yaml     # leaf (ring 1, canary)
        │   └── <region-b>/values.yaml     # leaf (ring 2)
        └── aws/
            └── <region>/values.yaml       # leaf (ring 3): ECR repository (regional), pin, IRSA role-arn

config/envs/<env>/<cloud>/<region>/        # team-owned ESO SecretStore + eso-reader ServiceAccount
                                           #   (per cluster; see platform-gitops tenants/_template/README.md)
```

Only the sbx leaves exist today; the dev/stg/prd rows show the convention (promotion creates them).

## How a cluster picks its values

A cluster is `environment + cloud + region` (labels on its in-cluster Argo CD secret). Its
workload ApplicationSet layers, **later wins**:

```
base  ->  envs/<env>  ->  envs/<env>/<cloud>  ->  envs/<env>/<cloud>/<region>
```

Every layer except `base` is optional. Put a setting at the **lowest layer where it is genuinely
shared**: sizing is env-wide, the Azure prod ACR hostname is cloud-wide (one geo-replicated
registry), an ECR hostname is regional so it lives in the leaf.

## The leaf opts a service in to a cluster

A service is deployed to a cluster only if `apps/<svc>/envs/<env>/<cloud>/<region>/values.yaml`
exists. That file holds the cluster's image pin (`image.tag` + `image.digest`) **and** the
`goldenChart` version to deploy with, so:

- both the image pin and the chart version **live only in the leaf** (never in `base`, env or cloud layers);
- a team can upgrade to a new golden chart version on one cluster without touching any other;
- **prd regions are promoted independently** (canary eastus, then westus, then aws);
- a new region renders nothing until the `promote` workflow creates the leaf for each service;
- rollback is reverting that cluster's promotion PR.

Don't use the top-level keys `name`, `server`, `metadata`, `environment`, `cloud`, `region` or
`addons` in a values file: the leaf's content is also visible to the ApplicationSet.

## Common tasks

| Task | How |
| --- | --- |
| New service | Add `apps/<svc>/base/values.yaml` and the sbx leaf (`goldenChart` + `route.host`); then CI pins the image. |
| Upgrade the golden chart | Change `goldenChart` in the leaf, sbx first, then dev, stg, prd (a ring may not be ahead of the one below it). Check `golden-helm-charts/CHANGELOG.md` for upgrade notes first. |
| Promote a service | Run **promote** with `from_cluster` / `to_cluster` as `<env>/<cloud>/<region>`, e.g. `stg/azure/eastus` -> `prd/azure/eastus`. |
| New region / cloud | Platform onboards the cluster; then promote each service into `prd/<cloud>/<region>`. The workflow creates the leaf; review `route.host` and identity annotations in the PR. |

Promotion into prd copies the exact digest into that cloud's registry (`az acr import` for Azure, or a
registry-to-registry copy such as `crane`/`cosign copy` into ECR for AWS), then pins it in the leaf.

## Checks and guardrails

- **PR check** (`.github/workflows/validate.yml`) calls the platform's reusable workflow. It fails a PR for a missing or invalid
  `goldenChart`, a leaf for a cluster that is not in `fleet.yaml`, an image pin outside the leaf, a chart upgrade that skips a ring,
  or values that no longer render with the pinned chart. Make `validate / check` a required check.
- **`promote`** goes one ring at a time (an emergency skip needs `skip_justification`), enters prd through the canary
  `prd/azure/eastus`, accepts only stable `X.Y.Z` into stg/prd, and verifies the digest was signed by that service's release build.
- A new leaf created by `promote` starts on the golden chart version of the source ring.

`CODEOWNERS` already covers every path under `apps/*/envs/prd/`, including the per-cluster leaves.
