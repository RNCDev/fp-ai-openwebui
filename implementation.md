# OpenWebUI Fork with Entra Authentication - Direct Integration Plan

## Overview: Fork and Modify OpenWebUI

This approach involves forking the OpenWebUI repository and directly integrating Entra ID authentication into the codebase before the main application logic.

---

## Phase 1: Repository Setup and Analysis (Week 1)

### 1.1 Fork and Clone OpenWebUI
```bash
# 1. Fork https://github.com/open-webui/open-webui to your account
# 2. Clone your fork
git clone https://github.com/RNCDev/fp-ai-openwebui.git
cd fp-ai-openwebui

# 3. Add upstream remote for updates
git remote add upstream https://github.com/open-webui/open-webui.git

# 4. Create feature branch
git checkout -b feature/entra-authentication
```

### 1.2 Analyze OpenWebUI Architecture
**Current OpenWebUI Structure:**
```
open-webui/
├── backend/                 # FastAPI Python backend
│   ├── apps/
│   │   ├── auth/           # Current auth system
│   │   ├── ollama/         # Model integration
│   │   └── webui/          # Main API endpoints
│   ├── main.py             # FastAPI app entry point
│   └── requirements.txt
├── src/                    # Svelte frontend
│   ├── lib/
│   │   ├── components/
│   │   └── stores/         # Auth state management
│   ├── routes/
│   │   ├── auth/           # Auth pages
│   │   └── +layout.svelte  # Main layout
│   └── app.html
└── docker/                 # Docker configs
```

### 1.3 Identify Integration Points
**Key Files to Modify:**
- `backend/apps/auth/` - Replace auth logic
- `backend/main.py` - Add Entra middleware
- `src/lib/stores/user.js` - Update user state management
- `src/routes/auth/` - Replace auth UI
- `src/routes/+layout.svelte` - Add auth check

---

## Phase 2: Backend Authentication Integration (Week 2)

### 2.1 Install Required Dependencies
```bash
# Add to backend/requirements.txt
msal==1.24.0
python-jose[cryptography]==3.3.0
requests==2.31.0
```

