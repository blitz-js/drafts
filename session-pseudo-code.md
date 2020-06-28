The pseudo code highlights the core logic of session management, the actual interface that is exposed to the user is built on top of this.

Items in `<>` are expected to come from some configuration.

## Notes:
These are points that are not obvious from the below implementation:
- The client can change the frontendData on the frontend and there will be no way to detecting that change. This has no security downsides since the backend is anyway not supposed to "blindly" trust data sent from the frontend.
- The expiry of the session is updated on use. This does not happen for each request since it would mean that we would have to update the set-cookie + the public data token on the frontend for each request which is inefficient (we will have to do a localstorage write for each request). Instead, it happens for all non-GET api requests that happen anytime after 1/4th the session expiry time has passed already.   

## Backend pseudo code:

### `createNewSession`
```ts
// will create a non-anonymous session.

function createNewSession(res: BlitzApiResponse, publicData: object, privateData: object = {}) {
    assert publicData.userId !== undefined;
    assert publicData.role !== undefined;
    let userId = publicData.userId;
    if (<essential method>) {
        let {
            sessionHandle, accessToken, antiCSRFToken, publicDataToken, expiresAt
        } = createNewSessionHelper(userId, publicData, privateDate);

        setCookie(res, "sSessionToken", accessToken, <API domain>, expiresAt, httpOnly: true, <secure: env !== dev mode>, <sameSite>);

        setHeader(res, "anti-csrf", antiCSRFToken);

        setHeader(res, "public-data-token", publicDataToken);

        return {
            sessionHandle, publicData
        }
    } else {
        // TODO: advanced method
    }
}

function createNewSessionHelper(userId: string, publicData: object, privateData: object) {
    let sessionHandle = UUID() + ":ots";  // ots is opaque token simple. ":" is a delimiter. This will be used for interoperability between the essential and advanced method.
    let publicDataToken = createPublicDataToken(userId, publicData, <session_expiry>);
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
        sessionHandle, accessToken, antiCSRFToken, publicDataToken, expiresAt
    }
}

```

### `createPublicDataToken`
```ts
function createPublicDataToken(publicData: object, expiry: number) {
    return base64(JSON.stringify(publicData) + ";" + expiry);
}
```

### `storeNewSessionInDb`
```ts
function storeNewSessionInDb(sessionHandle: string, userId: string, accessToken: string, antiCSRFToken: string, publicData: object, privateData: object, expiresAt: number, createdAtTime: number) {
    // table structure will be:
    //  session_handle: varchar(35) Primary key
    //  user_id TEXT 
    //  session_token_hash: TEXT 
    //  antiCSRFToken: TEXT
    //  public_data TEXT
    //  session_data TEXT
    //  expires_at BIGINT
    //  created_at_time BIGINT

    saveInDB(sessionHandle, userId, hash(accessToken), antiCSRFToken, JSON.stringify(publicData), JSON.stringify(privateData), expiresAt, createdAtTime);
}
```

### `getSession`
```ts
function getSession(req: BlitzApiRequest, res: BlitzApiResponse, enableCsrfProtection: boolean, httpMethod: string) {
    let sessionToken = req.cookie("sSessionToken"); // for essential method
    let idRefreshToken = req.cookie("sIdRefreshToken"); // for advanced method
    if (accessToken === undefined && idRefreshToken === undefined) {
        throw new UnauthorisedException("missing session tokens. Please login again");
    }
    if (sessionToken !== undefined) {
        let antiCSRFToken = req.headers("anti-csrf");
        let {sessionHandle, publicData, newAccessToken, newPublicDataToken,
            expiresAt} = getSessionHelper(sessionToken, antiCSRFToken, enableCsrfProtection), httpMethod;
        if (newAccessToken !== undefined) {
            setCookie(res, "sSessionToken", newAccessToken, <API domain>, expiresAt, httpOnly: true, <secure: env !== dev mode>, <sameSite>);
        }
        if (newPublicDataToken !== undefined) {
            setHeader(res, "public-data-token", newPublicDataToken);
        }
        return {
            sessionHandle, publicData
        }
    } else {
        // TODO: advanced method
    }
}

function getSessionHelper(sessionToken: string, inputAntiCSRFToken: string | undefined, enableCsrfProtection: boolean, httpMethod: string) {
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
    let newPublicDataToken = undefined

    // we generate new tokens if the public data has changed or if > 1/4th of the expiry time has passed (since we are doing a rolling expiry window).
    // the reason we have the httpMethod !== "get" check is cause we cannot reliably update the frontend expiry information in get requests since they could be from a browser level navigation (for example: user opening the site after a long time).
    if (hash(publicData) !== splittedToken[2] || ((expiresAt - Date.now()) < (1 - 1/4.0)*<session_expiry> &&
            httpMethod !== "get")) {
        let newExpiresAt = Date.now() + <session_expiry>;
        let publicDataToken = createPublicDataToken(publicData, newExpiresAt);
        newPublicDataToken = publicDataToken;
        newAccessToken = base64(sessionHandle +";"+ UUID() + ";" + hash(JSON.stringify(publicData)) + ";v0");
        updateSessionExpiryInDb(sessionHandle, newExpiresAt);   // should not wait for this to happen
    }
    return {sessionHandle, publicData: JSON.parse(publicData), newAccessToken, newPublicDataToken, newExpiresAt};
}
```

