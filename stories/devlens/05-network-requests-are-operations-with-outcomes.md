# Chapter 5: Network Requests Are Operations With Outcomes

Runtime errors and console calls are usually reported at a point in time. A network request is different: it starts, remains in flight, and later succeeds, fails, times out, or is aborted. That difference forced DevLens to make a deliberate modeling choice rather than treating network capture as another one-line event listener.

ADR-0010 records the decision: **one event per completed network operation**. DevLens does not emit a start event and a separate end event, and it does not mutate an event in place as the request progresses. The event is reported when the request settles.

## Why completion is the semantic unit

The research document evaluated alternatives against the existing architecture. A pair of correlated start/end events would add request-correlation concepts to Core, the Panel, and export/import. A mutable event would conflict with the event model's immutability and the Store's append-oriented behavior. One completion event fits the established model and keeps consumers simpler.

The choice has an accepted gap: an in-flight request produces no event until it settles. A request that hangs forever, or is still in flight when the page is torn down, may never produce an event. DevLens is describing completed operations, not exposing a live transport timeline. This is a product trade-off, not evidence that hanging requests are impossible.

## Fetch and asynchronous XHR

The Network package intercepts Fetch and asynchronous XMLHttpRequest. It classifies outcomes into explicit categories such as `success`, `http-error`, `network-error`, `aborted`, `timeout`, and `opaque`. Severity is derived from the outcome and, for HTTP errors, the response status range.

The split between classifiers and interceptors is valuable. Outcome classification can be tested as a pure function without invoking the browser API. The interceptor handles the lifecycle and delegates interpretation. This keeps request-specific mechanics from spreading through the EventBus and Panel.

The network capture design also treats non-interference as a hard constraint. The application should retain the same observable request behavior, including response access, errors, aborts, and timing semantics. The interceptor exists to describe the operation, not to become a new transport implementation.

## URL normalization is a privacy boundary

Captured request URLs can contain query parameters with access tokens, search terms, identifiers, or other sensitive values. The implementation canonicalizes URLs through the native URL parser: scheme and host casing normalize, default ports are removed, fragments are cleared, and relative URLs are resolved against a base URL when possible. Query parameter names are preserved while non-empty values are replaced with `***`. Empty flags remain empty.

If parsing fails, the normalizer returns the raw string unchanged instead of throwing. That is an explicit trade-off: a malformed or unusual URL should not cause DevLens to interfere with the request that already happened. It also means the redaction guarantee is strongest for values the parser can successfully interpret. The fallback path should be considered when evaluating privacy expectations for malformed inputs.

URL normalization is a leaf utility shared by Fetch and XHR. Both interceptors use it before events reach Core, so downstream packages receive already-normalized URLs and do not need interceptor-specific logic.

## What network metadata can tell you

The package records content-type and content-length metadata when exposed by the browser. It does not claim access to arbitrary response bodies, nor does it promise that every response is inspectable. Opaque responses and browser security boundaries remain real constraints. Diagnostics should report what the environment makes observable, not fabricate detail.

## The tests that matter

The repository contains dedicated tests for Fetch and XHR interceptors, outcome classifiers, HTTP status handling, URL normalization, content-length parsing, and the network package. This structure supports testing the hard boundary conditions separately. The suite was not run during this documentation pass, so its current pass state is not independently verified here.

**Sources:** [Network ADR](https://github.com/Mystify7777/devlens/blob/f9db916d6eed0c8804a76fe6ce07d0a8efb6b4ef/docs/adr/0010-network-plugin.md), [network capture research](https://github.com/Mystify7777/devlens/blob/f9db916d6eed0c8804a76fe6ce07d0a8efb6b4ef/docs/research/network-capture.md), [Fetch interceptor](https://github.com/Mystify7777/devlens/blob/f9db916d6eed0c8804a76fe6ce07d0a8efb6b4ef/packages/network/src/interceptors/fetch-interceptor.ts), [XHR interceptor](https://github.com/Mystify7777/devlens/blob/f9db916d6eed0c8804a76fe6ce07d0a8efb6b4ef/packages/network/src/interceptors/xhr-interceptor.ts), [URL normalizer](https://github.com/Mystify7777/devlens/blob/f9db916d6eed0c8804a76fe6ce07d0a8efb6b4ef/packages/network/src/normalize-url.ts).