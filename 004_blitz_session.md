# [RFC] Blitz Auth Session Management

Authentication has two key components:

1. Identity verification
   1. This is the process of verifying that someone is who they say they are, usually performed via username/password or OAuth.
2. Session management
   1. This is the process of securing multiple requests by the same user, usually performed via cookies or http headers.
   2. Once a session is created, subsequent requests verify the _session_, not the user.
   3. Without session management, the user would have to enter their username/password on each request.

This RFC only addresses the second part, session management. We plan to post another RFC for identity verification, but it will be fully swappable/pluggable with a default of self-hosted username/password.

## Preface

This RFC is almost entirely put together by [Rishabh Poddar](https://twitter.com/rishpoddar), the Co-founder and CTO of [Supertokens](https://supertokens.io/). He'll also lead the actual implementation. Thank you so much Rishabh!!!!

## Objectives

1. Enable login and logout functionality.
2. Ability to easily set session inactivity timeout length (potentially for an "infinite" amount of time).
3. By default, prevent against:
   - CSRF: Can be switch off on a per endpoint basis.
   - XSS
   - Brute force
   - Database session theft: Even if an attacker gets the session tokens from the db, they should not be able to hijack those user's accounts.
4. Optionally provide significantly more security by detecting token theft as per [this RFC](https://tools.ietf.org/html/rfc6749#section-10.4). This will be inspired by [SuperTokens.io](https://supertokens.io?s=bl)'s implementation of sessions. [Here](https://docs.google.com/spreadsheets/d/14h9qd2glE31HSGUofx43XwfJHZNzgkdCwEKl-3UcXLE/edit?usp=sharing) are all the ways tokens can be stolen.
5. Allow users to revoke a session
6. Allow multiple devices per logged in user
7. Anonymous sessions
8. Easily allow advanced session operations like:
   - Limit number of devices a user can login with
   - Sync session data across devices
   - Keep user session data intact across login / logout (to be thought about)

## Two Security Levels

Blitz session management will have two methods with different tradeoffs on complexity and security:

1. Default: Opaque token stored in the database
   1. This works the same as traditional fullstack frameworks like Rails
   2. Great security and usability for most apps
2. Advanced: Short lived JWTs plus refresh tokens
   1. For apps with special operational security needs
   2. For apps with extreme scalability needs where can't make a DB request to verify each session

Both methods send access tokens to the frontend via `httpOnly`, `secure` cookies. Separately, the anti-csrf token will be sent to the frontend via HTTP response headers.

## Default: Opaque token stored in the database

### Blitz Developer Interface

#### Login Mutation

```ts
// app/auth/mutations/login.ts

export default async function login(args: UserCredentials, ctx: Context) {
  // Perform identity verification here
  //   - username/password, auth0, OAuth, etc
  //   - This will covered in another RFC
  let user; // = ??? returned from identity verification

  try {
    await ctx.session.create({
      userId: user.id,
      role: "admin",
      // publicData will be accessible by frontend code and will
      // contain userId and role along with any other data provided here.
      // Put anything here you don't want to make a DB request for
      publicData: {
        teamIds: user.teams.map(team => team.id)
      },
      // privateData is stored in the DB with the session and is not directly
      // accessible by the frontend. A DB request is required to get this data
      // An example could be shopping cart items
      privateData: {
        /* ... */
      }
    });

    // successfully created a session.
  } catch (err) {
    // log and throw an error
  }
}
```

#### Logout Mutation

```ts
// app/auth/mutations/logout.ts

export default async function logout(args: SomArgs, ctx: Context) {
  // The following will take care of clearing all cookies.
  // for anonymous sessions, this is a noop.
  // if the session does not exist, the function below will not throw any error.
  await ctx.session.revoke();
}
```

#### Accessing Session Data in Queries/Mutations

```ts
// app/products/queries/getProducts.ts

export default async function getProducts(args: SomArgs, ctx: Context) {
  // Read the userId
  let userId = ctx.session.userId;

  // Example on how to read/set session publicData
  let publicSessionData = await ctx.session.getPublicData();
  await ctx.session.setPublicData({ ...publicSessionData /* some new data */ });

  // Example on how to read/set session privateData
  let privateSessionData = await ctx.session.getPrivateData();
  await ctx.session.setPrivateData({
    ...privateSessionData /* some new data */
  });

  if (ctx.session.role === "admin") {
    return await db.product.findMany();
  } else {
    return await db.product.findMany({ where: { published: true } });
  }
}
```

#### Using Session Handles

You can get all session handles belonging to a user. With these handles you can:

1. Get and set private data of that session handle
2. Revoke that session handle - logging the user out of that device
3. Revoke all sessions belonging to a user.

```ts
// app/auth/queries/exampleQuery.ts
import { Session } from "blitz";

export default async function exampleQuery(args: SomArgs, ctx: Context) {
  // this is a unique ID per session
  let sessionHandle = ctx.session.handle;

  // Can revoke all session for a user
  await Session.revokeAllSessionsForUser(ctx.session.userId);

  // Can get all sessions for a user and loop through them
  let allSessionsForThisUser: string[] = await Session.getAllSessionHandlesForUser(
    ctx.session.userId
  );
  for (let session of allSessionsForThisUser) {
    try {
      // Can use publicData to get the role or other public info of this session.
      let publicData = await Session.getPublicData(session);

      // get private session data for this session
      let privateData = await Session.getPrivateData(session);

      // Can change privateData for this session only
      await Session.setPrivateData(session, { ...privateData /* ... */ });

      // Log user out of this specific device
      await Session.revokeSessions([session]);
    } catch (err) {
      if (Session.Error.isUnauthorized(err)) {
        // This session has been revoked
      } else {
        // some generic error.
      }
    }
  }
}
```

#### Session Regeneration / Role Change

This is a security best practice when changing any `publicData`. This means that if a user's role has to change, then their session tokens will change too.

```ts
// app/users/mutations/changeRole.ts
import { Session } from "blitz";

export default async function changeRole(args: SomArgs, ctx: Context) {
  // Authorize request
  if (ctx.session.role !== "admin") {
    throw new AuthorizationError();
  }

  // Update role in DB
  const user = await db.user.update({
    where: { id: args.userId },
    data: { role: args.role },
    include: { teams: true }
  });

  let regeneratedSession = await ctx.regenerate({
    role: user.role,
    publicData: {
      ...ctx.session.getPublicData(),
      teamIds: user.teams.map(team => team.id)
    }
  });

  /*
    - New tokens are now set in the response header.
    - The old access token has not been revoked yet. This is so
      that in case of any failure of this API, the user doesn't get logged out.
    - The old access tokens will be revoked when the new one is used.
      Ideally this requires synchronisation from the frontend when calling this API,
      however, that can be ignored since regeneration is a very rare situation.
      In the worst case (very rare), the user may get logged out if multiple,
      parallel calls to this API are made.
    */
}
```

### Implementation Details

#### Session Creation

- After login, the server will create an opaque token. This will be a random, 32 character long `string`. It will also create an anti-csrf token which will also be 32 characters long. The final access token will be a concatenation of these two strings.
- This token will be sent to the frontend via `httpOnly`, `secure` cookies. Separately, the anti-csrf token will be sent to the frontend via response headers.
- The anti-csrf token must be stored in the localstorage on the frontend.
- The SHA256 hash of the access token will be stored in the database. This token will have the following properties mapped to it:
  - userId
  - anti-csrf token
  - expiry time
  - session data (this can be manipulated via the Blitz session API).
- Creating a new session while another one exists results in the headers / cookies changing. However, the older session will still be alive.
- For serious production apps, a cronjob will be needed to remove all expired tokens on a regular basis.

#### Session Verification

- For each request that requires CSRF protection, the frontend must read the localstorage and send the anti-csrf token in the request header.
- An incoming access token can be verified by checking that it's in the db and that it has not expired. After each verification, the expiry time of the access token updated asynchronously (and in a lock free way).
- CSRF attack protection can be done checking that the incoming anti-csrf token (from the header) is what is associated with the session.
- Once verified, the Blitz queries/mutations can easily get the userId associated with the session and also manipulate the session data.

#### Session Revocation/Logout

- This is easily done by deleting the session from the database.
- This is also how user logout will be implemented. Furthermore, cookies will be cleared, and a header will be sent signaling to the frontend to remove the anti-csrf token from the localstorage

#### Session Middleware

**Blitz developers don't have to implement this middleware.** This middleware will be provided by Blitz. It is shown here for those who are curious.

This is rough pseudo code, exact details will change.

```ts
// internal middleware file
import {Session, BlitzApiRequest, BlitzApiResponse} from 'blitz'

type SessionType = {
    userId: string | null, // will be null if anonymous
    role: string,  // will be "public" if session is anonymous.
    create: ({
        userId: string
        role: string
        privateData?: Object
        publicData?: Object
    }) => Promise<SessionType>,
    revoke: () => Promise<void>,    // if anonymous, this will fo nothing.
    getPrivateData: () => Promise<object>,
    setPrivateData: (data: object) => Promise<void>,
    getPublicData: () => object,
    handle: string,
    regenerate({
        role: string,
        publicData?: Object
    }) => Promise<SessionType>
}

// NOTE: Ignore this middleware API, the middleware API itself will likely change
export const middleware = [
  (req: BlitzApiRequest, res: BlitzApiResponse): SessionType => {
    try {
        let enableCsrfProtection = req.method !== "GET";

        // Adds session object to the ctx
        return await Session.getSession(req, res, enableCsrfProtection);
    } catch (err) {
        if (Session.Error.isUnauthorized(err)) {
            if (Session.Error.isAntiCSRFTokenFailed(err)) {
                throw err;
            }
        } else {
            throw err;
        }
    }

    // Create an anonymous session if enabled
    // the role here will be "public"
    // Adds session object to the ctx
    return Session.createAnonymousSession(req, res);
  }
]
```

## Advanced: Short lived JWTs plus refresh tokens

This is significantly more secure than the default method, but it is more inconvenient to the user than the above method because it requires refreshing of tokens.

### Blitz Developer Interface

TODO

### Implementation Details

#### Session Creation

- After login, the server will create a short lived JWT (access token) and a long lived opaque token (called refresh token). An opaque token is a random string that doesn't mean anything. It only acts as a pointer to the session data stored in the database.
- The JWT will be generated by a shared secret key. This secret key will be automatically changed from time to time for security purposes. It is very important to keep this key secure, because if it is compromised, then an attacker can easily assume the identity of any user in the system.
- A hashed version of the opaque token will be stored in the database similar to the default method described above.
- Both these tokens will be sent to the frontend client via `httpOnly`, `secure` cookies. The access token will be sent to all APIs, whereas the refresh token will only be sent to the refresh API.
- The server will also create an anti-csrf token that is sent to the frontend via headers and is also a part of the JWT claims. This token is stored in the localstorage on the frontend.

#### Session Verification

- The access token is sent for each API call.
- Verification is implemented by checking the signature of the JWT, as well as the expiration time.
- CSRF protection is done by checking the incoming anti-csrf token is the same as what's in the JWT. This can be disabled on a per API basis.

#### Session Refreshing

- When the access token expires, a call needs to be made to the refresh API with the refresh token.
- If the refresh token is valid (not expired, and in the database), then a new JWT and a new refresh token is sent to the frontend.
- From a security point of view, it is important that we also send a new refresh token. If we do not do that, then this flow is the same as the default method (from a security point of view).

#### Session Revocation/Logout

- This is easily done by deleting the refresh session from the database.
- This is also how user logout will be implemented. Furthermore, cookies will be cleared, and a header will be sent signaling to the frontend to remove the anti-csrf token from the localstorage.
- Because we are using JWTs, technically, the session is not completely revoked immediately. A malicious user can still access the APIs with that JWT. Once the JWT expires (which has a very short lifetime), then the session will be completely revoked. This seems to provide a good balance between security and scalability. The alternative is to use opaque tokens instead of JWTs as the access token.

#### Why is this more inconvenient to the user that the previous method?

For server side rendering, we require the access token in almost all backend calls that result from a browser navigation. If the access token is expired / missing, then these API calls cannot function without refreshing the session.

However, the refresh token is not available to these calls. The only way to refresh the session is to send some code to the frontend (JS & HTML that shows a spinner) that will call the refresh API, and on success redirect the page and call this API again (this time with a valid access token).

This extra implementation detail can be handled automatically by the framework in client side rendered apps, but not in server side rendered apps since it requires UI input.

#### Why is this more secure than the default method?

This question is answered in this 2 part [blog post](https://supertokens.io/blog/all-you-need-to-know-about-user-session-security). This method also detects session hijacking which can occur in all the following ways as mentioned [here](https://docs.google.com/spreadsheets/d/14h9qd2glE31HSGUofx43XwfJHZNzgkdCwEKl-3UcXLE/edit?usp=sharing).

## Anonymous Sessions

By default, anonymous users don't have sessions. The `ctx.session` object will be:

```js
// Default with anonymous sessions disabled:
{
  handle: null,
  userId: null,
  role: "public"
}
```

However, **you can turn on anonymous sessions**, in which case a session will be created (and stored in the DB for the default method). They will have a full session object on which you can set `publicData` and `privateData`.

One use case for this is saving shopping cart items for anonymous users. If an anonymous user later signs up or logs in, the anonymous session data can be merged into their new authenticated session.

## Configuration

```js
// blitz.config.js

module.exports {
  auth: {
    // Defaults:
    // anonymousRole: 'public'
    // anonymousSessions: false

    // Specific to default method
    TODO

    // Specific to advanced method
    TODO
  }
}
```

## Authorization

We want to be able to associate roles to sessions. There are many ways of doing this, but by default, we will allow just on role per session. This role information can be stored in the database along with the other session information.

We could have allowed a session to have multiple roles, but then this can easily cause conflicts and make the system vastly more complex. If the user needs to do this, they will have to use a third party service or build it themselves.

If a session's role changes, we should allow for the regeneration of the session token. This is a security best practice and is also recommended by OWASP.

**We plan to post a separate full RFC on Authorization**

## Frontend requirements

TODO
