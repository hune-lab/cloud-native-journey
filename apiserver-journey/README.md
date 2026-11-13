# apiserver journey

How a request travels through the kube-apiserver, read from the source code of Kubernetes v1.37.2: the handler chain, request info, authentication, priority and fairness, authorization, and the route to the resource.

Articles: [The Cloud Native Journey, apiserver](https://hune-lab.com/the-cloud-native-journey/apiserver/part-1)

## Cheatsheets

- [Authenticators: what they check, how they are enabled and assembled](cheatsheets/authenticators.md)
- [Built-in API groups, resources and subresources](cheatsheets/api-groups.md)
- [Default RBAC: roles, bindings and identities](cheatsheets/rbac-defaults.md)

## Design patterns

The patterns met along the way, each illustrated by the apiserver code that implements it.

- [Decorator, as a function: how a plain function becomes an HTTP handler](design-patterns/decorator-function.md)
- [Decorator, as a struct: a middleware written as a struct](design-patterns/decorator-struct.md)
- [Chain of responsibility: the authenticators and the impersonation modes](design-patterns/chain-of-responsibility.md)
- [Composite: the union of authenticators](design-patterns/composite.md)

## Under the hood

- [From the audit flags to WithAudit](under-the-hood/audit-activation.md)
- [The impersonation modes, one by one: what each checks, which RBAC rules it needs](under-the-hood/impersonation-modes.md)