### 2.2 Create Entra Authentication Module
```python
# backend/apps/auth/entra.py
import msal
from jose import jwt, JWTError
from fastapi import HTTPException, Depends, Request
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
import requests
from typing import Optional
import os

class EntraAuth:
    def __init__(self):
        self.tenant_id = os.getenv("ENTRA_TENANT_ID")
        self.client_id = os.getenv("ENTRA_CLIENT_ID") 
        self.client_secret = os.getenv("ENTRA_CLIENT_SECRET")
        self.authority = f"https://login.microsoftonline.com/{self.tenant_id}"
        self.redirect_uri = os.getenv("ENTRA_REDIRECT_URI", "http://localhost:3000/auth/callback")
        
        # MSAL app for server-side auth
        self.msal_app = msal.ConfidentialClientApplication(
            client_id=self.client_id,
            client_credential=self.client_secret,
            authority=self.authority
        )
        
        # JWT validation keys
        self.jwks_uri = f"https://login.microsoftonline.com/{self.tenant_id}/discovery/v2.0/keys"
        self._jwks_cache = None

    def get_auth_url(self, state: str = None):
        """Generate Entra ID authorization URL"""
        return self.msal_app.get_authorization_request_url(
            scopes=["User.Read", "openid", "profile", "email"],
            state=state,
            redirect_uri=self.redirect_uri
        )

    async def exchange_code_for_tokens(self, code: str, state: str = None):
        """Exchange authorization code for tokens"""
        result = self.msal_app.acquire_token_by_authorization_code(
            code,
            scopes=["User.Read", "openid", "profile", "email"],
            redirect_uri=self.redirect_uri
        )
        
        if "error" in result:
            raise HTTPException(status_code=400, detail=result.get("error_description"))
            
        return result

    async def validate_token(self, token: str) -> dict:
        """Validate JWT token and return user info"""
        try:
            # Get signing keys
            if not self._jwks_cache:
                response = requests.get(self.jwks_uri)
                self._jwks_cache = response.json()
            
            # Decode and validate JWT
            unverified_header = jwt.get_unverified_header(token)
            rsa_key = self._get_rsa_key(unverified_header["kid"])
            
            payload = jwt.decode(
                token,
                rsa_key,
                algorithms=["RS256"],
                audience=self.client_id,
                issuer=f"https://login.microsoftonline.com/{self.tenant_id}/v2.0"
            )
            
            return payload
            
        except JWTError as e:
            raise HTTPException(status_code=401, detail="Invalid token")

    def _get_rsa_key(self, kid: str):
        """Extract RSA key for token validation"""
        for key in self._jwks_cache["keys"]:
            if key["kid"] == kid:
                return {
                    "kty": key["kty"],
                    "kid": key["kid"],
                    "use": key["use"],
                    "n": key["n"],
                    "e": key["e"]
                }
        raise HTTPException(status_code=401, detail="Unable to find appropriate key")

    async def get_user_profile(self, access_token: str):
        """Get user profile from Microsoft Graph"""
        headers = {"Authorization": f"Bearer {access_token}"}
        response = requests.get("https://graph.microsoft.com/v1.0/me", headers=headers)
        
        if response.status_code == 200:
            return response.json()
        else:
            raise HTTPException(status_code=400, detail="Failed to get user profile")

# Global auth instance
entra_auth = EntraAuth()

# FastAPI dependency for protected routes
security = HTTPBearer()

async def get_current_user(credentials: HTTPAuthorizationCredentials = Depends(security)):
    """FastAPI dependency to get current authenticated user"""
    token = credentials.credentials
    user_data = await entra_auth.validate_token(token)
    return user_data
```

### 2.3 Modify Authentication Routes
```python
# backend/apps/auth/routes.py (replace existing auth routes)
from fastapi import APIRouter, Request, HTTPException, Depends, Response
from fastapi.responses import RedirectResponse
from .entra import entra_auth, get_current_user
from ..webui.models.users import Users, UserModel, UserResponse
import secrets

router = APIRouter()

@router.get("/login")
async def login():
    """Initiate Entra ID login"""
    state = secrets.token_urlsafe(32)
    auth_url = entra_auth.get_auth_url(state=state)
    
    response = RedirectResponse(url=auth_url)
    response.set_cookie("auth_state", state, httponly=True, secure=True)
    return response

@router.get("/callback")
async def auth_callback(request: Request, code: str, state: str = None):
    """Handle Entra ID callback"""
    # Verify state parameter
    stored_state = request.cookies.get("auth_state")
    if not stored_state or stored_state != state:
        raise HTTPException(status_code=400, detail="Invalid state parameter")
    
    # Exchange code for tokens
    token_result = await entra_auth.exchange_code_for_tokens(code, state)
    access_token = token_result["access_token"]
    id_token = token_result["id_token"]
    
    # Get user profile
    user_profile = await entra_auth.get_user_profile(access_token)
    
    # Create or update user in OpenWebUI database
    user = await create_or_update_user(user_profile)
    
    # Generate OpenWebUI session token
    session_token = create_session_token(user)
    
    response = RedirectResponse(url="/")
    response.set_cookie("token", session_token, httponly=True, secure=True)
    response.delete_cookie("auth_state")
    return response

@router.get("/user")
async def get_user_info(user_data = Depends(get_current_user)):
    """Get current user information"""
    user = Users.get_user_by_email(user_data["email"])
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    return UserResponse(**user.model_dump())

@router.post("/logout")
async def logout():
    """Logout user"""
    response = Response(content="Logged out successfully")
    response.delete_cookie("token")
    return response

async def create_or_update_user(user_profile: dict):
    """Create or update user in OpenWebUI database"""
    email = user_profile["mail"] or user_profile["userPrincipalName"]
    name = user_profile["displayName"]
    
    # Check if user exists
    existing_user = Users.get_user_by_email(email)
    
    if existing_user:
        # Update existing user
        updated_user = Users.update_user_by_id(
            existing_user.id,
            {"name": name, "email": email}
        )
        return updated_user
    else:
        # Create new user
        role = determine_user_role(user_profile)
        new_user = Users.insert_new_user(
            form_data={
                "name": name,
                "email": email,
                "password": secrets.token_urlsafe(32),  # Random password (not used)
                "role": role
            }
        )
        return new_user

def determine_user_role(user_profile: dict) -> str:
    """Determine user role based on Entra groups or other criteria"""
    # This would typically check group membership
    # For now, default to 'user', admins can be promoted manually
    return "user"

def create_session_token(user) -> str:
    """Create OpenWebUI compatible session token"""
    # Use OpenWebUI's existing token creation logic
    from ..webui.utils import create_token
    return create_token({"id": user.id, "email": user.email})
```

