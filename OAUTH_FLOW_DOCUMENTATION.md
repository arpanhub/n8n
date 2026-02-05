# n8n OAuth Flow Documentation

This document provides a comprehensive mapping of the code files and flow responsible for handling OAuth authentication when users connect their applications (Gmail, Notion, etc.) to n8n.

## Table of Contents
1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Key Components](#key-components)
4. [OAuth Flow Sequence](#oauth-flow-sequence)
5. [File Structure](#file-structure)
6. [Detailed Flow](#detailed-flow)

## Overview

n8n supports OAuth 1.0a and OAuth 2.0 authentication flows for connecting to third-party services. The OAuth implementation is split across frontend (Vue.js) and backend (Node.js/Express) components, with a dedicated OAuth client library.

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         Frontend (Vue.js)                        │
│                                                                   │
│  CredentialEdit.vue → CredentialConfig.vue → OAuthButton.vue    │
│           ↓                                                       │
│  credentials.store.ts → credentials.api.ts                       │
└────────────────────────────┬──────────────────────────────────────┘
                             │ HTTP Request
                             ↓
┌─────────────────────────────────────────────────────────────────┐
│                      Backend (Express/Node.js)                   │
│                                                                   │
│  OAuth Controllers → OAuth Service → Credentials Helper          │
│       ↓                   ↓                 ↓                     │
│  ClientOAuth2      Credentials Repo   Credentials Entity         │
└─────────────────────────────────────────────────────────────────┘
```

## Key Components

### Frontend Components

#### 1. **UI Components** (`packages/frontend/editor-ui/src/features/credentials/`)
- **`components/CredentialEdit/CredentialEdit.vue`**
  - Main modal for creating/editing credentials
  - Handles OAuth popup window management
  - Implements BroadcastChannel API for callback communication
  - Location: Lines 1084-1134

- **`components/CredentialEdit/CredentialConfig.vue`**
  - Displays credential configuration form
  - Shows OAuth connection button
  - Location: Lines 1-500+

- **`components/CredentialEdit/OauthButton.vue`**
  - Renders OAuth connection button
  - Special handling for Google OAuth
  - Location: Lines 1-28

#### 2. **Store and API** (`packages/frontend/editor-ui/src/features/credentials/`)
- **`credentials.store.ts`**
  - Pinia store for credential state management
  - `oAuth1Authorize()` - Line 377
  - `oAuth2Authorize()` - Line 373
  - Manages credential CRUD operations

- **`credentials.api.ts`**
  - API client functions
  - `oAuth1CredentialAuthorize()` - Lines 85-95
  - `oAuth2CredentialAuthorize()` - Lines 98-108
  - Makes REST API calls to backend

### Backend Components

#### 1. **OAuth Controllers** (`packages/cli/src/controllers/oauth/`)

- **`oauth2-credential.controller.ts`**
  - **Route**: `/oauth2-credential`
  - **Endpoints**:
    - `GET /auth` - Generates OAuth2 authorization URL (Line 26)
    - `GET /callback` - Handles OAuth2 callback with authorization code (Line 38)
  - Uses `@RestController` decorator
  - Implements PKCE support for enhanced security

- **`oauth1-credential.controller.ts`**
  - **Route**: `/oauth1-credential`
  - **Endpoints**:
    - `GET /auth` - Generates OAuth1 authorization URL (Line 24)
    - `GET /callback` - Handles OAuth1 callback with oauth tokens (Line 42)
  - Handles OAuth 1.0a three-legged flow

#### 2. **OAuth Service** (`packages/cli/src/oauth/oauth.service.ts`)

Core service handling OAuth logic:

**Key Methods**:
- `getCredential()` - Line 93: Retrieves and validates credential access
- `generateAOauth2AuthUri()` - Line 325: Creates OAuth2 authorization URL
  - Supports dynamic client registration
  - Handles PKCE flow
  - Creates CSRF state token
- `generateAOauth1AuthUri()` - Line 448: Creates OAuth1 authorization URL
  - Handles request token acquisition
  - Creates CSRF state token
- `resolveCredential()` - Line 266: Validates callback and retrieves credential data
- `createCsrfState()` - Line 196: Generates CSRF protection state
- `decodeCsrfState()` - Line 209: Validates and decodes CSRF state
- `verifyCsrfState()` - Line 253: Verifies CSRF token
- `encryptAndSaveData()` - Line 176: Encrypts and saves credential data
- `renderCallbackError()` - Line 297: Renders error page on failure

**Security Features**:
- CSRF protection with encrypted state
- Configurable skip auth for iframe embedding (`N8N_SKIP_AUTH_ON_OAUTH_CALLBACK`)
- URL validation for OAuth endpoints
- Support for dynamic credentials (external credential resolution)

#### 3. **OAuth Client Library** (`packages/@n8n/client-oauth2/`)

- **`src/client-oauth2.ts`**
  - Main OAuth2 client implementation
  - Handles authorization code flow
  - Token management and refresh

- **`src/client-oauth2-token.ts`**
  - Token storage and manipulation
  - Token refresh logic

- **`src/code-flow.ts`**
  - Authorization code flow implementation
  - `getUri()` - Generate authorization URL
  - `getToken()` - Exchange code for token

#### 4. **Credentials Management** (`packages/cli/src/credentials/`)

- **`credentials.controller.ts`**
  - Main CRUD operations for credentials
  - Route: `/credentials`
  - Methods: GET, POST, PATCH, DELETE

- **`credentials.service.ts`**
  - Business logic for credential operations
  - Credential encryption/decryption
  - Permission handling

- **`credentials-helper.ts`**
  - Helper methods for credential operations
  - Credential type management
  - Overwrites and defaults application

#### 5. **Database Entities** (`packages/@n8n/db/`)

- **`src/entities/credentials-entity.ts`**
  - Credential data model
  - Encrypted storage
  - Relationships with users/projects

- **OAuth-specific migrations**:
  - `1760116750277-CreateOAuthEntities.ts`
  - `1763572724000-ChangeOAuthStateColumnToUnboundedVarchar.ts`

#### 6. **Templates** (`packages/cli/templates/`)

- **`oauth-callback.handlebars`**
  - Success page shown after successful OAuth
  - Uses BroadcastChannel to notify parent window
  - Auto-closes after 5 seconds

- **`oauth-error-callback.handlebars`**
  - Error page shown when OAuth fails
  - Displays error message and reason

## OAuth Flow Sequence

### OAuth 2.0 Authorization Code Flow

```
┌─────────┐           ┌──────────┐           ┌─────────┐           ┌──────────┐
│ User UI │           │ Frontend │           │ Backend │           │ Provider │
└────┬────┘           └────┬─────┘           └────┬────┘           └────┬─────┘
     │                     │                      │                      │
     │ 1. Click Connect    │                      │                      │
     ├────────────────────>│                      │                      │
     │                     │                      │                      │
     │                     │ 2. Request Auth URL  │                      │
     │                     │ POST /credentials    │                      │
     │                     ├─────────────────────>│                      │
     │                     │                      │                      │
     │                     │                      │ 3. Generate CSRF     │
     │                     │                      │    Create State      │
     │                     │                      │    Save to DB        │
     │                     │                      │                      │
     │                     │ 4. Auth URL + State  │                      │
     │                     │<─────────────────────┤                      │
     │                     │                      │                      │
     │                     │ 5. GET /oauth2-      │                      │
     │                     │    credential/auth   │                      │
     │                     ├─────────────────────>│                      │
     │                     │                      │                      │
     │                     │ 6. Authorization URI │                      │
     │                     │    with PKCE/state   │                      │
     │                     │<─────────────────────┤                      │
     │                     │                      │                      │
     │ 7. Open Popup       │                      │                      │
     │    window.open()    │                      │                      │
     │<────────────────────┤                      │                      │
     │                     │                      │                      │
     │ 8. Redirect to Provider                    │                      │
     ├────────────────────────────────────────────────────────────────>│
     │                     │                      │                      │
     │ 9. User Authorizes  │                      │                      │
     │<────────────────────────────────────────────────────────────────┤
     │                     │                      │                      │
     │ 10. Callback with   │                      │                      │
     │     code & state    │                      │                      │
     ├─────────────────────────────────────────>│                      │
     │                     │                      │                      │
     │                     │                      │ 11. Verify State     │
     │                     │                      │     Decrypt Data     │
     │                     │                      │     Verify CSRF      │
     │                     │                      │                      │
     │                     │                      │ 12. Exchange Code    │
     │                     │                      │     for Token        │
     │                     │                      ├─────────────────────>│
     │                     │                      │                      │
     │                     │                      │ 13. Access Token +   │
     │                     │                      │     Refresh Token    │
     │                     │                      │<─────────────────────┤
     │                     │                      │                      │
     │                     │                      │ 14. Encrypt & Save   │
     │                     │                      │     Token to DB      │
     │                     │                      │                      │
     │ 15. Success Page    │                      │                      │
     │    (BroadcastChannel)                      │                      │
     │<─────────────────────────────────────────┤                      │
     │                     │                      │                      │
     │ 16. Message Parent  │                      │                      │
     │     'success'       │                      │                      │
     ├────────────────────>│                      │                      │
     │                     │                      │                      │
     │                     │ 17. Close Popup      │                      │
     │                     │     Update UI        │                      │
     │                     │                      │                      │
```

### OAuth 1.0a Flow

```
┌─────────┐           ┌──────────┐           ┌─────────┐           ┌──────────┐
│ User UI │           │ Frontend │           │ Backend │           │ Provider │
└────┬────┘           └────┬─────┘           └────┬────┘           └────┬─────┘
     │                     │                      │                      │
     │ 1. Click Connect    │                      │                      │
     ├────────────────────>│                      │                      │
     │                     │                      │                      │
     │                     │ 2. GET /oauth1-      │                      │
     │                     │    credential/auth   │                      │
     │                     ├─────────────────────>│                      │
     │                     │                      │                      │
     │                     │                      │ 3. Request Token     │
     │                     │                      ├─────────────────────>│
     │                     │                      │                      │
     │                     │                      │ 4. Request Token +   │
     │                     │                      │    Secret            │
     │                     │                      │<─────────────────────┤
     │                     │                      │                      │
     │                     │                      │ 5. Generate CSRF     │
     │                     │                      │    Save Secret       │
     │                     │                      │                      │
     │                     │ 6. Auth URL + Token  │                      │
     │                     │<─────────────────────┤                      │
     │                     │                      │                      │
     │ 7. Open Popup       │                      │                      │
     │<────────────────────┤                      │                      │
     │                     │                      │                      │
     │ 8. Redirect to Provider with oauth_token   │                      │
     ├────────────────────────────────────────────────────────────────>│
     │                     │                      │                      │
     │ 9. User Authorizes  │                      │                      │
     │<────────────────────────────────────────────────────────────────┤
     │                     │                      │                      │
     │ 10. Callback with   │                      │                      │
     │     verifier & token│                      │                      │
     ├─────────────────────────────────────────>│                      │
     │                     │                      │                      │
     │                     │                      │ 11. Verify State     │
     │                     │                      │                      │
     │                     │                      │ 12. Exchange for     │
     │                     │                      │     Access Token     │
     │                     │                      ├─────────────────────>│
     │                     │                      │                      │
     │                     │                      │ 13. Access Token +   │
     │                     │                      │     Token Secret     │
     │                     │                      │<─────────────────────┤
     │                     │                      │                      │
     │                     │                      │ 14. Encrypt & Save   │
     │                     │                      │                      │
     │ 15. Success Page    │                      │                      │
     │<─────────────────────────────────────────┤                      │
     │                     │                      │                      │
     │ 16. Message Parent  │                      │                      │
     ├────────────────────>│                      │                      │
     │                     │                      │                      │
```

## File Structure

### Frontend Files

```
packages/frontend/editor-ui/src/features/credentials/
├── components/
│   ├── CredentialEdit/
│   │   ├── CredentialEdit.vue           # Main credential modal, popup management
│   │   ├── CredentialConfig.vue         # Credential form, OAuth button display
│   │   ├── OauthButton.vue              # OAuth button component
│   │   └── GoogleAuthButton.vue         # Google-specific OAuth button
│   └── ...
├── credentials.store.ts                  # State management, OAuth methods
├── credentials.api.ts                    # API client methods
└── credentials.types.ts                  # TypeScript types
```

### Backend Files

```
packages/cli/src/
├── controllers/
│   └── oauth/
│       ├── oauth1-credential.controller.ts              # OAuth1 endpoints
│       ├── oauth2-credential.controller.ts              # OAuth2 endpoints
│       └── oauth2-dynamic-client-registration.schema.ts # Validation schemas
├── oauth/
│   ├── oauth.service.ts                  # Core OAuth business logic
│   ├── validate-oauth-url.ts             # URL validation
│   └── types.ts                          # OAuth type definitions
├── credentials/
│   ├── credentials.controller.ts         # Main credential CRUD
│   ├── credentials.service.ts            # Credential business logic
│   ├── credentials-helper.ts             # Helper functions
│   └── credentials-finder.service.ts     # Credential lookup
├── credentials-helper.ts                 # Global credential helper
└── templates/
    ├── oauth-callback.handlebars         # Success callback page
    └── oauth-error-callback.handlebars   # Error callback page
```

### OAuth Client Library

```
packages/@n8n/client-oauth2/
├── src/
│   ├── client-oauth2.ts                  # Main OAuth2 client
│   ├── client-oauth2-token.ts            # Token management
│   ├── code-flow.ts                      # Authorization code flow
│   ├── credentials-flow.ts               # Client credentials flow
│   ├── types.ts                          # Type definitions
│   └── utils.ts                          # Utility functions
└── test/
    └── client-oauth2.test.ts             # Tests
```

### Database

```
packages/@n8n/db/src/
├── entities/
│   └── credentials-entity.ts             # Credential data model
└── migrations/
    ├── 1760116750277-CreateOAuthEntities.ts
    └── 1763572724000-ChangeOAuthStateColumnToUnboundedVarchar.ts
```

## Detailed Flow

### 1. User Initiates OAuth Connection

**File**: `packages/frontend/editor-ui/src/features/credentials/components/CredentialEdit/CredentialEdit.vue`

```typescript
// Lines 1084-1094
const oauthPopup = window.open(url, 'OAuth Authorization', params);
```

The user clicks "Connect" button in the credential modal, which:
- Opens a popup window pointing to the OAuth authorization URL
- Sets up a BroadcastChannel listener for the callback
- Clears any existing OAuth token data

### 2. Frontend Requests Authorization URL

**File**: `packages/frontend/editor-ui/src/features/credentials/credentials.store.ts`

```typescript
// Line 373-378
const oAuth2Authorize = async (data: ICredentialsResponse): Promise<string> => {
  return await credentialsApi.oAuth2CredentialAuthorize(rootStore.restApiContext, data);
};
```

**File**: `packages/frontend/editor-ui/src/features/credentials/credentials.api.ts`

```typescript
// Lines 98-108
export async function oAuth2CredentialAuthorize(
  context: IRestApiContext,
  data: ICredentialsResponse,
): Promise<string> {
  return await makeRestApiRequest(
    context,
    'GET',
    '/oauth2-credential/auth',
    data as unknown as IDataObject,
  );
}
```

### 3. Backend Generates Authorization URL

**File**: `packages/cli/src/controllers/oauth/oauth2-credential.controller.ts`

```typescript
// Lines 24-35
@Get('/auth')
async getAuthUri(req: OAuthRequest.OAuth2Credential.Auth): Promise<string> {
  const credential = await this.oauthService.getCredential(req);
  
  const uri = await this.oauthService.generateAOauth2AuthUri(credential, {
    cid: credential.id,
    origin: 'static-credential',
    userId: req.user.id,
  });
  return uri;
}
```

**File**: `packages/cli/src/oauth/oauth.service.ts`

```typescript
// Lines 325-446
async generateAOauth2AuthUri(
  credential: CredentialsEntity,
  csrfData: CreateCsrfStateData,
): Promise<string> {
  // Get credential data with defaults
  const oauthCredentials = await this.getOAuthCredentials<OAuth2CredentialData>(credential);
  
  // Handle dynamic client registration if configured
  if (oauthCredentials.useDynamicClientRegistration && oauthCredentials.serverUrl) {
    // Fetch server metadata
    // Register client dynamically
    // Update credentials with client_id and client_secret
  }
  
  // Validate OAuth URLs
  this.validateOAuthUrlOrThrow(oauthCredentials.authUrl);
  this.validateOAuthUrlOrThrow(oauthCredentials.accessTokenUrl);
  
  // Generate CSRF state token
  const [csrfSecret, state] = this.createCsrfState(csrfData);
  
  // Configure OAuth options
  const oAuthOptions = {
    ...this.convertCredentialToOptions(oauthCredentials),
    state,
  };
  
  // Handle PKCE flow
  if (oauthCredentials.grantType === 'pkce') {
    const { code_verifier, code_challenge } = await pkceChallenge();
    oAuthOptions.query = {
      ...oAuthOptions.query,
      code_challenge,
      code_challenge_method: 'S256',
    };
    toUpdate.codeVerifier = code_verifier;
  }
  
  // Save CSRF secret
  await this.encryptAndSaveData(credential, toUpdate);
  
  // Generate authorization URI
  const oAuthObj = new ClientOAuth2(oAuthOptions);
  const returnUri = oAuthObj.code.getUri();
  
  return returnUri.toString();
}
```

### 4. User Authorizes on Provider Site

The popup window navigates to the provider's OAuth authorization page. The user logs in (if needed) and approves the requested permissions.

### 5. Provider Redirects to Callback URL

The OAuth provider redirects back to n8n's callback URL with an authorization code and the state parameter:

```
https://n8n-instance.com/rest/oauth2-credential/callback?code=AUTH_CODE&state=ENCRYPTED_STATE
```

### 6. Backend Handles Callback

**File**: `packages/cli/src/controllers/oauth/oauth2-credential.controller.ts`

```typescript
// Lines 38-134
@Get('/callback', { usesTemplates: true, skipAuth: skipAuthOnOAuthCallback })
async handleCallback(req: OAuthRequest.OAuth2Credential.Callback, res: Response) {
  try {
    const { code, state: encodedState } = req.query;
    
    // Validate parameters
    if (!code || !encodedState) {
      return this.oauthService.renderCallbackError(res, 'Insufficient parameters');
    }
    
    // Resolve and verify credential
    const [credential, decryptedDataOriginal, oauthCredentials, state] =
      await this.oauthService.resolveCredential<OAuth2CredentialData>(req);
    
    // Configure options based on grant type
    let options: Partial<ClientOAuth2Options> = {};
    const oAuthOptions = this.convertCredentialToOptions(oauthCredentials);
    
    if (oauthCredentials.grantType === 'pkce') {
      options = {
        body: { code_verifier: decryptedDataOriginal.codeVerifier },
      };
    } else if (oauthCredentials.authentication === 'body') {
      options = {
        body: {
          ...(oAuthOptions.body ?? {}),
          client_id: oAuthOptions.clientId,
          client_secret: oAuthOptions.clientSecret,
        },
      };
      delete oAuthOptions.clientSecret;
    }
    
    // Exchange code for token
    const oAuthObj = new ClientOAuth2(oAuthOptions);
    const queryParameters = req.originalUrl.split('?').splice(1, 1).join('');
    const oauthToken = await oAuthObj.code.getToken(
      `${oAuthOptions.redirectUri}?${queryParameters}`,
      options,
    );
    
    // Merge token data with existing data
    const { oauthTokenData: tokenData } = decryptedDataOriginal;
    const oauthTokenData = {
      ...(typeof tokenData === 'object' ? tokenData : {}),
      ...oauthToken.data,
    };
    
    // Save encrypted token data
    if (!state.origin || state.origin === 'static-credential') {
      await this.oauthService.encryptAndSaveData(
        credential,
        { oauthTokenData },
        ['csrfSecret']
      );
      return res.render('oauth-callback');
    }
    
    // Handle dynamic credentials
    if (state.origin === 'dynamic-credential') {
      await this.oauthService.saveDynamicCredential(
        credential,
        { oauthTokenData },
        state.authorizationHeader.split('Bearer ')[1],
        state.credentialResolverId,
      );
      return res.render('oauth-callback');
    }
  } catch (e) {
    return this.oauthService.renderCallbackError(res, error.message);
  }
}
```

### 7. Success Page Notifies Parent Window

**File**: `packages/cli/templates/oauth-callback.handlebars`

```html
<script type="text/javascript">
  (function messageParent() {
    const broadcastChannel = new BroadcastChannel('oauth-callback');
    broadcastChannel.postMessage('success');
  })();
  (function autoclose(){
    setTimeout(function() { window.close(); }, 5000);
  })();
</script>
```

The success page:
1. Posts a 'success' message via BroadcastChannel
2. Auto-closes after 5 seconds

### 8. Frontend Receives Success Message

**File**: `packages/frontend/editor-ui/src/features/credentials/components/CredentialEdit/CredentialEdit.vue`

```typescript
// Lines 1113-1134
const receiveMessage = (event: MessageEvent) => {
  if (event.data === 'success') {
    // Update credential data
    credentialData.value = {
      ...credentialData.value,
      oauthTokenData: {} as CredentialInformation,
    };
    
    // Close popup
    if (oauthPopup) {
      oauthPopup.close();
    }
  }
};
oauthChannel.addEventListener('message', receiveMessage);
```

The frontend:
1. Receives the success message
2. Updates the credential data state
3. Closes the popup window
4. Updates the UI to show connection status

## Security Features

### 1. CSRF Protection

**Implementation**: `packages/cli/src/oauth/oauth.service.ts`

- Generates a unique CSRF token using the `csrf` library
- Encrypts state data containing credential ID and user ID
- Verifies token and state on callback
- Maximum age of 10 minutes (MAX_CSRF_AGE)

```typescript
// Lines 196-207
createCsrfState(data: CreateCsrfStateData): [string, string] {
  const token = new Csrf();
  const csrfSecret = token.secretSync();
  const state: CsrfState = {
    token: token.create(csrfSecret),
    createdAt: Date.now(),
    data: this.cipher.encrypt(JSON.stringify(data)),
  };
  const base64State = Buffer.from(JSON.stringify(state)).toString('base64');
  return [csrfSecret, base64State];
}
```

### 2. PKCE (Proof Key for Code Exchange)

For enhanced security in OAuth 2.0, PKCE is supported:

```typescript
// Lines 426-433
if (oauthCredentials.grantType === 'pkce') {
  const { code_verifier, code_challenge } = await pkceChallenge();
  oAuthOptions.query = {
    ...oAuthOptions.query,
    code_challenge,
    code_challenge_method: 'S256',
  };
  toUpdate.codeVerifier = code_verifier;
}
```

### 3. URL Validation

**File**: `packages/cli/src/oauth/validate-oauth-url.ts`

All OAuth URLs are validated to prevent:
- SSRF attacks
- Local network access
- Malicious redirects

### 4. Token Encryption

**File**: `packages/cli/src/oauth/oauth.service.ts`

```typescript
// Lines 176-187
async encryptAndSaveData(
  credential: ICredentialsDb,
  toUpdate: ICredentialDataDecryptedObject,
  toDelete: string[] = [],
) {
  const credentials = new Credentials(credential, credential.type, credential.data);
  credentials.updateData(toUpdate, toDelete);
  await this.credentialsRepository.update(credential.id, {
    ...credentials.getDataToSave(),
    updatedAt: new Date(),
  });
}
```

All OAuth tokens are encrypted before storage using the `Credentials` class from `n8n-core`.

### 5. Permission Checks

User permissions are verified before:
- Generating authorization URLs
- Handling callbacks
- Accessing credential data

### 6. Skip Auth Mode

**Environment Variable**: `N8N_SKIP_AUTH_ON_OAUTH_CALLBACK`

For iframe/embed scenarios, authentication can be skipped on OAuth callback. This is controlled via:

```typescript
// Lines 56-61
export function shouldSkipAuthOnOAuthCallback() {
  const value = process.env.N8N_SKIP_AUTH_ON_OAUTH_CALLBACK?.toLowerCase() ?? 'false';
  return value === 'true';
}
export const skipAuthOnOAuthCallback = shouldSkipAuthOnOAuthCallback();
```

## Special Features

### 1. Dynamic Client Registration (OAuth 2.0)

For providers supporting RFC 7591, n8n can dynamically register as an OAuth client:

**File**: `packages/cli/src/oauth/oauth.service.ts` (Lines 333-405)

1. Fetches OAuth authorization server metadata from `.well-known/oauth-authorization-server`
2. Registers client with the provider's registration endpoint
3. Receives and stores client_id and client_secret
4. Automatically selects appropriate grant type and authentication method

### 2. Dynamic Credentials (External Resolution)

n8n supports resolving credentials from external sources:

**File**: `packages/cli/src/oauth/oauth.service.ts` (Lines 615-640)

```typescript
async saveDynamicCredential(
  credential: CredentialsEntity,
  oauthTokenData: ICredentialDataDecryptedObject,
  authHeader: string,
  credentialResolverId: string,
) {
  // Store credential via dynamic proxy
  await this.dynamicCredentialsProxy.storeIfNeeded(
    credentialStoreMetadata,
    oauthTokenData,
    { version: 1, identity: authHeader },
    credentials.getData(),
    { credentialResolverId },
  );
}
```

### 3. Multiple OAuth Origins

The OAuth flow supports different origin types:
- `static-credential`: Standard n8n credentials
- `dynamic-credential`: Externally resolved credentials (for iframe/embed scenarios)

### 4. Google-Specific OAuth

**File**: `packages/frontend/editor-ui/src/features/credentials/components/CredentialEdit/GoogleAuthButton.vue`

Google OAuth has special handling in the UI with a custom button component.

## Credential Types Supporting OAuth

### OAuth 2.0 Examples
Located in `packages/nodes-base/credentials/`:

- GoogleDriveOAuth2Api.credentials.ts
- MicrosoftOneDriveOAuth2Api.credentials.ts
- MicrosoftToDoOAuth2Api.credentials.ts
- ZoomOAuth2Api.credentials.ts
- NotionOAuth2Api.credentials.ts
- SlackOAuth2Api.credentials.ts
- And many more...

### OAuth 1.0a Examples

- TwitterOAuth1Api.credentials.ts
- TrelloOAuth1Api.credentials.ts

Each credential type defines:
- `extends: ['oAuth2Api']` or `['oAuth1Api']`
- OAuth-specific properties (authUrl, accessTokenUrl, scopes, etc.)
- Additional authentication parameters

## Testing

### Backend Tests

**OAuth Service Tests**: `packages/cli/src/oauth/__tests__/oauth.service.test.ts`
**OAuth1 Controller Tests**: `packages/cli/src/controllers/oauth/__tests__/oauth1-credential.controller.test.ts`
**OAuth2 Controller Tests**: `packages/cli/src/controllers/oauth/__tests__/oauth2-credential.controller.test.ts`

### Integration Tests

**OAuth2 API Tests**: `packages/cli/test/integration/controllers/oauth/oauth2.api.test.ts`
**OAuth2 Skip Auth Tests**: `packages/cli/test/integration/controllers/oauth/oauth2.skip-auth.api.test.ts`

## Error Handling

### Frontend Error Display

Errors are displayed in the credential modal with appropriate messages.

### Backend Error Rendering

**File**: `packages/cli/templates/oauth-error-callback.handlebars`

When OAuth fails, users see:
- Error message
- Detailed reason (if available)
- n8n branding
- Manual close option

**File**: `packages/cli/src/oauth/oauth.service.ts` (Line 297)

```typescript
renderCallbackError(res: Response, message: string, reason?: string) {
  res.render('oauth-error-callback', { error: { message, reason } });
}
```

## API Types

**File**: `packages/@n8n/api-types/src/dto/oauth/oauth-client.dto.ts`

Defines TypeScript types shared between frontend and backend for OAuth operations.

## Configuration

### Environment Variables

- `N8N_SKIP_AUTH_ON_OAUTH_CALLBACK`: Skip authentication on OAuth callback (for iframe embedding)

### Global Config

**File**: `packages/cli/src/oauth/oauth.service.ts`

```typescript
getBaseUrl(oauthVersion: OauthVersion) {
  const restUrl = `${this.urlService.getInstanceBaseUrl()}/${this.globalConfig.endpoints.rest}`;
  return `${restUrl}/oauth${oauthVersion}-credential`;
}
```

Callback URLs are dynamically constructed based on:
- Instance base URL
- REST endpoint configuration
- OAuth version (1 or 2)

## Summary

The n8n OAuth implementation is a robust, secure system that:

1. **Supports both OAuth 1.0a and OAuth 2.0** with separate controllers and flows
2. **Implements security best practices** including CSRF protection, PKCE, URL validation, and token encryption
3. **Provides a seamless user experience** with popup windows and automatic callback handling
4. **Scales to support dynamic credentials** for iframe/embed scenarios
5. **Supports advanced OAuth 2.0 features** like dynamic client registration
6. **Maintains clean separation** between frontend UI, API layer, business logic, and data storage

The codebase is well-organized with clear separation of concerns, making it maintainable and extensible for adding new OAuth providers.
