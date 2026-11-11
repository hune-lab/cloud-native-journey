# Kubernetes default RBAC: roles, bindings and identities

Every ClusterRole, Role, binding and well-known identity that a fresh kube-apiserver creates on startup, read from the source of **Kubernetes v1.37.2** (commit [`c50e5395f1ae`](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/)). Each entry links to the line that defines it and to the official documentation.

Part of [The Cloud Native Journey](https://hune-lab.com/the-cloud-native-journey), the apiserver journey.

## How to check it on your cluster

All these objects carry the label `kubernetes.io/bootstrapping: rbac-defaults` and the annotation `rbac.authorization.kubernetes.io/autoupdate: "true"` ([policy.go L42-43](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/policy.go#L42-L43)):

```sh
kubectl get clusterroles,clusterrolebindings -l kubernetes.io/bootstrapping=rbac-defaults
kubectl get roles,rolebindings -A -l kubernetes.io/bootstrapping=rbac-defaults
```

Because of the annotation, the apiserver recreates and completes them at every start: edit them and your changes come back. Docs: [Auto-reconciliation](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#auto-reconciliation).

Roles marked *feature gate* below exist only when that gate is on. The default value given is the one of v1.37.

## Well-known users and groups

There is no `User` or `Group` object in Kubernetes. These names are produced by the authenticators, and the default bindings below refer to them.

| Name | Kind | Meaning | Source |
|---|---|---|---|
| `system:masters` | group | full access: bound to `cluster-admin` | [user.go L71](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/authentication/user/user.go#L71) |
| `system:nodes` | group | kubelets | [user.go L72](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/authentication/user/user.go#L72) |
| `system:monitoring` | group | read metrics and health endpoints | [user.go L73](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/authentication/user/user.go#L73) |
| `system:unauthenticated` | group | every request no authenticator recognized | [user.go L74](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/authentication/user/user.go#L74) |
| `system:authenticated` | group | added to every authenticated identity | [user.go L75](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/authentication/user/user.go#L75) |
| `system:anonymous` | user | the identity of unauthenticated requests, when anonymous access is on | [user.go L77](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/authentication/user/user.go#L77) |
| `system:apiserver` | user | the apiserver talking to itself | [user.go L78](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/authentication/user/user.go#L78) |
| `system:kube-proxy` | user | kube-proxy | [user.go L81](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/authentication/user/user.go#L81) |
| `system:kube-controller-manager` | user | kube-controller-manager | [user.go L82](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/authentication/user/user.go#L82) |
| `system:kube-scheduler` | user | kube-scheduler | [user.go L83](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/authentication/user/user.go#L83) |
| `system:serviceaccount:<ns>:<name>` | user | a ServiceAccount | [util.go L30-31](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/authentication/serviceaccount/util.go#L30-L31) |
| `system:serviceaccounts:<ns>` | group | all ServiceAccounts of a namespace | [util.go L32](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/authentication/serviceaccount/util.go#L32) |
| `system:serviceaccounts` | group | all ServiceAccounts | [util.go L33](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/authentication/serviceaccount/util.go#L33) |

Docs: [Referring to subjects](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#referring-to-subjects).

## User-facing ClusterRoles

Ready to use, **not bound to anyone** by default, except `cluster-admin`. `admin`, `edit` and `view` hold no rules of their own: they are aggregated from the `system:aggregate-to-*` roles below, and chained, so `view` ⊂ `edit` ⊂ `admin`. Docs: [User-facing roles](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#user-facing-roles).

| ClusterRole | Grants | Bound by default to | Source |
|---|---|---|---|
| `cluster-admin` | `*` on every resource of every group, and `*` on every non-resource URL | group `system:masters` ([L719](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/policy.go#L719)) | [L340](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/policy.go#L340) |
| `admin` | everything in `edit`, plus read/write `roles` and `rolebindings`, and `create` `localsubjectaccessreviews` | nobody | [L392](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/policy.go#L392), [L422](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/policy.go#L422) |
| `edit` | everything in `view`, plus write on workloads, `configmaps`, `services`, `persistentvolumeclaims`, `leases`; read/write **`secrets`**; `pods/exec`, `pods/attach`, `pods/portforward`, `pods/proxy`; `impersonate` `serviceaccounts`; `create` `serviceaccounts/token` | nobody | [L403](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/policy.go#L403), [L162-211](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/policy.go#L162-L211) |
| `view` | read most namespaced objects, including `pods/log`, `*/status` and `*/scale`; **no `secrets`** | nobody | [L413](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/policy.go#L413), [L113-160](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/policy.go#L113-L160) |

Two consequences worth knowing:

- `edit` can `impersonate` the ServiceAccounts of its namespace and create tokens for them: it can act with their rights.
- `view` excludes `secrets`, but includes `pods/log`.

## Discovery ClusterRoles

Bound by default to broad groups, so that every client can discover the API. Docs: [API discovery roles](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#discovery-roles).

| ClusterRole | Grants | Bound by default to | Source |
|---|---|---|---|
| `system:basic-user` | `create` `selfsubjectaccessreviews`, `selfsubjectrulesreviews`, `selfsubjectreviews`: read your own rights and identity | `system:authenticated` ([L722](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/policy.go#L722)) | [L378](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/policy.go#L378) |
| `system:discovery` | `GET` `/api/*`, `/apis/*`, `/openapi/*`, `/version`, `/livez`, `/readyz`, `/healthz` | `system:authenticated` ([L721](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/policy.go#L721)) | [L349](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/policy.go#L349) |
| `system:public-info-viewer` | `GET` `/version`, `/livez`, `/readyz`, `/healthz` | `system:authenticated`, `system:unauthenticated` ([L723](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/policy.go#L723)) | [L383](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/policy.go#L383) |
| `system:monitoring` | `GET` `/metrics`, `/metrics/slis`, `/livez`, `/readyz`, `/healthz` and their sub-paths; `get` `nodes/metrics` | `system:monitoring` ([L720](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/policy.go#L720)) | [L365](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/policy.go#L365), [L316-335](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/policy.go#L316-L335) |
| `system:service-account-issuer-discovery` | `GET` `/.well-known/openid-configuration`, `/openid/v1/jwks` | `system:serviceaccounts` ([L747](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/policy.go#L747)) | [L585](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/policy.go#L585) |
| `system:cluster-trust-bundle-discovery` | read `clustertrustbundles` | `system:serviceaccounts` ([L752](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/policy.go#L752)) | [L703](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/policy.go#L703) (feature gate `ClusterTrustBundle`, on) |

## Core component ClusterRoles

The identity of each control-plane component. Docs: [Core component roles](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#core-component-roles).

| ClusterRole | Grants | Bound by default to | Source |
|---|---|---|---|
| `system:kube-scheduler` | read nodes, pods, workloads, PDBs, volumes and storage objects; `delete` pods; `create` `pods/binding`; `patch` `pods/status`; its own `leases`; DRA objects | user `system:kube-scheduler` ([L727](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/policy.go#L727)) | [L632-698](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/policy.go#L632-L698) |
| `system:volume-scheduler` | read/update `persistentvolumes`, `persistentvolumeclaims`; read `storageclasses` | user `system:kube-scheduler` ([L728](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/policy.go#L728)) | [L549](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/policy.go#L549) |
| `system:kube-controller-manager` | `list`/`watch` everything; manage `secrets` and `serviceaccounts`; `create` `serviceaccounts/token`; its own `leases` | user `system:kube-controller-manager` ([L725](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/policy.go#L725)) | [L493](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/policy.go#L493) |
| `system:node` | what a kubelet needs ([L214-312](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/policy.go#L214-L312)) | **no subject**: the binding exists but is empty ([L732-735](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/policy.go#L732-L735)); kubelets are authorized by the Node authorizer | [L452](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/policy.go#L452) |
| `system:node-proxier` | `list`/`watch` `services`, `endpoints`, `endpointslices`, `servicecidrs`; read nodes; events | user `system:kube-proxy` ([L724](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/policy.go#L724)) | [L616-630](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/policy.go#L616-L630) |

With `--use-service-account-credentials`, the kube-controller-manager runs each controller under its own ServiceAccount, with the controller roles below.

## Other component ClusterRoles

Ready for add-ons and extensions. **Not bound by default**, except `system:kube-dns`. Docs: [Other component roles](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#other-component-roles).

| ClusterRole | Grants | Source |
|---|---|---|
| `system:auth-delegator` | `create` `tokenreviews` and `subjectaccessreviews`: delegate authentication and authorization to the apiserver | [L475](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/policy.go#L475) |
| `system:kube-aggregator` | read `services`, `endpoints` | [L484](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/policy.go#L484) |
| `system:kube-dns` | `list`/`watch` `endpoints`, `services`. Bound to ServiceAccount `kube-system/kube-dns` ([L726](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/policy.go#L726)) | [L514](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/policy.go#L514) |
| `system:kubelet-api-admin` | full access to the kubelet API: `nodes/proxy`, `nodes/metrics`, `nodes/stats`, `nodes/log`, and `nodes/pods`, `nodes/healthz`, `nodes/configz` (feature gate `KubeletFineGrainedAuthz`, on) | [L597-613](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/policy.go#L597-L613) |
| `system:node-bootstrapper` | `create`/`get`/`list`/`watch` `certificatesigningrequests`: kubelet TLS bootstrap | [L467](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/policy.go#L467) |
| `system:node-problem-detector` | `get` nodes, `patch` `nodes/status`, events | [L458](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/policy.go#L458) |
| `system:persistent-volume-provisioner` | create/delete `persistentvolumes`, update `persistentvolumeclaims`, read `storageclasses` | [L521](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/policy.go#L521) |
| `system:heapster` | read events, pods, nodes, namespaces (legacy) | [L444](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/policy.go#L444) |
| `system:certificates.k8s.io:certificatesigningrequests:nodeclient` | `create` `certificatesigningrequests/nodeclient` | [L536](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/policy.go#L536) |
| `system:certificates.k8s.io:certificatesigningrequests:selfnodeclient` | `create` `certificatesigningrequests/selfnodeclient` | [L543](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/policy.go#L543) |
| `system:certificates.k8s.io:legacy-unknown-approver` | `approve` for signer `kubernetes.io/legacy-unknown` | [L557](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/policy.go#L557) |
| `system:certificates.k8s.io:kubelet-serving-approver` | `approve` for signer `kubernetes.io/kubelet-serving` | [L563](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/policy.go#L563) |
| `system:certificates.k8s.io:kube-apiserver-client-approver` | `approve` for signer `kubernetes.io/kube-apiserver-client` | [L569](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/policy.go#L569) |
| `system:certificates.k8s.io:kube-apiserver-client-kubelet-approver` | `approve` for signer `kubernetes.io/kube-apiserver-client-kubelet` | [L575](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/policy.go#L575) |

## Aggregation building blocks

The rules of `admin`, `edit` and `view` actually live here. A ClusterRole labeled `rbac.authorization.k8s.io/aggregate-to-view: "true"` (or `-edit`, `-admin`) is merged into the matching role: that is how an operator exposes its CRDs without touching the built-in roles. Docs: [Aggregated ClusterRoles](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#aggregated-clusterroles).

| ClusterRole | Label | Merged into | Source |
|---|---|---|---|
| `system:aggregate-to-view` | `aggregate-to-view` | `view` | [L439](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/policy.go#L439) |
| `system:aggregate-to-edit` | `aggregate-to-edit` | `edit` | [L433](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/policy.go#L433) |
| `system:aggregate-to-admin` | `aggregate-to-admin` | `admin` | [L422](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/policy.go#L422) |
| `view` | `aggregate-to-edit` | `edit` | [L413](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/policy.go#L413) |
| `edit` | `aggregate-to-admin` | `admin` | [L403](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/policy.go#L403) |

`cluster-admin` is not aggregated: it holds `*`.

## Controller ClusterRoles

One ClusterRole per controller of the kube-controller-manager, named `system:controller:<name>` and bound to the ServiceAccount `kube-system/<name>` ([controller_policy.go L36-53](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/controller_policy.go#L36-L53)). Each controller gets only the rights it needs. Docs: [Roles for built-in controllers](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#controller-roles).

| ClusterRole | ServiceAccount | Condition | Source |
|---|---|---|---|
| `system:controller:attachdetach-controller` | `kube-system/attachdetach-controller` | always | [L67](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/controller_policy.go#L67) |
| `system:controller:certificate-controller` | `kube-system/certificate-controller` | always | [L457](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/controller_policy.go#L457) |
| `system:controller:clusterrole-aggregation-controller` | `kube-system/clusterrole-aggregation-controller` | always | [L85](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/controller_policy.go#L85) |
| `system:controller:cronjob-controller` | `kube-system/cronjob-controller` | always | [L92](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/controller_policy.go#L92) |
| `system:controller:daemon-set-controller` | `kube-system/daemon-set-controller` | always | [L103](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/controller_policy.go#L103) |
| `system:controller:deployment-controller` | `kube-system/deployment-controller` | always | [L116](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/controller_policy.go#L116) |
| `system:controller:device-taint-eviction-controller` | `kube-system/device-taint-eviction-controller` | feature gate `DRADeviceTaints`, on | [L251](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/controller_policy.go#L251) |
| `system:controller:disruption-controller` | `kube-system/disruption-controller` | always | [L130](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/controller_policy.go#L130) |
| `system:controller:endpoint-controller` | `kube-system/endpoint-controller` | always | [L146](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/controller_policy.go#L146) |
| `system:controller:endpointslice-controller` | `kube-system/endpointslice-controller` | always | [L156](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/controller_policy.go#L156) |
| `system:controller:endpointslicemirroring-controller` | `kube-system/endpointslicemirroring-controller` | always | [L168](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/controller_policy.go#L168) |
| `system:controller:ephemeral-volume-controller` | `kube-system/ephemeral-volume-controller` | always | [L194](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/controller_policy.go#L194) |
| `system:controller:expand-controller` | `kube-system/expand-controller` | always | [L184](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/controller_policy.go#L184) |
| `system:controller:generic-garbage-collector` | `kube-system/generic-garbage-collector` | always | [L277](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/controller_policy.go#L277) |
| `system:controller:horizontal-pod-autoscaler` | `kube-system/horizontal-pod-autoscaler` | always | [L285](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/controller_policy.go#L285) |
| `system:controller:job-controller` | `kube-system/job-controller` | always | [L311](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/controller_policy.go#L311) |
| `system:controller:kube-apiserver-serving-clustertrustbundle-publisher` | `kube-system/kube-apiserver-serving-clustertrustbundle-publisher` | feature gate `ClusterTrustBundle`, on | [L535](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/controller_policy.go#L535) |
| `system:controller:legacy-service-account-token-cleaner` | `kube-system/legacy-service-account-token-cleaner` | always | [L570](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/controller_policy.go#L570) |
| `system:controller:namespace-controller` | `kube-system/namespace-controller` | always | [L315](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/controller_policy.go#L315) |
| `system:controller:node-controller` | `kube-system/node-controller` | always | [L324](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/controller_policy.go#L324) |
| `system:controller:persistent-volume-binder` | `kube-system/persistent-volume-binder` | always | [L338](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/controller_policy.go#L338) |
| `system:controller:pod-garbage-collector` | `kube-system/pod-garbage-collector` | always | [L355](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/controller_policy.go#L355) |
| `system:controller:podcertificaterequestcleaner` | `kube-system/podcertificaterequestcleaner` | feature gate `PodCertificateRequest`, on | [L474](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/controller_policy.go#L474) |
| `system:controller:podgroup-protection-controller` | `kube-system/podgroup-protection-controller` | feature gate `GenericWorkload`, off | [L510](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/controller_policy.go#L510) |
| `system:controller:pv-protection-controller` | `kube-system/pv-protection-controller` | always | [L490](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/controller_policy.go#L490) |
| `system:controller:pvc-protection-controller` | `kube-system/pvc-protection-controller` | always | [L481](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/controller_policy.go#L481) |
| `system:controller:replicaset-controller` | `kube-system/replicaset-controller` | always | [L365](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/controller_policy.go#L365) |
| `system:controller:replication-controller` | `kube-system/replication-controller` | always | [L375](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/controller_policy.go#L375) |
| `system:controller:resource-claim-controller` | `kube-system/resource-claim-controller` | feature gate `DynamicResourceAllocation`, on | [L224](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/controller_policy.go#L224) |
| `system:controller:resourcepoolstatusrequest-controller` | `kube-system/resourcepoolstatusrequest-controller` | feature gate `DRAResourcePoolStatus`, off | [L258](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/controller_policy.go#L258) |
| `system:controller:resourcequota-controller` | `kube-system/resourcequota-controller` | always | [L386](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/controller_policy.go#L386) |
| `system:controller:root-ca-cert-publisher` | `kube-system/root-ca-cert-publisher` | always | [L526](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/controller_policy.go#L526) |
| `system:controller:route-controller` | `kube-system/route-controller` | always | [L395](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/controller_policy.go#L395) |
| `system:controller:selinux-warning-controller` | `kube-system/selinux-warning-controller` | feature gate `SELinuxChangePolicy`, on | [L595](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/controller_policy.go#L595) |
| `system:controller:service-account-controller` | `kube-system/service-account-controller` | always | [L403](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/controller_policy.go#L403) |
| `system:controller:service-cidrs-controller` | `kube-system/service-cidrs-controller` | feature gate `MultiCIDRServiceAllocator`, on | [L420](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/controller_policy.go#L420) |
| `system:controller:service-controller` | `kube-system/service-controller` | always | [L410](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/controller_policy.go#L410) |
| `system:controller:statefulset-controller` | `kube-system/statefulset-controller` | always | [L432](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/controller_policy.go#L432) |
| `system:controller:storage-version-garbage-collector` | `kube-system/storage-version-garbage-collector` | feature gate `StorageVersionAPI`, off | [L558](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/controller_policy.go#L558) |
| `system:controller:storage-version-migrator-controller` | `kube-system/storage-version-migrator-controller` | feature gate `StorageVersionMigrator`, on | [L580](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/controller_policy.go#L580) |
| `system:controller:ttl-after-finished-controller` | `kube-system/ttl-after-finished-controller` | always | [L519](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/controller_policy.go#L519) |
| `system:controller:ttl-controller` | `kube-system/ttl-controller` | always | [L450](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/controller_policy.go#L450) |
| `system:controller:validatingadmissionpolicy-status-controller` | `kube-system/validatingadmissionpolicy-status-controller` | always | [L545](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/controller_policy.go#L545) |
| `system:controller:volumeattributesclass-protection-controller` | `kube-system/volumeattributesclass-protection-controller` | feature gate `VolumeAttributesClass`, on | [L498](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/controller_policy.go#L498) |

## Namespaced Roles

A few Roles live in `kube-system` and `kube-public`. Docs: [Default roles and role bindings](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#default-roles-and-role-bindings).

| Role | Namespace | Grants | Bound to | Source |
|---|---|---|---|---|
| `extension-apiserver-authentication-reader` | `kube-system` | read the configmap `extension-apiserver-authentication` (CA bundles for front-proxy and client certificates) | users `system:kube-controller-manager`, `system:kube-scheduler` ([L124](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/namespace_policy.go#L124)) | [L77](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/namespace_policy.go#L77) |
| `system::leader-locking-kube-controller-manager` | `kube-system` | read/write `leases`, `leasecandidates`: leader election | user `system:kube-controller-manager`, SAs `kube-controller-manager`, `leader-election-controller` ([L130](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/namespace_policy.go#L130)) | [L109](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/namespace_policy.go#L109) |
| `system::leader-locking-kube-scheduler` | `kube-system` | read/write `leases`, `leasecandidates`: leader election | user `system:kube-scheduler`, SA `kube-scheduler` ([L132](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/namespace_policy.go#L132)) | [L117](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/namespace_policy.go#L117) |
| `system:controller:bootstrap-signer` | `kube-system` | read `secrets` (bootstrap tokens) | SA `kube-system/bootstrap-signer` ([L134](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/namespace_policy.go#L134)) | [L85](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/namespace_policy.go#L85) |
| `system:controller:cloud-provider` | `kube-system` | create/read `configmaps` | SA `kube-system/cloud-provider` ([L137](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/namespace_policy.go#L137)) | [L93](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/namespace_policy.go#L93) |
| `system:controller:token-cleaner` | `kube-system` | read/delete `secrets` (expired bootstrap tokens) | SA `kube-system/token-cleaner` ([L139](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/namespace_policy.go#L139)) | [L100](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/namespace_policy.go#L100) |
| `system:controller:bootstrap-signer` | `kube-public` | read `configmaps`, update the configmap `cluster-info` | SA `kube-system/bootstrap-signer` ([L151](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/namespace_policy.go#L151)) | [L143](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/namespace_policy.go#L143) |

---

Found an error, or a line that moved in a newer release? Open an issue: every entry must point to the line that defines it.
