The pseudo code highlights the core logic of session management, the actual interface that is exposed to the user is built on top of this.

Items in `<>` are expected to come from some configuration.

## Pseudo code for essential method:

### `createNewSession`
```ts
// will create a non-anonymous session.

function createNewSession(res: BlitzApiResponse, publicData: object, privateData: object = {}) {
    assert publicData.userId !== undefined;
    assert publicData.role !== undefined;
    let userId = publicData.userId;
    if (<essential method>) {
        let {
            sessionHandle, accessToken, antiCSRFToken, idConnectToken, idConnectPublicKey, expiresAt
        } = createNewSessionHelper(userId, publicData, privateDate);

        setCookie(res, "sSessionToken", accessToken, <API domain>, expiresAt, httpOnly: true, <secure: env !== dev mode>, <sameSite>);

        setHeader(res, "anti-csrf", antiCSRFToken);

        setHeader(res, "id-connect-token", idConnectToken + ";" + idConnectPublicKey);

        return {
            sessionHandle, publicData, userId
        }
    } else {
        // TODO: advanced method
    }
}

function createNewSessionHelper(userId: string, publicData: object, privateData: object) {
    let sessionHandle = UUID();
    let {idConnectToken, publicKey} = createIdConnectToken(userId, publicData, <session_expiry>);
    let antiCSRFToken = UUID();

    /*
    - ";" is a delimiter. v0 is for versioning the structure to allow for future changes
    - We store the hashed public data in the opaque token so that when we verify, we can detect changes in it and return a new set of tokens if necessary. 
    */
    let accessToken = base64(sessionHandle +";"+ UUID() + ";" + hash(JSON.stringify(publicData)) + ";v0");

    let createdAtTime = Date.now();
    let expiresAt = createdAtTime + <session_expiry>;

    storeNewSessionInDb(sessionHandle, userId, accessToken, antiCSRFToken, publicData, privateData, expiresAt,
    createdAtTime);

    return {
        sessionHandle, accessToken, antiCSRFToken, idConnectToken, idConnectPublicKey: publicKey,
        expiresAt
    }
}

```

### `createIdConnectToken`
```ts
function createIdConnectToken(publicData: object, expiry: number) {
    let {publicKey, privateKey} = getCurrentIdConnectSigningKeys(); 

    let idConnectToken = jwt(privateKey, {
        ...publicData,
        expiry
    }); // payload is publicData + expiry

    return {idConnectToken, publicKey};
}
```

### `storeNewSessionInDb`
```ts
function storeNewSessionInDb(sessionHandle: string, userId: string, accessToken: string, antiCSRFToken: string, publicData: object, privateData: object, expiresAt: number, createdAtTime: number) {
    // table structure will be:
    //  sessionHandle: varchar(32) Primary key
    //  userId TEXT 
    //  sessionTokenHash: TEXT 
    //  antiCSRFToken: TEXT
    //  publicData TEXT
    //  sessionData TEXT
    //  expires_at BIGINT
    //  created_at_time BIGINT

    saveInDB(sessionHandle, userId, hash(accessToken), antiCSRFToken, JSON.stringify(publicData), JSON.stringify(privateData), expiresAt, createdAtTime);
}
```

### `getCurrentIdConnectSigningKeys`
```ts
function getCurrentIdConnectSigningKeys() {
    let keys = // this will fetch public / private key from memory. If not there, then will query DB. If not in DB, it will create one (in isolation).

    if (keys === undefined || keys.expired) {
        keys = // create new keys and save in db (in isolation).
    }
    return {
        keys.publicKey, keys.privateKey
    };
}
```

