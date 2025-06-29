# AI Implementation Guide for fp-ai-openwebui

## 🎯 Working with AI to Implement Entra Authentication

You can absolutely use AI assistants to implement the Entra ID integration! Here's how to set up the perfect collaboration:

---

## 1. Create Essential Context Files

### A. Create `IMPLEMENTATION.md` (Detailed Technical Specs)

```markdown
# Entra ID Implementation Specification

## Architecture Overview
Replace OpenWebUI's built-in authentication with Microsoft Entra ID while maintaining all original functionality.

## Integration Points

### Backend Changes Required
1. **Replace `backend/apps/auth/`** - Complete overhaul with MSAL integration
2. **Modify `backend/main.py`** - Add Entra middleware
3. **Update `backend/requirements.txt`** - Add MSAL dependencies

### Frontend Changes Required  
1. **Update `src/lib/stores/user.js`** - New auth flow
2. **Modify `src/routes/+layout.svelte`** - Auth gate
3. **Replace `src/routes/auth/`** - Entra-specific pages

## Technical Requirements

### Dependencies to Add
- `msal==1.24.0` (Python MSAL library)
- `python-jose[cryptography]==3.3.0` (JWT handling)
- `requests==2.31.0` (HTTP client)

### Environment Variables
- `ENTRA_TENANT_ID`
- `ENTRA_CLIENT_ID` 
- `ENTRA_CLIENT_SECRET`
- `ENTRA_REDIRECT_URI`

### Key Functions to Implement
1. `EntraAuth.get_auth_url()` - Generate login URL
2. `EntraAuth.exchange_code_for_tokens()` - Handle callback
3. `EntraAuth.validate_token()` - JWT validation
4. `EntraAuth.get_user_profile()` - Fetch user info
5. `create_or_update_user()` - User provisioning

## Authentication Flow
1. User visits app → Check for valid session
2. No session → Redirect to Entra login
3. User authenticates → Entra redirects with code
4. Exchange code for tokens → Validate & create session
5. Map Entra user to OpenWebUI user → Set session cookie

## File-by-File Implementation Plan

### Phase 1: Backend Core (Priority 1)
- [ ] `backend/apps/auth/entra.py` - New Entra auth module
- [ ] `backend/apps/auth/routes.py` - Replace auth routes
- [ ] `backend/requirements.txt` - Add dependencies
- [ ] `backend/main.py` - Add auth middleware

### Phase 2: Frontend Integration (Priority 2)  
- [ ] `src/lib/stores/user.js` - Update auth store
- [ ] `src/routes/+layout.svelte` - Add auth gate
- [ ] `src/routes/auth/callback/+page.svelte` - Callback handler

### Phase 3: Configuration (Priority 3)
- [ ] `.env.example` - Environment template
- [ ] `vercel.json` - Vercel deployment config
- [ ] `docs/ENTRA_SETUP.md` - Setup guide

## Success Criteria
- [ ] Users authenticate via Entra ID only
- [ ] All existing OpenWebUI features work unchanged
- [ ] User roles map from Entra groups
- [ ] Sessions persist correctly
- [ ] Deployable to Vercel
```

### B. Create `CONTEXT.md` (AI Assistant Context)

```markdown
# AI Assistant Context for fp-ai-openwebui

## Project Overview
This is a fork of open-webui/open-webui with Entra ID authentication integration. The goal is to replace the built-in auth system with Microsoft Entra ID while preserving all original functionality.

## Current State
- ✅ Repository forked and set up
- ✅ Custom README with Entra features listed  
- ❌ Entra authentication not yet implemented
- ❌ No code changes made yet

## Key Constraints
1. **Preserve OpenWebUI branding** (license requirement)
2. **Maintain all original features** 
3. **Enterprise-ready security**
4. **Vercel deployable**

## Architecture Decision
Direct integration approach - modify the OpenWebUI codebase rather than using a proxy.

## Tech Stack
- **Backend**: FastAPI (Python)
- **Frontend**: Svelte/SvelteKit  
- **Auth**: Microsoft Entra ID (Azure AD)
- **Deployment**: Vercel

## Implementation Strategy
Phase 1: Backend Entra integration
Phase 2: Frontend auth flow updates  
Phase 3: Configuration and deployment

## Files to Modify
See IMPLEMENTATION.md for detailed file-by-file breakdown.

## AI Assistant Guidelines
- Follow the implementation plan in IMPLEMENTATION.md
- Ask for clarification on Entra-specific requirements
- Suggest improvements while staying within scope
- Test each component before moving to next phase
- Maintain code quality and security standards
```

