# Chapter 4: Plugins Must Observe Without Owning the Application

Runtime and Console capture have a deceptively sharp constraint: they observe behavior owned by the host application. They must collect useful diagnostics without changing what the application does.

The common plugin contract is intentionally small:

```ts
interface Plugin {
  install(): void;
  uninstall(): void;
}
```

Both operations must be idempotent. Calling `install()` twice must not stack duplicate listeners or wrappers, and calling `uninstall()` twice must not break cleanup. This matters because development tools are often mounted and unmounted as the app reloads, routes change, or tests reset their environment.

## Runtime capture

The Runtime package listens for browser runtime failures, including `window.error` and unhandled promise rejections. It normalizes browser-specific data into a DevLens event and reports it to the EventBus. Its cleanup path removes the listeners that it installed.

The runtime normalizer must deal with imperfect values. JavaScript permits throwing values that are not Error instances, so a robust diagnostic message cannot assume every rejection reason has a conventional `message` or `stack`. The integration attempts to produce useful text while preserving the fact that the raw reason may be unusual.

## Console capture and native behavior

Console capture wraps selected console methods so DevLens can observe calls. This is a global monkey-patch, which makes restoration and non-interference core requirements rather than optional polish. The native console method must still be called with the original receiver and arguments, and uninstall must restore the original methods rather than leave DevLens installed in the host application.

The design goal is not to replace the browser console. It is to mirror activity into the event stream while allowing normal output to continue. Any capture failure that prevents application code from logging as it did before would violate the product's central promise.

## One lifecycle contract, not one forced implementation

The plugin ADR records a useful refinement: not every integration should be forced into the same literal lifecycle shape. Runtime and Console need install/uninstall because they attach global listeners or patch global methods. Network capture has similar global interception requirements.

React is different. A React error boundary is a component whose lifecycle is managed by React; it does not patch a global and does not need artificial install/uninstall methods. ADR-0013 corrects the earlier generalization that all capture integrations should share the same factory shape. The shared principle is a common event contract, not identical mechanics where they do not fit.

## Failure isolation and cleanup

A capture library should not become the reason an application fails. Wrappers should preserve native behavior, normalization should handle unexpected values defensively, and teardown should be tested. This is especially important for network interception, where throwing inside a wrapper could change a request's behavior or turn diagnostics into a production defect.

Tests should check repeated install/uninstall, listener removal, restoration of original functions, preserving method arguments and return values, and reporting capture events without recursively capturing DevLens's own internal diagnostics.

**Sources:** [Runtime plugin](https://github.com/Mystify7777/devlens/blob/f9db916d6eed0c8804a76fe6ce07d0a8efb6b4ef/packages/runtime/src/runtime.ts), [runtime listeners](https://github.com/Mystify7777/devlens/tree/f9db916d6eed0c8804a76fe6ce07d0a8efb6b4ef/packages/runtime/src/listeners), [console interceptor](https://github.com/Mystify7777/devlens/blob/f9db916d6eed0c8804a76fe6ce07d0a8efb6b4ef/packages/console/src/interceptors/console-interceptor.ts), [plugin contract ADR](https://github.com/Mystify7777/devlens/blob/f9db916d6eed0c8804a76fe6ce07d0a8efb6b4ef/docs/adr/0006-plugin-contract.md), [React role ADR](https://github.com/Mystify7777/devlens/blob/f9db916d6eed0c8804a76fe6ce07d0a8efb6b4ef/docs/adr/0013-react-integration-role.md).