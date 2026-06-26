---
status: provisional
stage: alpha
latest-milestone: "v0.x"
---

<!-- omit from toc -->

# Unified organizations

Related: [milo-os/milo#636](https://github.com/milo-os/milo/issues/636)

- [Summary](#summary)
- [What is an organization?](#what-is-an-organization)
  - [Definition](#definition)
  - [How organizations relate to the platform](#how-organizations-relate-to-the-platform)
  - [Organization lifecycle](#organization-lifecycle)
- [User experience](#user-experience)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Current State](#current-state)
- [Proposal](#proposal)
  - [API: drop type, add contactInfo](#1-api-drop-type-add-contactinfo)
  - [Organization naming](#organization-naming-metadataname)
  - [Implementation examples](#implementation-examples)
  - [Quota](#2-quota-one-grant-policy)
  - [Signup and onboarding](#3-signup-and-onboarding)
  - [Portal UX](#4-portal-ux)
  - [Cross-repo work](#5-cross-repo-work)
  - [Migration plan](#6-migration-plan)
- [Open Questions](#open-questions)
- [Risks](#risks)
- [Implementation History](#implementation-history)

## Summary

An organization is how a customer teams up on Datum Cloud. It groups people,
holds their projects, and ties together access, quota, and billing. After this
enhancement, every organization works the same way: one kind of org, no Personal
or Standard split, no second-class workspace created at signup.

Users get a default organization when they sign up. They complete a short
onboarding flow (contact details, then a billing account with a valid payment
method) before they can use the rest of the platform. The org they start with is
the org they keep. A solo developer who later incorporates updates contact
details instead of creating a new workspace. Team invites, billing settings, and
editable display names are available from the start.

Individual vs business is contact information on the org, not a permanent label
frozen at creation time.

## What is an organization?

### Definition

An organization is the top-level tenant in Datum Cloud. It is a group of people
working together on projects and consuming resources from the platform. Most
day-to-day work happens inside projects; the organization is where membership,
access, quota, and billing come together.

Each organization has:

- Members with roles scoped to that org.
- Projects that hold the workloads and resources users actually operate.
- Contact information (`spec.contactInfo`) for who the org is and how to reach
  them. Required: email and name. Optional: address and business name.
- One or more billing accounts, each with its own contact, currency, and payment
  method. Billing contact stays separate from org contact.
- Quota grants (for example, a default of 10 projects per org).

There is no org "type." Whether someone is an individual or a business is a
property of their contact details, and they can update those at any time.

### How organizations relate to the platform

```mermaid
flowchart TD
  user[User] -- membership + role --> org[Organization]
  org -- contains --> project[Projects]
  project -- contains --> workloads[Workloads / resources]
  org -- has --> contact[Org contact info]
  org -- granted --> quota[Quota grants]
  org -- has 0..n --> billing[Billing accounts]
  billing -- has --> payment[Payment method]
```

| Relationship | Cardinality | Notes |
|--------------|-------------|-------|
| User ↔ Organization | many-to-many via membership | A user can belong to many orgs with different roles in each. |
| Organization → Projects | one-to-many | Projects are where users do most of their work. Resources live in projects. |
| Organization → Contact info | one | Tenancy-level identity. Separate from any billing contact. |
| Organization → Quota | one-to-many grants | Allowances attach to the org, not the user. |
| Organization → Billing accounts | one-to-many | An org can have multiple billing accounts. There is no single "the billing address" to fall back on, which is why org contact carries its own optional address. |

Access and quota both resolve through the organization. That is why a user can
collaborate across several orgs with different roles and limits in each.

### Organization lifecycle

1. **Created** when a user signs up (default org) or when they create an
   additional org in the portal.
2. **Identified** by a stable, opaque system name and an editable display name.
   Users do not pick resource slugs.
3. **Onboarded** through contact details and a billing account with a valid
   payment method. Full platform access opens once onboarding is complete.
4. **Operated** with members, projects, and billing added over time.

Completion is recorded with a one-time annotation on the org
(`resourcemanager.miloapis.com/onboarding-completed-at`), stamped by a
controller when contact info is complete and a billing account has a usable
payment method. See [Signup and onboarding](#3-signup-and-onboarding) for
implementation detail.

## User experience

**Signup.** A user signs up and gets a default organization automatically, with
a neutral display name they can change later.

**Onboarding.** Before they can use the platform, they provide org contact info
(email and name required; address and business name optional) and set up a
billing account with a valid payment method. The portal walks them through
contact first, then billing.

**Day to day.** Every org has the same capabilities: invite teammates, manage
billing, rename the org, create projects up to the org quota. There is no
Personal badge, no hidden settings, no cap that forces a user to recreate their
workspace to grow.

**Additional orgs.** Users can create more organizations from the portal. They
provide a display name; the system assigns the internal identifier.

**What goes away.** The Personal/Standard split, locked display names on
auto-created orgs, the 2-project Personal cap, and the pattern where a user
must abandon their signup workspace and start a new org to invite a teammate or
run a serious workload.

## Motivation

Today we maintain two organization types, Personal and Standard, even though
they are the same resource underneath. The type drives quota, portal UX, and
provisioning rules, and it stands in loosely for "individual vs business" even
though that information belongs on contact and billing records.

That causes a few user-visible problems:

- A user who outgrows a Personal workspace cannot convert it. They create a new
  Standard org and leave projects behind.
- Signup creates a constrained Personal org before the user has chosen how they
  want to work.
- The same enum branches behavior across controllers, quota policies, admission
  rules, and both portals for what is essentially one kind of object.

Billing identity already lives on `BillingAccount.spec.contactInfo`. Org type
was never the billing record; it was a product label that could not be changed.
This enhancement removes the label and puts identity where it belongs.

### Goals

- Remove `Organization.spec.type` entirely. An org is an org.
- Add `Organization.spec.contactInfo` with the same general shape as
  `BillingContactInfo` (name, businessName, email, address). `email` and `name`
  are required; `address` and `businessName` are optional and can be filled in
  during onboarding or later in the portal. Billing contact stays on
  `BillingAccount`; org contact is the tenancy-level identity and communication
  record.
- Use opaque Kubernetes names for `metadata.name`. No user names, business
  names, or semantic prefixes like `personal-org-`. Human-readable names live in
  `metadata.annotations.kubernetes.io/display-name` and `spec.contactInfo`.
- Migrate all existing orgs to Standard-equivalent behavior: quota grants,
  rename, invites, settings pages currently hidden for Personal.
- Keep auto-creating a default org on signup (datum controller), but gate full
  platform access until onboarding is complete: org contact details and a
  billing account with a valid payment method.
- Redirect users with no org membership to onboarding (edge cases, failed
  provisioning, legacy accounts).

### Non-Goals

- Final CEL field-level validation rules (defer to implementation PRs).
- Stripe, Avalara, or tax-engine integration details.
- Syncing org contact and billing contact. They may share shape and the portal
  may pre-fill a billing form from org data, but there is no automatic copy,
  shared CRD, or controller that keeps them in sync.
- Plan-tier member limits (discussed in #636; out of scope).

## Current State

| Area                | What branches on org type today                                                                                                                                                                                      |
| ------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **milo**            | `organization_types.go`: required immutable enum; CRD + API docs; membership status caches `status.organization.type`; telemetry metrics emit `spec.type`                                                            |
| **datum**           | `personal_organization_controller.go`: auto-creates Personal org + membership + default project; `organization-update-policy.yaml`: blocks Personal display name changes; separate grant policies (2 vs 10 projects) |
| **cloud-portal**    | Personal badge/sort-first; hides team + billing settings nav; blocks rename UI; hard-coded 2 vs 10 project limits                                                                                                    |
| **staff-portal**    | Org list type filter and membership columns show Personal/Standard                                                                                                                                                   |
| **billing**         | `BillingAccount.spec.contactInfo` unchanged and independent; optional portal pre-fill only when user creates a billing account manually                                                                              |
| **zitadel / infra** | No org-type coupling found; infra changes are deploy-order and CRD schema rollout only                                                                                                                               |

Member invites on Personal orgs are enforced in the **portal** (team nav hidden),
not in the Milo API. After this change, invites follow normal RBAC for all
orgs.

What `spec.type` controls today in code:

- **Quota tier.** datum `GrantCreationPolicy` triggers: Personal = 2 projects,
  Standard = 10.
- **Lifecycle / provisioning.** datum auto-creates only Personal orgs on User
  reconcile; Standard orgs are user-created.
- **Validation.** Admission policy blocks display-name changes on Personal orgs.
- **Portal UX.** cloud-portal badge, sort-first, hidden team/billing nav,
  blocked rename UI, hard-coded project-limit copy; staff-portal type filter/column.
- **Observability.** Membership status caches `status.organization.type`;
  metrics emit `spec.type`.

## Proposal

### 1. API: drop type, add contactInfo

Remove `OrganizationSpec.type` and its CEL immutability rule. The spec holds
contact info (and any fields that remain).

Reuse or extract shared Go types from `billing/api/v1alpha1` (`BillingContactInfo`,
`BillingAddress`) to avoid drift. Use distinct type names (e.g.
`OrganizationContactInfo`, `PostalAddress`) so org and billing APIs can evolve
independently.

Remove the Type print column from Organization and OrganizationMembership CRDs.
Drop `status.organization.type` from membership status (keep displayName cache).

#### Organization naming (`metadata.name`)

**Today:** auto-created orgs use `personal-org-{hash(userUID)}`; user-created
orgs in cloud-portal let the user pick a slug (often derived from company name).

**Proposed:** `metadata.name` is an opaque, system-assigned identifier. It must
not embed the user's name, business name, or product semantics (`personal`,
`acme`, etc.).

| Field                                             | Purpose                                                                                              |
| ------------------------------------------------- | ---------------------------------------------------------------------------------------------------- |
| `metadata.name`                                   | Stable API identity, namespace suffix (`organization-{name}`), URLs `/org/{name}`. Server-generated. |
| `metadata.annotations.kubernetes.io/display-name` | Human-readable label shown in UI. Editable.                                                          |
| `spec.contactInfo.businessName`                   | Legal/trading name for contact/compliance. Not used as the k8s name.                                 |

**Format (decided):** `org-{12 lowercase hex chars}` derived deterministically
from `sha256(user.UID)` for auto-created default orgs (idempotent reconcile).
User-created orgs via the portal use the same format with `crypto/rand` (or the
same hash helper if tied to the acting user). Example: `org-7f3a9c2b1d4e`.

**User-created orgs (portal):** stop asking for a resource slug. Collect display
name (and contact in onboarding); server generates `metadata.name` before POST.

**Migration:** do not rename existing Organization CRs (`metadata.name` is
immutable). Legacy names like `personal-org-abc123` and user-chosen slugs remain
until we run a multi-step rename migration (out of scope for v1). New orgs only
follow the new rule.

**Default org controller cutover:** changing the name formula alone would create
a second org for every existing user (today `personal-org-{hash}`, tomorrow
`org-{hash}` are different objects). The controller must first look up the
user's existing default org (via existing membership or `controllerRef`) and
reconcile it as-is. Only create `org-{suffix}` when no default org exists yet
(new signups after deploy).

Proposed shape:

```yaml
apiVersion: resourcemanager.miloapis.com/v1alpha1
kind: Organization
metadata:
  name: org-7f3a9c2b1d4e
  annotations:
    kubernetes.io/display-name: Acme Corp
spec:
  contactInfo:
    email: admin@acme.com
    name: Jane Doe
    businessName: Acme Corp Ltd
    address:
      country: GB
      line1: 1 Example Street
      city: London
      region: Greater London
      postalCode: EC1A 1BB
```

### Implementation examples

These snippets show the intended shape across repos. Reference implementations
for the enhancement; not shipped code yet.

#### Shared types (milo)

Org contact and billing contact share field shapes but are separate resources
with separate lifecycles. Do not auto-sync them in controllers. The portal may
suggest billing values when creating a first `BillingAccount`; it must not write
billing contact back to the org or vice versa.

**Why org contact has its own (optional) address, not just billing.** An org
can have **multiple** `BillingAccount`s, each with its own `contactInfo` and
address (e.g. different departments, currencies, or cost centers). There is no
single "the billing address" to fall back on, and an org may have zero billing
accounts at signup. So when an organization records an address, it belongs on
the org-level contact, the canonical tenancy-level identity and communication
record, independent of how many billing accounts exist or which one is active.
The org address is optional (added during onboarding or later); billing
addresses remain per-`BillingAccount` for invoicing and tax.

```go
// pkg/apis/resourcemanager/v1alpha1/organization_types.go (proposed)

type OrganizationSpec struct {
	// ContactInfo is the tenancy-level contact for this organization.
	// Independent of BillingAccount.spec.contactInfo.
	//
	// +kubebuilder:validation:Optional
	ContactInfo *OrganizationContactInfo `json:"contactInfo,omitempty"`
}

type OrganizationContactInfo struct {
	// +kubebuilder:validation:Required
	Email string `json:"email"`

	// +kubebuilder:validation:Required
	Name string `json:"name"`

	// BusinessName is optional: individuals may have no legal/trading name.
	//
	// +kubebuilder:validation:Optional
	BusinessName string `json:"businessName,omitempty"`

	// Address is optional: it can be supplied during onboarding or later in the
	// portal. When present, Country, Line1, and City are required.
	//
	// +kubebuilder:validation:Optional
	Address *PostalAddress `json:"address,omitempty"`
}

type PostalAddress struct {
	// +kubebuilder:validation:Required
	Country string `json:"country"`
	// +kubebuilder:validation:Required
	Line1 string `json:"line1"`
	Line2 string `json:"line2,omitempty"`
	// +kubebuilder:validation:Required
	City       string `json:"city"`
	Region     string `json:"region,omitempty"`
	PostalCode string `json:"postalCode,omitempty"`
}
```

Billing stays unchanged:

```yaml
# billing.miloapis.com/v1alpha1: separate namespace, separate object
apiVersion: billing.miloapis.com/v1alpha1
kind: BillingAccount
metadata:
  name: acme-billing
  namespace: organization-org-7f3a9c2b1d4e
spec:
  currencyCode: USD
  contactInfo:
    email: finance@acme.com
    name: Accounts Payable
    businessName: Acme Corp Ltd
    address:
      country: GB
      line1: 10 Billing Lane
```

#### Example Organization after migration

```yaml
apiVersion: resourcemanager.miloapis.com/v1alpha1
kind: Organization
metadata:
  name: personal-org-a1b2c3
  annotations:
    kubernetes.io/display-name: My organization
spec:
  contactInfo:
    email: jane@example.com
    name: Jane Doe
    address:
      country: US
      line1: 123 Main St
      city: Austin
      region: TX
      postalCode: "78701"
```

Legacy slug in `metadata.name` is not renamed in v1 migration.

#### datum: default org controller

Today the controller uses `personal-org-{hash(userUID)}` and sets
`Spec.Type = "Personal"`. After the change:

```go
// datum/internal/controller/resourcemanager/default_organization_controller.go

func orgNameForUser(uid types.UID) string {
	sum := sha256.Sum256([]byte(uid))
	return fmt.Sprintf("org-%s", hex.EncodeToString(sum[:])[:12])
}

// Reconcile must NOT blindly use orgNameForUser on every run; look up the
// user's existing default org first (membership or ownerRef). Only assign
// orgNameForUser when creating a net-new default org for a new user.

_, err := controllerutil.CreateOrUpdate(ctx, r.Client, defaultOrg, func() error {
	if defaultOrg.Name == "" {
		defaultOrg.Name = orgNameForUser(user.UID)
	}

	displayName := "My organization"
	if g := strings.TrimSpace(user.Spec.GivenName); g != "" {
		displayName = fmt.Sprintf("%s's organization", g)
	}
	metav1.SetMetaDataAnnotation(&defaultOrg.ObjectMeta, "kubernetes.io/display-name", displayName)

	if defaultOrg.Spec.ContactInfo == nil {
		defaultOrg.Spec.ContactInfo = &resourcemanagerv1alpha1.OrganizationContactInfo{
			Email: user.Spec.Email,
			Name:  strings.TrimSpace(user.Spec.GivenName + " " + user.Spec.FamilyName),
		}
	}
	return controllerutil.SetControllerReference(user, defaultOrg, r.Scheme)
})
```

#### datum: single quota grant policy

Replace `personal-org-grant-policy.yaml` and `standard-org-grant-policy.yaml`:

```yaml
apiVersion: quota.miloapis.com/v1alpha1
kind: GrantCreationPolicy
metadata:
  name: organization-project-quota-policy
spec:
  trigger:
    resource:
      apiVersion: resourcemanager.miloapis.com/v1alpha1
      kind: Organization
  target:
    resourceGrantTemplate:
      metadata:
        name: default-project-quota
        namespace: "organization-{{ trigger.metadata.name }}"
      spec:
        consumerRef:
          apiGroup: resourcemanager.miloapis.com
          kind: Organization
          name: "{{ trigger.metadata.name }}"
        allowances:
          - resourceType: resourcemanager.miloapis.com/projects
            buckets:
              - amount: 10
```

#### datum: delete Personal-only admission policy

Remove `disallow-personal-org-name-change` entirely. Display names become
editable for all orgs.

#### milo: membership status cache (drop type)

```go
// internal/controllers/resourcemanager/organization_membership_controller.go

organizationMembership.Status.Organization = OrganizationMembershipOrganizationStatus{
	DisplayName: displayName,
}
```

#### milo: optional contactInfo validation webhook

```go
func contactInfoComplete(c *resourcemanagerv1alpha1.OrganizationContactInfo) bool {
	// Onboarding requires email + name only. Address is optional and can be
	// added later in the portal.
	if c == nil || c.Email == "" || c.Name == "" {
		return false
	}
	return true
}

// Used by portal gating logic; webhook only validates field formats on write,
// not onboarding completeness (that stays in cloud-portal middleware).
```

#### cloud-portal: onboarding gate middleware

Extend the pattern in `fraud-status.middleware.ts`. Run after fraud checks,
before org routes:

```typescript
// app/utils/middlewares/org-onboarding.middleware.ts

const ONBOARDING_PATHS = new Set([
  paths.onboarding.completeProfile,
  paths.onboarding.organizationContact,
  paths.onboarding.billing,
  paths.auth.logOut,
]);

const ONBOARDING_COMPLETED_AT =
  "resourcemanager.miloapis.com/onboarding-completed-at";

export async function orgOnboardingMiddleware(
  ctx: MiddlewareContext,
  next: NextFunction,
) {
  const pathname = new URL(ctx.request.url).pathname;
  if (ONBOARDING_PATHS.has(pathname)) {
    return next();
  }

  const memberships = await createOrganizationService().listForUser(
    session.sub,
  );
  if (memberships.length === 0) {
    return redirect(paths.onboarding.organizationContact);
  }

  const defaultOrg = memberships[0].organization;

  // Fast path: controller has confirmed contact + valid billing.
  if (defaultOrg.metadata?.annotations?.[ONBOARDING_COMPLETED_AT]) {
    return next();
  }

  // Otherwise route to the first incomplete step.
  if (!isOrgContactComplete(defaultOrg.spec?.contactInfo)) {
    return redirect(paths.onboarding.organizationContact);
  }
  return redirect(paths.onboarding.billing);
}

function isOrgContactComplete(contact?: OrganizationContactInfo): boolean {
  return Boolean(contact?.email && contact?.name);
}
```

#### cloud-portal: onboarding writes org contact only

```typescript
await organizationService.update(orgId, {
  spec: {
    contactInfo: {
      email: form.email,
      name: form.name,
      businessName: form.businessName,
      address: {
        country: form.country,
        line1: form.line1,
        city: form.city,
        postalCode: form.postalCode,
      },
    },
  },
});
navigate(paths.account.organizations.root);
```

#### cloud-portal: billing account create (separate form, optional pre-fill)

Pre-fill from org contact is client-side only. Submit creates a distinct object:

```typescript
const defaults = {
  email: org.contactInfo?.email ?? user.email,
  name: org.contactInfo?.name,
  businessName: org.contactInfo?.businessName,
  address: org.contactInfo?.address,
};

await billingAccountService.create(orgNamespace, {
  currencyCode: "USD",
  contactInfo: formValues,
});
```

No controller watches `Organization` and copies contact into `BillingAccount`.

#### cloud-portal: create org (server-generated name)

Remove the user-facing slug field from `createOrganizationSchema`:

```typescript
import { randomBytes } from "crypto";

function generateOrgName(): string {
  return `org-${randomBytes(6).toString("hex")}`;
}

export const createOrganizationSchema = z.object({
  displayName: z.string().min(1).max(100),
  description: z.string().max(500).optional(),
});
```

Optionally add a milo validating webhook rule that rejects create requests where
`metadata.name` matches display name, contact business name, or common patterns
(`personal-org-*`, email local-part, etc.).

#### Migration: quota bump for existing Personal orgs

```bash
kubectl get resourcegrants -A -l quota.miloapis.com/policy=personal-organization-project-quota-policy \
  -o json | jq -r '.items[] | "\(.metadata.namespace)/\(.metadata.name)"' | while read ns name; do
  kubectl patch resourcegrant "$name" -n "$ns" --type=merge -p \
    '{"spec":{"allowances":[{"resourceType":"resourcemanager.miloapis.com/projects","buckets":[{"amount":10}]}]}}'
done
```

#### Migration: strip type from existing Organization CRs

```bash
kubectl get organizations -o json | jq -r '.items[] | select(.spec.type != null) | .metadata.name' | while read name; do
  kubectl patch organization "$name" --type=json -p='[{"op":"remove","path":"/spec/type"}]'
done
```

### 2. Quota: one grant policy

Replace the Personal + Standard `GrantCreationPolicy` pair with a single policy
triggered on Organization create (no type constraint). Default: 10 projects
(today's Standard allowance).

Migration: patch existing Personal org grants from 2 to 10 (prefer in-place
patch if the controller supports it).

### 3. Signup and onboarding

Keep the datum default-org controller, but change what it creates:

- Still auto-create org + owner membership + default project on User reconcile.
- Stop setting `spec.type = Personal` (field removed).
- Use a neutral default display name ("My organization" or "{Given}'s
  organization"), editable after onboarding.
- **Onboarding gate:** full platform access is blocked until onboarding is
  complete. Onboarding requires complete `spec.contactInfo` (email + name;
  address optional) **and** a `BillingAccount` with a valid payment method.
  cloud-portal middleware keeps the user in the onboarding flow (contact step,
  then billing step) until both hold, extending today's `complete-profile`
  name-review flow.
- Users with zero org memberships redirect to onboarding.
- **Onboarding-completed marker:** a controller stamps an annotation on the
  Organization once both conditions hold: contact info is complete and at least
  one `BillingAccount` with a valid payment method is attached in the org
  namespace. The portal gate reads this annotation. See
  [Onboarding completion annotation](#onboarding-completion-annotation).

```mermaid
flowchart TD
  signup[User signs up via Zitadel] --> userCR[User CR created in Milo]
  userCR --> datumCtrl[datum default org controller]
  datumCtrl --> orgCR[Organization + membership + project]
  orgCR --> portal[cloud-portal session]
  portal --> checkMembership{any org membership?}
  checkMembership -->|no| onboardContact[Onboarding: org contact form]
  checkMembership -->|yes| checkContact{contactInfo complete?}
  checkContact -->|no| onboardContact
  checkContact -->|yes| checkBilling{billing account with<br/>valid payment method?}
  checkBilling -->|no| onboardBilling[Onboarding: add billing +<br/>valid payment method]
  onboardContact --> checkBilling
  onboardBilling --> stamp[Controller stamps<br/>onboarding-completed annotation]
  checkBilling -->|yes| stamp
  stamp --> app[Full platform access]
```

#### Onboarding completion annotation

Full platform access requires completed onboarding: org contact info (email +
name) and a billing account whose `DefaultPaymentMethodReady` condition is
`True`. A billing account without a usable payment method does not count. Until
both hold, the portal keeps the user in the onboarding flow.

| Annotation | Value | Set by | Meaning |
|------------|-------|--------|---------|
| `resourcemanager.miloapis.com/onboarding-completed-at` | RFC3339 timestamp | controller | Contact info is complete and the org has a billing account with a valid payment method. |

The controller watches `Organization` and `BillingAccount` resources and
stamps the annotation once. It is set once and not removed if billing later
changes. The portal gate reads the annotation rather than recomputing the
condition on every request.

Payment readiness comes from the existing billing API: the
`DefaultPaymentMethodReady` condition on `BillingAccount`, set when the
referenced `PaymentMethod` is `Active`. The stripe-provider handles the card
SetupIntent flow that moves a payment method to `Active`.

Example:

```yaml
apiVersion: resourcemanager.miloapis.com/v1alpha1
kind: Organization
metadata:
  name: org-7f3a9c2b1d4e
  annotations:
    kubernetes.io/display-name: Acme Corp
    resourcemanager.miloapis.com/onboarding-completed-at: "2026-06-24T10:42:00Z"
spec:
  contactInfo:
    email: admin@acme.com
    name: Jane Doe
```

Controller logic (sketch):

```go
// Watches Organization and BillingAccount; reconciles the Organization.
const onboardingCompletedAt = "resourcemanager.miloapis.com/onboarding-completed-at"

func (r *OnboardingReconciler) Reconcile(ctx context.Context, org *resourcemanagerv1alpha1.Organization) error {
	// Idempotent: once stamped, never unset.
	if _, ok := org.Annotations[onboardingCompletedAt]; ok {
		return nil
	}
	if !contactInfoComplete(org.Spec.ContactInfo) {
		return nil
	}

	var billing billingv1alpha1.BillingAccountList
	if err := r.List(ctx, &billing, client.InNamespace(orgNamespace(org.Name))); err != nil {
		return err
	}
	if !hasValidPaymentMethod(billing.Items) {
		return nil
	}

	patch := client.MergeFrom(org.DeepCopy())
	metav1.SetMetaDataAnnotation(&org.ObjectMeta, onboardingCompletedAt, time.Now().UTC().Format(time.RFC3339))
	return r.Patch(ctx, org, patch)
}

// hasValidPaymentMethod reports whether any billing account is usable for
// charges. The billing service already exposes this signal: it sets the
// DefaultPaymentMethodReady condition to True when the account's referenced
// default PaymentMethod is in the Active phase. We read that condition rather
// than inspecting Stripe/provider data directly.
func hasValidPaymentMethod(accounts []billingv1alpha1.BillingAccount) bool {
	for i := range accounts {
		// billingv1alpha1.BillingAccountConditionDefaultPaymentMethodReady
		if meta.IsStatusConditionTrue(accounts[i].Status.Conditions, "DefaultPaymentMethodReady") {
			return true
		}
	}
	return false
}
```

### 4. Portal UX

**cloud-portal**

- Remove Personal badge, sort-first, and type-conditional copy.
- Enable team settings, billing settings nav, and rename for all orgs.
- Replace hard-coded project limits with quota API reads (or platform default 10
  until quota is wired).
- Add org contact editor under org settings.
- Extend onboarding route(s) to PATCH organization contactInfo.
- Create-org flow: collect display name only; generate opaque `metadata.name`
  server-side.
- Read `resourcemanager.miloapis.com/onboarding-completed-at` to drive
  onboarding banners and the "add a billing account with a valid payment method"
  prompt; stop showing them once the annotation is present.

**staff-portal**

- Remove Personal/Standard type filter and column.
- Show org contact fields on org detail (read from `spec.contactInfo`).

### 5. Cross-repo work

| Repo                | Work                                                                                                                                                                                                                                                       |
| ------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **milo**            | API + CRD + codegen; remove type from membership status; optional webhook to reject semantic/slug-like org names on create; update tests/samples/docs                                                                                                      |
| **datum**           | Rewrite default org controller (no type); delete Personal name-change policy; merge quota grant policies; add onboarding-completion controller that watches Organization + BillingAccount and stamps `onboarding-completed-at`                             |
| **billing**         | No schema change; no org-to-billing contact sync; portal may pre-fill billing form as a one-time UX default. Onboarding consumes the existing `DefaultPaymentMethodReady` `BillingAccount` condition (no new billing API needed)                           |
| **stripe-provider** | Already implemented and deployed (`milo-os/stripe-provider`, alpha). Moves a `PaymentMethod` to `Active` via the Stripe SetupIntent flow, which drives `DefaultPaymentMethodReady`. No new work required; onboarding only consumes the resulting condition |
| **zitadel**         | None expected                                                                                                                                                                                                                                              |
| **infra**           | Roll out CRD + datum/milo/cloud-portal/staff-portal in coordinated order; migration job or one-shot patch for existing orgs                                                                                                                                |

### 6. Migration plan

1. **Schema:** deploy CRD that adds `contactInfo` and removes `type` in one
   breaking version (preferred per #636: drop entirely).
2. **Data:** batch job sets `contactInfo.email` and `contactInfo.name` from the
   org owner's User record where missing; does not overwrite billing data.
   `address` is optional and is not backfilled; users add it during onboarding
   or later in the portal. Note that `spec.contactInfo` itself is optional at
   the top level, so legacy orgs that have no `contactInfo` block remain valid.
   The required sub-fields (`email`, `name`) only apply once `contactInfo` is
   set. This lets admin patches (type removal, quota bump) run against legacy
   orgs without tripping validation, while every org submitting contact data,
   and every new org, must provide email + name. The onboarding gate is what
   enforces completeness before full platform access.
3. **Quota:** upgrade Personal org grants to Standard amounts.
4. **Cleanup:** remove deprecated policies, portal branches, metrics labels
   keyed on `organization_type`.

Backward compatibility: clients reading `spec.type` break. Regenerate
cloud-portal/staff-portal OpenAPI types. GraphQL schema drops `OrganizationType`
enum.

## Open Questions

1. Plan-based member limits instead of type-based (raised in #636; defer unless
   product wants it in v1).
2. Deprecation window: single breaking release vs one release with deprecated
   `type` still accepted.

## Risks

- **Quota migration:** Personal orgs jumping from 2 to 10 projects may surprise
  users who relied on the lower cap as a soft limit.
- **Breaking API change:** external integrations or scripts filtering on
  `spec.type` need audit.
- **Onboarding friction:** gating all platform access on contact info **and** a
  billing account with a valid payment method is a hard gate. Requiring payment
  details before any usage will increase signup drop-off and blocks "try before
  you pay" exploration. Mitigate with pre-filled contact defaults from the User
  profile and a fast, low-friction billing step; revisit whether a limited trial
  or grace period should precede the billing requirement.

## Implementation History

- **2026-06-19**: Initial enhancement document (provisional), derived from
  [milo-os/milo#636](https://github.com/milo-os/milo/issues/636).
