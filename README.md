# The Cloud Native Journey

Kubernetes cheatsheets, design patterns and under-the-hood notes, sourced line by line from the code. Companion to [The Cloud Native Journey](https://hune-lab.com/the-cloud-native-journey), a series that reads the Kubernetes source code.

Every entry links to the exact line of the source that defines it, at a pinned commit, and to the official documentation.

## Journeys

### [apiserver journey](apiserver-journey/)

- [Authenticators: what they check, how they are enabled and assembled](apiserver-journey/cheatsheets/authenticators.md) (Kubernetes v1.37.2)
- [Built-in API groups, resources and subresources](apiserver-journey/cheatsheets/api-groups.md) (Kubernetes v1.37.2)
- [Default RBAC: roles, bindings and identities](apiserver-journey/cheatsheets/rbac-defaults.md) (Kubernetes v1.37.2)
- Design pattern: [Decorator, as a function](apiserver-journey/design-patterns/decorator-function.md) (Go 1.26.10, Kubernetes v1.37.2)
- Design pattern: [Decorator, as a struct](apiserver-journey/design-patterns/decorator-struct.md) (Go 1.26.10, Kubernetes v1.37.2)
- Design pattern: [Chain of responsibility: the authenticators and the impersonation modes](apiserver-journey/design-patterns/chain-of-responsibility.md) (Kubernetes v1.37.2)
- Design pattern: [Composite: the union of authenticators](apiserver-journey/design-patterns/composite.md) (Kubernetes v1.37.2)
- Under the hood: [From the audit flags to WithAudit](apiserver-journey/under-the-hood/audit-activation.md) (Kubernetes v1.37.2, RKE2 v1.37.2+rke2r1)
- Under the hood: [The impersonation modes, one by one](apiserver-journey/under-the-hood/impersonation-modes.md) (Kubernetes v1.37.2)
