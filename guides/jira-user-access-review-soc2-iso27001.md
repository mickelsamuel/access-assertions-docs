---
title: "Running a Jira Cloud user access review for SOC 2 or ISO 27001"
description: "What auditors want from a Jira access review, how to pull the population natively in Jira Cloud, and which evidence to keep before the audit log expires."
layout: page
---

If Jira holds anything your auditors care about (customer data in tickets, security incidents, change records), it's in scope for your access reviews. Jira Cloud doesn't have a built-in review or certification feature that I could find in Atlassian's documentation, so the review is something you assemble. This guide covers what the evidence needs to show, how to pull it with native tools, and where those tools stop.

One naming note: Jira Cloud menus now say "space" for project. The REST API still says project.

## What the auditor is actually testing

SOC 2's logical access criteria (the CC6 series) and ISO 27001:2022's access rights control (5.18) both come down to the same thing in practice: access is granted on purpose, checked on a schedule, and removed when it's no longer needed. For a review, the evidence an auditor usually asks for is:

1. **The population.** A list of who had access, as of a stated date, that clearly came out of the system and not out of someone's memory. Auditors test whether the list is complete, so "exported from Jira on 2026-10-01 by this admin" matters more than formatting.
2. **The decision.** Someone accountable for each project looked at the list and said keep or remove, line by line, with a date.
3. **The follow-through.** Every "remove" actually happened, with proof.
4. **The schedule.** The review recurs at the frequency your own policy states. Quarterly is common. Whatever you pick, missing a cycle is a finding.

Privileged access gets looked at separately and harder. In Jira that means project administrators, Jira administrators and organization admins.

Check the exact wording with your auditor. Firms differ on sampling and on what counts as a sign-off.

## Step 1: decide the scope

List the projects that hold in-scope data. Ten sensitive projects reviewed properly beat four hundred reviewed badly. Write the scope down and say why each project is in or out, because the auditor will ask.

## Step 2: pull the population

For each in-scope project you want two lists: who can browse it, and who can administer it.

**Who can browse.** Access comes from the project's permission scheme, not just the People page. The Browse spaces permission can go to project roles, groups, "Any logged in user", application access or single users. The fastest native pull is one REST call per project:

```
GET /rest/api/3/user/permission/search?projectKey=ABC&permissions=BROWSE_PROJECTS&query=
```

It resolves groups and roles for you. It also returns app accounts (`accountType: app`); on a stock site that's about ten Atlassian apps per project, so drop them before the population goes to a reviewer. Atlassian notes it only looks "up to the thousandth user", so on larger sites you'll need the slower route: read the scheme (`GET /rest/api/3/permissionscheme/{id}/permission?expand=all`), the role actors (`GET /rest/api/3/project/{key}/role/{id}`) and group members (`GET /rest/api/3/group/member`).

**Who can administer.** Same call with `permissions=ADMINISTER_PROJECTS`. Jira admins can administer every project, but this search won't list them (I checked on a test site: a Jira admin with no project role came back without the permission), so add them from Global permissions yourself.

**Site-level privilege.** Jira admins are listed under **Settings > System > Global permissions** (look at Administer Jira). Organization admins show up in the org's user export, in the "Org role" column.

**Dormant accounts.** The user export from admin.atlassian.com (**Directory > Users > Export users**) includes a "Last seen in" column per app. It's the easiest way to spot people who still have access and haven't logged in for months. Only organization admins can run that export.

Save every raw response with the date and the account that ran it.

## Step 3: get the decisions recorded

Native Jira is fine for this part. Create one work item per project per cycle, attach the population CSV, and assign it to the project's owner. Their decision goes in a comment or a field, and the transition to Done is the sign-off. The work item's history gives you who and when without extra effort.

Don't send the list to managers as a spreadsheet over email unless you also keep the replies. "Approved in Slack" is weak evidence.

## Step 4: remove access and prove it

Make the removals, then capture proof two ways:

- **The audit log.** Jira's audit log (**Settings > System > Audit Log**) records when users are added to or removed from projects, groups or roles, and every permission scheme edit. Export it, or pull `GET /rest/api/3/auditing/record` with `from` and `to`.
- **A second pull.** Re-run Step 2 for the affected projects. The before-and-after pair is cleaner evidence than a log line, because it shows the end state.

## Step 5: keep the evidence longer than Jira does

This is where teams get caught. Jira's own audit log offers 1, 3 or 6 months of retention and starts at 3 on a new site, and the organization-level audit log keeps 180 days. If your audit period is twelve months, the events from the first half are gone by fieldwork. Export the audit log at every review cycle and store it with the review records.

What to keep per cycle:

- the raw population pull (JSON) and the CSV built from it, dated
- the review work items with their decisions and history
- the audit log export covering the changes
- the second pull showing removals took effect

## Where native Jira stops

- No access review or certification workflow. You build it from work items.
- No "who can access this project" report or export in the UI.
- The org user export is organization-admin only and doesn't include project roles or permission schemes.
- The Permission Helper answers one user, one work item, one permission at a time, which is a troubleshooting tool, not a review.
- Audit log retention tops out at six months (Jira) or 180 days (organization log, which also needs Atlassian Guard or an Enterprise plan).
- The REST user search stops at the thousandth user.

## Doing it with Access Assertions

I build Access Assertions, so I'm biased here. It turns the "who should have access" part of your policy into rules written as sentences: who may browse a project, who must not, who may administer it, and which groups issue security may grant to. It checks them every night and gives each rule PASS, FAIL or UNKNOWN. PASS only appears when every group and role behind the rule was read in full, so a partial read never looks clean. A failure opens one Jira ticket naming the path from project through scheme, grant, role and group to the account. Results and evidence are kept for 400 days, so a twelve-month audit period is still there at fieldwork, and they export as CSV and JSON, ready to file with the rest of your review records. It's read-only, runs entirely on Atlassian and covers company-managed projects. [Access Assertions on the Atlassian Marketplace](https://marketplace.atlassian.com/apps/370337795).

## Sources

- Space access and configuration permissions: https://support.atlassian.com/jira-cloud-administration/docs/space-access-and-configuration-permissions/
- Grant or revoke permissions in a scheme: https://support.atlassian.com/jira-cloud-administration/docs/grant-or-revoke-permissions-in-a-scheme/
- View, grant, revoke global permissions: https://support.atlassian.com/jira-cloud-administration/docs/view-grant-revoke-global-permissions-in-jira/
- Export users from a site: https://support.atlassian.com/organization-administration/docs/export-users-from-a-site/
- Audit activities in Jira: https://support.atlassian.com/jira-cloud-administration/docs/audit-activities-in-jira-applications/
- Organization audit log retention (180 days): https://support.atlassian.com/security-and-access-policies/docs/view-audit-log-activities/
- Jira audit log retention options (1, 3 or 6 months): https://jira.atlassian.com/browse/JRACLOUD-70433
- Use the Jira Admin Helper: https://support.atlassian.com/jira-cloud-administration/docs/use-the-jira-admin-helper/
- Jira Cloud REST API v3: https://developer.atlassian.com/cloud/jira/platform/rest/v3/
- Community article on SOC 2 access reviews ("typically quarterly"; vendor-written): https://community.atlassian.com/forums/App-Central-articles/How-to-Audit-Atlassian-User-Access-for-SOC-2-Compliance/ba-p/3219533
