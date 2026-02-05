# n8n OAuth Flow Visual Diagrams

This document contains visual diagrams representing the OAuth flow in n8n using Mermaid.

## OAuth 2.0 Complete Flow Diagram

```mermaid
sequenceDiagram
    participant User as User Browser
    participant UI as Frontend (Vue)
    participant Store as Credentials Store
    participant API as Backend API
    participant OAuth as OAuth Service
    participant DB as Database
    participant Provider as OAuth Provider

    User->>UI: Click "Connect Account"
    UI->>Store: Call oAuth2Authorize()
    Store->>API: GET /oauth2-credential/auth
    API->>OAuth: getCredential(req)
    OAuth->>DB: Find credential by ID
    DB-->>OAuth: Credential entity
    OAuth->>OAuth: getOAuthCredentials()
    OAuth->>OAuth: createCsrfState()
    Note over OAuth: Generate CSRF token<br/>Encrypt state data<br/>Create PKCE challenge
    OAuth->>DB: Save CSRF secret & code_verifier
    OAuth->>OAuth: generateAOauth2AuthUri()
    OAuth-->>API: Authorization URL + state
    API-->>Store: Authorization URL
    Store-->>UI: Authorization URL
    UI->>User: Open popup window.open()
    
    User->>Provider: Navigate to authorization URL
    Provider->>User: Show login & consent screen
    User->>Provider: Approve authorization
    Provider->>User: Redirect to callback URL
    Note over Provider,User: /rest/oauth2-credential/callback<br/>?code=AUTH_CODE&state=STATE
    
    User->>API: GET /oauth2-credential/callback
    API->>OAuth: resolveCredential()
    OAuth->>OAuth: decodeCsrfState()
    Note over OAuth: Decode base64 state<br/>Decrypt state data<br/>Verify user ID match
    OAuth->>DB: Find credential by ID
    DB-->>OAuth: Credential + encrypted data
    OAuth->>OAuth: getDecryptedDataForCallback()
    OAuth->>OAuth: verifyCsrfState()
    Note over OAuth: Verify CSRF token<br/>Check timestamp < 10min
    OAuth-->>API: Credential, data, state
    
    API->>API: convertCredentialToOptions()
    API->>Provider: POST /token (exchange code)
    Note over API,Provider: Include:<br/>- authorization_code<br/>- code_verifier (PKCE)<br/>- client credentials
    Provider-->>API: Access token + Refresh token
    
    API->>OAuth: encryptAndSaveData()
    OAuth->>DB: Update credential with tokens
    DB-->>OAuth: Success
    
    API->>User: Render oauth-callback.html
    Note over User: BroadcastChannel message<br/>"success"<br/>Auto-close after 5s
    
    User->>UI: BroadcastChannel receives message
    UI->>UI: Update credential state
    UI->>User: Close popup, show success
```

## OAuth 1.0a Complete Flow Diagram

```mermaid
sequenceDiagram
    participant User as User Browser
    participant UI as Frontend (Vue)
    participant Store as Credentials Store
    participant API as Backend API
    participant OAuth as OAuth Service
    participant DB as Database
    participant Provider as OAuth Provider

    User->>UI: Click "Connect Account"
    UI->>Store: Call oAuth1Authorize()
    Store->>API: GET /oauth1-credential/auth
    API->>OAuth: getCredential(req)
    OAuth->>DB: Find credential by ID
    DB-->>OAuth: Credential entity
    
    OAuth->>OAuth: getOAuthCredentials()
    OAuth->>OAuth: createCsrfState()
    Note over OAuth: Generate CSRF token<br/>Encrypt state data
    
    OAuth->>Provider: POST /request_token
    Note over OAuth,Provider: OAuth 1.0a signature<br/>oauth_callback URL
    Provider-->>OAuth: Request token + secret
    
    OAuth->>DB: Save CSRF secret
    OAuth->>OAuth: Build authorization URL
    OAuth-->>API: Auth URL + oauth_token
    API-->>Store: Authorization URL
    Store-->>UI: Authorization URL
    UI->>User: Open popup window.open()
    
    User->>Provider: Navigate to auth URL
    Provider->>User: Show login & consent
    User->>Provider: Approve authorization
    Provider->>User: Redirect to callback
    Note over Provider,User: /rest/oauth1-credential/callback<br/>?oauth_token=TOKEN<br/>&oauth_verifier=VERIFIER<br/>&state=STATE
    
    User->>API: GET /oauth1-credential/callback
    API->>OAuth: resolveCredential()
    OAuth->>OAuth: decodeCsrfState()
    OAuth->>DB: Find credential
    OAuth->>OAuth: verifyCsrfState()
    OAuth-->>API: Credential, data, state
    
    API->>Provider: POST /access_token
    Note over API,Provider: oauth_token<br/>oauth_verifier
    Provider-->>API: Access token + token secret
    
    API->>OAuth: encryptAndSaveData()
    OAuth->>DB: Update credential with tokens
    
    API->>User: Render oauth-callback.html
    User->>UI: BroadcastChannel message
    UI->>User: Close popup, show success
```

