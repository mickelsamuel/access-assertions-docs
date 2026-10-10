---
title: "How to list every project administrator in Jira Cloud"
description: "Jira Cloud can't export project admins across all projects from the UI. A short REST script can, once you know where Administer spaces really comes from."
layout: page
---

You want one list: every project, and every person who can administer it. Maybe it's for a privileged-access review, maybe you just want to email them all. Jira Cloud can show you this one project at a time, but there's no cross-project report or export in the UI. A short script against the REST API gets you there.

Jira Cloud's menus now say "space" for project, so the permission is labelled **Administer spaces**. In the API it's still `ADMINISTER_PROJECTS`.

## First, what makes someone a project admin

It isn't the "Administrator" role by itself. A person can administer a company-managed project when that project's permission scheme grants them **Administer spaces**, and the scheme can grant it to:

- a project role (most schemes give it to the Administrators role)
- a group
- a single user
- other grantee types such as application access or a user picker field, which are rare for this permission but possible

**Jira admins** have it everywhere without appearing on any project. Atlassian's permission reference says "Jira admins automatically inherit the Administer spaces permission." They're listed under **Settings > System > Global permissions** against *Administer Jira*.

Being **space lead** doesn't count. Atlassian's grant page says the lead "is not the same as the Administer spaces permission". Leads often are admins too, but check rather than assume.

And role membership can be a group, so "Administrators: jira-project-admins" means everyone in that group.

## The UI route, one project at a time

1. Open the project and select More actions (•••) next to its name, then **Space settings > Permissions**. Find Administer spaces and note who it's granted to.
2. Go to **Space settings > People** and note who holds the roles from step 1, including any groups.
3. For each group, an organization admin can export members from admin.atlassian.com under **Directory > Users > Export users**, choosing "Users from selected groups only".

There's also a per-person view. In admin.atlassian.com, open a user under **Directory > Users**, then under Jira select **... (More options) > View Jira space roles**. It shows that person's role memberships across all Jira spaces. It's handy for one person, but it shows roles, so it won't show a grant made straight to a group in the scheme.

## The REST route, all projects at once

Run this as a Jira admin so every project is visible.

**1. Get the project list.**

```
GET /rest/api/3/project/search?startAt=0&maxResults=100
```

Page through with `startAt`. Each project has a `style` field: `classic` is company-managed, `next-gen` is team-managed. Handle the two separately (see below).

**2. For each company-managed project, ask who has the permission.**

```
GET /rest/api/3/user/permission/search?projectKey=ABC&permissions=ADMINISTER_PROJECTS&query=
```

This resolves roles, groups and direct grants for you. It's the shortest path to a correct list, with two catches. First, it leaves out Jira admins. On a test site in October 2026, a Jira admin who held no project role could open and save a project's settings, yet this search, `GET /rest/api/3/mypermissions` and `POST /rest/api/3/permissions/check` all reported no `ADMINISTER_PROJECTS` for them on that project. Step 4 covers adding them back. It also returns app accounts (`accountType: app`), about ten on a stock site, so filter those out. Second, Atlassian documents that it only considers users "up to the thousandth user" and then filters, so it returns fewer results than `maxResults` per page and won't see everyone on a site with more than about 1,000 users.

**3. If you need to show *why*, or you're over 1,000 users, walk the chain instead.**

```
GET /rest/api/3/project/{key}/permissionscheme
GET /rest/api/3/permissionscheme/{schemeId}/permission?expand=all
GET /rest/api/3/project/{key}/role/{roleId}
GET /rest/api/3/group/member?groupId={groupId}
```

Keep the grants where `permission` is `ADMINISTER_PROJECTS`. Many projects share a scheme, so cache scheme lookups and you'll make far fewer calls. Role actors come back as users and groups. Groups need the last call, at most 50 members per page.

**4. Add the Jira admins.** They can administer every project, and none of the API checks above will show it, so either list them once at the top or add them to each project's rows. Say which you did.

**5. Write the CSV.** One row per project and person, with the route: `project key, project name, account ID, display name, via` where "via" is something like `role Administrators > group jira-project-admins`. Keep the raw JSON next to it with the run date.

## Team-managed projects

They don't use permission schemes, so steps 2 and 3 don't apply. Admin rights come from the project's Administrator role, which is the default role that gets "Administer the space". Jira admins always have access to a team-managed space's settings. I haven't verified a REST route for team-managed role members that I'd recommend, so for a handful of team-managed projects the dependable method is to open each project's space settings and read the people list there.

## Where native Jira stops

- No cross-project report or export of project admins in the UI.
- The "users with permission" search stops at the thousandth user.
- The org user export is organization-admin only and doesn't include project roles or permission schemes.
- The per-user "View Jira space roles" page shows roles, not every grant.
- Group-usage reporting ("where is this group used?") is a long-standing feature request, JRACLOUD-71967, with over 1,000 votes. Atlassian's November 2024 update said it wouldn't be built in the next 12 to 18 months.

## Doing it with Access Assertions

I build Access Assertions, so weigh this accordingly. One of its four rule types is "who may administer this project", written as a sentence and checked nightly across up to 25 company-managed projects per rule. Each rule comes back PASS, FAIL or UNKNOWN. PASS needs every group and role behind the rule to have been read in full; if anything couldn't be read, you get UNKNOWN and the reason. When someone outside the allowed set can administer a project, it opens one Jira ticket naming the path from project to scheme, grant, role, group and account. It's read-only and runs entirely on Atlassian. [Access Assertions on the Atlassian Marketplace](https://marketplace.atlassian.com/apps/370337795).

## Sources

- Space access and configuration permissions (Administer spaces; Jira admins inherit it): https://support.atlassian.com/jira-cloud-administration/docs/space-access-and-configuration-permissions/
- Grant or revoke permissions in a scheme (grantee types; space lead is not Administer spaces): https://support.atlassian.com/jira-cloud-administration/docs/grant-or-revoke-permissions-in-a-scheme/
- Change which permission scheme a space uses (Space settings > Permissions): https://support.atlassian.com/jira-cloud-administration/docs/change-which-permission-scheme-a-space-uses/
- View, grant, revoke global permissions: https://support.atlassian.com/jira-cloud-administration/docs/view-grant-revoke-global-permissions-in-jira/
- See all project roles a user belongs to (View Jira space roles): https://support.atlassian.com/jira/kb/see-all-project-roles-a-jira-cloud-user-belongs-to-across-all-projects/
- Export users from a site: https://support.atlassian.com/organization-administration/docs/export-users-from-a-site/
- Team-managed space permissions: https://support.atlassian.com/jira-software-cloud/docs/next-gen-permissions/
- Jira Cloud REST API v3 (project search, user permission search, permission schemes, roles, groups): https://developer.atlassian.com/cloud/jira/platform/rest/v3/
- JRACLOUD-71967: https://jira.atlassian.com/browse/JRACLOUD-71967
- Community thread this guide answers: https://community.atlassian.com/forums/discussion/2581367/finding-all-project-administrators-for-specific-projects
