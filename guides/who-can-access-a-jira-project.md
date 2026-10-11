---
title: "Who can access a Jira Cloud project? How to list it and export it"
description: "There's no export button for project users in Jira Cloud. Here's how to build the real list from permission schemes, roles, groups and the REST API."
layout: page
---

Short answer: Jira Cloud has no "export users" button for a project, and the People page doesn't show everyone who can get in. You can still build an accurate list. It takes either a few screens plus a group export, or one REST call per project.

A note on names first. Jira Cloud's menus now say "space" where they used to say "project" (Space settings, Browse spaces, Administer spaces). The REST API still says project, and the permission keys are still `BROWSE_PROJECTS` and `ADMINISTER_PROJECTS`. This guide covers company-managed projects.

## Why the People page isn't the answer

Access to a company-managed project comes from its permission scheme. The permission that matters is **Browse spaces**: Atlassian calls it "a prerequisite for almost all permissions in a space." That permission can be granted to a project role, but it can also go to a group directly, to everyone with application access, to "Any logged in user", to a single user, to the reporter or assignee, or to whoever is named in a user or group picker field.

Space settings > People only shows who sits in which project role. If the scheme also grants Browse spaces to a group, or to everyone with Jira access, those people never appear on the People page. And when a role holds a group, you see the group's name, not its members.

So the full list is: everyone the scheme reaches through Browse spaces, with every group expanded into people.

## Method 1: the screens (no scripting)

This works if you administer the project. For the group step you'll need an organization admin.

1. **Find the scheme and its grants.** Open the project, select More actions (•••) next to its name in the sidebar, then **Space settings > Permissions**. Project admins can view this page, though only Jira admins can change the scheme. Find **Browse spaces** and note every grantee.
2. **List role members.** Go to **Space settings > People**. Note which users and which groups hold each role that Browse spaces is granted to.
3. **Expand the groups.** An organization admin can export group members from admin.atlassian.com: **Directory > Users > Export users**, choose "Users from selected groups only", and tick Group membership. Atlassian emails a CSV link. Only organization admins can run this export.
4. **Check for the broad grants.** If Browse spaces goes to "Any logged in user" or to application access, the honest answer is "everyone with access to the site" (or "everyone with Jira"). The user export with the App access option tells you who that is.

Fine for one project. It doesn't scale to fifty, and you'll redo it every time someone changes a group.

## Method 2: one REST call per project

The quickest native route is **Find users with permissions**:

```
GET /rest/api/3/user/permission/search?projectKey=ABC&permissions=BROWSE_PROJECTS&query=
```

It returns users who hold the permission in that project however they got it, so groups and scheme grants are already resolved. The permission you need to call it is *Administer Projects* for that project (or *Administer Jira* for any project). A project admin can run it without being a Jira admin.

Two catches in Atlassian's own description of this endpoint:

- It "takes the users in the range defined by `startAt` and `maxResults`, up to the thousandth user", then filters. Pages come back with fewer results than you asked for, so keep paging until the range runs out. On sites with more than about 1,000 users it won't see everyone. Atlassian's suggestion for that case is to pull all users with `GET /rest/api/3/users/search` and filter in your own code.
- Privacy settings apply, so email addresses can be hidden. Keep the `accountId` as the key in your export.
- App accounts come back too. On a fresh site I tested in October 2026, one project returned 12 accounts: one person and 11 app accounts (Jira Outlook, Microsoft Teams for Jira Cloud and so on). Each result has an `accountType`, so keep `atlassian` (people) and set `app` aside before you count anyone.

If you're a Jira admin and want the answer per user rather than per project, `POST /rest/api/3/permissions/check` takes an `accountId`, a list of project permissions and up to 1,000 project IDs, and returns the projects where that user has the permission.

## Method 3: rebuild the path (when someone asks "why")

An auditor, or your security team, will sometimes want to know *how* a person got in. For that, walk the chain yourself:

```
GET /rest/api/3/project/{projectKeyOrId}/permissionscheme
GET /rest/api/3/permissionscheme/{schemeId}/permission?expand=all
GET /rest/api/3/project/{projectIdOrKey}/role/{roleId}
GET /rest/api/3/group/member?groupId={groupId}&includeInactiveUsers=true
```

