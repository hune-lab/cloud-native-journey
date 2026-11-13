# Decorator, as a struct: a middleware written as a struct

In most languages a middleware is an object that holds the next handler. In Go, that object is a struct. This note builds that form step by step, from a single handler to a chain, then shows where the kube-apiserver uses it. The other form, a plain function turned into a handler, is in [Decorator, as a function](decorator-function.md). Sources: **Go 1.26.10** (commit [`4e8c85f5319d`](https://github.com/golang/go/tree/4e8c85f5319d999bfd7617fdc191631e5634e3c5)) and **Kubernetes v1.37.2** (commit [`c50e5395f1ae`](https://github.com/kubernetes/kubernetes/tree/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32)).

Part of [The Cloud Native Journey](https://hune-lab.com/the-cloud-native-journey), the apiserver journey.

## A struct is a handler as soon as it has `ServeHTTP`

A struct is an `http.Handler` as soon as it has the method `ServeHTTP(ResponseWriter, *Request)`. Go asks for nothing more: there is nothing to declare, the compiler sees that the method is there.

Take the simplest HTTP server, with one handler that answers 404:

```go
package server

import (
	"fmt"
	"net/http"
	"net/http/httptest"
	"slices"
)

type Route string

const (
	RouteDeployments Route = "/apis/apps/v1/namespaces/demo/deployments"
	RouteHealthz     Route = "/healthz"
)

type notFoundHandler struct {
	routes []Route
}

func (h notFoundHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	if !slices.Contains(h.routes, Route(r.URL.Path)) {
		w.Header().Set("Content-Type", "text/plain; charset=utf-8")
		w.WriteHeader(http.StatusNotFound)
		fmt.Fprintln(w, "404 page not found")
		return
	}
	fmt.Fprintln(w, "ok")
}

func serve(r *http.Request) *http.Response {
	handler := notFoundHandler{routes: []Route{RouteDeployments, RouteHealthz}}
	responseWriter := httptest.NewRecorder()
	handler.ServeHTTP(responseWriter, r)
	return responseWriter.Result()
}
```

`serve` does not build the response itself: it hands it to the `ResponseWriter`, provided by Go's standard library. If the handler set a code with `WriteHeader`, that code is sent; otherwise the response goes out as `200 OK`, as Go's real server does ([`server.go` L1674-1678](https://github.com/golang/go/blob/4e8c85f5319d999bfd7617fdc191631e5634e3c5/src/net/http/server.go#L1674-L1678)).

Every handler follows the same logic: its job is to prepare the response through the `ResponseWriter`, by writing it itself or by handing it to another handler. For that, it needs the request it receives.

## Two handlers called one after the other

Now add a second handler. It gives each request an audit ID, sent back in the `Audit-ID` header, as the apiserver does. The obvious way is to call both handlers in turn, with the same `ResponseWriter`:

```go
type auditIDHandler struct{}

func (h auditIDHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	id := make([]byte, 16)
	rand.Read(id)
	w.Header().Set("Audit-ID", fmt.Sprintf("%x", id))
}

func serve(r *http.Request) *http.Response {
	auditID := auditIDHandler{}
	notFound := notFoundHandler{routes: []Route{RouteDeployments, RouteHealthz}}
	responseWriter := httptest.NewRecorder()
	auditID.ServeHTTP(responseWriter, r)
	notFound.ServeHTTP(responseWriter, r)
	return responseWriter.Result()
}
```

Swap the two calls and the `Audit-ID` header disappears from both responses: headers leave with the status code, and anything set afterwards is lost. So `serve` must know the right order of the handlers, and nothing guarantees it.

## The handler keeps the next one

Instead of letting `serve` call the handlers in turn, the audit handler can keep the next one in a field, and call it itself:

```go
type auditIDHandler struct {
	next http.Handler
}

func (h auditIDHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	id := make([]byte, 16)
	rand.Read(id)
	w.Header().Set("Audit-ID", fmt.Sprintf("%x", id))
	h.next.ServeHTTP(w, r)
}

func serve(r *http.Request) *http.Response {
	handler := auditIDHandler{next: notFoundHandler{routes: []Route{RouteDeployments, RouteHealthz}}}
	responseWriter := httptest.NewRecorder()
	handler.ServeHTTP(responseWriter, r)
	return responseWriter.Result()
}
```

Look at `h.next.ServeHTTP(w, r)`. One handler calls another, and both share the same interface. The order is now guaranteed by construction: the audit handler sets its header, then calls `next`. And `serve` is back to its first shape: one handler, one call, without knowing how many others are chained behind it.

This is a *middleware*, an implementation of the *decorator* pattern: every handler implements the same interface, `http.Handler`, and may or may not hold a `next` field, the handler it wraps. The audit handler has one; `notFoundHandler`, at the end of the chain, does not. In Go nothing enforces it, since an interface only describes methods; other languages express it in the type itself, with an abstract class that carries the `next` property.

```text
request: serve calls handler.ServeHTTP(w, r)
                       │
                       ▼
┌─ auditIDHandler ──────────── http.Handler ─┐
│  ServeHTTP(w, r)                           │
│    w.Header().Set("Audit-ID", …)           │
│    h.next.ServeHTTP(w, r)                  │
└──────────────────────┬─────────────────────┘
                       │ next
                       ▼
┌─ another middleware ──────── http.Handler ─┐
│  ServeHTTP(w, r)                           │
│    …                                       │
│    h.next.ServeHTTP(w, r)                  │
└──────────────────────┬─────────────────────┘
                       │ next
                       ▼
┌─ notFoundHandler ─────────── http.Handler ─┐
│  ServeHTTP(w, r)            no next        │
│    404 or "ok", written to w               │
└──────────────────────┬─────────────────────┘
                       │
                       ▼
response: what the handlers wrote to w,
returned by serve once the chain is done
```

## Ten handlers, nested

It is a little verbose, and above all it nests structs. With ten handlers:

```go
handler := auditIDHandler{
	next: panicRecoveryHandler{
		next: requestInfoHandler{
			next: waitGroupHandler{
				next: deadlineHandler{
					next: timeoutHandler{
						next: authenticationHandler{
							next: auditHandler{
								next: authorizationHandler{
									next: notFoundHandler{routes: []Route{RouteDeployments, RouteHealthz}},
								},
							},
						},
					},
				},
			},
		},
	},
}
```

One way out is an array of middlewares, as in [Symfony Messenger](https://github.com/symfony/symfony/blob/0989c3650c8f52dff75ba5000bf51d593de24120/src/Symfony/Component/Messenger/Middleware/StackMiddleware.php#L46-L55) or [Spring Security](https://github.com/spring-projects/spring-security/blob/e17858065427ef265d649e2e281a9cba7e5ff319/web/src/main/java/org/springframework/security/web/FilterChainProxy.java#L376-L387). The middlewares are stored in order, and the framework keeps a cursor on that array, which it passes to each middleware. To move on, a middleware calls `$stack->next()->handle($envelope, $stack)` in Symfony, `chain.doFilter(request, response)` in Spring. The price: every handler receives the chain as a parameter, must know about the stack, and its signature is no longer that of a plain handler.

## What the kube-apiserver does

The Kubernetes developers did not choose the array. For most of their filters they chose the function form of [decorator-function.md](decorator-function.md): each `With…` takes the next handler and returns an `http.HandlerFunc`, and `DefaultBuildHandlerChain` wraps them one after the other, without nesting literals.

A few filters remain structs, when they need more than the next handler:

- `WithTimeout` returns a `timeoutHandler`, which keeps the next handler in a field, exactly like our `auditIDHandler` ([`server/filters/timeout.go` L75-82](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/server/filters/timeout.go#L75-L82)):

  ```go
  func WithTimeout(h http.Handler, timeoutFunc timeoutFunc) http.Handler {
  	return &timeoutHandler{h, timeoutFunc}
  }

  type timeoutHandler struct {
  	handler http.Handler
  	timeout timeoutFunc
  }
  ```

- `WithConstrainedImpersonation` returns a `constrainedImpersonationHandler`, whose `handler` field is the next one, and which also keeps state from one request to the next ([`endpoints/filters/impersonation/constrained_impersonation.go` L46-79](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/endpoints/filters/impersonation/constrained_impersonation.go#L46-L79)):

  ```go
  func WithConstrainedImpersonation(handler http.Handler, a authorizer.UnconditionalAuthorizer, s runtime.NegotiatedSerializer) http.Handler {
  	// …
  	return &constrainedImpersonationHandler{
  		handler: handler,
  		tracker: newImpersonationModesTracker(ma),
  		s:       s,
  		// …
  	}
  }

  type constrainedImpersonationHandler struct {
  	handler http.Handler
  	tracker *impersonationModesTracker
  	s       runtime.NegotiatedSerializer
  	// …
  }

  func (c *constrainedImpersonationHandler) ServeHTTP(w http.ResponseWriter, req *http.Request) {
  ```

Both forms are the same thing seen from two sides: a struct with a `ServeHTTP` method, or a function whose type has that method. Either way, the value is an `http.Handler`.

Official documentation: [`net/http` Handler](https://pkg.go.dev/net/http#Handler).
