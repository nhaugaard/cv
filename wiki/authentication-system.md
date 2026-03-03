# Page: Authentication System

# Authentication System

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [.env.example](.env.example)
- [.gitignore](.gitignore)
- [.vscode/settings.json](.vscode/settings.json)
- [README.md](README.md)
- [docs/changelog/index.mdx](docs/changelog/index.mdx)
- [docs/guides/setting-up-passkeys.mdx](docs/guides/setting-up-passkeys.mdx)
- [docs/spec.json](docs/spec.json)
- [knip.json](knip.json)
- [package.json](package.json)
- [pnpm-lock.yaml](pnpm-lock.yaml)
- [scripts/fonts/generate.ts](scripts/fonts/generate.ts)
- [scripts/fonts/types.ts](scripts/fonts/types.ts)
- [src/components/resume/preview.module.css](src/components/resume/preview.module.css)
- [src/components/typography/combobox.tsx](src/components/typography/combobox.tsx)
- [src/components/typography/webfontlist.json](src/components/typography/webfontlist.json)
- [src/integrations/auth/client.ts](src/integrations/auth/client.ts)
- [src/integrations/auth/config.ts](src/integrations/auth/config.ts)
- [src/routes/auth/-components/social-auth.tsx](src/routes/auth/-components/social-auth.tsx)
- [src/routes/auth/login.tsx](src/routes/auth/login.tsx)
- [src/routes/auth/register.tsx](src/routes/auth/register.tsx)
- [src/routes/builder/$resumeId/-sidebar/right/sections/typography.tsx](src/routes/builder/$resumeId/-sidebar/right/sections/typography.tsx)
- [src/routes/dashboard/settings/authentication/-components/hooks.tsx](src/routes/dashboard/settings/authentication/-components/hooks.tsx)
- [vite.config.ts](vite.config.ts)

</details>



## Purpose and Scope

This document describes the authentication system in Reactive Resume, which is built on the Better Auth framework. The system provides comprehensive user authentication and authorization capabilities including email/password authentication, social login providers, custom OAuth integration, API key authentication, two-factor authentication (2FA), and passkeys (WebAuthn). 