## Component Architecture Diagram

```mermaid
graph TB
    subgraph "Frontend (Vue.js)"
        A[CredentialEdit.vue] --> B[CredentialConfig.vue]
        B --> C[OauthButton.vue]
        A --> D[credentials.store.ts]
        D --> E[credentials.api.ts]
    end
    
    subgraph "Backend API Layer"
        E -->|HTTP Request| F[OAuth2CredentialController]
        E -->|HTTP Request| G[OAuth1CredentialController]
        F --> H[OauthService]
        G --> H
    end
    
    subgraph "Business Logic Layer"
        H --> I[CredentialsHelper]
        H --> J[CredentialsRepository]
        H --> K[ClientOAuth2]
        I --> J
    end
    
    subgraph "Data Layer"
        J --> L[(Database)]
        M[CredentialsEntity] -.stored in.- L
    end
    
    subgraph "OAuth Provider"
        N[OAuth Provider API]
    end
    
    K -->|Token Exchange| N
    F -->|Callback| O[oauth-callback.handlebars]
    G -->|Callback| O
    O -->|BroadcastChannel| A
    
    style A fill:#e1f5ff
    style F fill:#ffe1e1
    style G fill:#ffe1e1
    style H fill:#fff3e1
    style K fill:#e8ffe1
    style N fill:#ffe1f5
```

## OAuth 2.0 PKCE Flow Diagram

```mermaid
sequenceDiagram
    participant UI as Frontend
    participant Backend as OAuth Service
    participant DB as Database
    participant Provider as OAuth Provider

    Note over UI,Provider: PKCE (Proof Key for Code Exchange) Flow
    
    UI->>Backend: Request authorization URL
    Backend->>Backend: Generate code_verifier (random)
    Backend->>Backend: Generate code_challenge = SHA256(code_verifier)
    Backend->>DB: Save code_verifier encrypted
    
    Backend-->>UI: Auth URL with code_challenge
    Note over Backend,UI: URL includes:<br/>code_challenge=HASH<br/>code_challenge_method=S256
    
    UI->>Provider: Navigate to auth URL
    Provider->>Provider: Store code_challenge
    Provider->>UI: Redirect with code
    
    UI->>Backend: Callback with code
    Backend->>DB: Retrieve code_verifier
    Backend->>Provider: POST /token
    Note over Backend,Provider: Include:<br/>- code<br/>- code_verifier
    Provider->>Provider: Verify SHA256(code_verifier) == code_challenge
    Provider-->>Backend: Access token
    Backend->>DB: Save encrypted token
```

## Dynamic Client Registration Flow

```mermaid
sequenceDiagram
    participant UI as Frontend
    participant Backend as OAuth Service
    participant Provider as OAuth Provider
    participant DB as Database

    Note over UI,DB: Dynamic Client Registration (RFC 7591)
    
    UI->>Backend: Request OAuth connection
    Backend->>Provider: GET /.well-known/oauth-authorization-server
    Provider-->>Backend: Server metadata
    Note over Backend: Includes:<br/>- authorization_endpoint<br/>- token_endpoint<br/>- registration_endpoint<br/>- grant_types_supported
    
    Backend->>Backend: Select grant type & auth method
    Note over Backend: Preferences:<br/>1. PKCE (S256)<br/>2. Authorization Code + Basic Auth<br/>3. Client Credentials
    
    Backend->>Provider: POST /register
    Note over Backend,Provider: Registration request:<br/>- redirect_uris<br/>- grant_types<br/>- token_endpoint_auth_method<br/>- client_name: "n8n"
    Provider-->>Backend: Client credentials
    Note over Provider: Response:<br/>- client_id<br/>- client_secret (if applicable)
    
    Backend->>DB: Save client credentials
    Backend-->>UI: Continue OAuth flow
```

## File Organization Diagram

