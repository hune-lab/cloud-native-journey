# Kubernetes built-in API groups and resources

The API groups a kube-apiserver serves out of the box, and their resources, read from the source of **Kubernetes v1.37.2** (commit [`c50e5395f1ae`](https://github.com/kubernetes/kubernetes/tree/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32)). Each row links to the code that registers the resources.

Part of [The Cloud Native Journey](https://hune-lab.com/the-cloud-native-journey), the apiserver journey.

## What is listed

Resources served **by default**: stable versions only, and no resource behind a feature gate that is off by default. Alpha and beta versions are disabled unless you enable them with `--runtime-config` ([instance.go L538-547](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/pkg/controlplane/instance.go#L538-L547)).

CRDs add their own groups, so the set is open. To see what your cluster actually serves:

```sh
kubectl api-resources
kubectl api-versions
```

Docs: [The Kubernetes API](https://kubernetes.io/docs/concepts/overview/kubernetes-api/), [API groups and versioning](https://kubernetes.io/docs/concepts/overview/kubernetes-api/#api-groups-and-versioning).

## Groups

| Group | Resources | Source |
|---|---|---|
| `""` (core) | `pods`, `services`, `endpoints`, `nodes`, `namespaces`, `configmaps`, `secrets`, `serviceaccounts`, `persistentvolumes`, `persistentvolumeclaims`, `replicationcontrollers`, `events`, `resourcequotas`, `limitranges`, `podtemplates`, `bindings`, `componentstatuses` | [`storage_core.go` L248-330](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/pkg/registry/core/rest/storage_core.go#L248-L330), [`storage_core_generic.go` L125-151](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/pkg/registry/core/rest/storage_core_generic.go#L125-L151) |
| `apps` | `deployments`, `statefulsets`, `daemonsets`, `replicasets`, `controllerrevisions` | [`storage_apps.go` L56-99](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/pkg/registry/apps/rest/storage_apps.go#L56-L99) |
| `batch` | `jobs`, `cronjobs` | [`storage_batch.go` L51-61](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/pkg/registry/batch/rest/storage_batch.go#L51-L61) |
| `autoscaling` | `horizontalpodautoscalers` | [`storage_autoscaling.go` L70](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/pkg/registry/autoscaling/rest/storage_autoscaling.go#L70), [`storage_autoscaling.go` L86](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/pkg/registry/autoscaling/rest/storage_autoscaling.go#L86) |
| `policy` | `poddisruptionbudgets` | [`storage_policy.go` L49](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/pkg/registry/policy/rest/storage_policy.go#L49) |
| `networking.k8s.io` | `networkpolicies`, `ingresses`, `ingressclasses`, `ipaddresses`, `servicecidrs` | [`storage_settings.go` L61-98](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/pkg/registry/networking/rest/storage_settings.go#L61-L98) |
| `discovery.k8s.io` | `endpointslices` | [`storage_discovery.go` L51](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/pkg/registry/discovery/rest/storage_discovery.go#L51) |
| `events.k8s.io` | `events` | [`storage_events.go` L54](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/pkg/registry/events/rest/storage_events.go#L54) |
| `storage.k8s.io` | `storageclasses`, `volumeattachments`, `volumeattributesclasses`, `csinodes`, `csidrivers`, `csistoragecapacities` | [`storage_storage.go` L86-125](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/pkg/registry/storage/rest/storage_storage.go#L86-L125) |
| `rbac.authorization.k8s.io` | `roles`, `rolebindings`, `clusterroles`, `clusterrolebindings` | [`storage_rbac.go` L109-124](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/pkg/registry/rbac/rest/storage_rbac.go#L109-L124) |
| `apiextensions.k8s.io` | `customresourcedefinitions` | [`apiserver.go` L150](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiextensions-apiserver/pkg/apiserver/apiserver.go#L150) |
| `apiregistration.k8s.io` | `apiservices` | [`storage_apiservice.go` L38](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/kube-aggregator/pkg/registry/apiservice/rest/storage_apiservice.go#L38) |
| `admissionregistration.k8s.io` | `validatingwebhookconfigurations`, `mutatingwebhookconfigurations`, `validatingadmissionpolicies`, `validatingadmissionpolicybindings`, `mutatingadmissionpolicies`, `mutatingadmissionpolicybindings` | [`storage_apiserver.go` L82-130](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/pkg/registry/admissionregistration/rest/storage_apiserver.go#L82-L130) |
| `certificates.k8s.io` | `certificatesigningrequests`, `clustertrustbundles`, `podcertificaterequests` | [`storage_certificates.go` L79-101](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/pkg/registry/certificates/rest/storage_certificates.go#L79-L101) |
| `coordination.k8s.io` | `leases` | [`storage_coordination.go` L65](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/pkg/registry/coordination/rest/storage_coordination.go#L65) |
| `authentication.k8s.io` | `tokenreviews`, `selfsubjectreviews` | [`storage_authentication.go` L63-67](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/pkg/registry/authentication/rest/storage_authentication.go#L63-L67) |
| `authorization.k8s.io` | `subjectaccessreviews`, `selfsubjectaccessreviews`, `localsubjectaccessreviews`, `selfsubjectrulesreviews` | [`storage_authorization.go` L59-74](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/pkg/registry/authorization/rest/storage_authorization.go#L59-L74) |
| `scheduling.k8s.io` | `priorityclasses` | [`storage_scheduling.go` L82](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/pkg/registry/scheduling/rest/storage_scheduling.go#L82) |
| `node.k8s.io` | `runtimeclasses` | [`runtime_class.go` L49](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/pkg/registry/node/rest/runtime_class.go#L49) |
| `flowcontrol.apiserver.k8s.io` | `flowschemas`, `prioritylevelconfigurations` | [`storage_flowcontrol.go` L72-82](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/pkg/registry/flowcontrol/rest/storage_flowcontrol.go#L72-L82) |
| `resource.k8s.io` | `deviceclasses`, `devicetaintrules`, `resourceclaims`, `resourceclaimtemplates`, `resourceslices` | [`storage_resource.go` L84-118](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/pkg/registry/resource/rest/storage_resource.go#L84-L118) |
| `storagemigration.k8s.io` | `storageversionmigrations` | [`storage_storagemigration.go` L77](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/pkg/registry/storagemigration/rest/storage_storagemigration.go#L77) |

The core group has no name: its resources are served under `/api/v1`, all the others under `/apis/<group>/<version>`.
