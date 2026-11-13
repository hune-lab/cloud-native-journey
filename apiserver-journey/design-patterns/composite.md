# Composite: the union of authenticators

A group of objects is used through the same interface as a single one. Whoever calls it does not know, and does not need to know, whether it talks to one object or to ten. In the kube-apiserver, `WithAuthentication` calls one authenticator, and that one authenticator may be a union of several. Source: **Kubernetes v1.37.2** (commit [`c50e5395f1ae`](https://github.com/kubernetes/kubernetes/tree/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32)).

Part of [The Cloud Native Journey](https://hune-lab.com/the-cloud-native-journey), the apiserver journey.

## One call, one authenticator

`WithAuthentication` makes a single call, `auth.AuthenticateRequest(req)`, on a single `authenticator.Request` ([`endpoints/filters/authentication.go` L68](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/endpoints/filters/authentication.go#L68)). Yet the apiserver usually has several authenticators. The union reconciles the two.

## The union is an authenticator

[`request/union/union.go` L27-71](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/authentication/request/union/union.go#L27-L71), shortened:

```go
type unionAuthRequestHandler struct {
	// Handlers is a chain of request authenticators to delegate to
	Handlers []authenticator.Request
	// FailOnError determines whether an error returns short-circuits the chain
	FailOnError bool
}

func New(authRequestHandlers ...authenticator.Request) authenticator.Request {
	if len(authRequestHandlers) == 1 {
		return authRequestHandlers[0] // one? no need to wrap it
	}
	return &unionAuthRequestHandler{Handlers: authRequestHandlers, FailOnError: false}
}

func (authHandler *unionAuthRequestHandler) AuthenticateRequest(req *http.Request) (*authenticator.Response, bool, error) {
	// … calls AuthenticateRequest on each of its Handlers
}
```

Three things make it a composite:

- it **holds** authenticators: `Handlers []authenticator.Request`;
- it **is** an authenticator: it has the `AuthenticateRequest` method, and `New` returns an `authenticator.Request`;
- its method **calls the same method** on each element. How it walks them is the [chain of responsibility](chain-of-responsibility.md).

`New` adds an original touch: with a single authenticator it returns that authenticator unchanged, with no union around it.

## At startup: none, one or several

[`pkg/kubeapiserver/authenticator/config.go` L231-238](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/pkg/kubeapiserver/authenticator/config.go#L231-L238):

```go
if len(authenticators) == 0 {
	if config.Anonymous.Enabled {
		return anonymous.NewAuthenticator(config.Anonymous.Conditions), nil, &securityDefinitionsV2, securitySchemesV3, nil
	}
	return nil, nil, &securityDefinitionsV2, securitySchemesV3, nil
}

authenticator := union.New(authenticators...)
```

No authenticator configured: anonymous alone, with no union. One or several: `union.New`, which returns the authenticator itself when it is alone, and a union otherwise. In every case `WithAuthentication` receives one `authenticator.Request` and never sees the difference.

## The same shape, one level down: tokens

`bearertoken` and `websocket` extract a token and hand it to `authenticator.Token`, which has its own union, built the same way, shortened here ([`token/union/union.go` L27-71](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/authentication/token/union/union.go#L27-L71)):

```go
type unionAuthTokenHandler struct {
	Handlers    []authenticator.Token
	FailOnError bool
}

func New(authTokenHandlers ...authenticator.Token) authenticator.Token {
	if len(authTokenHandlers) == 1 {
		return authTokenHandlers[0]
	}
	return &unionAuthTokenHandler{Handlers: authTokenHandlers, FailOnError: false}
}
```

So a union of request authenticators can contain `bearertoken`, which calls a union of token authenticators: a composite inside a composite.

Official documentation: [Authenticating](https://kubernetes.io/docs/reference/access-authn-authz/authentication/).
