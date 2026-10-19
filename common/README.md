# common/ — team-owned namespaced manifests applied to EVERY cluster

Put cloud/env/region-agnostic namespaced resources the team owns here. They are applied by the
platform `tenants-config` ApplicationSet (first of its two sources) into the team namespace, under
the team's AppProject, on every cluster the team runs on.

Good fits (namespaced, same on all clusters):
- ServiceAccounts (workload-identity annotations that are constant), RBAC Role/RoleBinding
- Istio `AuthorizationPolicy` / `PeerAuthentication`, extra `NetworkPolicy`
- Shared `ConfigMap`, `PodDisruptionBudget`

Per-cluster resources (vault URL, env/region-specific identity) go in
`clusters/<cloud>/<env>/<region>/` instead. Cluster-scoped resources (e.g. `ClusterSecretStore`)
are platform-only and rejected by the team AppProject.