For information about API design and request validation, see [API Design](#2.4). For details about database schema and persistence, see [Data Layer](#2.3).

**Sources:** [src/integrations/auth/config.ts:1-238](), [.env.example:18-36]()

---

## System Architecture

The authentication system is built on Better Auth (v1.5.0-beta.18) and integrates with the application through a layered architecture consisting of server-side configuration, client-side hooks, and UI components.

### Authentication Architecture

The diagram below maps major code entities to their roles in the auth stack.

**Authentication system code entity map**

```mermaid
graph TB
    subgraph "Client Layer (src/integrations/auth/client.ts)"
        authClient["authClient\n(createAuthClient)"]
        AuthHooks["Hooks\nuseAuthAccounts()\nuseAuthProviderActions()\nuseEnabledProviders()"]
        AuthUI["UI Components\nSocialAuth\nAuthentication Settings"]
    end

    subgraph "Server Layer (src/integrations/auth/config.ts)"
        getAuthConfig["getAuthConfig()"]
        betterAuth["betterAuth()\nfrom better-auth/minimal"]
        emailAndPassword["emailAndPassword\nplugin"]
        socialProviders["socialProviders\nGoogle, GitHub"]
        genericOAuth["genericOAuth()\nplugin"]
        apiKey["apiKey()\nplugin"]
        twoFactor["twoFactor()\nplugin"]
        username["username()\nplugin"]
        openAPI["openAPI()\nplugin"]
        getTrustedOrigins["getTrustedOrigins()"]
        isCustomOAuthEnabled["isCustomOAuthProviderEnabled()"]
    end

    subgraph "Utilities"
        PasswordUtils["hashPassword()\nverifyPassword()\nsrc/utils/password"]
        EmailSvc["sendEmail()\nsrc/integrations/email/service"]
        GenerateId["generateId()\nsrc/utils/string"]
    end

    subgraph "Data Layer"
        drizzleAdapter["drizzleAdapter(db)\nbetter-auth/adapters/drizzle"]
        schema["schema\nusers, sessions,\naccounts, twoFactor, apiKeys"]
        PostgreSQL[("PostgreSQL")]
    end

    AuthUI --> AuthHooks
    AuthHooks --> authClient
    authClient --> betterAuth

    getAuthConfig --> isCustomOAuthEnabled
    getAuthConfig --> getTrustedOrigins
    getAuthConfig --> betterAuth

    betterAuth --> emailAndPassword
    betterAuth --> socialProviders
    betterAuth --> genericOAuth
    betterAuth --> apiKey
    betterAuth --> twoFactor
    betterAuth --> username
    betterAuth --> openAPI

    emailAndPassword --> PasswordUtils
    emailAndPassword --> EmailSvc
    twoFactor --> EmailSvc

    betterAuth --> drizzleAdapter
    drizzleAdapter --> schema
    schema --> PostgreSQL

    betterAuth --> GenerateId
```

Sources: [src/integrations/auth/config.ts:1-253](), [src/integrations/auth/client.ts:1-31](), [src/routes/dashboard/settings/authentication/-components/hooks.tsx:1-107]()

---

## Core Authentication Configuration

The authentication system is configured in `src/integrations/auth/config.ts` through the `betterAuth()` function, which returns a configured auth instance.

### Better Auth Initialization

`getAuthConfig()` in [src/integrations/auth/config.ts:40-251]() initializes Better Auth with the following core configuration:

| Configuration | Value | Description |
|--------------|-------|-------------|
| `appName` | `"Reactive Resume"` | Application identifier |
| `baseURL` | `env.APP_URL` | Base URL for the application |
| `secret` | `env.AUTH_SECRET` | Secret key for signing tokens |
| `database` | `drizzleAdapter(db, { schema, provider: "pg" })` | PostgreSQL adapter via Drizzle ORM |
| `telemetry.enabled` | `false` | Telemetry disabled |
| `trustedOrigins` | Result of `getTrustedOrigins()` | Allowed CORS origins |
| `advanced.useSecureCookies` | `APP_URL.startsWith("https://")` | Secure cookies when using HTTPS |
| `advanced.database.generateId` | `generateId` from `src/utils/string` | Custom ID generation |

#### Trusted Origins

`getTrustedOrigins()` [src/integrations/auth/config.ts:22-38]() always includes `APP_URL` as a trusted origin. When `APP_URL` is a localhost hostname, it also adds the sibling (`localhost` ↔ `127.0.0.1`) to the trusted set, ensuring development environments work regardless of which loopback alias is used.

Sources: [src/integrations/auth/config.ts:80-93]()

---

## Authentication Methods

### Email and Password Authentication

Email/password authentication is configured through the `emailAndPassword` plugin with the following features:

```mermaid
graph LR
    SignUp["Sign Up Flow"]
    SignIn["Sign In Flow"]
    ResetPassword["Reset Password Flow"]
    EmailVerification["Email Verification"]
    
    SignUp --> EmailVerification
    SignUp --> PasswordHash["hashPassword()"]
    SignIn --> PasswordVerify["verifyPassword()"]
    ResetPassword --> SendEmail["sendResetPassword()"]
    EmailVerification --> SendEmail2["sendVerificationEmail()"]
    
    PasswordHash --> bcrypt["bcrypt library"]
    PasswordVerify --> bcrypt
    
    SendEmail --> SMTPService["SMTP/Email Service"]
    SendEmail2 --> SMTPService
```

**Configuration:**

| Setting | Value | Description |
|---------|-------|-------------|
| `enabled` | `!env.FLAG_DISABLE_EMAIL_AUTH` | Can be disabled via flag |
| `autoSignIn` | `true` | Auto sign-in after registration |
| `minPasswordLength` | `6` | Minimum password length |
| `maxPasswordLength` | `64` | Maximum password length |
| `requireEmailVerification` | `false` | Email verification optional |
| `disableSignUp` | `env.FLAG_DISABLE_SIGNUPS \|\| env.FLAG_DISABLE_EMAIL_AUTH` | Can disable signups |
| `password.hash` | `hashPassword()` | Custom bcrypt-based hashing |
| `password.verify` | `verifyPassword()` | Custom password verification |

The password reset and email verification flows use the `sendEmail()` service to send notification emails to users.

**Sources:** [src/integrations/auth/config.ts:78-108](), [.env.example:66-67]()

---

### Social Authentication Providers

Reactive Resume supports Google and GitHub as built-in social authentication providers, configured through the `socialProviders` object.

#### Google OAuth

```typescript
google: {
  enabled: !!env.GOOGLE_CLIENT_ID && !!env.GOOGLE_CLIENT_SECRET,
  disableSignUp: env.FLAG_DISABLE_SIGNUPS,
  clientId: env.GOOGLE_CLIENT_ID,
  clientSecret: env.GOOGLE_CLIENT_SECRET,
  mapProfileToUser: async (profile) => {...}
}
```

The `mapProfileToUser` function extracts:
- `name` from profile or email prefix
- `email` from profile
- `image` from profile picture
- `username` from email prefix
- `emailVerified` set to `true`

**Sources:** [src/integrations/auth/config.ts:137-156]()

#### GitHub OAuth

```typescript
github: {
  enabled: !!env.GITHUB_CLIENT_ID && !!env.GITHUB_CLIENT_SECRET,
  disableSignUp: env.FLAG_DISABLE_SIGNUPS,
  clientId: env.GITHUB_CLIENT_ID,
  clientSecret: env.GITHUB_CLIENT_SECRET,
  mapProfileToUser: async (profile) => {...}
}
```

GitHub authentication includes special handling for legacy accounts that may have stored usernames in different formats. The system queries the database to check for existing accounts with matching `username` or `displayUsername` fields and preserves them during migration.

**Sources:** [src/integrations/auth/config.ts:158-209]()

#### Social Auth UI Flow

```mermaid
sequenceDiagram
    participant User
    participant SocialAuthComponent as "SocialAuth Component"
    participant AuthClient as "authClient"
    participant Provider as "OAuth Provider"
    participant Server as "Better Auth Server"
    
    User->>SocialAuthComponent: Click provider button
    SocialAuthComponent->>AuthClient: authClient.signIn.social()
    AuthClient->>Provider: Redirect to OAuth
    Provider->>User: Show consent screen
    User->>Provider: Approve
    Provider->>Server: OAuth callback
    Server->>Server: mapProfileToUser()
    Server->>Server: Create/update user
    Server->>SocialAuthComponent: Redirect to callbackURL
    SocialAuthComponent->>User: Navigate to dashboard
```

**Sources:** [src/routes/auth/-components/social-auth.tsx:1-99]()

---

### Custom OAuth Provider

For organizations with their own OAuth providers, Reactive Resume supports a generic OAuth integration through the `genericOAuth` plugin.

#### Configuration

Custom OAuth is enabled when the following environment variables are set:
- `OAUTH_CLIENT_ID` and `OAUTH_CLIENT_SECRET` (required)
- Either `OAUTH_DISCOVERY_URL` (OpenID Connect discovery) or manual URLs:
  - `OAUTH_AUTHORIZATION_URL`
  - `OAUTH_TOKEN_URL`
  - `OAUTH_USER_INFO_URL`

The `isCustomOAuthProviderEnabled()` function validates this configuration:

```typescript
function isCustomOAuthProviderEnabled() {
  const hasDiscovery = Boolean(env.OAUTH_DISCOVERY_URL);
  const hasManual = Boolean(env.OAUTH_AUTHORIZATION_URL) && 
                    Boolean(env.OAUTH_TOKEN_URL) && 
                    Boolean(env.OAUTH_USER_INFO_URL);
  
  return Boolean(env.OAUTH_CLIENT_ID) && 
         Boolean(env.OAUTH_CLIENT_SECRET) && 
         (hasDiscovery || hasManual);
}
```

#### Profile Mapping

The custom OAuth provider uses `mapProfileToUser` to extract user information:

| Field | Source | Fallback |
|-------|--------|----------|
| `email` | `profile.email` | Required, throws error if missing |
| `name` | `profile.name` | `profile.preferred_username` or email prefix |
| `username` | `profile.preferred_username` | email prefix |
| `image` | `profile.image` | `profile.picture` or `profile.avatar_url` |
| `emailVerified` | - | Always `true` |

**Sources:** [src/integrations/auth/config.ts:15-61](), [.env.example:30-36]()

---

### API Key Authentication

API keys provide programmatic access to the application with rate limiting.

#### Configuration

```typescript
apiKey({
  enableSessionForAPIKeys: true,
  rateLimit: {
    enabled: true,
    timeWindow: 1000 * 60 * 60 * 24, // 1 day
    maxRequests: 500, // 500 requests per day
  },
})
```

API keys are rate-limited to 500 requests per 24-hour window. Each API key can be associated with a session for maintaining state across requests.

**Sources:** [src/integrations/auth/config.ts:214-221]()

---

### Two-Factor Authentication (2FA)

Two-factor authentication is provided through the `twoFactor` plugin using Time-based One-Time Passwords (TOTP).

#### Server Configuration

```typescript
twoFactor({ issuer: "Reactive Resume" })
```

#### Client Configuration

The client-side 2FA plugin includes a redirect handler:

```typescript
twoFactorClient({
  onTwoFactorRedirect() {
    if (typeof window !== "undefined") {
      window.location.href = "/auth/verify-2fa";
    }
  },
})
```

When a user with 2FA enabled attempts to sign in, they are redirected to `/auth/verify-2fa` to enter their TOTP code.

**Sources:** [src/integrations/auth/config.ts:230](), [src/integrations/auth/client.ts:16-23]()

---

### Passkeys (WebAuthn)

Passkey support was introduced in v5.0.0 and subsequently **removed in v5.0.8** due to an upstream issue with the Better Auth provider (see [docs/guides/setting-up-passkeys.mdx]() for historical context). The `passkey` plugin is not present in the current server config [src/integrations/auth/config.ts:229-249]() or client config [src/integrations/auth/client.ts:1-31]().

Sources: [docs/changelog/index.mdx:42-45]()

---

### OpenAPI Plugin

The `openAPI()` plugin [src/integrations/auth/config.ts:231]() exposes machine-readable API documentation for the auth endpoints. This is used by the `docs/spec.json` OpenAPI specification and makes auth endpoints discoverable alongside the ORPC API. See page [2.4]() for more on the API layer.

---

### Username System

The `username` plugin provides username-based identification with normalization and validation.

#### Configuration

| Setting | Value | Description |
|---------|-------|-------------|
| `minUsernameLength` | `3` | Minimum username length |
| `maxUsernameLength` | `64` | Maximum username length |
| `usernameNormalization` | `toUsername()` | Converts to lowercase alphanumeric with `._-` |
| `displayUsernameNormalization` | `toUsername()` | Same normalization for display usernames |
| `usernameValidator` | `/^[a-z0-9._-]+$/` | Regex validation pattern |
| `validationOrder` | Post-normalization | Validates after normalization |

The system maintains two username fields:
- `username`: Normalized, unique identifier
- `displayUsername`: Original username for display purposes

**Sources:** [src/integrations/auth/config.ts:222-229]()

---

## Session Management

### Cookie-based Sessions

Sessions are managed using secure cookies handled by Better Auth's core session layer. No additional session plugin is required.

**Session cookie flow**

```mermaid
graph LR
    Browser["Client Browser"]
    Cookie["Session Cookie"]
    Server["betterAuth server handler"]
    DB[("session table\n(PostgreSQL)")]

    Browser -->|"HTTP Request + Cookie"| Server
    Server -->|"Validate session token"| DB
    DB -->|"Session + user record"| Server
    Server -->|"Response"| Browser
```

Cookie settings are derived from:
- **Secure**: `true` when `env.APP_URL` starts with `"https://"` (`advanced.useSecureCookies`)
- **HttpOnly**: `true` by Better Auth default (not accessible via JavaScript)
- **SameSite**: Better Auth defaults

Sources: [src/integrations/auth/config.ts:90-93]()

---

## Account Linking

Account linking allows users to connect multiple authentication providers to a single account.

### Configuration

```typescript
account: {
  accountLinking: {
    enabled: true,
    trustedProviders: ["google", "github"],
  },
}
```

### Account Management Hooks

The `useAuthAccounts()` hook provides access to linked accounts:

```typescript
const { accounts, hasAccount, getAccountByProviderId } = useAuthAccounts();
```

The `useAuthProviderActions()` hook provides link/unlink functionality:

```typescript
const { link, unlink } = useAuthProviderActions();

// Link a new provider
await link("google");

// Unlink an existing provider
await unlink("github", accountId);
```

**Workflow:**

```mermaid
sequenceDiagram
    participant User
    participant UI as "Settings UI"
    participant Hook as "useAuthProviderActions"
    participant AuthClient as "authClient"
    participant Server as "Better Auth"
    
    User->>UI: Click "Link Google"
    UI->>Hook: link("google")
    Hook->>AuthClient: authClient.linkSocial()
    AuthClient->>Server: Link account request
    Server->>Server: Validate & create account entry
    Server->>AuthClient: Success
    AuthClient->>Hook: Refresh accounts
    Hook->>UI: Update UI
```

**Sources:** [src/integrations/auth/config.ts:129-134](), [src/routes/dashboard/settings/authentication/-components/hooks.tsx:39-96]()

---

## Client Integration

### Auth Client

The client-side authentication is initialized through `createAuthClient()` with plugins:

```typescript
export const authClient = createAuthClient({
  plugins: [
    apiKeyClient(),
    usernameClient(),
    twoFactorClient({ onTwoFactorRedirect() {...} }),
    genericOAuthClient(),
    inferAdditionalFields<typeof auth>(),
  ],
});
```

**Plugin Purposes:**

| Plugin | Purpose |
|--------|---------|
| `apiKeyClient` | Client-side API key authentication |
| `usernameClient` | Username-based operations |
| `twoFactorClient` | 2FA verification flow |
| `genericOAuthClient` | Custom OAuth provider support |
| `inferAdditionalFields` | Type inference for custom user fields |

**Sources:** [src/integrations/auth/client.ts:11-28]()

---

### Authentication Hooks

#### useEnabledProviders

Fetches the list of enabled authentication providers from the server:

```typescript
const { enabledProviders } = useEnabledProviders();
// Returns: string[] e.g., ["credential", "google", "github", "custom"]
```

This hook queries the `orpc.auth.providers.list` endpoint to determine which providers are available based on environment configuration.

**Sources:** [src/routes/dashboard/settings/authentication/-components/hooks.tsx:102-107]()

#### Provider Utilities

Helper functions for displaying provider information:

```typescript
// Get display name
getProviderName("google"); // Returns: "Google"

// Get provider icon
getProviderIcon("github"); // Returns: <GithubLogoIcon />
```

Supported providers and their display information:

| Provider ID | Display Name | Icon |
|------------|--------------|------|
| `credential` | "Password" | `PasswordIcon` |
| `google` | "Google" | `GoogleLogoIcon` |
| `github` | "GitHub" | `GithubLogoIcon` |
| `custom` | "Custom OAuth" | `VaultIcon` |

**Sources:** [src/routes/dashboard/settings/authentication/-components/hooks.tsx:15-34]()

---

## Environment Configuration

### Required Variables

| Variable | Purpose | Example |
|----------|---------|---------|
| `AUTH_SECRET` | Secret key for token signing | Generated via `openssl rand -hex 32` |
| `DATABASE_URL` | PostgreSQL connection string | `postgresql://user:pass@host:5432/db` |
| `APP_URL` | Application base URL | `http://localhost:3000` |

### Optional Social Providers

| Provider | Variables | Notes |
|----------|-----------|-------|
| Google | `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET` | OAuth 2.0 credentials from Google Cloud Console |
| GitHub | `GITHUB_CLIENT_ID`, `GITHUB_CLIENT_SECRET` | OAuth App credentials from GitHub |
| Custom | `OAUTH_CLIENT_ID`, `OAUTH_CLIENT_SECRET`, `OAUTH_DISCOVERY_URL` or manual URLs | Generic OpenID Connect / OAuth 2.0 |

### Email Configuration (Optional)

If SMTP variables are not set, emails are logged to the console instead of being sent:

| Variable | Purpose |
|----------|---------|
| `SMTP_HOST` | SMTP server hostname |
| `SMTP_PORT` | SMTP server port |
| `SMTP_USER` | SMTP username |
| `SMTP_PASS` | SMTP password |
| `SMTP_FROM` | Sender email address |
| `SMTP_SECURE` | Use TLS/SSL (`true`/`false`) |

### Feature Flags

| Flag | Default | Purpose |
|------|---------|---------|
| `FLAG_DISABLE_SIGNUPS` | `false` | Disable new user registration |
| `FLAG_DISABLE_EMAIL_AUTH` | `false` | Disable email/password authentication |

**Sources:** [.env.example:18-67]()

---

## Database Schema

Better Auth uses `drizzleAdapter(db, { schema, provider: "pg" })` to persist authentication data in PostgreSQL. The Drizzle schema is shared with the rest of the application (see page [2.3]() for the full schema). The following tables are owned or shared by the auth system:

### Core Tables

| Table | Purpose | Key Fields |
|-------|---------|------------|
| `user` | User accounts | `id`, `email`, `username`, `displayUsername`, `emailVerified`, `image` |
| `session` | Active sessions | `id`, `userId`, `expiresAt`, `token` |
| `account` | Linked provider accounts | `id`, `userId`, `providerId`, `accountId` |
| `verification` | Email verification tokens | `id`, `identifier`, `value`, `expiresAt` |

### Security Tables

| Table | Purpose |
|-------|---------|
| `twoFactor` | 2FA TOTP secrets and backup codes |
| `apiKey` | API key tokens and metadata |

> Note: The `passkey` table was relevant when passkey support was active but is no longer used since passkeys were removed in v5.0.8.

**User Additional Fields:**

The `user.additionalFields` configuration adds:
- `username` (string, required): Normalized unique username
- `displayUsername` (string): Original username for display

Sources: [src/integrations/auth/config.ts:127-143]()

---

## Summary

The authentication system provides:

1. **Multiple Authentication Methods**: Email/password, Google, GitHub, custom OAuth, API keys, passkeys, and 2FA
2. **Account Linking**: Users can link multiple providers to one account
3. **Session Management**: Secure cookie-based sessions with PostgreSQL persistence
4. **Customization**: Environment-based configuration with feature flags
5. **Email Integration**: Password reset and email verification flows
6. **Rate Limiting**: API key requests limited to 500 per day
7. **Username System**: Normalized usernames with validation
8. **Client Hooks**: React hooks for authentication state management

**Sources:** [src/integrations/auth/config.ts:1-238](), [src/integrations/auth/client.ts:1-31]()

---

