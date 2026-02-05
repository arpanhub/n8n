# OAuth Flow Code Mapping

This directory contains comprehensive documentation mapping out the OAuth authentication flow in n8n when users connect their applications (Gmail, Notion, etc.).

## 📂 Documentation Files

### 1. [OAUTH_FLOW_QUICK_REFERENCE.md](./OAUTH_FLOW_QUICK_REFERENCE.md) 
**Start here!** Quick reference guide with:
- Key files organized by category
- 5-step flow summaries
- API endpoints
- Entry points for developers
- Debugging tips

### 2. [OAUTH_FLOW_DOCUMENTATION.md](./OAUTH_FLOW_DOCUMENTATION.md)
Comprehensive technical documentation with:
- Detailed architecture overview
- Complete file structure
- Step-by-step code flow with line numbers
- Security features explanation
- Code examples from actual implementation
- OAuth 1.0a and OAuth 2.0 flows

### 3. [OAUTH_FLOW_DIAGRAM.md](./OAUTH_FLOW_DIAGRAM.md)
Visual diagrams using Mermaid:
- Sequence diagrams (OAuth 2.0 and OAuth 1.0a)
- Component architecture
- PKCE flow
- Dynamic client registration
- State management
- Security layers
- Error handling flow
- BroadcastChannel communication

## 🎯 Quick Navigation

### For Different Roles

**Product Managers / Non-Technical**
- Start with the sequence diagrams in [OAUTH_FLOW_DIAGRAM.md](./OAUTH_FLOW_DIAGRAM.md)
- Then read the "OAuth Flow Sequence" section in [OAUTH_FLOW_DOCUMENTATION.md](./OAUTH_FLOW_DOCUMENTATION.md)

**Frontend Developers**
- Start with [OAUTH_FLOW_QUICK_REFERENCE.md](./OAUTH_FLOW_QUICK_REFERENCE.md) → "Entry Points" → "For Frontend Developers"
- Key file: `packages/frontend/editor-ui/src/features/credentials/components/CredentialEdit/CredentialEdit.vue`

**Backend Developers**
- Start with [OAUTH_FLOW_QUICK_REFERENCE.md](./OAUTH_FLOW_QUICK_REFERENCE.md) → "Entry Points" → "For Backend Developers"
- Key files: 
  - `packages/cli/src/controllers/oauth/oauth2-credential.controller.ts`
  - `packages/cli/src/oauth/oauth.service.ts`

**DevOps / Security Engineers**
- Focus on "Security Features" section in [OAUTH_FLOW_DOCUMENTATION.md](./OAUTH_FLOW_DOCUMENTATION.md)
- Review "Security Layers Diagram" in [OAUTH_FLOW_DIAGRAM.md](./OAUTH_FLOW_DIAGRAM.md)

## 🔍 What's Covered

### OAuth Flows
- ✅ OAuth 2.0 Authorization Code Flow
- ✅ OAuth 2.0 with PKCE (Proof Key for Code Exchange)
- ✅ OAuth 2.0 Dynamic Client Registration (RFC 7591)
- ✅ OAuth 1.0a Three-Legged Flow

### Components
- ✅ Frontend UI (Vue.js components)
- ✅ Frontend State Management (Pinia stores)
- ✅ Backend API Controllers
- ✅ OAuth Business Logic Service
- ✅ OAuth Client Library (@n8n/client-oauth2)
- ✅ Database Entities and Migrations
- ✅ Credential Encryption/Decryption
- ✅ Callback Templates

### Features
- ✅ CSRF Protection
- ✅ PKCE Support
- ✅ Token Encryption
- ✅ URL Validation
- ✅ Dynamic Credentials (External Resolution)
- ✅ BroadcastChannel Communication
- ✅ Error Handling
- ✅ Permission Checks

## 📊 File Statistics

- **Total Lines of Documentation**: ~1,600 lines
- **Key Backend Files Mapped**: 15+
- **Key Frontend Files Mapped**: 5+
- **Diagrams**: 11 Mermaid diagrams
- **Code Examples**: 20+ code snippets with line references

## 🚀 Common Use Cases

### Adding a New OAuth Provider

1. Review [OAUTH_FLOW_DOCUMENTATION.md](./OAUTH_FLOW_DOCUMENTATION.md) → "Credential Types Supporting OAuth"
2. Create new credential file in `packages/nodes-base/credentials/`
3. Extend `oAuth2Api` or `oAuth1Api`
4. Define provider-specific URLs and scopes

### Debugging OAuth Issues

1. Check [OAUTH_FLOW_QUICK_REFERENCE.md](./OAUTH_FLOW_QUICK_REFERENCE.md) → "Debugging Tips"
2. Review sequence diagrams to understand flow
3. Check error handling in `oauth.service.ts`

### Understanding Security

1. Read [OAUTH_FLOW_DOCUMENTATION.md](./OAUTH_FLOW_DOCUMENTATION.md) → "Security Features"
2. Review "Security Layers Diagram" in [OAUTH_FLOW_DIAGRAM.md](./OAUTH_FLOW_DIAGRAM.md)
3. Check CSRF and PKCE implementation

## 🔗 Related Resources

### In Repository
- Credential types: `packages/nodes-base/credentials/*OAuth*.credentials.ts`
- Tests: `packages/cli/src/oauth/__tests__/`
- Integration tests: `packages/cli/test/integration/controllers/oauth/`

### External
- OAuth 2.0 RFC: https://tools.ietf.org/html/rfc6749
- OAuth 1.0a RFC: https://tools.ietf.org/html/rfc5849
- PKCE RFC: https://tools.ietf.org/html/rfc7636
- Dynamic Client Registration RFC: https://tools.ietf.org/html/rfc7591

## 📝 Document Status

- **Created**: 2026-02-05
- **Last Updated**: 2026-02-05
- **Repository**: arpanhub/n8n
- **Branch**: copilot/map-oauth-code-flow
- **Status**: ✅ Complete

## 🤝 Contributing

If you find any missing information or errors in these documents:
1. The documentation reflects the codebase as of commit `304195f1`
2. For updates, review the actual source files referenced
3. Key files are in:
   - `packages/cli/src/oauth/`
   - `packages/cli/src/controllers/oauth/`
   - `packages/frontend/editor-ui/src/features/credentials/`

## 📄 License

This documentation follows the same license as the n8n project.

---

**Need Help?**
- Start with [OAUTH_FLOW_QUICK_REFERENCE.md](./OAUTH_FLOW_QUICK_REFERENCE.md)
- For details, see [OAUTH_FLOW_DOCUMENTATION.md](./OAUTH_FLOW_DOCUMENTATION.md)
- For visuals, see [OAUTH_FLOW_DIAGRAM.md](./OAUTH_FLOW_DIAGRAM.md)
