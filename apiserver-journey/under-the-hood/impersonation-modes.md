# The impersonation modes, one by one

When a request carries `Impersonate-*` headers, the constrained impersonation handler walks five modes, from the narrowest to the widest, and stops at the first one that grants the impersonation. This note shows what each mode checks, which RBAC rules it needs, and what the caller ends up with. Source: **Kubernetes v1.37.2** (commit [`c50e5395f1ae`](https://github.com/kubernetes/kubernetes/tree/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32)).

Part of [The Cloud Native Journey](https://hune-lab.com/the-cloud-native-journey), the apiserver journey. The walk itself is a [chain of responsibility](../design-patterns/chain-of-responsibility.md).

The five modes, in order ([`mode.go` L60-68](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/endpoints/filters/impersonation/mode.go#L60-L68)): `associated-node`, `arbitrary-node`, `serviceaccount`, `user-info`, then `legacy`.

## What a constrained mode checks

The first four modes are constrained. They all run the same `check`; only their filter differs ([`mode.go` L272-294](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/endpoints/filters/impersonation/mode.go#L272-L294)):

```go
func (c *constrainedImpersonationModeState) check(ctx context.Context, key *impersonationCacheKey, wantedUser *user.DefaultInfo, attributes authorizer.Attributes) (*impersonatedUserInfo, error) {
	requestor := attributes.GetUser()
	if !c.filter(wantedUser, requestor) {
		return nil, nil // 1. this mode does not apply
	}

	if impersonatedUser := c.cache.get(key); impersonatedUser != nil {
		return impersonatedUser, nil // granted recently
	}

	if err := checkAuthorization(ctx, c.authorizer, &impersonateOnAttributes{mode: c.mode, Attributes: attributes}); err != nil {
		return nil, err // 2. refused: impersonate-on:<mode>:<verb>
	}

	impersonatedUser, err := c.state.check(ctx, key, wantedUser, requestor)
	if err != nil || impersonatedUser == nil {
		return nil, err // 3. refused: impersonate:<mode>
	}
	c.cache.set(key, impersonatedUser)
	return impersonatedUser, nil
}
```

`requestor` is the caller, the identity that authentication stored in the request context. Every question below is asked on its behalf ([L536-546](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/endpoints/filters/impersonation/mode.go#L536-L546)).

1. **Does this mode apply?** `c.filter` is not a method: it is a struct field holding a function, and each mode gets its own when it is built ([L56-58](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/endpoints/filters/impersonation/mode.go#L56-L58), [L269](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/endpoints/filters/impersonation/mode.go#L269)). It looks at the wanted identity and at the caller, and calls no authorizer. If it says no, the mode returns `nil, nil` and the loop moves on.

   ```go
   type constrainedImpersonationModeFilter func(wantedUser *user.DefaultInfo, requestor user.Info) bool
   ```

2. **May the caller perform this action through this mode?** `checkAuthorization` asks the apiserver's authorizer and returns an error unless the answer is allow ([L559-581](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/endpoints/filters/impersonation/mode.go#L559-L581)). The authorizer receives the real request (same resource, same namespace, the real caller) with a rewritten verb ([L555-557](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/endpoints/filters/impersonation/mode.go#L555-L557)):

   ```go
   func (i *impersonateOnAttributes) GetVerb() string {
   	return "impersonate-on:" + i.mode + ":" + i.Attributes.GetVerb()
   }
   ```

   So `create` becomes `impersonate-on:<mode>:create`. The authorizer does not check that the caller may perform the action itself, only that it was granted the right to perform it while impersonating through this mode.

3. **May the caller impersonate this identity?** `c.state` is another field of the mode. For each field of the wanted identity (its name and, when requested, its UID, groups and extras), the caller needs the verb `impersonate:<mode>` in the `authentication.k8s.io` group ([L378-412](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/endpoints/filters/impersonation/mode.go#L378-L412)). The resource for the name depends on the identity: `nodes`, `serviceaccounts` or `users`. UIDs are checked on `uids`, groups on `groups`, extras on `userextras`.

The second question bounds the actions, the third the identities. When everything is granted, the borrowed identity gets `system:authenticated` ([L365-369](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/endpoints/filters/impersonation/mode.go#L365-L369)). Then, once the identity is replaced, `WithAuthorization` checks the request again as the borrowed identity: the caller can only do what is allowed both by its `impersonate-on` verbs and to the identity it borrows.

## The helper functions

The filters are built from a few small functions. They only check the shape of names: nothing is read from the cluster.

- `onlyUsernameSet` ([L635-637](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/endpoints/filters/impersonation/mode.go#L635-L637)): only the name was requested, no UID, no group, no extra. A request that also carries `Impersonate-Uid`, `Impersonate-Group` or `Impersonate-Extra-*` fails it. It is not rejected for all that: the mode just does not apply, and the loop moves on.
- `isNodeUsername` ([L604-614](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/endpoints/filters/impersonation/mode.go#L604-L614)): the name is `system:node:<name>`, and `<name>` is a valid node name, a DNS subdomain (lowercase letters, digits, dashes and dots, at most 253 characters).
- `isServiceAccountUsername` ([L595-598](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/endpoints/filters/impersonation/mode.go#L595-L598)): it hands the name to `SplitUsername` ([`serviceaccount/util.go` L107-124](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/authentication/serviceaccount/util.go#L107-L124)), which checks the `system:serviceaccount:` prefix, then exactly two parts, `<namespace>:<name>`: a DNS label for the namespace (lowercase letters, digits and dashes, at most 63 characters), a DNS subdomain for the name. The service account does not need to exist.
- `getExtraValue` ([L627-633](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/endpoints/filters/impersonation/mode.go#L627-L633)): the single value of an extra key of the caller.

## associated-node: an agent impersonates its own node

A DaemonSet runs on every node and needs the rights of the node that hosts it, and of that node only. Instead of giving its service account broad rights, it impersonates `system:node:<its node>`. The agent running on `node-1` sends:

```http
GET /api/v1/pods?fieldSelector=spec.nodeName%3Dnode-1 HTTP/2
Authorization: Bearer <token of the agent's service account>
Impersonate-User: system:node:node-1
```

**The filter** ([L89-92](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/endpoints/filters/impersonation/mode.go#L89-L92), [L616-625](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/endpoints/filters/impersonation/mode.go#L616-L625)):

```go
func(wantedUser *user.DefaultInfo, requestor user.Info) bool {
	wantedNodeName := wantedUser.Name
	return onlyUsernameSet(wantedUser) && requesterAssociatedWithRequestedNodeUsername(requestor, wantedNodeName)
}

func requesterAssociatedWithRequestedNodeUsername(requestor user.Info, username string) bool {
	nodeName, ok := isNodeUsername(username)
	if !ok {
		return false
	}
	if _, _, ok := isServiceAccountUsername(requestor.GetName()); !ok {
		return false
	}
	return getExtraValue(requestor, serviceaccount.NodeNameKey) == nodeName
}
```

Four conditions: only the name is requested; it is a node name; the caller is a service account; the node in the caller's extras, under `authentication.kubernetes.io/node-name`, is the requested node. That node comes from the service account token, which the apiserver itself issued and signed for this pod ([`pkg/serviceaccount/claims.go` L104-117](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/pkg/serviceaccount/claims.go#L104-L117), [`serviceaccount/util.go` L178-186](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/authentication/serviceaccount/util.go#L178-L186)): the agent cannot claim to run elsewhere without forging the token. If it asks for `system:node:node-2`, this mode is not a refusal: it just does not apply, and the loop moves to `arbitrary-node`, which needs much wider rights. The filter grants nothing yet.

**The two authorizations**, asked on behalf of the agent's service account:

| Verb | Group, resource, name | Question |
|---|---|---|
| `impersonate-on:associated-node:list` | `""`, `pods`, all namespaces | may it list pods while impersonating its node? |
| `impersonate:associated-node` | `authentication.k8s.io`, `nodes`, `*` | may it impersonate its node? |

The name `*` is not a mistake: for this check, `associated-node` replaces the node name with `*` ([L143-148](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/endpoints/filters/impersonation/mode.go#L143-L148)). The rule must hold whatever node the pod runs on, since the filter already checked that it is its own. Without a rule, both are refused:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-agent-impersonation
rules:
- apiGroups: ["authentication.k8s.io"]
  resources: ["nodes"]
  verbs: ["impersonate:associated-node"]       # impersonate its node
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["impersonate-on:associated-node:list", "impersonate-on:associated-node:get"]   # and only for this
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: node-agent-impersonation
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: node-agent-impersonation
subjects:
- kind: ServiceAccount
  name: node-agent
  namespace: monitoring
```

The agent may then list and read pods as its node, and nothing else: deleting a pod or reading a secret as the node stays refused.

**What it gets.** The identity `system:node:node-1`, with groups fixed by the apiserver, `system:nodes` then `system:authenticated` ([L382-392](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/endpoints/filters/impersonation/mode.go#L382-L392)). `WithAuthorization` then judges the request as that node. With the `Node` authorizer (`--authorization-mode=Node,RBAC`), a node may only list the pods whose `spec.nodeName` is its own, hence the field selector ([`node_authorizer.go` L467-482](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/plugin/pkg/auth/authorizer/node/node_authorizer.go#L467-L482)).

Three barriers: the token says which node, RBAC says which actions, the node restrictions say on which objects.

## arbitrary-node: impersonate any node

A central component, an admin or test tool, sometimes needs to see the cluster as the kubelet of a given node, whichever it is.

```shell
kubectl get node node-2 --as=system:node:node-2
```

**The filter** ([L177-187](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/endpoints/filters/impersonation/mode.go#L177-L187)):

```go
func(wantedUser *user.DefaultInfo, _ user.Info) bool {
	if !onlyUsernameSet(wantedUser) {
		return false
	}
	_, ok := isNodeUsername(wantedUser.Name)
	return ok
}
```

The caller is not looked at: anyone may enter this mode, provided they ask for a node by its name alone. Everything rests on the authorizations.

**The two authorizations**, if the tool reads the pod `web` in `demo` as `node-2`:

| Verb | Group, resource, namespace, name | Question |
|---|---|---|
| `impersonate-on:arbitrary-node:get` | `""`, `pods`, `demo`, `web` | may it read this pod while impersonating a node? |
| `impersonate:arbitrary-node` | `authentication.k8s.io`, `nodes`, `node-2` | may it impersonate that node? |

Unlike `associated-node`, the checked name is the requested node: a rule can be limited to some nodes with `resourceNames`, or opened to all. It is a wider right, hence its place right after `associated-node` in the loop.

**What it gets.** As with `associated-node`, `system:node:node-2` with `system:nodes` and `system:authenticated`, fixed by the apiserver.

## serviceaccount: act as a service account

A deployment tool applies a team's manifests with the rights of that team's service account, not its own. Flux, for instance, does so when a `Kustomization` sets `serviceAccountName`.

```shell
kubectl apply -n demo -f deployment.yaml --as=system:serviceaccount:demo:deployer
```

**The filter** ([L191-201](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/endpoints/filters/impersonation/mode.go#L191-L201)):

```go
func(wantedUser *user.DefaultInfo, _ user.Info) bool {
	if !onlyUsernameSet(wantedUser) {
		return false
	}
	_, _, ok := isServiceAccountUsername(wantedUser.Name)
	return ok
}
```

`--as=system:serviceaccount:demo:web` passes. `--as=alice` does not, nor does `--as=system:serviceaccount:demo:web --as-group=team-a`, which asks for a group on top of the name.

**The two authorizations**, for a tool that creates a deployment in `demo` as `system:serviceaccount:demo:deployer`:

| Verb | Group, resource, namespace, name | Question |
|---|---|---|
| `impersonate-on:serviceaccount:create` | `apps`, `deployments`, `demo` | may it create a deployment while impersonating a service account? |
| `impersonate:serviceaccount` | `authentication.k8s.io`, `serviceaccounts`, `demo`, `deployer` | may it impersonate that service account? |

Both checks are in the `demo` namespace, so a team can grant them itself, with a plain `Role`, without any right on the rest of the cluster:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deploy-as-deployer
  namespace: demo
rules:
- apiGroups: ["authentication.k8s.io"]
  resources: ["serviceaccounts"]
  resourceNames: ["deployer"]
  verbs: ["impersonate:serviceaccount"]          # impersonate deployer, and only it
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["impersonate-on:serviceaccount:create", "impersonate-on:serviceaccount:update", "impersonate-on:serviceaccount:patch"]
```

**What it gets.** `system:serviceaccount:demo:deployer`, with the groups the apiserver derives from the namespace, `system:serviceaccounts` and `system:serviceaccounts:demo`, then `system:authenticated` ([L395-408](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/endpoints/filters/impersonation/mode.go#L395-L408)). No extras: no pod, no node, since only the name could be requested.

## user-info: act as a user and their groups

An authenticating proxy in front of the apiserver authenticates users itself, with OIDC or an SSO, then calls the apiserver with its own identity and passes the user's on. An administrator checking someone's rights does the same:

```shell
kubectl auth can-i create deployments -n demo --as=alice --as-group=devs --as-group=team-a
```

**The filter** ([L205-220](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/endpoints/filters/impersonation/mode.go#L205-L220)):

```go
func(wantedUser *user.DefaultInfo, _ user.Info) bool {
	// nodes and service accounts cannot be impersonated in this mode.
	if _, ok := isNodeUsername(wantedUser.Name); ok {
		return false
	}
	if _, _, ok := isServiceAccountUsername(wantedUser.Name); ok {
		return false
	}
	return true
}
```

It is the only constrained mode that accepts every field: name, UID, groups and extras. In return it excludes nodes and service accounts, whose groups are fixed by the apiserver.

**The authorizations.** If the proxy relays Alice creating a deployment in `demo`, the action question is about `create`; the identity question splits into one `impersonate:user-info` check per requested field, in `authentication.k8s.io`:

| Verb | Resource, name | Question |
|---|---|---|
| `impersonate-on:user-info:create` | `deployments`, in `demo` | may it create a deployment while impersonating a user? |
| `impersonate:user-info` | `users`, `alice` | may it impersonate the name `alice`? |
| `impersonate:user-info` | `groups`, `devs` | may it impersonate the group `devs`? |
| `impersonate:user-info` | `groups`, `team-a` | may it impersonate the group `team-a`? |

A UID would add a check on `uids`, each extra a check on `userextras`. From four groups on, the apiserver first tries `groups/*`, one call instead of four. Two groups are refused whatever RBAC says: the empty group, and `system:masters`, which would bypass all authorization ([L433-442](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/endpoints/filters/impersonation/mode.go#L433-L442)).

**What it gets.** Exactly the requested identity, plus `system:authenticated`: here `alice` with `devs`, `team-a` and `system:authenticated`.

## legacy: the historical impersonation

Existing clusters have always granted the verb `impersonate` on `users`, `groups` or `serviceaccounts`, and many tools rely on it. The `legacy` mode keeps these rules working unchanged ([L222-235](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/endpoints/filters/impersonation/mode.go#L222-L235)):

```go
// legacyImpersonationMode is a complete reimplementation of the original impersonation mode that has
// existed in kube since v1.3.  The behavior is expected to be identical to the original implementation.

func (l *legacyImpersonationCheck) check(ctx context.Context, key *impersonationCacheKey, wantedUser *user.DefaultInfo, attributes authorizer.Attributes) (*impersonatedUserInfo, error) {
	requestor := attributes.GetUser()
	return l.m.check(ctx, key, wantedUser, requestor)
}
```

**No filter, no question on the action.** Its `check` goes straight to the identity check, the same as the third question of the constrained modes, set with the verb `impersonate` ([L224-226](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/endpoints/filters/impersonation/mode.go#L224-L226)). It applies to any identity. Names, groups and service accounts are checked in the core API group, as the old rules did ([L321-327](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/endpoints/filters/impersonation/mode.go#L321-L327)):

```yaml
rules:
- apiGroups: [""]
  resources: ["users", "groups", "serviceaccounts"]
  verbs: ["impersonate"]
```

**What it gets.** The requested identity, and all its rights with it: once it is borrowed, the caller can do anything as that identity. The safeguards of the constrained modes do not exist here, not even the refusal of `system:masters`, which would break existing uses. This is why `impersonate`, on `groups` in particular, is among the most sensitive rights in a cluster.

This mode is tried last. A caller holding both constrained and historical rules goes through the constrained mode, and the audit event records which one granted the impersonation (`authenticationMetadata.impersonationConstraint`).

Official documentation: [User impersonation](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#user-impersonation).