The second call lists every grant in the scheme. Keep the `BROWSE_PROJECTS` ones. The third returns a role's "actors" for that project, which are users and groups, not expanded. The fourth expands a group, 50 members per page at most. It skips inactive users by default, so set `includeInactiveUsers=true` if your counts have to match the directory.

You end up with rows like: project > scheme > grant > role > group > person. That's slower than Method 2, but it's the only version that explains itself.

## Things that change the answer

- **Work item security.** Security levels can hide individual work items from people who can browse the project. They're set under **Settings > Work items > Work item security schemes**, and a project-level list overstates what those people actually see.
- **Team-managed projects** have no permission scheme screen, but the REST API still returns one (`GET /rest/api/3/project/{projectIdOrKey}/permissionscheme`), scoped to that one project and rewritten by Jira when the access level (Open, Limited or Private) changes. Access comes from that level plus roles, so they need their own pass.
- **Free plan.** Permission schemes, project roles and work item security aren't available on Free Jira sites.
- **Jira admins** automatically inherit Administer spaces, per Atlassian's permission reference. Decide whether your list should include them.

## Exporting the result

None of these endpoints writes a CSV. Save the raw JSON, convert it to CSV, and put the run date in the file name. If this is for an audit, keep both files: the CSV for the reviewer, the JSON as the evidence that the CSV came from Jira.

Native limits, in one place: no project-level export in the UI; the user export is organization-admin only and doesn't include project roles or permission schemes; the "users with permission" search stops at the thousandth user; and the group-usage feature request (JRACLOUD-71967, over 1,000 votes) got an Atlassian update in November 2024 saying it wouldn't be built in the next 12 to 18 months.

## Doing it with Access Assertions

I build Access Assertions, a Jira Cloud app for exactly this check, so treat this paragraph accordingly. You write a rule as a sentence, for example "only these groups may browse these projects", and it checks the rule every night. Each rule gets PASS, FAIL or UNKNOWN. PASS only appears when every group and role behind the rule was read in full; otherwise you get UNKNOWN with the reason. If you turn findings on, a failure opens one Jira ticket naming the full path from project to account, and the results export as CSV and JSON. It's read-only and runs entirely on Atlassian. It handles company-managed projects and team-managed software spaces, up to 25 projects per rule. [Access Assertions on the Atlassian Marketplace](https://marketplace.atlassian.com/apps/370337795).

## Sources

- Grant or revoke permissions in a scheme (grantee types): https://support.atlassian.com/jira-cloud-administration/docs/grant-or-revoke-permissions-in-a-scheme/
- Change which permission scheme a space uses (Space settings > Permissions): https://support.atlassian.com/jira-cloud-administration/docs/change-which-permission-scheme-a-space-uses/
- Restrict space access to certain groups or users (Space settings > People): https://support.atlassian.com/jira/kb/restrict-project-access-to-certain-groups-or-users-in-jira-cloud/
- Space access and configuration permissions (Browse spaces, Administer spaces): https://support.atlassian.com/jira-cloud-administration/docs/space-access-and-configuration-permissions/
- Export users from a site (organization admin only, columns): https://support.atlassian.com/organization-administration/docs/export-users-from-a-site/
- Grant users access to security levels: https://support.atlassian.com/jira-cloud-administration/docs/grant-users-access-to-security-levels-in-a-work-item-security-scheme/
- Team-managed space permissions: https://support.atlassian.com/jira-software-cloud/docs/next-gen-permissions/
- Jira Cloud REST API v3 (user search, permissions, permission schemes, project roles, groups): https://developer.atlassian.com/cloud/jira/platform/rest/v3/
- Community thread, project admins can view Space settings > Permissions: https://community.atlassian.com/forums/Jira-questions/How-can-I-quickly-tell-whether-a-specific-user-can-access-a/qaq-p/2879498
- JRACLOUD-71967, group usage feature request: https://jira.atlassian.com/browse/JRACLOUD-71967