```mermaid
graph LR
    subgraph "Frontend Package"
        A[editor-ui/src/features/credentials/]
        A --> A1[components/CredentialEdit/]
        A --> A2[credentials.store.ts]
        A --> A3[credentials.api.ts]
        A1 --> A11[CredentialEdit.vue]
        A1 --> A12[CredentialConfig.vue]
        A1 --> A13[OauthButton.vue]
    end
    
    subgraph "Backend Package"
        B[cli/src/]
        B --> B1[controllers/oauth/]
        B --> B2[oauth/]
        B --> B3[credentials/]
        B --> B4[templates/]
        B1 --> B11[oauth2-credential.controller.ts]
        B1 --> B12[oauth1-credential.controller.ts]
        B2 --> B21[oauth.service.ts]
        B2 --> B22[validate-oauth-url.ts]
        B3 --> B31[credentials.controller.ts]
        B3 --> B32[credentials.service.ts]
        B3 --> B33[credentials-helper.ts]
        B4 --> B41[oauth-callback.handlebars]
        B4 --> B42[oauth-error-callback.handlebars]
    end
    
    subgraph "OAuth Client Library"
        C[@n8n/client-oauth2/src/]
        C --> C1[client-oauth2.ts]
        C --> C2[client-oauth2-token.ts]
        C --> C3[code-flow.ts]
    end
    
    subgraph "Database Package"
        D[@n8n/db/src/]
        D --> D1[entities/credentials-entity.ts]
        D --> D2[migrations/OAuth*.ts]
    end
    
    A3 -.HTTP.-> B11
    A3 -.HTTP.-> B12
    B11 --> B21
    B12 --> B21
    B21 --> B33
    B21 --> C1
    B33 --> D1
    
    style A fill:#e1f5ff
    style B fill:#ffe1e1
    style C fill:#e8ffe1
    style D fill:#fff3e1
```

## State Management Flow

```mermaid
stateDiagram-v2
    [*] --> CredentialCreated: User creates credential
    CredentialCreated --> AuthorizationRequested: User clicks Connect
    
    state AuthorizationRequested {
        [*] --> GenerateCSRF
        GenerateCSRF --> SaveCSRFSecret
        SaveCSRFSecret --> GeneratePKCE: If OAuth2 PKCE
        GeneratePKCE --> SaveCodeVerifier
        SaveCodeVerifier --> BuildAuthURL
        GenerateCSRF --> BuildAuthURL: If OAuth1 or standard OAuth2
        BuildAuthURL --> [*]
    }
    
    AuthorizationRequested --> PopupOpened: Return auth URL
    PopupOpened --> UserAuthorizing: Navigate to provider
    UserAuthorizing --> CallbackReceived: User approves
    
    state CallbackReceived {
        [*] --> ValidateState
        ValidateState --> DecodeState
        DecodeState --> VerifyCSRF
        VerifyCSRF --> RetrieveCredential
        RetrieveCredential --> ExchangeToken: If OAuth2
        ExchangeToken --> SaveTokens
        RetrieveCredential --> RequestAccessToken: If OAuth1
        RequestAccessToken --> SaveTokens
        SaveTokens --> [*]
    }
    
    CallbackReceived --> CredentialConnected: Tokens saved
    CredentialConnected --> [*]: Success page shown
    
    UserAuthorizing --> AuthorizationFailed: User denies
    CallbackReceived --> AuthorizationFailed: Error occurs
    AuthorizationFailed --> [*]: Error page shown
```

## Security Layers Diagram

```mermaid
graph TB
    subgraph "Security Measures"
        A[User Request] --> B{Authentication}
        B -->|Authenticated| C[Permission Check]
        B -->|Skip Auth Mode| D[Dynamic Credential Check]
        C -->|Authorized| E[Generate CSRF Token]
        D -->|Valid Bearer Token| E
        
        E --> F[Encrypt State Data]
        F --> G[Create OAuth URL]
        
        G --> H[URL Validation]
        H -->|Valid| I[User Authorization]
        H -->|Invalid| J[Reject - Security Error]
        
        I --> K[Callback Received]
        K --> L[Decode State]
        L --> M[Verify CSRF Token]
        M -->|Valid & Fresh| N[Verify User Match]
        M -->|Invalid/Expired| O[Reject - CSRF Failed]
        
        N -->|Match| P{OAuth Version}
        N -->|No Match| Q[Reject - Unauthorized]
        
        P -->|OAuth2 PKCE| R[Verify Code Verifier]
        P -->|OAuth2 Standard| S[Use Client Secret]
        P -->|OAuth1| T[Use Token Secret]
        
        R --> U[Exchange Code]
        S --> U
        T --> U
        
        U --> V[Encrypt Token]
        V --> W[Save to Database]
        
        style A fill:#e1f5ff
        style E fill:#fff3e1
        style F fill:#fff3e1
        style H fill:#ffe1e1
        style M fill:#ffe1e1
        style V fill:#e8ffe1
        style J fill:#ffcccc
        style O fill:#ffcccc
        style Q fill:#ffcccc
    end
```

## BroadcastChannel Communication

