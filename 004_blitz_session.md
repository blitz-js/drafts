# RFC 004: Blitz Session Management

## Objectives
1) Enable login and logout functionality.
2) Ability to easily set session inactivity timeout length (potentially for an "infinite" amount of time).
3) Easy to use
4) By default, prevent against:
    - CSRF: Can be switch off on a per endpoint basis.
    - XSS
    - Brute force
    - Database session theft: Even if an attacker gets the session tokens from the db, they should not be able to hijack those user's accounts.
5) Optionally provide significantly more security by detecting token theft as per [this RFC](https://tools.ietf.org/html/rfc6749#section-10.4). This will be inspired by [SuperTokens.io](https://supertokens.io?s=bl)'s implementation of sessions. [Here](https://docs.google.com/spreadsheets/d/14h9qd2glE31HSGUofx43XwfJHZNzgkdCwEKl-3UcXLE/edit?usp=sharing) are all the ways tokens can be stolen.
6) Allow users to revoke a session
7) Allow multiple devices per logged in user
8) Anonymous sessions (to be thought about)
9) Easily allow advanced session operations like:
    - Limit number of devices a user can login with
    - Sync session data across devices
    - Keep user session data intact across login / logout (to be thought about)

## Default methodology:
#### Creation of a session
- After login, the backend will issue an opaque token. This will be a random, 32 character long `string`. The backend will also issue an anti-csrf token which will also be 32 characters long. The final access token will be a concatenation of these two strings. 
- This token will be sent to the frontend via `httpOnly`, `secure` cookies. If the user does not have `https` for their app, then they can switch off the secure flag via some config. Separately, the anti-csrf token will be sent to the frontend via response headers.
- The anti-csrf token must be stored in the localstorage on the frontend.
- The SHA256 hash of the access token will be stored in the database. This token will have the following properties mapped to it:
    - userId
    - anti-csrf token
    - expiry time
    - session data (this can be manipulated by the user).
- A cronjob will remove all expired tokens on a regular basis.

#### Session verification
- For each request that requires CSRF protection, the frontend must read the localstorage and send the anti-csrf token in the request header.
- An incoming access token can be verified by checking that it's in the db and that it has not expired. After each verification, the expiry time of the access token updated asynchronously (and in a lock free way).
- CSRF attack protection can be done checking that the incoming anti-csrf token (from the header) is what is associated with the session.
- Once verified, the API can easily get the userId associated with the session and also manipulate the session data.

#### Session revocation
- This is easily done by revoking a session from the database.
- This is also how user logout will be implemented. Furthermore, cookies will be cleared, and a header will be sent signalling to the frontend to remove the anti-csrf token from the localstorage

## More secure methodology:
This is significantly more secure than the above, but it is slightly more inconvenient than the above method because it requires refreshing of tokens.

TODO