### `getSession`
```ts
function getSession(req: BlitzApiRequest, res: BlitzApiResponse, enableCsrfProtection: boolean) {
    let sessionToken = req.cookie("sSessionToken"); // for essential method
    let idRefreshToken = req.cookie("sIdRefreshToken"); // for advanced method
    if (accessToken === undefined && idRefreshToken === undefined) {
        throw new UnauthorisedException("missing session tokens. Please login again");
    }
    if (sessionToken !== undefined) {
        let antiCSRFToken = req.headers("anti-csrf");
        let {sessionHandle, userId, publicData, newAccessToken, newOpenIDConnectToken,
            newOpenIDConnectTokenPublicKey, expiresAt} = getSessionHelper(sessionToken, antiCSRFToken, enableCsrfProtection);
        if (newAccessToken !== undefined) {
            setCookie(res, "sSessionToken", newAccessToken, <API domain>, expiresAt, httpOnly: true, <secure: env !== dev mode>, <sameSite>);
        }
        if (newOpenIDConnectToken !== undefined) {
            setHeader(res, "id-connect-token", newOpenIDConnectToken + ";" + newOpenIDConnectTokenPublicKey);
        }
        return {
            sessionHandle, publicData, userId
        }
    } else {
        // TODO: advanced method
    }
}

function getSessionHelper(sessionToken: string, inputAntiCSRFToken: string | undefined, enableCsrfProtection: boolean) {
    let splittedToken = sessionToken.split(";");
    let sessionHandle = splittedToken[0]; // we need to check for the version too at the end and make sure its v0 (see creating a new session function)
    let sessionInfo = readFromDb(sessionHandle);
    if (sessionInfo === undefined) {
        throw new UnauthorisedException("Input session ID doesn't exist");
    }
    let {sessionHandle, antiCSRFToken, publicData, sessionTokenHash, expiresAt, userId} = sessionInfo;
    if (enableCsrfProtection && antiCSRFToken !== inputAntiCSRFToken) {
        throw new AntiCSRFTokenMismatchException();
    }
    if (hash(sessionToken) !== sessionTokenHash) {
        throw new UnauthorisedException("Input session ID doesn't exist");
    }
    if (Date.now() > expiresAt) {
        revokeSession(sessionHandle);
        throw new UnauthorisedException("Input session ID doesn't exist");
    }
    let newAccessToken = undefined;
    let newOpenIDConnectToken = undefined
    let newOpenIDConnectTokenPublicKey = undefined;
    if (hash(publicData) !== splittedToken[2]) {
        // public data has changed somehow.. so we must issue new tokens.
        let {idConnectToken, publicKey} = createIdConnectToken(userId, publicData, expiresAt);
        newOpenIDConnectToken = idConnectToken;
        newOpenIDConnectTokenPublicKey = publicKey;
        newAccessToken = base64(sessionHandle +";"+ UUID() + ";" + hash(JSON.stringify(publicData)) + ";v0");
    }
    return {sessionHandle, userId, publicData: JSON.parse(publicData), newAccessToken, newOpenIDConnectToken,
            newOpenIDConnectTokenPublicKey, expiresAt};
}
```

### `revokeAllSessionsForUser`
```ts
// returns list of revoked session for this user
function revokeAllSessionsForUser(userId: string): string[] {
    // multiple ways to implement this. Ideally we want to delete from the db directly based on userId, however, that may not give us the list of sessionHandles...?
    let sessionHandles: string[] = getAllSessionHandlesForUser(userId);
    return revokeMultipleSessions(sessionHandles);
}
```

### `getAllSessionHandlesForUser`
```ts
function getAllSessionHandlesForUser(userId: string): string[] {
    return readFromDb(userId);  // select sessionHandle from <table> where user_id = userId;
}
```

### `revokeSession`
```ts
function revokeSession(sessionHandle: string): boolean {
    return deleteFromDb(sessionHandle) === 1;  // delete from <table> where session_handle = sessionHandle;
}
```

### `revokeMultipleSessions`
```ts
function revokeMultipleSessions(sessionHandles: string[]): string[] {
    let revoked = [];
    sessionHandles.forEach(session => {
        if (revokeSession(session)) {
            revoked.push(session);
        }
    });
    return revoked;
}
```

### `getPrivateData`
```ts
function getPrivateData(sessionHandle: string) {
    let data = readFromDb(sessionHandle);   // select sessionData from <table> where session_handle = sessionHandle;
    if (data === undefined) {
        throw new UnauthorisedException("sessionHandle doesn't exist");
    }
    return JSON.parse(data);
}
```

### `setPrivateData`
```ts
function setPrivateData(sessionHandle: string, newData) {
    let numberOfRowsUpdated = updateInDb(sessionHandle, JSON.stringify(newData));   // update <table> set sessionData = newData where session_handle = sessionHandle;
    if (numberOfRowsUpdated === 0) {
        throw new UnauthorisedException("sessionHandle doesn't exist");
    }
}
```

### `getPublicData`
```ts
function getPublicData(sessionHandle: string) {
    let data = readFromDb(sessionHandle);   // select publicData from <table> where session_handle = sessionHandle;
    if (data === undefined) {
        throw new UnauthorisedException("sessionHandle doesn't exist");
    }
    return JSON.parse(data);
}
```

### `setPublicData`
```ts
function setPublicData(sessionHandle: string, newData) {
    let numberOfRowsUpdated = updateInDb(sessionHandle, JSON.stringify(newData));   // update <table> set publicData = newData where session_handle = sessionHandle;
    if (numberOfRowsUpdated === 0) {
        throw new UnauthorisedException("sessionHandle doesn't exist");
    }
}
```
