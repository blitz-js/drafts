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

    let {
        sessionHandle, accessToken, antiCSRFToken, idConnectToken, idConnectPublicKey
    } = createNewSessionHelper(userId, publicData, privateDate);

    setCookie("sSessionToken", accessToken, <API domain>, <session_expiry>, <httpOnly: true>, <secure: env !== dev mode>, <sameSite: lax>);

    setHeader("anti-csrf", antiCSRFToken);

    setHeader("id-connect-token", idConnectToken + ";" + idConnectPublicKey);

    return {
        sessionHandle, publicData
    }
}

function createNewSessionHelper(userId: string, publicData: object, privateData: object) {
    let sessionHandle = UUID();
    let {idConnectToken, publicKey} = createIdConnectToken(userId, publicData, <session_expiry>);
    let antiCSRFToken = UUID();
    let accessToken = base64(sessionHandle +";"+ UUID() + ";v0");   // ";" is a delimiter. v0 is for versioning the structure to allow for future changes 

    storeNewSessionInDb(sessionHandle, accessToken, antiCSRFToken, publicData, privateData);

    return {
        sessionHandle, accessToken, antiCSRFToken, idConnectToken, idConnectPublicKey: publicKey
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
function storeNewSessionInDb(sessionHandle: string, accessToken: string, antiCSRFToken: string, publicData: object, privateData: object) {
    // table structure will be:
    //  sessionHandle: varchar(32) Primary key
    //  accessToken: TEXT 
    //  antiCSRFToken: TEXT
    //  publicData TEXT
    //  privateData TEXT

    saveInDB(sessionHandle, hash(accessToken), antiCSRFToken, JSON.stringify(publicData), JSON.stringify(privateData));
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