### `updateSessionExpiryInDb`
```ts
function updateSessionExpiryInDb(sessionHandle: string, expiresAt: number) {
    updateInDb(sessionHandle, expiresAt);   // update <table> set expires_at = expiresAt where session_handle = sessionHandle;
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
    let oldData = getPrivateData(sessionHandle);
    newData = {
        ...oldData,
        ...newData
    };
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
    assert newData.userId === undefined; // no changing of userId for a session.

    let oldData = getPublicData(sessionHandle);
    newData = {
        ...oldData,
        ...newData
    };
    let numberOfRowsUpdated = updateInDb(sessionHandle, JSON.stringify(newData));   // update <table> set publicData = newData where session_handle = sessionHandle;
    if (numberOfRowsUpdated === 0) {
        throw new UnauthorisedException("sessionHandle doesn't exist");
    }
}
```

### `refreshSession`
```ts
function refreshSession(req: BlitzApiRequest, res: BlitzApiResponse) {
    // TODO: advanced method
}
```

### Session class in context object
```ts
    class Session {
        private sessionHandle: string | undefined;
        private publicData: object | undefined;
        public userId: string | undefined;
        public role: string | undefined;

        constructor(sessionHandle, publicData) {
            if (sessionHandle !== undefined) {
                this.sessionHandle = sessionHandle;
            }
            if (publicData !== undefined) {
                this.publicData= publicData;
                this.userId = publicData.userId;
                this.role = publicData.role;
            }
        }

        function create(publicData, privateData) {
            // TODO:
        }

        function revoke() {
            if (this.sessionHandle !== undefined) {
                // TODO:
            }
        }

        function getPrivateData() {
            if (this.sessionHandle !== undefined) {
                // TODO:
            } else {
                throw Error("no session exists");
            }
        }

        function setPrivateData() {
            if (this.sessionHandle !== undefined) {
                // TODO:
            } else {
                throw Error("no session exists");
            }
        }

        function getPublicData() {
            if (this.sessionHandle !== undefined) {
                // TODO:
            } else {
                throw Error("no session exists");
            }
        }

        function setPublicData() {
            if (this.sessionHandle !== undefined) {
                // TODO:
            } else {
                throw Error("no session exists");
            }
        }
    }
```

## Frontend pseudo code:
We would add interceptors to `fetch`.
All requests (GET / POST etc..) will go through a helper function:
```ts
// to be called once on app load.
addSessionInterception = () => {
    originalFetch = global.fetch
    global.fetch = (url: RequestInfo, config?: RequestInit): Promise<Response> => {
        return await doRequest(
            (config?: RequestInit) => {
                return originalFetch(url, {
                    ...config
                });
            },
            config,
            url
        );
    };
}


// httpCall is the actual call to `fetch`. It takes the fetch config, and returns the response promise.
// config is the fetch config
// url is the url to which the request is going to.
doRequest = (httpCall: (config?: RequestInit) => Promise<Response>, config?: RequestInit, url?: any): Promise<Response> => {
    if (/*url is not app's API domain*/) {
        return await httpCall(config)
    }

    let antiCsrfToken = getAntiCsrfToken()
    if (antiCsrfToken !== null) {
        // <add anti csrf to config's header with key "anti-csrf">
    }
    let response = await httpCall(config);
    
    if (response === session expired || response === unauthorised) {
        deleteAntiCsrfToken()
        deletePublicToken()
        // cookies should be cleared automatically from the backend.
    } else {
        foreach (key, value in response.headers) {
            if (key === "anti-csrf") {
                setAntiCsrfToken(value)
            } else if (key === "public-data-token") {
                storePublicToken(value)
            }
        }
    }
    return response
}
```

## doesSessionExist
```ts
    doesSessionExist() {
        return getSessionInfo() !== null;
    }
```

## getSessionInfo
```ts
getSessionInfo() {
    let item = localstorage.getItem("public-token")
    if (item === null) {
        return null;
    }
    item = undo base64(item)
    let expiry = item.split(";")[1]
    if (Date.now() > expiry) {
        deleteAntiCsrfToken()
        deletePublicToken()
        return null;
    }
    return JSON.parse(item.split(";")[0]);
}
```

## getAntiCsrfToken
```ts
getAntiCsrfToken() {
    return localstorage.getItem("anti-csrf");
}
```

## setAntiCsrfToken
```ts
setAntiCsrfToken(token) {
    localstorage.setItem("anti-csrf", token);
}
```

## storePublicToken
```ts
storePublicToken(token) {
     localstorage.setItem("public-token", token);
}
```

## deleteAntiCsrfToken
```ts
deleteAntiCsrfToken() {
     localstorage.removeItem("anti-csrf");
}
```

## deletePublicToken
```ts
deletePublicToken() {
    localstorage.removeItem("public-token");
}
```