# OAuth Flow Code Mapping - Quick Reference

This file provides a quick reference to the OAuth flow implementation in n8n.

## 📚 Documentation Files

1. **[OAUTH_FLOW_DOCUMENTATION.md](./OAUTH_FLOW_DOCUMENTATION.md)** - Comprehensive technical documentation
   - Detailed file structure
   - Complete flow sequences
   - Security features
   - Code examples from actual files

2. **[OAUTH_FLOW_DIAGRAM.md](./OAUTH_FLOW_DIAGRAM.md)** - Visual diagrams
   - Sequence diagrams
   - Architecture diagrams
   - State flow diagrams
   - Security layer diagrams

## 🔑 Key Files by Category

### Frontend (User Interface)

| File | Purpose | Key Lines |
|------|---------|-----------|
| `packages/frontend/editor-ui/src/features/credentials/components/CredentialEdit/CredentialEdit.vue` | Main credential modal, manages OAuth popup window | 1084-1134 |
| `packages/frontend/editor-ui/src/features/credentials/components/CredentialEdit/CredentialConfig.vue` | Displays OAuth connection button and form | Full file |
| `packages/frontend/editor-ui/src/features/credentials/components/CredentialEdit/OauthButton.vue` | OAuth button component | Full file |
| `packages/frontend/editor-ui/src/features/credentials/credentials.store.ts` | Pinia store with OAuth methods | 373-378 |
| `packages/frontend/editor-ui/src/features/credentials/credentials.api.ts` | API client for OAuth endpoints | 85-108 |

### Backend (API Controllers)

| File | Purpose | Key Lines |
|------|---------|-----------|
| `packages/cli/src/controllers/oauth/oauth2-credential.controller.ts` | OAuth 2.0 endpoints (/auth, /callback) | Full file |
| `packages/cli/src/controllers/oauth/oauth1-credential.controller.ts` | OAuth 1.0a endpoints (/auth, /callback) | Full file |

### Backend (Business Logic)

| File | Purpose | Key Lines |
|------|---------|-----------|
| `packages/cli/src/oauth/oauth.service.ts` | Core OAuth service with all business logic | Full file (642 lines) |
| `packages/cli/src/oauth/validate-oauth-url.ts` | URL security validation | Full file |
| `packages/cli/src/credentials-helper.ts` | Credential encryption/decryption helpers | Full file |

### Backend (Templates)

| File | Purpose |
|------|---------|
| `packages/cli/templates/oauth-callback.handlebars` | Success page with BroadcastChannel communication |
| `packages/cli/templates/oauth-error-callback.handlebars` | Error page display |

### OAuth Client Library

| File | Purpose |
|------|---------|
| `packages/@n8n/client-oauth2/src/client-oauth2.ts` | Main OAuth2 client |
| `packages/@n8n/client-oauth2/src/client-oauth2-token.ts` | Token management |
| `packages/@n8n/client-oauth2/src/code-flow.ts` | Authorization code flow |

### Database

| File | Purpose |
|------|---------|
| `packages/@n8n/db/src/entities/credentials-entity.ts` | Credential data model |
| `packages/@n8n/db/src/migrations/common/1760116750277-CreateOAuthEntities.ts` | OAuth entities migration |

## 🔄 Quick Flow Reference

### OAuth 2.0 in 5 Steps

1. **User clicks "Connect"** → Opens popup window
2. **Frontend requests auth URL** → `GET /oauth2-credential/auth`
3. **User authorizes on provider** → Provider redirects to callback
4. **Backend exchanges code for token** → `GET /oauth2-credential/callback`
5. **Success page notifies parent** → BroadcastChannel → UI updates

### OAuth 1.0a in 5 Steps

1. **User clicks "Connect"** → Opens popup window
2. **Backend requests token** → Gets request token from provider
3. **User authorizes** → Provider shows consent screen
4. **Provider redirects** → Sends oauth_verifier
5. **Backend exchanges for access token** → Saves encrypted token

## 🛡️ Security Features

