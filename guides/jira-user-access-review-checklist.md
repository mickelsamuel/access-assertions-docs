---
title: "Jira Cloud user access review checklist (with a CSV template)"
description: "A quarterly access review checklist for Jira Cloud: what to pull, what to flag, how to record each decision and how to prove removals happened. Includes a plain CSV template."
layout: page
---

This is the checklist I'd use to run a quarterly user access review on Jira Cloud. It's written for the person who actually has to do it, usually a Jira admin with a deadline and an auditor waiting. There's a plain CSV template at the end, no sign-up needed: [jira-access-review-template.csv](jira-access-review-template.csv).

If you need the background on what SOC 2 and ISO 27001 auditors expect from a review, read [Running a Jira Cloud user access review for SOC 2 or ISO 27001](jira-user-access-review-soc2-iso27001.md) first. This page is the working list.

Jira Cloud's menus say "space" for project now. The REST API still says project.

## Before you start

- **Write down the scope.** Which projects hold data the review covers, and why the others are out. Better a short list done properly than every project done badly.
- **Name an owner for each project.** The person who can say "yes, she should still have this". Usually not the Jira admin.
- **Fix the "as of" date.** Every pull below should come from the same day, and the file names should carry that date.
- **Check your plan.** Permission schemes, space roles and work item security schemes aren't available on Free Jira sites, and Atlassian notes that on Free sites all users have Administer spaces.
- **Check the audit log retention.** Jira's audit log is at **Settings > System > Audit Log**; how long it keeps events is set under **Actions > Audit Log Settings**. If it won't cover the whole quarter, export it now as well as at the end.

## Pull the population

**Site-level access first. These people can reach everything.**

- **Jira admins.** **Settings > System > Global permissions**, then look at Administer Jira. Atlassian's permission reference says Jira admins "automatically inherit the Administer spaces permission", so they can administer every project without appearing on any of them.
- **Organization admins and everyone with Jira.** In admin.atlassian.com, **Directory > Users > Export users** (or, in the original admin experience, open the site, then **Users > Export users**; organization admins only). Include Group membership and App access. The CSV gives you User status (Active, Suspended or Deactivated), Org role, a role per app, and a "Last seen in" date per app. Atlassian counts a view of "a minimum of 2 seconds" as being seen. The file arrives by email.

**Then each in-scope project.**

- **Who can browse it.** The permission is Browse spaces (`BROWSE_PROJECTS` in the API). It comes from the project's permission scheme, which can grant it to roles, groups, application access, Public, Any logged in user and more. The People page shows only role members, so don't use it as the list.
- **Who can administer it.** Administer spaces (`ADMINISTER_PROJECTS`), plus the Jira admins from above.
- **Groups expanded to people.** A role that holds a group shows the group's name. The review needs the members.

The [permissions report guide](jira-permissions-report.md) has a short script that does all three for every company-managed project in one run and writes the CSV the template below is built around. For a single project, `GET /rest/api/3/user/permission/search?projectKey=ABC&permissions=BROWSE_PROJECTS&query=` resolves everything in one call, though it won't tell you how each person got in.

- **Team-managed projects.** They don't use permission schemes. Note each one's access setting (Open, Limited or Private) and its people list by hand.
- **Work item security.** If an in-scope project uses security levels, list each level's grantees too. Browse spaces gets someone into the project; the level decides whether they see the secured work items. See [Jira Cloud issue security levels](jira-issue-security-levels.md).
- **Save the raw output** with the date and the account that ran it.

## Flag before you send it out

Owners review faster when the obvious cases are already marked. Sort and flag:

- **Deactivated or suspended accounts** still in groups or roles. The group members API hides inactive accounts unless you ask for them, which is how they survive reviews. The grant is still there if the account comes back.
- **Not seen in Jira for 90 days** or whatever threshold your policy states. Join the user export's "Last seen in" column on account ID.
- **Contractors, vendors and former team members.** Usually a group name gives them away.
- **Broad grants.** Application access, Any logged in user or Public on Browse spaces means "everyone", and no owner can review "everyone" line by line. Treat the grant itself as the finding.
- **App accounts.** The API returns them with `accountType: app`. On my test site there were 11 per project. Review them as a separate list, not mixed in with people.
- **Single-user grants in the scheme.** They're easy to forget and they don't show on the People page.
- **Empty groups and roles** that hold a grant. Harmless today. Note them so a future member doesn't surprise anyone.

## Get the decisions

