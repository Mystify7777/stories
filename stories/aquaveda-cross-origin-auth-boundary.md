# AquaVeda #145: When Browser Auth Tests Failed at the Boundary, Not the Auth Logic

**Story status:** Evidence-backed engineering story  
**Repository:** [Mystify7777/aquaveda-v2](https://github.com/Mystify7777/aquaveda-v2)  
**Pull request:** [#145](https://github.com/Mystify7777/aquaveda-v2/pull/145)  
**Issue:** #29  
**Review date:** 2026-11-12

## The problem

AquaVeda's frontend and backend are deployed on different origins. Authentication therefore cannot be considered correct merely because the server-side login and logout handlers work.

The browser must correctly coordinate:

- credentialed CORS;
- preflight responses;
- HttpOnly authentication cookies;
- cookie clearing on logout;
- protected-route behavior after the session disappears.

PR #145 was created to verify that boundary with real browser coverage.

## Where the browser tests went wrong

The PR's corrective commit history is unusually useful because it shows exactly where the first version of the browser assertions was too optimistic.

The implementation required fixes for:

- Playwright refresh-cookie URL assertions;
- the post-logout anonymous route guard assertion;
- navigation through the actual sign-in link role;
- the session-failure state reached during logout/bootstrap.

These failures did not point to a broken authentication protocol.

They pointed to a test that was asserting an internal or intermediate representation instead of the observable browser behavior that actually defines the contract.

## The first mistake: asserting the wrong cookie location

The browser test needed to verify that authentication cookies existed after login and were removed after logout.

The initial Playwright assertion around refresh-cookie behavior was corrected to use the actual cookie URL/path semantics.

That matters because browser cookie state is not simply:

```text
"cookie exists"
```

It is closer to:

```text
name + domain + path + security attributes + value
```

A cookie can therefore be present while a test still looks in the wrong place.

The fix was to assert against the browser's actual cookie representation rather than assuming that the refresh cookie would appear under the same URL/path interpretation as another cookie.

## The second mistake: asserting an internal logout outcome

Logout is not complete merely because the logout request returned successfully.

The meaningful browser contract is:

```text
authenticated
    |
logout
    |
cookies cleared
    |
session bootstrap
    |
anonymous
    |
protected route
    |
authentication gate
```

The test was strengthened to verify the actual anonymous route guard after logout rather than stopping at the HTTP operation.

A further correction made the test navigate using the actual sign-in link role. That makes the test depend on the user-visible navigation contract instead of a brittle implementation detail.

## The third mistake: forgetting the bootstrap state

After logout, the application still has to discover that the authenticated session is gone.

That can produce a transient session-failure/bootstrap state before the final anonymous UI appears.

The test therefore had to recognize the application's real state transition rather than assuming that logout immediately meant "anonymous page rendered."

This is a subtle but important distinction:

> clearing a cookie is an authentication operation; rendering the anonymous application is a client-state consequence of that operation.

The browser test needs to verify both when the product contract depends on both.

## The final verification boundary

The completed browser scenario verifies the full lifecycle:

```text
register/login
      |
      v
authentication cookies present
      |
      v
authenticated application
      |
      v
logout
      |
      v
cookies removed
      |
      v
session bootstrap detects absence
      |
      v
anonymous route guard
      |
      v
sign-in navigation available
```

The backend tests separately cover the CORS preflight contract:

- allowed origins receive the expected `Access-Control-Allow-Origin`;
- credentialed requests receive `Access-Control-Allow-Credentials: true`;
- disallowed origins do not receive the credentialed CORS grant;
- allowed methods include the authentication POST operation.

Logout route tests also verify the actual clear-cookie contract, including empty values and the expected paths for the access and refresh cookies.

## Why the resolution was better than weakening the tests

The tempting response to a browser test failure is to make the assertion less specific.

That would have defeated the purpose of this PR.

Instead, the tests became **more specific about the real contract**:

- inspect actual browser cookie state;
- use the application's real route guard;
- use semantic navigation;
- account for the session bootstrap transition;
- verify CORS at the HTTP boundary;
- verify cookie clearing at the browser boundary.

The test suite therefore became stronger rather than merely quieter.

## Verification

The final PR passed the full repository CI pipeline:

- frontend tests, ESLint, TypeScript and production build;
- backend verification with MongoDB;
- Playwright browser smoke tests with MongoDB.

The PR was squash-merged as:

```text
c15842d3353bd7a519b6562078bca34f52eb56d2
```

## The lesson

Authentication bugs often hide at the boundaries between systems.

The server can be correct while the browser contract is wrong.

The browser can have the correct cookies while the application still has stale session state.

The logout endpoint can succeed while the protected route still renders because the client has not completed its bootstrap transition.

And a test can fail while all of those things are working because the assertion is observing the wrong layer.

The durable testing strategy is therefore to define the contract as a sequence of observable states:

> **request → browser state → application state → user-visible route behavior**

That gives the test enough specificity to catch real regressions without tying it to incidental implementation details.

## Broader engineering takeaway

Cross-origin authentication is not one feature. It is a chain:

```text
CORS
  +
credentials
  +
cookie attributes
  +
session bootstrap
  +
route protection
  =
browser authentication contract
```

PR #145 turned that chain into executable evidence.

That is much more valuable than another unit test proving that a login function returns 200.