| Feature | Implementation |
|---------|----------------|
| CSRF Protection | Encrypted state token with 10-minute expiry |
| PKCE Support | Code challenge/verifier for OAuth 2.0 |
| Token Encryption | AES-256 encryption before database storage |
| URL Validation | Prevents SSRF and local network access |
| Permission Checks | User authorization before OAuth operations |

## 📡 API Endpoints

### OAuth 2.0
- `GET /rest/oauth2-credential/auth?id={credentialId}` - Get authorization URL
- `GET /rest/oauth2-credential/callback?code={code}&state={state}` - Handle callback

### OAuth 1.0a
- `GET /rest/oauth1-credential/auth?id={credentialId}` - Get authorization URL
- `GET /rest/oauth1-credential/callback?oauth_token={token}&oauth_verifier={verifier}&state={state}` - Handle callback

## 🎯 Entry Points

### For Frontend Developers
Start here: `packages/frontend/editor-ui/src/features/credentials/components/CredentialEdit/CredentialEdit.vue`

Key method: `openOAuthWindow()` (around line 1084)

### For Backend Developers
Start here: `packages/cli/src/controllers/oauth/oauth2-credential.controller.ts`

Key methods:
- `getAuthUri()` - Line 26
- `handleCallback()` - Line 38

### For Understanding Business Logic
Start here: `packages/cli/src/oauth/oauth.service.ts`

Key methods:
- `generateAOauth2AuthUri()` - Line 325
- `resolveCredential()` - Line 266
- `createCsrfState()` - Line 196

## 🔍 Debugging Tips

1. **Frontend popup issues**: Check `CredentialEdit.vue` BroadcastChannel setup
2. **Auth URL generation**: Check `oauth.service.ts` → `generateAOauth2AuthUri()`
3. **Callback errors**: Check `oauth.service.ts` → `resolveCredential()` and CSRF verification
4. **Token save failures**: Check `credentials-helper.ts` encryption logic
5. **Provider-specific issues**: Check credential type files in `packages/nodes-base/credentials/`

## 🧪 Testing

### Backend Tests
- `packages/cli/src/oauth/__tests__/oauth.service.test.ts`
- `packages/cli/src/controllers/oauth/__tests__/oauth2-credential.controller.test.ts`
- `packages/cli/src/controllers/oauth/__tests__/oauth1-credential.controller.test.ts`

### Integration Tests
- `packages/cli/test/integration/controllers/oauth/oauth2.api.test.ts`
- `packages/cli/test/integration/controllers/oauth/oauth2.skip-auth.api.test.ts`

## 📦 Example Credential Types

OAuth 2.0: `packages/nodes-base/credentials/`
- GoogleDriveOAuth2Api.credentials.ts
- SlackOAuth2Api.credentials.ts
- NotionOAuth2Api.credentials.ts
- MicrosoftOneDriveOAuth2Api.credentials.ts

OAuth 1.0a: `packages/nodes-base/credentials/`
- TwitterOAuth1Api.credentials.ts
- TrelloOAuth1Api.credentials.ts

## 🌐 Environment Variables

| Variable | Purpose | Default |
|----------|---------|---------|
| `N8N_SKIP_AUTH_ON_OAUTH_CALLBACK` | Skip user auth on callback (for iframe scenarios) | `false` |

## 💡 Key Concepts

- **CSRF State**: Encrypted, time-limited token preventing cross-site attacks
- **PKCE**: Enhanced OAuth 2.0 flow without client secret
- **Dynamic Credentials**: External credential resolution for iframe scenarios
- **BroadcastChannel**: Cross-window communication without postMessage
- **Dynamic Client Registration**: Automatic OAuth client registration per RFC 7591

## 📖 Further Reading

For complete details, see:
- [OAUTH_FLOW_DOCUMENTATION.md](./OAUTH_FLOW_DOCUMENTATION.md) - Full technical documentation
- [OAUTH_FLOW_DIAGRAM.md](./OAUTH_FLOW_DIAGRAM.md) - Visual diagrams

---

**Generated**: 2026-02-05  
**Repository**: n8n  
**Purpose**: OAuth flow code mapping for developer reference