---

## Phase 3: Frontend Integration (Week 3)

### 3.1 Update Svelte Auth Store
```javascript
// src/lib/stores/user.js (modify existing)
import { writable } from 'svelte/store';
import { goto } from '$app/navigation';

export const user = writable(null);

class UserService {
  constructor() {
    this.baseUrl = '/api/v1/auths';
  }

  async getSessionUser() {
    try {
      const res = await fetch(`${this.baseUrl}/user`, {
        credentials: 'include'
      });
      
      if (res.ok) {
        const userData = await res.json();
        user.set(userData);
        return userData;
      } else {
        // Not authenticated, redirect to login
        this.login();
        return null;
      }
    } catch (error) {
      console.error('Auth error:', error);
      this.login();
      return null;
    }
  }

  login() {
    // Redirect to Entra login
    window.location.href = `${this.baseUrl}/login`;
  }

  async logout() {
    try {
      await fetch(`${this.baseUrl}/logout`, {
        method: 'POST',
        credentials: 'include'
      });
      user.set(null);
      goto('/');
    } catch (error) {
      console.error('Logout error:', error);
    }
  }
}

export const userService = new UserService();

// Auto-check authentication on app load
if (typeof window !== 'undefined') {
  userService.getSessionUser();
}
```

### 3.2 Update Main Layout
```svelte
<!-- src/routes/+layout.svelte (modify existing) -->
<script>
  import { onMount } from 'svelte';
  import { user, userService } from '$lib/stores/user.js';
  
  onMount(async () => {
    // Check authentication status
    await userService.getSessionUser();
  });
  
  $: if ($user === null && typeof window !== 'undefined') {
    // Show loading or redirect to login
    userService.login();
  }
</script>

{#if $user}
  <!-- Existing OpenWebUI layout -->
  <div class="app">
    <slot />
  </div>
{:else}
  <!-- Loading state -->
  <div class="loading">
    <p>Authenticating...</p>
  </div>
{/if}

<style>
  .loading {
    display: flex;
    justify-content: center;
    align-items: center;
    height: 100vh;
    font-family: system-ui;
  }
</style>
```

### 3.3 Remove/Replace Existing Auth Pages
```bash
# Remove existing auth pages (they'll redirect to Entra)
rm -rf src/routes/auth/signin
rm -rf src/routes/auth/signup

# Keep callback route for Entra redirect
# src/routes/auth/callback/+page.svelte can be a simple loading page
```

---

## Phase 4: Configuration and Deployment (Week 4)

### 4.1 Environment Configuration
```bash
# .env (for development)
ENTRA_TENANT_ID=your-tenant-id
ENTRA_CLIENT_ID=your-client-id
ENTRA_CLIENT_SECRET=your-client-secret
ENTRA_REDIRECT_URI=http://localhost:3000/api/v1/auths/callback

# Claude API
OPENAI_API_BASE_URL=https://api.anthropic.com
OPENAI_API_KEY=your-claude-api-key
```

