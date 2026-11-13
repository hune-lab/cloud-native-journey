# Chain of responsibility: the authenticators and the impersonation modes

A request goes through a list of handlers, in order. Each one is asked the same question, "is this mine?", and answers yes, with a result, or no. The first yes ends the walk. In the kube-apiserver, this is how a request finds out who sent it. Source: **Kubernetes v1.37.2** (commit [`c50e5395f1ae`](https://github.com/kubernetes/kubernetes/tree/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32)).

Part of [The Cloud Native Journey](https://hune-lab.com/the-cloud-native-journey), the apiserver journey.

## The pattern

Four parts:

- a collection of links, walked in order (an array, a list);
- one question asked to each of them;
- an answer, yes or no, with the result when it is yes;
- a conductor that does the walking.

## The question: the authenticator contract

Every request authenticator answers the same question with three values ([`authenticator/interfaces.go` L34-36](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/authentication/authenticator/interfaces.go#L34-L36)):

```go
type Request interface {
	AuthenticateRequest(req *http.Request) (*Response, bool, error)
}
```

The three values carry the answer:

| Return | Meaning |
|---|---|
| `resp, true, nil` | "It is me, here is the identity" |
| `nil, false, nil` | "Not for me", for example no client certificate was presented |
| `nil, false, err` | "It was for me, but it is invalid", for example an expired token |

Token authenticators answer the same way, with a token instead of a request ([L28-30](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/authentication/authenticator/interfaces.go#L28-L30)):

```go
type Token interface {
	AuthenticateToken(ctx context.Context, token string) (*Response, bool, error)
}
```

## The links

The request authenticators ([cheatsheet](../cheatsheets/authenticators.md)):

| Authenticator | What it checks |
|---|---|
| front-proxy | the `X-Remote-User`, `X-Remote-Group` headers, accepted only if the connection presents the proxy's certificate |
| x509 | the client certificate, against the CA of `--client-ca-file` |
| bearertoken | the `Authorization: Bearer` header, whose token it hands to the `Token` authenticators |
| websocket | the token passed in the WebSocket protocol, handed to the same `Token` authenticators |
| anonymous | nothing: it produces `system:anonymous` when no one recognised the request |

## The conductor: the union

The walk is done by the union ([`request/union/union.go` L53-71](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/authentication/request/union/union.go#L53-L71)):

```go
func (authHandler *unionAuthRequestHandler) AuthenticateRequest(req *http.Request) (*authenticator.Response, bool, error) {
	var errlist []error
	for _, currAuthRequestHandler := range authHandler.Handlers {
		resp, ok, err := currAuthRequestHandler.AuthenticateRequest(req) // ask each link
		if err != nil {
			if authHandler.FailOnError {
				return resp, ok, err
			}
			errlist = append(errlist, err)
			continue // "mine, but invalid": try the next one
		}
		if ok {
			return resp, ok, err // the first yes wins
		}
	}
	return nil, false, utilerrors.NewAggregate(errlist)
}
```

Two details differ from the textbook pattern:

- **An error does not stop the walk.** With `FailOnError: false`, the default set by `New` ([L36-41](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/authentication/request/union/union.go#L36-L41)), an invalid credential is recorded and the next link is asked. The errors are returned together only if no one says yes.
- **The conductor is itself a link.** The union implements `authenticator.Request`, so whoever calls it cannot tell a chain from a single authenticator. That is the [composite](composite.md).

The token union does the same walk over `authenticator.Token` ([`token/union/union.go` L53-71](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/authentication/token/union/union.go#L53-L71)).

## A second chain: the impersonation modes

The same pattern appears one filter later. When a request carries `Impersonate-*` headers, the constrained impersonation handler must decide whether the caller may borrow that identity. It walks five modes, from the narrowest to the widest ([`endpoints/filters/impersonation/mode.go` L60-68](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/endpoints/filters/impersonation/mode.go#L60-L68)):

```go
func allImpersonationModes(a authorizer.UnconditionalAuthorizer) []impersonationMode {
	return []impersonationMode{
		associatedNodeImpersonationMode(a),
		arbitraryNodeImpersonationMode(a),
		serviceAccountImpersonationMode(a),
		userInfoImpersonationMode(a),
		legacyImpersonationMode(a),
	}
}
```

**The question.** Every mode implements one method ([L44-54](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/endpoints/filters/impersonation/mode.go#L44-L54)):

```go
type impersonationMode interface {
	check(ctx context.Context, key *impersonationCacheKey, wantedUser *user.DefaultInfo, attributes authorizer.Attributes) (*impersonatedUserInfo, error)

	// all methods below are only used in unit tests
	verbForTests() string
	cachesForTests() (outer, inner *impersonationCache)
}
```

Its answer has the same three shapes as the authenticators:

| Return | Meaning |
|---|---|
| `info, nil` | "This mode grants the impersonation", with the borrowed identity |
| `nil, nil` | "This mode does not apply to that identity": try the next one |
| `nil, err` | "This mode applies, but the caller is not allowed" |

**The conductor.** The tracker walks the modes ([`constrained_impersonation.go` L235-276](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/endpoints/filters/impersonation/constrained_impersonation.go#L235-L276)), shortened:

```go
func (t *impersonationModesTracker) getImpersonatedUser(ctx context.Context, wantedUser *user.DefaultInfo, attributes authorizer.Attributes) (*impersonatedUserInfo, error) {
	// …
	// try the last successful mode first to reduce the amortized cost of impersonation
	modeIdx, modeIdxOk := t.idxCache.get(attributes)
	// …
	for i, mode := range t.modes {
		// …
		impersonatedUser, err := mode.check(ctx, key, wantedUser, attributes) // ask each link
		if err != nil {
			// …
			continue // refused by this mode: try the next one
		}
		if impersonatedUser == nil {
			continue // this mode does not apply
		}
		t.idxCache.set(attributes, i) // remember the mode that worked
		return impersonatedUser, nil  // the first yes wins
	}
	// …
	return nil, errors.New("all impersonation modes failed")
}
```

Two differences with the authenticators:

- **The conductor remembers.** It first tries the mode that succeeded last time for that caller, then the others in order. The chain stays the same; only the starting point moves.
- **A link does not decide on its own.** A mode says whether it applies, through a filter, then turns the impersonation into authorization questions (`impersonate:<mode>`, `impersonate-on:<mode>:<verb>`) and asks the apiserver's authorizer, which decides ([`mode.go` L284](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/endpoints/filters/impersonation/mode.go#L284)).

Official documentation: [Authenticating](https://kubernetes.io/docs/reference/access-authn-authz/authentication/), [User impersonation](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#user-impersonation).
