# Chapter 9: React Integration Without Making Core a React Package

Framework support is often where a supposedly framework-agnostic library quietly becomes framework-dependent. DevLens keeps React integration in `@devlens/react`, outside Core and separate from the Panel. This is more than package hygiene: it preserves the option to use the event engine in applications that do not use React.

## The boundary is a component, not a global plugin

The React integration exposes a client-side error-boundary component bound to an EventBus. When a descendant throws an error caught by the boundary, it reports a normalized event with the message and React's `componentStack`. The boundary can accept an optional fallback, defaulting to rendering nothing. It reports the error but does not prescribe the application's error UI.

That role differs from Runtime, Console, and Network capture. Those packages install global listeners or wrap global APIs and therefore need explicit install/uninstall cleanup. A React component is mounted and unmounted by React itself. Giving it artificial `install()` and `uninstall()` methods would create semantics that do not match its lifecycle.

ADR-0013 explicitly corrects the earlier broad implication in the plugin-contract ADR that React should follow the same factory shape as the global interceptors. The shared contract is that integrations report normalized events through Core, not that every integration must have identical lifecycle methods.

## Normalize runtime values defensively

TypeScript's React lifecycle signature declares the thrown value as an Error, but JavaScript permits throwing any value. The integration therefore normalizes the runtime value into a displayable message, handling Error objects, strings, JSON-serializable values, and fallback string conversion. It does not import Runtime's normalizer merely to reuse a small helper, because that would introduce a dependency between capture packages.

This is a deliberate trade-off: a few duplicated lines are preferable to a dependency edge that violates the package architecture. Shared code is not automatically better if it makes otherwise independent packages depend on each other.

## Client-only behavior is a real constraint

The component is marked client-only. Error boundaries operate through React's client-side lifecycle, so this integration should not be described as a general server-rendering error collector. Its scope is client-side component errors caught by the boundary, not every possible failure in a React application.

It also does not replace the Panel or capture runtime errors by itself. It is one source feeding the same EventBus. Applications can combine it with Runtime, Console, Network, and Panel according to their needs.

## What to test

Tests should cover caught errors, component stack inclusion, non-Error thrown values, fallback rendering, and whether the integration leaves host error handling consistent with the documented contract. The package's test files exist in the reviewed repository, but this documentation pass did not run them.

**Sources:** [React error boundary](https://github.com/Mystify7777/devlens/blob/f9db916d6eed0c8804a76fe6ce07d0a8efb6b4ef/packages/react/src/error-boundary.tsx), [React tests](https://github.com/Mystify7777/devlens/tree/f9db916d6eed0c8804a76fe6ce07d0a8efb6b4ef/packages/react/src), [React integration ADR](https://github.com/Mystify7777/devlens/blob/f9db916d6eed0c8804a76fe6ce07d0a8efb6b4ef/docs/adr/0013-react-integration-role.md), [plugin contract ADR](https://github.com/Mystify7777/devlens/blob/f9db916d6eed0c8804a76fe6ce07d0a8efb6b4ef/docs/adr/0006-plugin-contract.md).