### 4.2 Docker Configuration (for local testing)
```dockerfile
# Dockerfile (modify existing)
FROM node:alpine as build

WORKDIR /app
COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build

FROM python:3.11-slim as runtime

WORKDIR /app

# Install Python dependencies
COPY backend/requirements.txt .
RUN pip install -r requirements.txt

# Copy built frontend
COPY --from=build /app/build ./static
COPY backend/ ./backend/

# Environment variables
ENV ENTRA_TENANT_ID=${ENTRA_TENANT_ID}
ENV ENTRA_CLIENT_ID=${ENTRA_CLIENT_ID}
ENV ENTRA_CLIENT_SECRET=${ENTRA_CLIENT_SECRET}

EXPOSE 8080
CMD ["python", "backend/main.py"]
```

### 4.3 Vercel Deployment Configuration
```json
// vercel.json
{
  "builds": [
    {
      "src": "backend/main.py",
      "use": "@vercel/python"
    },
    {
      "src": "package.json",
      "use": "@vercel/static-build",
      "config": { "distDir": "build" }
    }
  ],
  "routes": [
    {
      "src": "/api/(.*)",
      "dest": "/backend/main.py"
    },
    {
      "src": "/(.*)",
      "dest": "/build/$1"
    }
  ],
  "env": {
    "ENTRA_TENANT_ID": "@entra-tenant-id",
    "ENTRA_CLIENT_ID": "@entra-client-id", 
    "ENTRA_CLIENT_SECRET": "@entra-client-secret"
  }
}
```

---

## Phase 5: Maintaining Updates from Upstream (Ongoing)

### 5.1 Regular Sync Strategy
```bash
# Weekly upstream sync
git fetch upstream
git checkout main
git merge upstream/main

# Resolve conflicts (mainly in auth-related files)
# Test integration
# Push to your repo
```

### 5.2 Automated Conflict Resolution
Create a script to handle common merge conflicts:
```bash
#!/bin/bash
# sync-upstream.sh

# Fetch latest upstream
git fetch upstream

# Checkout feature branch
git checkout feature/entra-authentication

# Attempt merge
git merge upstream/main

# Handle known conflicts
if [ $? -ne 0 ]; then
    echo "Merge conflicts detected. Resolving known patterns..."
    
    # Keep our auth changes
    git checkout --ours backend/apps/auth/
    git checkout --ours src/lib/stores/user.js
    git checkout --ours src/routes/+layout.svelte
    
    # Accept upstream changes for other files
    git add .
    git commit -m "Merge upstream changes, preserve Entra auth"
fi
```

---

## Benefits of This Approach

### ✅ Advantages
- **Clean Integration**: Authentication is part of the main app flow
- **Full Feature Access**: All OpenWebUI features work seamlessly  
- **Maintainable**: Clear separation of auth logic
- **Performance**: No proxy overhead
- **Vercel Compatible**: Can be deployed directly to Vercel

### ✅ License Compliance
- **Branding Preserved**: OpenWebUI branding remains intact
- **Attribution**: Original copyright notices maintained
- **BSD-3 Compliance**: All requirements met

### ✅ Security Benefits
- **Single Authentication Point**: No token passing between services
- **Direct Integration**: Fewer attack vectors
- **Enterprise Ready**: Full Entra ID integration

## Implementation Timeline

- **Week 1**: Fork, analyze, and plan integration points
- **Week 2**: Implement backend Entra authentication
- **Week 3**: Update frontend auth flow and UI
- **Week 4**: Testing, deployment config, and go-live
- **Ongoing**: Maintain upstream sync and updates

This approach gives you a production-ready, enterprise-authenticated OpenWebUI that can be deployed anywhere, including Vercel, while maintaining full compatibility with the original codebase.
