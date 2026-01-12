---
title: "Enhancement: Create CRM People and Companies Components"
labels: enhancement
---

### High-Level Summary

Implement the **People (Contacts)** and **Companies** components for the Datum Cloud CRM. This enhancement will provide the core contact and company management capabilities, including list views, detail views, label-based source tracking, subscription management, activity logging, and notes.

 To clarify, there is no expectation to automatically link Companies to Organizations or Persons to Users. One of the reasons we changed the naming convention was specifically to avoid confusion between these APIs and the existing concepts.

Following a discussion with @kaleygel , the goal here is for this data to be manually managed by staff users rather than being auto-populated or linked by the system. We aren't expecting users or organization owners to fill this out themselves; we want the data to be entered precisely based on staff knowledge of the companies and people involved.

---

### Motivation

Currently, there is no centralized system within Datum Cloud to manage contacts and companies. This creates challenges for:

- **Tracking leads and contacts** — Sales and marketing teams have no unified view of people interacting with the platform
- **Understanding acquisition sources** — Without labels, we cannot measure which channels (waitlist, newsletter, events, etc.) drive the most engagement
- **Managing relationships** — No way to associate contacts with their companies or track interaction history
- **Running campaigns** — Unable to manage subscriptions for newsletters, investor updates, and other communications
- **Scaling operations** — Manual processes will not scale as the user base grows

Building these foundational CRM components will enable the team to effectively manage relationships and support growth.

---

### Goals

**Create the People module with:**

*Overview Page displaying:*

- Person Name (required)
- Company (required)
- Email Addresses (required)
- Job Title (optional)
- Created By (required)
- Labels (optional) — extensible label system including:
  - Website Waitlist
  - Newsletter Signup
  - Investors
  - Alt Cloud Meetups
  - Events
  - Organic Search

*Detail View displaying:*

- Person (required)
- Company (required)
- Email Addresses (required)
- Job Title (optional)
- Created By (required)
- Labels (required) — tied to how the user was added
- Subscriptions (required) — contact lists:
  - Website Waitlist
  - Newsletter Signup
  - Investor Updates
- Activities section for product-level actions/events
- Notes section

*Workflow:* manually managed.

---

**Create the Companies module with:**

*Overview Page displaying:*

- Company Name
- Domains
- Description
- LinkedIn (company page URL)
- Labels — extensible label system including:
  - Product Signup
  - Newsletter Subscribe
  - Referral
  - Inbound Inquiry
  - Website
  - Existing Client Expansion
  - Partner Introduction
  - Organic Search
  - Marketing Campaign
  - Event
- Last Interaction (date)
- Created At (date)
- Created By

*Detail View displaying:*

- All fields from overview (Company Name, Domains, Description, LinkedIn, Labels, Last Interaction, Created At, Created By)
- People section — list of contacts associated with this company

---

**Design for Future Expansion:**

The architecture should be built to accommodate the following future capabilities:

- Email activity tracking (inbox sync, email history per contact)
- Email sending (direct outreach from CRM)
- Bulk import/export functionality (CSV, integrations)
- Calendar integration (meetings, scheduling)
- Advanced automation workflows

---

### Non-Goals

The following are out of scope for this enhancement:

- Deal/Opportunity pipeline functionality
- Email sending or inbox integration
- Calendar/meeting integration
- Advanced analytics dashboards
- Bulk import/export functionality
- Automatic duplicate detection
- Custom field builder
- Third-party integrations (Salesforce, HubSpot, etc.)
- Native mobile application
- Complex automation rules beyond the newsletter auto-add
