# Chapter 8: Authentication Is More Than a Token

Authentication is often presented as a short sequence: register, log in, receive a JWT, attach it to requests. HaloTask Pro's current code contains that flow, but its more interesting security work concerns what happens after a token has been issued.

The server signs tokens with a pinned algorithm and includes a token-version claim. The authentication middleware verifies the bearer token, validates the decoded payload, then loads the account's current token version. A token whose version no longer matches is rejected. The middleware also refuses access if the account no longer exists, and fails closed if the account lookup fails. This makes a session revocable at the account level rather than relying only on token expiration.

The password-reset path increments the token version alongside the password update. That can invalidate sessions issued before the reset. The client sees a 401, clears local authentication, and redirects to login. The offline queue must preserve pending work instead of repeatedly sending it without valid credentials or discarding it as though the task itself were malformed.

## Recovery has two separate questions

The code contains reset-token generation, hashing, expiration, email construction, and SMTP/Resend transport setup. The reset code is generated with a cryptographically secure random integer and stored as a hash. The README documents a configurable expiration window and a single-use flow. Those implementation details are distinct from whether the deployed email transport is configured correctly.

The README itself warns that forgot-password/password-reset is not working at present because of an SMTP and Resend conflict and an OTP-related requirement. That warning conflicts with other sections that list password recovery as delivered. A public narrative should preserve the conflict: the server code has a recovery implementation, but the repository owner has documented an operational failure. Without exercising the deployed flow and checking provider configuration, it would be irresponsible to call recovery production-ready.

The server also uses a neutral forgot-password response so callers are not told whether an email address belongs to an account. The source includes rate-limiting middleware and request validation. These are useful controls, but their presence in source does not automatically establish the exact rate limits, proxy configuration, or behavior of the deployed service.

## Security is a chain, not a badge

The task controller validates request bodies and scopes database operations by user ID. The API client binds credentials to the queue's original session for replay. The auth middleware checks the current session generation. Each control addresses a different failure mode: malformed input, cross-account resource access, credential switching during an asynchronous request, or reuse of a revoked token.

Security claims should name the property and its evidence. Source and tests can support the claim that reset increments a session generation and middleware rejects old versions. They cannot prove that every production secret is configured safely or that a third-party email provider is delivering messages.

One public-facing concern is intentionally not repeated in this book: the source repository's README has contained a credential-like demo login. A public project narrative should never copy those values. The repository owner should remove the credential from public documentation and rotate or revoke it if it is still valid. That remediation belongs in the source repository, outside this documentation-only change.

### Evidence trail

- [Authentication controller](https://github.com/Mystify7777/Halotask-Pro/blob/main/halotasks-server/src/controllers/auth.controller.ts)
- [Authentication middleware](https://github.com/Mystify7777/Halotask-Pro/blob/main/halotasks-server/src/middleware/auth.middleware.ts)
- [Session invalidation tests](https://github.com/Mystify7777/Halotask-Pro/blob/main/halotasks-server/tests/sessionInvalidation.test.ts)
- [Rate-limit middleware](https://github.com/Mystify7777/Halotask-Pro/blob/main/halotasks-server/src/middleware/rateLimit.ts)
- [README's reset-flow warning](https://github.com/Mystify7777/Halotask-Pro/blob/main/README.md)

### The standard for the next revision

Run the auth and session-invalidation tests, inspect the reset route's complete request/response path, and verify the deployed recovery flow without exposing actual codes or secrets. Until then, code-level controls and operational readiness remain separate conclusions.
