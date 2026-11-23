# Chapter 9: Authentication Is a Lifecycle

Authentication in CyberShield is more than checking a password and issuing a JWT. The public API documents registration, login, OTP verification and resend, forgot-password and reset-password flows, plus protected account routes. The reviewed implementation also has a refresh endpoint and logout behavior. Together, these flows form an account lifecycle: prove control of an address, establish a session, renew it, and provide ways to recover or end access.

## Short-lived access, cookie-backed refresh

The recorded hardening work moves away from storing the access JWT as the long-lived session artifact in browser local storage. The intended pattern is an in-memory access token paired with an HTTP-only refresh-token cookie. The token utility defaults access tokens to 15 minutes and refresh tokens to seven days, and signs them with separate secrets, explicit token types, an issuer, and HS256. Refresh tokens include a version value associated with the user record.

The refresh-token version gives the server a way to invalidate older refresh tokens by incrementing the version in persistent user state. Refresh rotation updates that version and issues a replacement token. Logout clears the refresh cookie. This is a practical session-revocation mechanism for a system using stateless signed tokens, though it is not the same as maintaining a full per-device session inventory.

Cookie options matter because browsers automatically attach cookies in eligible requests. The active changelog records tightening the refresh cookie's same-site policy and matching cookie-clearing options during logout. A cookie security change is only complete when creation and deletion use compatible scope and attributes.

## Verification and recovery are attack surfaces

Registration and password recovery need to avoid becoming account-enumeration endpoints. The changelog records anti-enumeration responses, OTP verification hardening, refresh/session protections, and a stronger password minimum. OTP generation was changed from `Math.random` to Node's cryptographic `randomInt`, with an OTP hash secret required and plaintext fallback removed in the recorded hardening work.

Those controls have operational dependencies. Email delivery must be configured; secrets must be present and rotated appropriately; expiry and attempt limits must be enforced consistently; and responses must not reveal whether an arbitrary address has an account. A source-level helper or changelog line does not prove all of these conditions hold in the deployed environment.

There is also a useful evidence caveat in the reviewed controller: the module-level OTP hash secret expression includes fallback behavior. The changelog describes the hardened intended state, but a public technical narrative should not claim the fallback is absent from this exact snapshot without reconciling the code and configuration. The discrepancy is itself worth recording for review rather than smoothing over it.

## Authorization is separate from identity

A valid token establishes identity, not permission to perform every action. CyberShield models `USER`, `ADMIN`, and `SUPER_ADMIN` roles, has role middleware, and exposes administrative operations for user management, report moderation, and content review. Some actions explicitly protect the super-admin account or prevent an administrator from modifying an equal-or-higher privilege target. These checks are the difference between “authenticated” and “authorized.”

Suspension is another account state distinct from role. A user can have a role and still be suspended. Authentication middleware and account-sensitive controllers must consistently respect that state, including refresh flows and requests made with a token issued before suspension.

## Security claims need negative tests

A strong auth test suite should cover more than successful registration and login. It should test expired access tokens, malformed bearer headers, missing or invalid refresh cookies, replayed refresh tokens after rotation, logout followed by refresh, suspended accounts, role escalation attempts, password-reset expiry, and generic responses for unknown accounts. The repository includes route-level tests for auth and refresh/logout behavior, but this documentation pass did not execute them or verify current production settings.

**Engineering lesson:** session security is a protocol shared by the client, cookie configuration, token utilities, database state, middleware, and deployment secrets. A secure helper cannot compensate for an endpoint that bypasses the lifecycle.

**Sources:** [Auth controller](https://github.com/Mystify7777/CyberShield/blob/740a612024491ecfb0df19f869a02be68d2f55b7/server/src/controllers/authController.js), [token utilities](https://github.com/Mystify7777/CyberShield/blob/740a612024491ecfb0df19f869a02be68d2f55b7/server/src/utils/generateToken.js), [cookie utilities](https://github.com/Mystify7777/CyberShield/blob/740a612024491ecfb0df19f869a02be68d2f55b7/server/src/utils/cookies.js), [auth route tests](https://github.com/Mystify7777/CyberShield/blob/740a612024491ecfb0df19f869a02be68d2f55b7/server/tests/routes/auth.routes.test.js), [security guide](https://github.com/Mystify7777/CyberShield/blob/740a612024491ecfb0df19f869a02be68d2f55b7/docs/security/SECURITY.md).