```mermaid
sequenceDiagram
    participant Popup as OAuth Popup Window
    participant Channel as BroadcastChannel
    participant Parent as Parent Window (Modal)

    Note over Popup,Parent: Same-origin communication mechanism
    
    Parent->>Channel: Create BroadcastChannel('oauth-callback')
    Parent->>Channel: addEventListener('message')
    Parent->>Popup: window.open(authUrl)
    
    Popup->>Provider: User authorizes
    Provider->>Popup: Redirect to callback URL
    
    Popup->>Popup: Render success page
    Popup->>Channel: new BroadcastChannel('oauth-callback')
    Popup->>Channel: postMessage('success')
    
    Channel->>Parent: Broadcast 'success' message
    Parent->>Parent: Update credential state
    Parent->>Popup: Close popup (or auto-close)
    Parent->>Parent: Update UI to show connected
    
    Note over Popup: Auto-closes after 5 seconds
```

## Error Handling Flow

```mermaid
graph TD
    A[OAuth Flow Start] --> B{Step}
    
    B -->|Auth URL Generation| C{Error?}
    C -->|Invalid Credentials| D[NotFoundError]
    C -->|Invalid OAuth URLs| E[ValidationError]
    C -->|No Error| F[Generate URL]
    
    B -->|User Authorization| G{Error?}
    G -->|User Denies| H[Provider Redirects with error]
    G -->|Network Error| I[Timeout/Connection Error]
    G -->|No Error| J[Authorization Code Returned]
    
    B -->|Callback Processing| K{Error?}
    K -->|Missing Parameters| L[Render Error Page: Insufficient params]
    K -->|Invalid State| M[Render Error Page: Invalid state]
    K -->|CSRF Verification Failed| N[Render Error Page: CSRF failed]
    K -->|Token Exchange Failed| O[Render Error Page: Provider error]
    K -->|No Error| P[Save Tokens]
    
    D --> Q[Frontend: Show Error in Modal]
    E --> Q
    H --> R[Render oauth-error-callback.html]
    I --> R
    L --> R
    M --> R
    N --> R
    O --> R
    
    P --> S[Render oauth-callback.html]
    S --> T[BroadcastChannel: success]
    T --> U[Frontend: Update UI]
    
    R --> V[BroadcastChannel: error OR manual close]
    V --> W[Frontend: Show error message]
    
    style D fill:#ffcccc
    style E fill:#ffcccc
    style H fill:#ffcccc
    style I fill:#ffcccc
    style L fill:#ffcccc
    style M fill:#ffcccc
    style N fill:#ffcccc
    style O fill:#ffcccc
    style S fill:#ccffcc
    style U fill:#ccffcc
```

## Credential Type Inheritance

```mermaid
graph TD
    A[ICredentialType] --> B[oAuth2Api]
    A --> C[oAuth1Api]
    
    B --> D[GoogleDriveOAuth2Api]
    B --> E[MicrosoftOneDriveOAuth2Api]
    B --> F[NotionOAuth2Api]
    B --> G[SlackOAuth2Api]
    B --> H[ZoomOAuth2Api]
    B --> I[...]
    
    C --> J[TwitterOAuth1Api]
    C --> K[TrelloOAuth1Api]
    C --> L[...]
    
    style A fill:#e1f5ff
    style B fill:#fff3e1
    style C fill:#fff3e1
    style D fill:#e8ffe1
    style E fill:#e8ffe1
    style F fill:#e8ffe1
    style G fill:#e8ffe1
    style H fill:#e8ffe1
    style J fill:#ffe1f5
    style K fill:#ffe1f5
```

## Token Storage and Encryption

```mermaid
graph LR
    A[OAuth Token Received] --> B[Credentials Class]
    B --> C[updateData method]
    C --> D{Encryption}
    
    D --> E[Cipher Service]
    E --> F[Encrypt with AES-256]
    F --> G[Base64 Encode]
    
    G --> H[CredentialsEntity]
    H --> I[(Database)]
    
    J[Retrieve Credentials] --> K[Load from Database]
    K --> L[CredentialsEntity]
    L --> M[Credentials Class]
    M --> N[Cipher Service]
    N --> O[Decrypt]
    O --> P[Decrypted Token Data]
    P --> Q[Use in Workflow]
    
    style A fill:#e1f5ff
    style F fill:#fff3e1
    style I fill:#ffe1e1
    style O fill:#e8ffe1
    style Q fill:#ccffcc
```

## Summary

These diagrams provide visual representations of:

1. **Sequence Diagrams**: Show the step-by-step flow of OAuth 1.0a and OAuth 2.0 authentication
2. **Architecture Diagrams**: Display the component structure and relationships
3. **Flow Diagrams**: Illustrate specialized flows like PKCE and dynamic client registration
4. **State Diagrams**: Show the state transitions during the OAuth process
5. **Security Diagrams**: Highlight the security layers and measures
6. **Communication Diagrams**: Explain BroadcastChannel usage for popup-parent communication
7. **Error Handling**: Map out error scenarios and handling
8. **Data Flow**: Show token encryption and storage

All diagrams use Mermaid syntax and can be rendered in GitHub, GitLab, or any Mermaid-compatible viewer.
