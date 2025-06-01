# OAuth 2.0 and OpenID Connect Integration

Delegate authentication to external identity providers (Google, Azure AD, Auth0) using standard protocols.

## Why It Matters

- Offloads password management and security (MFA, breach detection) to the provider
- Social login reduces registration friction for end users
- Enterprise SSO via Azure AD / Okta is often a hard requirement for B2B

## Key Recommendations

- **OIDC setup** (Auth0 example):

```csharp
builder.Services.AddAuthentication(options =>
{
    options.DefaultScheme = CookieAuthenticationDefaults.AuthenticationScheme;
    options.DefaultChallengeScheme = OpenIdConnectDefaults.AuthenticationScheme;
})
.AddCookie()
.AddOpenIdConnect(options =>
{
    options.Authority = "https://your-domain.auth0.com";
    options.ClientId = config["Auth0:ClientId"];
    options.ClientSecret = config["Auth0:ClientSecret"];
    options.ResponseType = OpenIdConnectResponseType.Code;
    options.Scope.Add("openid"); options.Scope.Add("profile"); options.Scope.Add("email");
    options.SaveTokens = true;
});
```

- **Azure AD**: `AddMicrosoftIdentityWebApi(config.GetSection("AzureAd"))`
- **Multiple providers**: Chain `.AddGoogle()`, `.AddMicrosoftAccount()`, `.AddGitHub()` with login route `/login/{provider}`
- **PKCE**: Required for SPAs and mobile apps -- `options.UsePkce = true`
- **JWT exchange**: Convert OIDC session to custom JWT for API consumption via `/auth/exchange` endpoint
- **Claims mapping**: Enrich external claims with local roles/permissions in `OnCreatingTicket` event
- **Account linking**: Map external provider IDs to local user records; support multiple providers per user
- **Secrets**: Store client IDs/secrets in AWS Secrets Manager, different per environment

## User Provisioning Pattern

1. User authenticates via provider
2. Check local DB for existing user by email
3. If new, create local user record + link external identity
4. Generate custom JWT with local roles/claims

## Pitfalls to Avoid

- Storing client secrets in `appsettings.json` (use secrets manager)
- Not validating issuer and audience on API tokens
- Missing CSRF protection (always use state parameter -- enabled by default in ASP.NET)
- Skipping PKCE for public clients (SPAs, mobile)
- Assuming email from provider is verified without checking `email_verified` claim
- Not handling the case where a user signs in with a different provider but same email