- **One line per person, per project, per permission.** That's what the template is.
- **Every line gets keep, remove or change**, with a reason for anything that isn't an obvious keep.
- **Record the decision in Jira**, not in email or chat. One work item per project per cycle, assigned to the owner, with the filled-in CSV attached. The transition to Done is the sign-off, and the history shows who and when.
- **Chase the stragglers.** A review cycle with one project unsigned is an incomplete review.

## Make the changes and prove them

- **Remove access at the source.** If someone got in through a group, remove them from the group (or remove the group from the grant). Taking them off the People page does nothing if the scheme also grants their group.
- **Fill in the change ticket and date** on each removed line.
- **Re-run the pull** for affected projects and confirm the lines are gone. Fill in `verified_on`. This before-and-after pair is the strongest evidence you'll have.
- **Export the audit log.** It records users added to or removed from groups and roles, and permission scheme edits, and the export takes up to 100,000 events as CSV.

## Archive

- The dated raw pull and the CSV built from it
- The completed template with every decision
- The review work items, with history
- The second pull showing removals took effect
- The audit log export

Store them somewhere that outlives Jira's own retention, with the quarter in the folder name.

## The template

[Download jira-access-review-template.csv](jira-access-review-template.csv). It's a header row and two example rows:

```
review_cycle,project,permission,via,account_id,display_name,account_type,user_status,last_seen_in_jira,owner,decision,reason,change_ticket,changed_on,verified_on
2026-Q4,ABC,BROWSE_PROJECTS,group contractors,<account-id>,<person>,atlassian,Active,2026-06-02,<project owner>,remove,Contract ended in August,ABC-123,2026-10-14,2026-10-15
```

The six columns after `review_cycle` are the first six columns of the permissions report script's output, so its rows paste straight in (drop its last column, `active`). `user_status` and `last_seen_in_jira` come from the org user export, joined on account ID. The rest is filled in during the review. Keep `via`: when an auditor picks a line and asks why that person had access, it's the answer.

## Doing it with Access Assertions

I build Access Assertions, so weigh this accordingly. A quarterly review tells you what access looked like on one day. The app checks the rules behind the review every night in between: who may browse a project, who must never browse it, who may administer it, and which groups issue security may grant to. Each rule covers up to 25 company-managed projects or team-managed software spaces on sites of up to 4,000 active accounts (1,250 for admin rules). With findings on, a break opens one Jira ticket naming the path to the account, and results are kept for 400 days and export as CSV or JSON, so the evidence for the whole year is there at audit time. [Access Assertions on the Atlassian Marketplace](https://marketplace.atlassian.com/apps/370337795).

Since 3.20.0 the app can also run the review on this checklist. You name reviewers and a review period on a rule. The app opens each review on schedule and lists everyone the rule covers and how each person got in, as of that day. That's the population pull and the `via` column from one capture instead of a script. Reviewers mark each row keep, remove or exception (an exception needs a reason and an end date no more than a year out), then sign. With findings on, the removals go into one Jira ticket by row number and access path, with no names in it. Later scans check each removal, and the signed review shows whether it happened, which covers most of "make the changes and prove them". A signed review copies out as CSV, JSON or HTML for the archive. It only sees what its rule covers, so the site-level pulls and the last-seen dates from the user export are still yours to do.

## Sources

- View, grant, revoke global permissions: https://support.atlassian.com/jira-cloud-administration/docs/view-grant-revoke-global-permissions-in-jira/
- Space access and configuration permissions (Browse spaces; Jira admins inherit Administer spaces): https://support.atlassian.com/jira-cloud-administration/docs/space-access-and-configuration-permissions/
- Export users from a site (columns, "Last seen in"): https://support.atlassian.com/organization-administration/docs/export-users-from-a-site/
- Grant or revoke permissions in a scheme (grantee types): https://support.atlassian.com/jira-cloud-administration/docs/grant-or-revoke-permissions-in-a-scheme/
- Permissions limitations in Free Jira sites: https://support.atlassian.com/jira-cloud-administration/docs/permissions-limitations-in-free-jira-sites/
- Audit activities in Jira (location, what's logged, export): https://support.atlassian.com/jira-cloud-administration/docs/audit-activities-in-jira-applications/
- Team-managed space permissions: https://support.atlassian.com/jira-software-cloud/docs/next-gen-permissions/
- Jira Cloud REST API v3: https://developer.atlassian.com/cloud/jira/platform/rest/v3/
