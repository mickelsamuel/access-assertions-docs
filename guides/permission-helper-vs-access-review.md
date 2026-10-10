---
title: "Jira Permission Helper vs an access review: which question each one answers"
description: "Jira's Permission Helper explains why one user can or can't act on one work item. An access review asks who can. Here's when to use each, natively."
layout: page
---

People reach for the Permission Helper when someone asks "who can see this project?", and it doesn't fit. It was built for a different question. Both are worth knowing, and mixing them up costs time in an audit.

## What the Permission Helper does

It's in Jira under **Settings > System**, then **Admin helper > Permission helper**. You enter three things:

- a user (leave it blank to test anonymous access)
- a work item key
- a permission, such as Browse spaces or Edit work items

Select Submit and it lists the permissions that user has on that work item. Tick **Show failed conditions** and it explains why a permission is missing. Atlassian's description is that it "can help you figure out why a user can or can't see a certain work item."

Only Jira administrators can open it, and it's meant for company-managed projects.

For support tickets it's excellent. "Priya can't see FIN-212" goes from twenty minutes of clicking through schemes to one screen.

## Why it can't run an access review

An access review asks the opposite question. Not "can this person do this?" but "who can do this, and should they?" The Helper doesn't fit that for a few concrete reasons.

**You have to know whom to ask about.** The Helper tests a name you type. A review exists to find the names you *didn't* think of: the contractor in a group someone added to a role two years ago, or everyone with Jira access because Browse spaces went to "Any logged in user".

**It's one combination at a time.** One user, one work item, one permission. Fifty projects and 300 users is 15,000 checks before you even pick a second permission.

**It leaves no record.** There's no export and no history, so an auditor has nothing to look at afterwards.

**It's one work item, not the project.** Work item security levels can make two items in the same project behave differently, so a single item doesn't stand in for the whole project.

## The native tools that do fit a review

Jira Cloud has no review feature, but the REST API covers the "who" question reasonably well.

**Who can access one project:**

```
GET /rest/api/3/user/permission/search?projectKey=ABC&permissions=BROWSE_PROJECTS&query=
```

It returns the users who hold the permission, with groups and roles already resolved. Swap in `ADMINISTER_PROJECTS` for admins. Callers need Administer Projects on that project or Administer Jira. Atlassian documents a limit: it only considers users "up to the thousandth user", so very large sites have to work through the permission scheme, role actors and group members instead.

**Which projects one user can access:**

```
POST /rest/api/3/permissions/check
{
  "accountId": "<user>",
  "projectPermissions": [
    { "permissions": ["BROWSE_PROJECTS"], "projects": [10001, 10002] }
  ]
}
```

Up to 1,000 projects per call, and you need Administer Jira to check someone other than yourself. This is the scalable version of what people try to do with the Helper.

**Role memberships for one user:** in admin.atlassian.com, open the user under **Directory > Users**, then under Jira select **... (More options) > View Jira space roles**. It covers roles, not grants made straight to groups in a scheme.

## Side by side

| | Permission Helper | Access review |
|---|---|---|
| Question | Why can or can't this user do this? | Who can do this, and should they? |
| Input | One user, one work item, one permission | A set of projects |
| Finds people you didn't name | No | That's the point |
| Leaves a record | No | Has to, or it didn't happen |
| Native support | Built in (Jira admins) | REST API plus your own process |

## Using both well

Use the Helper when a person reports a problem, or right after a change, to confirm one specific path. Use a review on a schedule, project by project, with the population pulled from the API and saved with a date. When the review turns up something odd ("why is this person on the list?"), that's the moment to open the Helper and ask it why.

## Doing it with Access Assertions

I build Access Assertions, so this is my own product. It sits on the review side of the table. You write rules as sentences (who may browse a project, who must not, who may administer it, which groups issue security may grant to) and it checks them nightly. Each rule gets PASS, FAIL or UNKNOWN, and it only says PASS when every group and role behind the rule was read in full. A failure opens one Jira ticket that names the path from project to account, which is the "why" the Helper would give you, for everyone at once. Results export as CSV and JSON. It's read-only and runs on Atlassian, for company-managed projects. [Access Assertions on the Atlassian Marketplace](https://marketplace.atlassian.com/apps/370337795).

## Sources

- Use the Jira Admin Helper: https://support.atlassian.com/jira-cloud-administration/docs/use-the-jira-admin-helper/
- Check a user's access from a work item (Show failed conditions): https://support.atlassian.com/jira-cloud-administration/docs/check-a-users-access-from-a-work-item/
- Grant or revoke permissions in a scheme ("Any logged in user" and other grantees): https://support.atlassian.com/jira-cloud-administration/docs/grant-or-revoke-permissions-in-a-scheme/
- See all project roles a user belongs to: https://support.atlassian.com/jira/kb/see-all-project-roles-a-jira-cloud-user-belongs-to-across-all-projects/
- Jira Cloud REST API v3 (Find users with permissions, Get bulk permissions): https://developer.atlassian.com/cloud/jira/platform/rest/v3/
