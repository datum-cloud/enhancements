---
status: provisional
stage: alpha
latest-milestone: "v0.1"
---

<!-- omit from toc -->
# Platform Feedback

- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [Where it shows up](#where-it-shows-up)
  - [Browse and upvote](#browse-and-upvote)
  - [Share feedback](#share-feedback)
  - [Report a security issue](#report-a-security-issue)
  - [Delivery tracker](#delivery-tracker)
  - [Staff review and insights](#staff-review-and-insights)
  - [Product rules](#product-rules)
- [Rollout](#rollout)
- [Alternatives](#alternatives)

## Summary

Datum Feedback is one place for users to see what we're building, tell us what
they want, and follow along as it ships. It brings together the roadmap,
releases, changelog and feature requests, and shows them the same way on the
website, in the cloud portal, in datumctl and in Patch.

GitHub stays the source of truth for the work itself. Feedback pulls public
roadmap items from our two GitHub orgs (`datum-cloud` and `milo-os`) and adds
what GitHub can't give us: votes and submissions tied to Datum users and
organizations, a staff review step before anything goes public, and updates
for the people who asked.

Tracking issue: [datum-cloud/enhancements#889][issue]. Web designs:
[Figma: Roadmap + Backlog][figma]. Technical design:
[Platform Feedback: Architecture](../../../architecture/design/feedback/README.md).

[issue]: https://github.com/datum-cloud/enhancements/issues/889
[figma]: https://www.figma.com/design/bBEQ8YeTP4SngNl5EkkQdH/Datum---Master-Design-File?node-id=17642-52938

## Motivation

Today users send feedback by email, support ticket or Discord. That works,
but it puts the effort on them: explaining context, checking whether someone
already asked, and finding out whether it's on the roadmap. We get feedback
we can't easily de-duplicate, count, or tie back to a customer.

Each surface has also started solving this on its own. The website talks to
GitHub directly for the roadmap and changelog, the portal has a feedback form
that goes nowhere yet, and datumctl and Patch have nothing. One shared
service means one experience and one set of data.

### Goals

- Anyone can browse the roadmap, releases, changelog and existing feedback,
  and search, filter and sort it, without signing in.
- Signed-in users can upvote items and see what they've voted for.
- Users can share feedback from the website, portal, datumctl or Patch, and
  are shown similar existing items first so they can upvote instead.
- Every submission is reviewed by Datum staff before anything is public.
- Users can track everything they've voted for or submitted, and hear back
  when work on it is picked up, when it's released, or if it's declined.
- Staff can see who is asking for what, by user and organization.
- People and agents can report security issues privately from the same
  places, follow the report through to a fix, and be credited and rewarded
  for responsible disclosure.
- The website's Releases, Roadmap, Changelog and Feedback pages are built on
  this service, replacing the website's own GitHub integration.

### Non-Goals

- Comment threads in Datum. Discussion stays on GitHub; every item links to
  it.
- Replacing GitHub for planning. Staff keep working in GitHub issues and the
  Enhancements project.
- Real-time bug fixing or support SLAs. This is not a support channel.
- Exporting feedback to the CRM in the first version.
- Turning Patch's capability-gap reports into submissions. Those stay a
  separate signal for the teams that own each service.

## Proposal

### Where it shows up

![One feedback hub shared by the website, cloud portal, datumctl, Patch and staff, synced with GitHub](./diagrams/surfaces.svg)

*Editable sources for all diagrams are the `.excalidraw` files in
[`diagrams/`](./diagrams).*

| Surface | What users can do |
| --- | --- |
| **datum.net** | Releases, Roadmap, Changelog and Feedback pages per the Figma. Browse without signing in; sign in to vote or submit. |
| **Cloud portal** | A feedback widget available on every page. Organization, project, page and recent errors are attached automatically. Browse and vote in place. |
| **datumctl** | `datumctl feedback search`, `show`, `vote`, `submit` and `mine`. |
| **Patch** | When a user says something is missing or broken, Patch searches existing feedback, offers to upvote a match, or drafts a submission for the user to confirm. |
| **Staff portal** | Review queue for submissions, and insight into who is asking for what. |

Content comes from three places on GitHub:

- **Roadmap items**: public issues in `datum-cloud` and `milo-os` that are
  tracked on the Enhancements project (project 22). The project's Status
  column drives the item's status.
- **Releases**: milestones in `datum-cloud/enhancements` (Hedy 2.0,
  Lovelace 2.1, ...), with the items in each.
- **Changelog**: posts in the `changelog` Discussions category of
  `datum-cloud/datum`. Items referenced in a post are marked as shipped by
  it.

### Browse and upvote

![Browse, open an item, upvote, sign in if needed](./diagrams/browse-and-vote.svg)

- **Browse**: the Roadmap groups items by Planning, On deck and In progress.
  The Feedback page lists everything with search, filters for status and
  category, and sorting by most upvotes, newest or trending.
- **Item detail**: title, categories, full description, vote count, voters
  and a link to the GitHub issue.
- **Upvote**: one click when signed in. Signed-out users are sent through
  login or sign-up and returned to the item. Upvoting also follows the item.
  Users can remove their vote.

### Share feedback

![Describe it, see similar items, submit, staff review, outcome](./diagrams/submit-and-review.svg)

1. The user describes the problem or idea, optionally adding screenshots or
   logs. In the portal and Patch, context is attached for them.
2. While they type, similar existing items appear. If one fits, they upvote
   it instead.
3. Otherwise they submit. The submission shows as "Under review" and is
   visible only to them and Datum staff.
4. Staff review it, helped by Patch's suggested duplicates and category. The
   outcome is one of:
   - **Approved**: published as a new public item on the Enhancements
     project, using only the text meant to be public. The submitter is
     counted as its first vote.
   - **Duplicate**: merged into an existing item; the submitter's vote moves
     to it.
   - **Declined**: the submitter is told why.

Nothing a user submits is ever published automatically.

### Report a security issue

Security reports get their own track. They start in the same places as
feedback, but nothing about them is ever public until the Security Team
says so.

**Where.** Every surface that takes feedback offers "Report a security
issue", which opens a separate private form:

- datum.net and the cloud portal, signed in;
- `datumctl security report`;
- Patch, when a user describes something that looks like a vulnerability.
  Patch drafts the report and the user confirms it, the same as feedback.

`security@datum.net` and GitHub private vulnerability reporting stay open
for people who'd rather not sign in. The Security Team logs those reports in
the same queue, so every report is tracked in one place. datum.net
publishes a [`security.txt`][security-txt] and a disclosure policy page
pointing to all three, so researchers and agents that look for one find it.

**Agents.** Agents acting for a user report through Patch, datumctl or the
MCP tools like anyone else, and the report records that an agent filed it.
The disclosure policy says what agents may and may not do while testing.

**How it differs from feedback.**

- No similar-items step and no voting. Reports are never matched against
  public items or shown to anyone but the reporter and the Security Team.
- Reports go to the Security Team, not the feedback review queue. General
  feedback reviewers can't see them.
- Reports are never published as GitHub issues. A fix that needs a public
  notice gets a GitHub security advisory when the Security Team is ready.
- The reporter hears about every change straight away, not in the daily
  digest.

**Tracking.** The reporter sees a private tracker:

| Stage | What it means |
| --- | --- |
| Received | Logged and acknowledged |
| Triaged | Confirmed, with a severity, or closed as not applicable or a duplicate, with a reason |
| Fixing | A fix is in progress |
| Fixed | The fix is deployed |
| Disclosed | Advisory published, if one was needed |

**Credit and rewards.** Valid reports are credited and rewarded:

- **Credit**: the reporter is named in the advisory and on a datum.net
  security acknowledgements page, if they opt in, with the name and link of
  their choice.
- **Rewards**: swag, platform credits on the reporter's organization, or
  just a public thank-you, scaled by severity. The reporter sees the reward
  on their tracker once it's granted.

The disclosure policy sets out scope (what may be tested), rules (no
testing against other customers' resources, no denial of service, no social
engineering), safe harbour for good-faith research, and reward tiers.

<<[UNRESOLVED disclosure policy]>>
Reward tiers and amounts, whether cash rewards or a bounty platform
(HackerOne, Bugcrowd) come later, and the safe harbour wording, which needs
legal review.
<<[/UNRESOLVED]>>

[security-txt]: https://www.rfc-editor.org/rfc/rfc9116

### Delivery tracker

![Stages from requested to released, with related work and notifications](./diagrams/delivery-tracker.svg)

Every item shows where it is on the way to users, like a parcel tracker.
Anyone can see an item's tracker; signed-in users also get one view of
everything they follow.

| Stage | What it means | Followers told? |
| --- | --- | --- |
| Under review | A submission waiting for staff (only the submitter sees this) | Outcome only |
| Requested | It's on the board: someone asked for it | No |
| Planned | On the roadmap: Planning or On deck | No |
| Picked up | Work has started on the item or any piece of it | Yes |
| Released | Shipped in a release or a changelog post | Yes |
| Closed | Won't be built, with a reason | Yes |

**Related work.** Bigger items are usually split into smaller pieces of
work, often in other repos: GitHub sub-issues and the pull requests that
close them. The tracker lists the public ones with their state ("3 of 5
done") and links to each, so users can see progress between Picked up and
Released. When a changelog post ships one piece before the whole item is
done, followers get a "part of this shipped" update that links to the post.

**Who follows.** Everyone who voted for an item, everyone whose submission
was approved as it or merged into it, and anyone who followed it by hand.

**How they hear.** Tracker updates go in the daily digest email, alongside
submission outcomes. Each stage is announced once: if work pauses and
starts again, the tracker shows it but nobody is emailed twice. Edits,
label changes and GitHub comments are never sent. Every digest links to
unfollow an item or all feedback updates.

**My feedback.** Signed-in users see everything they follow in one list,
with each item's stage and what changed last:

- a My feedback page on datum.net and in the cloud portal;
- `datumctl feedback mine`;
- Patch, when asked "what's happening with my requests?".

### Staff review and insights

Staff get, in the staff portal:

- a review queue of submissions with the submitter's private context
  (identity, organization, project, page, logs, attachments), Patch's
  suggestions, and approve / merge / decline actions;
- for any item, who voted and which organizations they belong to;
- top items by number of organizations asking, not just votes;
- for any organization, everything it has voted for or submitted.

On GitHub, the Enhancements project gets a Votes field so staff can sort
by demand where they already plan.

### Product rules

- **Public vs private.** Only public GitHub issues appear. Submissions and
  everything attached to them stay private to the submitter and staff, even
  after approval.
- **Votes.** The count shown is Datum upvotes plus GitHub 👍 reactions. A
  person is counted once where we can match their Datum and GitHub accounts.
  Someone who 👍s on GitHub without a linked Datum account and also votes in
  Datum is counted twice; we accept that.
- **Voter avatars** are only shown for users who opt in. Counts are always
  shown.
- **Statuses**: None (not yet triaged), Backlog, Planning, On deck, In
  progress, Shipped, Closed. Submissions also have Under review, Merged and
  Declined. The delivery tracker groups these into the stages above.
- **Related work** only shows public issues and pull requests. Work in
  private repos still moves the tracker through the item's own status but
  isn't listed or counted.
- **Categories** come from GitHub labels: Build, Deliver, Connect, Platform,
  Infrastructure, and AX/DX/UX on changelog posts. An Infrastructure label
  doesn't exist yet and needs adding.
- **Hiding.** Staff can hide an item from Datum Feedback with a GitHub label
  without closing the issue.
- **Security reports** are visible only to the reporter and the Security
  Team, never become public items, and never count as votes.

## Rollout

1. **Read.** Roadmap, Releases, Changelog and Feedback lists on datum.net,
   from both orgs, replacing the website's direct GitHub integration.
2. **Vote and track.** Upvoting, login gate, following, the delivery tracker
   with related work, My feedback, and digest emails.
3. **Submit.** Similar-feedback suggestions, submissions and the staff review
   queue. Security reports, their tracker, `security.txt` and the
   disclosure policy page.
4. **Everywhere.** Portal widget, datumctl commands, Patch, staff insights.

## Alternatives

**Keep each surface wrapping GitHub.** This is where we are today. Every
client re-implements fetching, caching and filtering; votes can't be tied to
Datum users; nothing reaches datumctl or Patch.

**Canny or a similar SaaS.** Good voting and roadmap UX, but a second source
of truth next to GitHub, a separate login, and no way into datumctl or
Patch.

**GitHub Discussions or reactions only.** Free, but requires a GitHub
account, can't hold private context, and can't tie demand to Datum
organizations.
