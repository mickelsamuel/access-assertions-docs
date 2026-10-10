---
title: "Jira Cloud audit log retention: the 180-day limit and how to keep access evidence longer"
description: "Jira Cloud has two audit logs and neither keeps much more than six months. What each one holds, and how to keep access evidence for a full audit year."
layout: page
---

"The audit log only goes back 180 days" is half right. Jira Cloud has two audit logs, they live in different places and have different rules, and in practice neither one gets you past about six months. If your auditors want twelve months of access history, you have to start keeping it yourself, and nothing you set up today recovers what has already expired.

## The two logs

**Jira's own audit log** (product level)

- Where: **Settings > System > Audit Log** in Jira.
- What it records: configuration changes, which Atlassian describes as changes "that can impact users and spaces". That includes permission scheme edits (created, copied, edited, removed, assigned to or unassigned from a space), permissions added to or removed from a scheme, global permission changes per group, and users being added to or removed from projects, groups or roles.
- Retention: set under **Actions > Audit Log Settings**. The only choices are 1, 3 or 6 months, and a new site starts at 3 months (I checked on a fresh Jira Cloud site in October 2026). So unless someone changed it, your log only reaches back about 90 days. The request for a 1-year option (JRACLOUD-70433) was closed as a duplicate, and the open request to raise the ceiling, JRACLOUD-68267, is still waiting.
- Export: CSV, up to 100,000 events. If there are more, you get the newest ones, and the export ignores whatever filter you had applied on the page.
- API: `GET /rest/api/3/auditing/record`, with `from`, `to`, `filter`, `offset` and `limit`. Needs Administer Jira.
- Plan: not available if all your Jira apps are on the Free plan.

**The organization audit log** (admin.atlassian.com)

- Where: **Atlassian Administration > Insights > Audit log**.
- What it records: in Atlassian's words, "organization admin activity, app admin activity, and user-created activity", the last one depending on your plan.
- Retention: "stored for up to 180 days. Any activity older than 180 days is removed from the audit log and can't be recovered." Atlassian's own advice right after that line is to export regularly.
- Plan: it needs Atlassian Guard (Standard or Premium) or at least one Enterprise plan. Without one, Atlassian says organization-level logs "will not store or display any events."
- Export: an emailed CSV with "up to 10,000 activities from the past 180 days."
- API: `GET https://api.atlassian.com/admin/v1/orgs/{orgId}/events` for filtered queries, or `/events-stream` for simple time-based paging.
- Webhooks: on an Enterprise plan or Guard Premium you can register up to three webhooks that push each event to another tool as it happens.

For access questions about a Jira project, the product-level log usually has the events you need, because that's where scheme and role changes land.

## Why exporting the log isn't enough by itself

The log tells you what changed. An auditor's question is usually about state: "who could see the Payroll project on 31 March?" To answer that from events alone, you need a known starting point and every change since. Miss one event, or start collecting after the window you're asked about, and you can't rebuild the answer.

So keep two things:

1. **The event stream**, exported on a schedule shorter than the retention period.
2. **Regular snapshots of current access**, as of a date, for the projects you care about.

The snapshot is what answers the auditor's question. The events explain how you got from one snapshot to the next.

## A native setup that works

**Pull the Jira audit log on a schedule.** A small script that calls `GET /rest/api/3/auditing/record?from=<last run>` weekly is plenty. Each record has an `id` and a `created` timestamp, so overlapping pulls are easy to de-duplicate. Use an account or token with Administer Jira and keep the output somewhere your Jira admins can't edit.

**Pull the organization log if you have it.** Same idea against `/admin/v1/orgs/{orgId}/events-stream`. If you're on Guard Premium or Enterprise, a webhook into your log tool or SIEM is less work than polling.

**Snapshot access monthly or at each review.** For each in-scope project:

```
GET /rest/api/3/user/permission/search?projectKey=ABC&permissions=BROWSE_PROJECTS&query=
GET /rest/api/3/user/permission/search?projectKey=ABC&permissions=ADMINISTER_PROJECTS&query=
```

Atlassian documents that this search only covers users "up to the thousandth user", so larger sites need to read the permission scheme, role actors and group members instead. The response also lists app accounts (their `accountType` is `app`), so filter those out before you count people. Save the JSON with the date in the file name.

**Store it where it can't quietly change.** Write-once storage, or at minimum a folder with its own access controls and a checksum file per export. Evidence that the Jira admins could edit is weaker evidence.

**Don't wait.** Whatever you set up only covers you from the day it starts running. If an audit period started five months ago, the oldest part of it is about to fall off.

## Limits to know before you rely on any of this

- Product log: three months by default, six at most, newest-first CSV cap of 100,000 events, not on Free.
- Organization log: 180 days, 10,000-activity CSV, needs Guard or Enterprise; webhooks need Guard Premium or Enterprise.
- Neither log is a record of who *had* access. They record changes. A permission granted before the window started leaves no trace inside it.
- Feature requests to extend retention (JRACLOUD-68267 for Jira, ACCESS-991 for the organization log) are still at "Gathering Interest".

## Doing it with Access Assertions

Disclosure: I build Access Assertions. It covers the snapshot half of this. You write rules as sentences, such as who may browse a project or who may administer it, and it checks them nightly with a PASS, FAIL or UNKNOWN result per rule. It only says PASS when every group and role behind the rule was read in full. Each failure opens a Jira ticket naming the path from project to account, and since those are ordinary work items they don't age out with the audit log. Results and the evidence behind them are kept for 400 days, which covers a full audit year with room to spare, and they export as CSV and JSON so you can file them wherever you keep audit records. It's read-only and runs entirely on Atlassian, for company-managed projects. [Access Assertions on the Atlassian Marketplace](https://marketplace.atlassian.com/apps/370337795).

## Sources

- Audit activities in Jira (menu path, retention setting, 100,000-event export, REST, Free plan): https://support.atlassian.com/jira-cloud-administration/docs/audit-activities-in-jira-applications/
- View audit log activities (organization log, 180 days, Guard or Enterprise required): https://support.atlassian.com/security-and-access-policies/docs/view-audit-log-activities/
- Export audit log (10,000 activities, past 180 days): https://support.atlassian.com/security-and-access-policies/docs/export-audit-logs/
- Audit log webhooks (up to 3, Enterprise or Guard Premium): https://support.atlassian.com/security-and-access-policies/docs/learn-more-about-audit-log-webhooks/
- Understand Atlassian Guard (what the organization log tracks): https://support.atlassian.com/security-and-access-policies/docs/understand-atlassian-guard/
- JRACLOUD-70433 (1, 3 or 6 months; 1 year closed as duplicate): https://jira.atlassian.com/browse/JRACLOUD-70433
- JRACLOUD-68267 (retention increase request): https://jira.atlassian.com/browse/JRACLOUD-68267
- ACCESS-991 (organization audit log storage request): https://jira.atlassian.com/browse/ACCESS-991
- Jira Cloud REST API v3, Get audit records and Find users with permissions: https://developer.atlassian.com/cloud/jira/platform/rest/v3/
- Atlassian Admin REST API, events: https://developer.atlassian.com/cloud/admin/organization/rest/api-group-events/
