---
status: provisional
stage: alpha
latest-milestone: "v0.1"
---

# Email-Based Login for Datum Cloud

- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories](#user-stories)
  - [System Architecture](#system-architecture)
  - [Authentication Flow Changes](#authentication-flow-changes)
  - [Account Management Changes](#account-management-changes)
  - [Notes and Constraints](#notes-and-constraints)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Zitadel LoginPolicy Changes](#zitadel-loginpolicy-changes)
  - [AuthProvider Enum Extension](#authprovider-enum-extension)
  - [Actions Server: Local Provider Handling](#actions-server-local-provider-handling)
  - [Cloud Portal Account Management UI](#cloud-portal-account-management-ui)
  - [Implementation Sequence](#implementation-sequence)
- [Production Readiness Review Questionnaire](#production-readiness-review-questionnaire)
  - [Feature Enablement and Rollback](#feature-enablement-and-rollback)
  - [Monitoring Requirements](#monitoring-requirements)
  - [Dependencies](#dependencies)
  - [Troubleshooting](#troubleshooting)
- [Implementation History](#implementation-history)
- [Alternatives](#alternatives)

## Summary

Datum Cloud currently requires every user to authenticate through Google or
GitHub OAuth. This enhancement adds Zitadel's native email/password
authentication so that Datum Cloud itself becomes a first-party identity
provider. Users will be able to sign up and sign in using only an email address
and password, reset their password by email, and link or unlink OAuth providers
from their account settings page.

The work is largely a configuration change to the existing Zitadel deployment.
The login UI, password routes, and email verification routes already exist in
the `auth-ui` login v2 application. The core additions are: enabling the feature
in the production Pulumi-managed LoginPolicy, extending the `AuthProvider` enum
on the Milo `User` resource, updating the actions server to handle the Zitadel
local-provider event shape, and replacing the mock data in the cloud-portal
account security settings card with live data from the Milo Identity API.

## Motivation

### Goals

- A user with only an email address can create a Datum Cloud account without
  linking a Google or GitHub account.
- An existing OAuth user can add an email/password credential to their account
  from the security settings page.
- An existing email user can add a Google or GitHub account to their profile.
- A user can reset their password using their registered email address.
- `User.Status.LastLoginProvider` accurately reflects when a user authenticated
  with email/password.
- All changes are safe to roll back by reverting the Pulumi LoginPolicy.

### Non-Goals

- Passwordless email ("magic link") login is not in scope for this release.
- Passkey-only accounts (eliminating the password entirely) are not in scope.
- Organization-level custom email domains or SAML/SSO configuration are not in
  scope.
- Per-project Zitadel organization login policy overrides are not in scope.
- Phone/SMS second factor is not in scope.

## Proposal

### User Stories

#### Story 1: New user signs up with email

A developer wants to create a Datum Cloud account but does not want to use their
personal GitHub or Google account. They visit `cloud.datum.net`, click "Create
Account", enter their first name, last name, and work email address, and choose
a password. They receive a verification email, click the link, and are routed to
the organizations page.

#### Story 2: Existing GitHub user adds email credentials

A user previously signed up with GitHub. They want to be able to log in with
their email when they are not on a machine where GitHub OAuth is convenient.
They navigate to Account Settings → Security, see an "Email" row in the
Sign-in Methods card, and click "Manage". They are taken to the Zitadel
self-service page where they set a password. On their next login they can use
either GitHub or email/password.

#### Story 3: Email user links their Google account

A user signed up with email. After noticing the Sign-in Methods card shows
"Google — Connect your Google account", they click "Connect" and complete the
Google OAuth flow. The card now shows both methods as linked.

#### Story 4: Forgotten password

A user with an email account cannot remember their password. On the login page,
they click "Forgot password?", enter their email, and receive a reset link. They
set a new password and are redirected to the portal.

### System Architecture

The current authentication path is unchanged for OAuth users. Email/password
authentication follows the same path through the Zitadel login v2 UI but uses
Zitadel's internal user store rather than an external OAuth provider.

```
User browser
  → auth.datum.net/ui/v2/login  (auth-ui, Zitadel login v2)
  → Zitadel gRPC UserService v2 (email/password check)
  → user.human.added event fires (on first registration)
  → zitadel-actions-receiver sidecar (auth-provider-zitadel)
  → creates iam.miloapis.com/v1alpha1 User in Milo control plane
  → cloud-portal fraud check / onboarding
  → organizations dashboard
```

Account linking (adding an OAuth provider to an email account, or adding email
to an OAuth account) is handled entirely within the Zitadel login v2 UI and the
Zitadel IDP-link API. No new Milo controllers are required.

### Authentication Flow Changes

The Zitadel login v2 app (`auth-ui`) already has all the routes needed:

| Route | Purpose |
|-------|---------|
| `/register` | Email signup form (first name, last name, email) |
| `/register/password` | Password-setting step after signup |
| `/password` | Password entry on existing account login |
| `/password/set` | Set initial password |
| `/password/change` | Change current password |
| `/verify` | Email verification code entry |
| `/verify/success` | Post-verification landing page |

These routes already render when `loginSettings.allowUsernamePassword` is
`true`. The only change to the login UI itself is that the password-reset link
will become visible after `hidePasswordReset` is set to `false` in the
LoginPolicy.

### Account Management Changes

The cloud-portal security settings page
(`app/routes/account/settings/security.tsx`) shows an
`AccountSignInMethodSettingsCard`. This card currently renders hardcoded mock
data. The change is to wire it to live data from the Milo Identity API.

The Milo `UserIdentity` resource (`identity.milo.io/v1alpha1`) already exists
and is served by the auth-provider-zitadel virtual API server. It exposes the
user's linked providers with `ProviderID`, `ProviderName`, and `Username`
fields.

The updated card will:

1. Fetch `UserIdentity` resources for the authenticated user.
2. Display each linked provider with its provider name and username.
3. For unlinked providers, show a "Connect" button that initiates the Zitadel
   IDP-linking flow.
4. For the email provider, show a "Manage" button that links to the Zitadel
   self-service password-change page (`/security` route in the login app).

### Notes and Constraints

**Staging already works.** The staging environment has `enable-dev-mode: true`,
which enables email login today. The Pulumi configuration for staging does not
fully reflect this; the LoginPolicy changes bring the Pulumi source of truth in
line with the already-running configuration.

**Single Zitadel organization.** The LoginPolicy change applies to the default
Datum Cloud organization in Zitadel. All OIDC applications (cloud-portal,
staff-portal, CLI, desktop app) share this organization. Staff portal users will
also gain the ability to log in with email/password after this change.

**Registration approval.** New email signups go through the same user creation
path as OAuth signups: a `User` resource is created in Milo, the cloud-portal
fraud check runs, and the `RegistrationApproval` field determines access. This
behavior is unchanged; operators retain the ability to hold email signups in the
`Pending` state if needed.

<<[UNRESOLVED draines]>>
Should email self-registration be immediately open in production, or should it
continue to go through the existing waitlist/approval gate? If we want open
registration, the fraud-check service must be configured to auto-approve
email-signup users. If we want gated registration, no code change is needed —
the existing `UserWaitlistController` handles email notifications for pending
users.
<<[/UNRESOLVED]>>

### Risks and Mitigations

| Risk | Likelihood | Mitigation |
|------|-----------|------------|
| Email spam / fake account creation | Medium | The existing fraud gate holds new users in `Pending` state; open registration can be enabled gradually after monitoring baseline signup rates |
| LoginPolicy rollback disrupts active email sessions | Low | Rolling back the LoginPolicy does not invalidate active Zitadel sessions; existing email users can finish their current session; new logins will fail until re-enabled |
| `parseIDPUserData` change breaks Google or GitHub avatar handling | Low | The Google and GitHub detection code runs before the fallback; add or update unit tests to cover all three branches |
| Staff portal unexpectedly gains email login | Low | Test on staging (which already has email login) before promoting to production; staff portal does not show a register button so self-signup is not exposed |
| Password reset emails not delivered | Medium | Zitadel uses its own SMTP configuration; verify email delivery works in staging before enabling production |

## Design Details

### Zitadel LoginPolicy Changes

The production Pulumi program at
`infra/apps/datum-iam-system/base/zitadel/zitadel-setup/pulumi/index.ts` must
be updated to remove the `enable-dev-mode` gates on email login:

```typescript
new zitadel.LoginPolicy("datum-cloud-login-policy", {
  orgId: org.id,
  idps: [datumCloudGoogleIdp.id, datumCloudGitHubIdp.id],
  allowExternalIdp: true,
  allowRegister: true,               // was: config.enableDevMode
  userLogin: true,                   // was: config.enableDevMode
  allowUsernamePassword: true,       // add explicitly
  disableLoginWithEmail: false,      // was: !config.enableDevMode
  disableLoginWithPhone: true,       // phone remains off
  hidePasswordReset: false,          // was: true
  forceMfa: false,
  forceMfaLocalOnly: false,
  ignoreUnknownUsernames: false,
  mfaInitSkipLifetime: '720h0m0s',
  passwordCheckLifetime: '240h0m0s',
  multiFactorCheckLifetime: '12h0m0s',
  secondFactorCheckLifetime: '18h0m0s',
  externalLoginCheckLifetime: '240h0m0s',
  passwordlessType: 'PASSWORDLESS_TYPE_NOT_ALLOWED',
  multiFactors: [],
  secondFactors: [],
  allowDomainDiscovery: true,
  defaultRedirectUri: config.cloudPortal.url,
  themeMode: 'THEME_MODE_AUTO',
});
```

A `PasswordComplexityPolicy` should also be added for the Datum Cloud
organization to set minimum password requirements:

```typescript
new zitadel.PasswordComplexityPolicy("datum-cloud-password-policy", {
  orgId: org.id,
  minLength: 12,
  hasUppercase: true,
  hasLowercase: true,
  hasNumber: true,
  hasSymbol: false,
});
```

### AuthProvider Enum Extension

The `AuthProvider` type in Milo must be extended to include `email` as a valid
value. The kubebuilder enum marker and the constant block both need updating.

**File**: `milo-os/demo/.milo/pkg/apis/iam/v1alpha1/user_types.go`

```go
// AuthProvider represents an external or first-party identity provider.
// +kubebuilder:validation:Enum=github;google;email
type AuthProvider string

const (
    AuthProviderGitHub AuthProvider = "github"
    AuthProviderGoogle AuthProvider = "google"
    AuthProviderEmail  AuthProvider = "email"
)
```

After this change, `task generate` must be run to regenerate the CRD manifests
and deep copy functions.

### Actions Server: Local Provider Handling

The `idpIntentSucceededHandler` in
`auth-provider-zitadel/internal/httpactionsserver/server.go` processes
`idpintent.succeeded` events from Zitadel to update `User.Status.AvatarURL` and
`User.Status.LastLoginProvider`. The helper function `parseIDPUserData` detects
the provider from the event payload shape.

Currently, when Zitadel's local email provider fires this event, the payload
contains neither the `User.picture` field (Google) nor a top-level `avatar_url`
(GitHub). The function returns an error for unrecognized payloads.

The fix extends `parseIDPUserData` to return `AuthProviderEmail` with an empty
avatar string instead of an error for this case:

```go
func parseIDPUserData(raw []byte) (iammiloapiscomv1alpha1.AuthProvider, string, error) {
    var m map[string]interface{}
    if err := json.Unmarshal(raw, &m); err != nil {
        return "", "", err
    }

    // Google format: {"User": {"picture": "..."}}
    if user, ok := m["User"].(map[string]interface{}); ok {
        if pic, ok := user["picture"].(string); ok && pic != "" {
            return iammiloapiscomv1alpha1.AuthProviderGoogle, pic, nil
        }
    }

    // GitHub format: top-level avatar_url
    if avatar, ok := m["avatar_url"].(string); ok && avatar != "" {
        return iammiloapiscomv1alpha1.AuthProviderGitHub, avatar, nil
    }

    // Google picture can also be top-level
    if pic, ok := m["picture"].(string); ok && pic != "" {
        if strings.Contains(pic, "googleusercontent.com") {
            return iammiloapiscomv1alpha1.AuthProviderGoogle, pic, nil
        }
    }

    // Local Zitadel email provider: no external avatar.
    // Return email provider with empty avatar rather than an error.
    return iammiloapiscomv1alpha1.AuthProviderEmail, "", nil
}
```

The `idpIntentSucceededHandler` already skips the status patch if `avatarURL`
is empty when `idpProvider` is known. No additional changes are needed in the
handler itself.

### Cloud Portal Account Management UI

The `AccountSignInMethodSettingsCard` component
(`cloud-portal/app/features/account/cards/sign-in-method-card.tsx`) currently
renders a hardcoded static list. It must be wired to real data.

**Data source**: `UserIdentity` resources from the Milo Identity API
(`identity.milo.io/v1alpha1/useridentities`), accessed through the existing
cloud-portal API client. The `UserIdentityStatus` fields `ProviderName`,
`ProviderID`, and `Username` provide the information needed to render each row.

**Rendering logic**:

- For each `UserIdentity` resource found, show the provider icon, provider name,
  and linked username. If the user last authenticated with this provider, show
  "Last used" with the relative timestamp.
- For providers not found in the `UserIdentity` list (Google and GitHub are
  always shown), show a "Connect" button. Clicking it starts the Zitadel
  IDP-linking flow by redirecting to the Zitadel login app with an
  `idpIntent` parameter.
- For the email provider, if no email `UserIdentity` is found, show a "Set
  password" button. If found, show a "Manage" button that links to the login
  app's self-service security page (`/security?requestId=...`).

### Implementation Sequence

The following order minimizes risk and allows staging validation at each step.

1. **`datum-cloud/infra`** — Add `PasswordComplexityPolicy`; update
   `LoginPolicy` to enable email login in production (targets the existing
   staging-first deploy pipeline).

2. **`datum-cloud/milo-os`** (upstream) — Extend `AuthProvider` enum;
   run code generation; open PR to upstream.

3. **`datum-cloud/auth-provider-zitadel`** — Fix `parseIDPUserData` to handle
   the email provider event payload; add unit test.

4. **`datum-cloud/cloud-portal`** — Wire `AccountSignInMethodSettingsCard` to
   live `UserIdentity` data; add "Manage" / "Connect" / "Set password" actions.

Steps 1 and 2–3 are independent and can proceed in parallel. Step 4 depends on
step 2–3 being deployed so the `UserIdentity` records for email users are
created correctly.

## Production Readiness Review Questionnaire

### Feature Enablement and Rollback

#### How can this feature be enabled / disabled in a live cluster?

- **Enable**: Apply the updated Pulumi program in `datum-iam-system`
  (`pulumi up` for the `zitadel-setup` stack). This updates the Zitadel
  LoginPolicy via the Zitadel API and does not require any cluster restart.

- **Disable**: Revert the LoginPolicy changes in Pulumi and run `pulumi up`.
  Existing email users will be unable to log in with email/password until
  re-enabled; their accounts and data are unaffected. Active sessions remain
  valid until they expire.

#### Does enabling the feature change any default behavior?

Yes. The login page at `auth.datum.net` will display a username/email input
field and a "Register" link in addition to the existing Google and GitHub
buttons. Password reset links will become visible on the password entry step.

#### Can the feature be disabled once it has been enabled?

Yes. Reverting the LoginPolicy is safe. Existing email users' accounts remain in
Milo and in Zitadel; they are simply unable to log in with email/password while
the feature is disabled.

### Monitoring Requirements

#### How can an operator determine if the feature is in use?

- Query Milo for `User` resources where `status.lastLoginProvider == "email"`.
- Review Zitadel audit logs for `user.human.selfregistered` and
  `session.terminated` events where the session's authentication method was
  `PASSWORD`.
- Zitadel exposes a Prometheus metrics endpoint; the
  `zitadel_auth_requests_total` counter distinguishes authentication methods.

#### What are the reasonable SLOs for the enhancement?

Email login should meet the same latency and availability targets as the
existing OAuth login flow:

- `p99` login completion time < 3 seconds (excluding the user's email
  client latency for verification)
- Email verification delivery < 60 seconds for 99% of requests
- Password reset email delivery < 60 seconds for 99% of requests

### Dependencies

- **Zitadel** (`datum-iam-system`): Must be running and healthy. Email login
  does not add new Zitadel dependencies; it uses existing session and user
  service APIs.
- **SMTP / Email delivery**: Zitadel must be configured with a working SMTP
  provider to send verification and password-reset emails. Verify in staging
  before promoting to production.
- **Milo control plane**: The `User` resource creation path via the actions
  server must be functioning. This is the same dependency as the existing
  OAuth login flow.

### Troubleshooting

#### How does this feature react if the API server is unavailable?

If the Milo API server is unavailable, the actions server cannot create the
`User` resource when an email user self-registers. The Zitadel action will
return a 500 response, causing the registration to fail with a user-visible
error. This is the same behavior as for OAuth registrations today.

#### What are other known failure modes?

- **SMTP not configured or failing**: Email verification and password reset
  emails are not delivered. Users will not be able to complete registration or
  recover their password. Detection: `zitadel_notification_errors_total` metric;
  Zitadel logs show `SMTP error`.
- **`idpintent.succeeded` event for email provider not handled**: Before the
  `parseIDPUserData` fix is deployed, the `User.Status.LastLoginProvider` will
  not be updated for email logins. The handler logs an error at `Error` level
  with the message `unknown idp provider`. This does not block login; it only
  affects the display in the account settings card.
- **`UserIdentity` API returns empty list**: If the virtual API server in
  `auth-provider-zitadel` is unavailable, the account settings card falls back
  to an empty state. The user can still log in; they just cannot see their
  linked providers in the UI.

## Implementation History

- 2026-05-10: Discovery brief written; enhancement document created as
  provisional.

## Alternatives

### Alternative 1: Build a custom email/password service

Build a separate microservice that handles email/password credentials outside
of Zitadel. This was rejected because Zitadel already implements this
functionality, including email verification, password complexity enforcement,
rate limiting, and brute-force protection.

### Alternative 2: Use a third-party email authentication provider (Auth0, Clerk)

Add a third OAuth provider whose specialty is email/password (e.g., Clerk).
This was rejected because it adds a new external dependency and a new billing
relationship. Zitadel already provides all required capabilities.

### Alternative 3: Passkey-first, no password

Skip passwords entirely and offer passkey registration after email verification.
This is a better long-term approach but requires changes to `passwordlessType`
and additional UX work. It is tracked as a future enhancement and is not blocked
by this work.