### C. Create `.cursorrules` (For Cursor AI)

```markdown
# Cursor Rules for fp-ai-openwebui

## Project Context
OpenWebUI fork with Entra ID authentication integration. Preserve all original functionality while replacing auth system.

## Key Files
- `IMPLEMENTATION.md` - Technical specification
- `CONTEXT.md` - Project overview  
- Original OpenWebUI codebase in all other directories

## Coding Guidelines
1. **Security First**: Validate all tokens, sanitize inputs
2. **Preserve Branding**: Keep OpenWebUI attribution per license
3. **Maintain Compatibility**: All existing features must work
4. **Enterprise Ready**: Support group-based roles, SSO

## Implementation Priority
1. Backend auth module (`backend/apps/auth/entra.py`)
2. Auth routes (`backend/apps/auth/routes.py`) 
3. Frontend auth store (`src/lib/stores/user.js`)
4. Layout auth gate (`src/routes/+layout.svelte`)

## When Working on Files
- Reference IMPLEMENTATION.md for technical specs
- Test each component thoroughly
- Ask about Entra configuration details if unclear
- Suggest security improvements
- Maintain existing code style and patterns
```

---

## 2. AI Collaboration Commands

### For Claude Code (Command Line)
```bash
# Clone your repo locally first
git clone https://github.com/RNCDev/fp-ai-openwebui.git
cd fp-ai-openwebui

# Then use Claude Code to implement
claude code "Implement the Entra authentication module as specified in IMPLEMENTATION.md. Start with backend/apps/auth/entra.py"

claude code "Review the current OpenWebUI auth structure and create the replacement routes in backend/apps/auth/routes.py"

claude code "Update the frontend auth store in src/lib/stores/user.js to work with the new Entra flow"
```

### For Cursor AI
1. Open the repository in Cursor
2. Chat: "Review IMPLEMENTATION.md and start implementing Phase 1: Backend Core"
3. Use `@IMPLEMENTATION.md` and `@CONTEXT.md` in your prompts
4. Let Cursor suggest and implement the code step by step

### For Other AI Assistants
Provide the context files and ask:
> "I have a fork of OpenWebUI that needs Entra ID authentication integration. Please review my IMPLEMENTATION.md and CONTEXT.md files, then help me implement the backend authentication module first."

---

## 3. Implementation Workflow with AI

### Step 1: Backend Foundation
```
AI Task: "Create backend/apps/auth/entra.py with MSAL integration"
- Review existing OpenWebUI auth patterns
- Implement EntraAuth class with required methods
- Add proper error handling and security
```

### Step 2: Routes Integration  
```
AI Task: "Replace backend/apps/auth/routes.py with Entra-compatible routes"
- Keep same API endpoints where possible
- Add Entra-specific endpoints (/login, /callback)
- Integrate with existing user management
```

### Step 3: Frontend Updates
```
AI Task: "Update src/lib/stores/user.js for Entra authentication flow"
- Modify existing auth store
- Add Entra-specific methods
- Maintain backward compatibility
```

### Step 4: Testing & Deployment
```
AI Task: "Create Vercel deployment configuration and test setup"
- Add vercel.json configuration
- Create environment setup guide
- Test authentication flow
```

---

## 4. Quality Checkpoints

After each AI implementation session:

✅ **Security Review**: Token validation, secure cookies, HTTPS
✅ **Feature Preservation**: All OpenWebUI features still work  
✅ **License Compliance**: OpenWebUI branding maintained
✅ **Code Quality**: Follows existing patterns and standards
✅ **Documentation**: Updates match implementation

---

## 5. Example AI Prompt

Here's a perfect starting prompt for your AI assistant:

> "I have a fork of OpenWebUI at https://github.com/RNCDev/fp-ai-openwebui that needs Microsoft Entra ID authentication integration. 
>
> Please review my repository structure and the IMPLEMENTATION.md file I'll share. I want to replace the built-in auth system with Entra ID while preserving all original OpenWebUI functionality and branding.
>
> Let's start with Phase 1: implementing the backend Entra authentication module. Can you help me create `backend/apps/auth/entra.py` with proper MSAL integration based on the specifications?"

This approach gives you:
- 🎯 **Clear specifications** for the AI to follow
- 🔄 **Iterative implementation** with checkpoints  
- 📋 **Quality assurance** at each step
- 🚀 **Fast development** with AI assistance

The key is providing comprehensive context upfront so the AI understands exactly what you're building and can implement it correctly!
