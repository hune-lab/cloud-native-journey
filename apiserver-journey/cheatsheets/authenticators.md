# kube-apiserver authenticators

How the kube-apiserver decides who sent a request: the authenticators it can load, the options that enable them, and how they are assembled into the single object the authentication handler calls. Read from the source of **Kubernetes v1.37.2** (commit [`c50e5395f1ae`](https://github.com/kubernetes/kubernetes/tree/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32)). Each row links to the code.

Part of [The Cloud Native Journey](https://hune-lab.com/the-cloud-native-journey), the apiserver journey.

Docs: [Authenticating](https://kubernetes.io/docs/reference/access-authn-authz/authentication/).

## The contract

Every authenticator implements one of two single-method interfaces ([`interfaces.go` L28-65](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/authentication/authenticator/interfaces.go#L28-L65)):

```go
type Token interface {
	AuthenticateToken(ctx context.Context, token string) (*Response, bool, error)
}

type Request interface {
	AuthenticateRequest(req *http.Request) (*Response, bool, error)
}

type Response struct {
	Audiences Audiences
	User      user.Info
}
```

| Return | Meaning |
|---|---|
| `resp, true, nil` | recognized: here is the identity |
| `nil, false, nil` | not mine: nothing I know how to check |
| `nil, false, err` | mine, but invalid |

## Request authenticators

Tried in this order by the request union ([`union.New`, config.go L238](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/pkg/kubeapiserver/authenticator/config.go#L238)). Each is added only when its option is set.

| Authenticator | Checks | Enabled by | Package | Added |
|---|---|---|---|---|
| front-proxy | `X-Remote-User`, `X-Remote-Group`, `X-Remote-Extra-*` headers, trusted only if the connection presents a client certificate signed by the request-header CA, with an allowed name | `--requestheader-client-ca-file` ([L146](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/server/options/authentication.go#L146)), `--requestheader-allowed-names` ([L151](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/server/options/authentication.go#L151)) | `authentication/request/headerrequest` | [L115-124](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/pkg/kubeapiserver/authenticator/config.go#L115-L124) |
| x509 | the client certificate, against the client CA; `CN` becomes the name, `O` the groups | `--client-ca-file` ([L205](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/server/options/authentication.go#L205)) | `authentication/request/x509` | [L128-130](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/pkg/kubeapiserver/authenticator/config.go#L128-L130) |
| bearertoken | extracts the token from `Authorization: Bearer` and hands it to the token authenticators | at least one token authenticator | `authentication/request/bearertoken` | [L211](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/pkg/kubeapiserver/authenticator/config.go#L211) |
| websocket | extracts the token from `Sec-WebSocket-Protocol` (prefix `base64url.bearer.authorization.k8s.io.`) and hands it to the same token authenticators | at least one token authenticator | `authentication/request/websocket` | [L211](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/pkg/kubeapiserver/authenticator/config.go#L211) |
| anonymous | nothing: produces `system:anonymous` in group `system:unauthenticated` when no one recognized the request | `--anonymous-auth` ([L357](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/pkg/kubeapiserver/options/authentication.go#L357)) | `authentication/request/anonymous` | [L243-245](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/pkg/kubeapiserver/authenticator/config.go#L243-L245) |

Package paths are relative to `staging/src/k8s.io/apiserver/pkg/`.

## Token authenticators

Tried in this order by the token union ([`tokenunion.New`, config.go L206](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/pkg/kubeapiserver/authenticator/config.go#L206)), reached only through bearertoken or websocket.

| Authenticator | Checks | Enabled by | Package | Added |
|---|---|---|---|---|
| token-file | a static token, looked up in a CSV file | `--token-auth-file` ([L470](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/pkg/kubeapiserver/options/authentication.go#L470)) | `apiserver/pkg/authentication/token/tokenfile` | [L134-139](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/pkg/kubeapiserver/authenticator/config.go#L134-L139) |
| ServiceAccount (legacy) | a ServiceAccount token stored in a Secret | `--service-account-key-file` ([L430](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/pkg/kubeapiserver/options/authentication.go#L430)), `--service-account-lookup` ([L437](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/pkg/kubeapiserver/options/authentication.go#L437)) | `pkg/serviceaccount` | [L141-146](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/pkg/kubeapiserver/authenticator/config.go#L141-L146) |
| ServiceAccount | a projected ServiceAccount token, bound to a pod, with audience and expiry | `--service-account-issuer` ([L440](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/pkg/kubeapiserver/options/authentication.go#L440)) | `pkg/serviceaccount` | [L148-153](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/pkg/kubeapiserver/authenticator/config.go#L148-L153) |
| bootstrap token | a bootstrap token, stored in a Secret of `kube-system` | `--enable-bootstrap-token-auth` ([L365](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/pkg/kubeapiserver/options/authentication.go#L365)) | `plugin/pkg/auth/authenticator/token/bootstrap` | [L156-157](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/pkg/kubeapiserver/authenticator/config.go#L156-L157) |
| JWT / OIDC | a token issued by an external identity provider; one authenticator per issuer | `--authentication-config` ([L345](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/pkg/kubeapiserver/options/authentication.go#L345)) or `--oidc-issuer-url` ([L375](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/pkg/kubeapiserver/options/authentication.go#L375)) | `apiserver/plugin/pkg/authenticator/token/oidc` | [L167-193](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/pkg/kubeapiserver/authenticator/config.go#L167-L193) |
| webhook | nothing itself: sends a `TokenReview` to an external service | `--authentication-token-webhook-config-file` ([L476](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/pkg/kubeapiserver/options/authentication.go#L476)) | `apiserver/plugin/pkg/authenticator/token/webhook` | [L195-201](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/pkg/kubeapiserver/authenticator/config.go#L195-L201) |

Package paths starting with `apiserver/` are under `staging/src/k8s.io/`, the others under the repository root.

OIDC is kept after the ServiceAccount authenticators on purpose: both verify JWTs, and OIDC may query its provider to refresh keys on a cache miss ([config.go L159-164](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/pkg/kubeapiserver/authenticator/config.go#L159-L164)).

## Wrappers

Pieces that hold exactly one authenticator and add a behavior around it.

| Wrapper | Wraps | Adds | Source |
|---|---|---|---|
| group adder | the request union | `system:authenticated` to any recognized identity, unless it is `system:anonymous` or already has `system:authenticated` or `system:unauthenticated` | [`authenticated_group_adder.go` L27-66](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/authentication/group/authenticated_group_adder.go#L27-L66), added at [config.go L240](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/pkg/kubeapiserver/authenticator/config.go#L240) |
| token cache | the token union | caches results by token string | [config.go L208-209](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/pkg/kubeapiserver/authenticator/config.go#L208-L209) |
| webhook cache | the webhook authenticator | caches `TokenReview` answers (`--authentication-token-webhook-cache-ttl`, [L483](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/pkg/kubeapiserver/options/authentication.go#L483)) | [config.go L411-416](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/pkg/kubeapiserver/authenticator/config.go#L411-L416) |
| CA verifier | the front-proxy header reader | checks the proxy's client certificate before trusting the headers | [`requestheader.go` L115-119](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/authentication/request/headerrequest/requestheader.go#L115-L119) |
| audience-agnostic | front-proxy, token-file, bootstrap, JWT | the API audiences to the response | [config.go L124, L139, L157, L292](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/pkg/kubeapiserver/authenticator/config.go#L124) |

## Assembly

Everything is built once, at startup, by `Config.New` ([config.go L107-249](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/pkg/kubeapiserver/authenticator/config.go#L107-L249)):

```go
var authenticators []authenticator.Request      // L108
var tokenAuthenticators []authenticator.Token   // L109
// L115-201: each authenticator appended to its list, if enabled

tokenAuth := tokenunion.New(tokenAuthenticators...)                   // L206
tokenAuth = tokencache.New(tokenAuth, ...)                            // L208-209, optional
authenticators = append(authenticators,
	bearertoken.New(tokenAuth), websocket.NewProtocolAuthenticator(tokenAuth)) // L211

if len(authenticators) == 0 {                                         // L231-236
	// anonymous alone, or nil
}
authenticator := union.New(authenticators...)                          // L238
authenticator = group.NewAuthenticatedGroupAdder(authenticator)        // L240
if config.Anonymous.Enabled {                                          // L242
	authenticator = union.NewFailOnError(authenticator, anonymous.NewAuthenticator(...)) // L245
}
```

The result is the single `authenticator.Request` handed to `WithAuthentication`.

## The object in memory

With every authenticator enabled:

```text
union (FailOnError)
 ├─ group adder
 │   └─ union
 │       ├─ front-proxy ── CA verifier ── header reader
 │       ├─ x509
 │       ├─ bearertoken ─┐
 │       └─ websocket ───┴─ token cache ── token union
 │                                          ├─ token-file
 │                                          ├─ ServiceAccount (legacy)
 │                                          ├─ ServiceAccount
 │                                          ├─ bootstrap
 │                                          ├─ JWT ── union (FailOnError), one OIDC per issuer
 │                                          └─ webhook cache ── webhook
 └─ anonymous
```

## How the unions decide

Both unions run the same loop ([request union L53-71](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/authentication/request/union/union.go#L53-L71), [token union L53-71](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/authentication/token/union/union.go#L53-L71)). With a single authenticator, `New` returns it as is, without a union.

| Answer from a member | `New` | `NewFailOnError` |
|---|---|---|
| recognized | stop, return the identity | stop, return the identity |
| not mine | try the next one | try the next one |
| invalid | record the error, try the next one | stop, return the error |
| end of list | not recognized, with the recorded errors | same |

`New` is used for the request union and the token union. `NewFailOnError` is used at the top, so that a bad token never becomes anonymous, and for the JWT issuers.

bearertoken turns "no token authenticator recognized the token" into an error, `invalid bearer token` ([bearertoken.go L71-73](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/authentication/request/bearertoken/bearertoken.go#L71-L73)): a request that presented a token nobody accepts is rejected, not treated as anonymous.

## Outcomes

| Request | Result |
|---|---|
| valid client certificate | x509 answers, tokens are never checked |
| valid bearer token | bearertoken answers with the identity found by a token authenticator |
| bearer token nobody accepts, or expired | error, `401 Unauthorized`; anonymous is not consulted |
| no credentials, anonymous enabled | `system:anonymous`, group `system:unauthenticated` |
| no credentials, anonymous disabled | `401 Unauthorized` |
