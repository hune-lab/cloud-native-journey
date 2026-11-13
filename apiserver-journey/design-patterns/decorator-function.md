# Decorator, as a function: how a plain function becomes an HTTP handler

Most handlers in the kube-apiserver chain are written the same way: a function that returns `http.HandlerFunc(func(w, r) {…})`. A few are structs instead; that form is in [Decorator, as a struct](decorator-struct.md). This note rebuilds that mechanism from scratch, then shows it is exactly what Go's `net/http` provides. Sources: **Go 1.26.10** (commit [`4e8c85f5319d`](https://github.com/golang/go/tree/4e8c85f5319d999bfd7617fdc191631e5634e3c5)) and **Kubernetes v1.37.2** (commit [`c50e5395f1ae`](https://github.com/kubernetes/kubernetes/tree/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32)).

Part of [The Cloud Native Journey](https://hune-lab.com/the-cloud-native-journey), the apiserver journey.

## Our own interface and function type

Let us write our own `Handler` interface, run by a method called `ExecuteHandler`, and a type `ToHandler` that turns a function into a handler:

```go
type Handler interface {
	ExecuteHandler(http.ResponseWriter, *http.Request)
}

type ToHandler func(http.ResponseWriter, *http.Request)

func (f ToHandler) ExecuteHandler(w http.ResponseWriter, r *http.Request) {
	f(w, r)
}
```

`ToHandler` is a function type that has the method `ExecuteHandler`, so it is a `Handler`. And that method does one thing only: it calls the function itself.

If this looks surprising and you come from another language: we declare a type whose values are functions of the form `func(http.ResponseWriter, *http.Request)`. A type in Go can carry methods whatever its underlying kind, a struct, an integer or a function. Here the method simply runs `f`, the function itself.

## A handler that prints hello

```go
var hello Handler = ToHandler(func(w http.ResponseWriter, r *http.Request) {
	fmt.Println("hello")
})

hello.ExecuteHandler(w, r) // prints hello
```

`hello` is a `Handler`: calling its `ExecuteHandler` method calls the function, which prints `hello`.

## A function becomes a handler

Write the code we want to run, and turn it into a handler:

```go
func withAuditID() Handler {
	return ToHandler(func(w http.ResponseWriter, r *http.Request) {
		id := make([]byte, 16)
		rand.Read(id)
		w.Header().Set("Audit-ID", fmt.Sprintf("%x", id))
	})
}
```

`ToHandler(func…)` runs nothing: it is a conversion. The function stays the same, it only takes the type `ToHandler`, and with it the method `ExecuteHandler`. The result is a handler, which anyone can run through `ExecuteHandler`.

## A handler that can be chained

To chain handlers, pass the next one as a parameter of `withAuditID`:

```go
func withAuditID(next Handler) Handler {
	return ToHandler(func(w http.ResponseWriter, r *http.Request) {
		id := make([]byte, 16)
		rand.Read(id)
		w.Header().Set("Audit-ID", fmt.Sprintf("%x", id))
		next.ExecuteHandler(w, r)
	})
}
```

The inner function does not receive `next` as a parameter: it keeps it from `withAuditID`, as a closure, the way a struct would keep it in a field.

If `router`, the handler that serves the route, is a `Handler`, then `withAuditID(router).ExecuteHandler(w, r)` amounts to:

```go
ToHandler(func(w http.ResponseWriter, r *http.Request) {
	// …
	next.ExecuteHandler(w, r)
}).ExecuteHandler(w, r)
```

By definition of `ToHandler`, this runs the function itself, which ends with `next.ExecuteHandler(w, r)` and hands over to the next handler.

## This is `net/http`

Replace `ToHandler` with `HandlerFunc` and `ExecuteHandler` with `ServeHTTP`, and you get Go's own abstraction ([`server.go` L88-90](https://github.com/golang/go/blob/4e8c85f5319d999bfd7617fdc191631e5634e3c5/src/net/http/server.go#L88-L90), [L2302-2307](https://github.com/golang/go/blob/4e8c85f5319d999bfd7617fdc191631e5634e3c5/src/net/http/server.go#L2302-L2307)):

```go
type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}

type HandlerFunc func(ResponseWriter, *Request)

// ServeHTTP calls f(w, r).
func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
	f(w, r)
}
```

## In the kube-apiserver

`WithAuditInit`, the first handler of the chain, is our `withAuditID`, with `http.HandlerFunc` in place of `ToHandler` and `ServeHTTP` in place of `ExecuteHandler` ([`audit_init.go` L41-63](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/endpoints/filters/audit_init.go#L41-L63)).

The same idiom appears well beyond HTTP. The authenticators use it too: `authenticator.TokenFunc` and `authenticator.RequestFunc` turn a plain function into an `authenticator.Token` or an `authenticator.Request` ([`interfaces.go` L38-52](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/authentication/authenticator/interfaces.go#L38-L52)).

## A method as the function: WithPriorityAndFairness

The function handed to `http.HandlerFunc` does not have to be a plain function. `WithPriorityAndFairness` builds a struct that holds the handler's state, then hands one of its methods to `http.HandlerFunc` ([`priority-and-fairness.go` L347-355](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/server/filters/priority-and-fairness.go#L347-L355)):

```go
priorityAndFairnessHandler := &priorityAndFairnessHandler{
	handler:       handler,       // the next handler
	fcIfc:         fcIfc,         // the APF controller
	workEstimator: workEstimator, // the request cost estimator
	// …
}
return http.HandlerFunc(priorityAndFairnessHandler.Handle)
```

`priorityAndFairnessHandler.Handle` is a *method value*: the method `Handle`, bound to that struct, used as a plain function of type `func(http.ResponseWriter, *http.Request)`. `http.HandlerFunc` turns it into a handler, exactly as it does for a function literal. When its turn comes, the chain calls `ServeHTTP`, which calls `Handle` on that struct, with all its fields at hand ([L88](https://github.com/kubernetes/kubernetes/blob/c50e5395f1ae2d4a2c05f28fdcc38c0916cc1a32/staging/src/k8s.io/apiserver/pkg/server/filters/priority-and-fairness.go#L88)).

It sits between the two forms: the state lives in a struct, as in the [struct form](decorator-struct.md), but the struct does not implement `ServeHTTP` itself; `http.HandlerFunc` does that part.
