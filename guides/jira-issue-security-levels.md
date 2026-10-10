---
title: "Jira Cloud issue security levels: who can actually see a secured work item"
description: "How Jira Cloud security levels work with Browse spaces, who ends up seeing a work item that has a level, which plans include them, and the mistakes that leave secured work visible."
layout: page
---

Security levels are how Jira Cloud hides individual work items inside a project that a lot of people can browse. HR complaints in an otherwise open HR project, a security incident in the engineering project, a customer escalation only managers should read. The feature is simple once you see the two checks it runs, and most of the surprises come from forgetting one of them.

Some naming first, because Jira has renamed things. Issues are now "work items" in the UI, so the feature appears as **work item security schemes** and the permission as **Set work item security**. Projects are "spaces". The REST API hasn't moved: the permission key is `SET_ISSUE_SECURITY`, the system field is called "Security Level", the endpoints live under `/rest/api/3/issuesecurityschemes`, and in JQL the field is `level`.

## Two checks, both required

To see a work item that has a security level, a person has to pass two checks.

1. **Browse spaces on the project.** It comes from the project's permission scheme. Atlassian describes it as letting users "view a space, see its work items, and search for them", with one exception: "unless work items have been hidden by work-level security settings".
2. **Membership of that work item's level.** The level's own grantee list decides this. Atlassian's support page says: "If someone isn't assigned to a security scheme, they won't be able to view those work items, even if they can access the rest of the space." The API's description of Set Issue Security says the same in fewer words: "only people in that security level can see the issue."

So a level can only narrow access. It never lets someone into a project they can't browse. If you put a contractor group on a level but the project's permission scheme doesn't give them Browse spaces, they still see nothing.

A work item without a level is visible to everyone who can browse the project.

## How levels are put together

A **work item security scheme** holds one or more **security levels**. Each project uses at most one security scheme, and each work item has at most one level. The Security Level field is single-valued (its API type is `securitylevel`), so levels don't stack on one work item. If two audiences need to see the same item, they both have to be grantees of one level.

Each level has its own list of grantees, picked from the same options a permission scheme offers:

- space roles
- groups and single users
- application access, Public and Any logged in user
- current assignee and reporter
- the space lead
- user and group picker field values

Schemes are managed under **Settings > Work items**, in the Work item attributes section, at **Work item security schemes**. It needs the Administer Jira global permission. From there you add levels, add grantees to each level, and mark one level as the default if you want one. Then, in each project's **Space settings**, expand **Work items**, select **Security**, and use **Actions > Select a scheme** (**Use a different scheme** if it already has one), then **Associate**.

## Who can set a level

Changing the level on a work item needs the **Set work item security** permission in the project's permission scheme. Atlassian also says the Work item security field has to be on the space's Layout page.

Keep that permission narrow. Whoever holds it decides which level a work item sits in, and a wider level means a wider audience.

## Which plans have it

Not Free. Atlassian's page on Free plan limits says Free sites can't "configure work item security schemes", and the only way to change existing ones there is "to upgrade to Standard, Premium, or Enterprise versions of Jira." Standard, Premium and Enterprise all have it.

## Common mistakes

**Leaving the reporter out.** Someone raises a sensitive ticket, it gets a level, and they can no longer see their own work item. Add Reporter to any level used for things people report about themselves.

**A broad grantee on the level.** Application access, Any logged in user or Public on a security level means anyone who can browse the project passes the second check. That's the same as having no level.

**No default level.** A level only protects work items that carry it, and people forget to set it. If everything in a project should be secured, mark a level as the default; Atlassian describes that as applying the level "to all work items in the spaces that use this scheme". Then use the third API call below to look for strays.

**Expecting sub-tasks to have their own level.** They don't. Atlassian's knowledge base says "Sub-tasks inherit the issue security level of their parent" and "it's not possible to assign security levels to sub-tasks". A sub-task under an unsecured parent is unsecured.

**Moving work items between spaces.** A moved work item takes the default security level of its new space. If the new space has no security scheme, Atlassian says it becomes "visible to anyone with access to the space." Check the level after every move of sensitive work.

**Trusting the project-level list.** A list of who can browse a project overstates who can see secured work. When someone asks "who can see the incident tickets?", the answer is the overlap of the Browse spaces population and the level's grantees, not either one alone.

**Using levels to hide a whole project.** If no one outside a group should see any of the project, don't secure every work item. Restrict Browse spaces in the permission scheme instead. Atlassian also points to data classification for restricting an entire space.

## Checking it with the API

Three calls cover most audits:

```
GET /rest/api/3/project/{key}/issuesecuritylevelscheme
GET /rest/api/3/issuesecurityschemes/level/member?schemeId={id}&expand=all
GET /rest/api/3/search/jql?jql=project%3DABC%20AND%20level%20is%20EMPTY
```

The first returns the project's security scheme. It needs Administer Jira or Administer spaces on that project. When a project has no scheme you get a 404, which on a test site on 10 October 2026 looked like this:

```
404
{"errorMessages": ["Security level for project SEC does not exist."], "errors": {}}
```

The second lists the grantees of every level in a scheme, 50 per page by default; keep raising `startAt` until `isLast` is true. It needs Administer Jira, and Atlassian notes it only returns members "in the context of classic projects", meaning company-managed ones. Group and role grantees come back as groups and roles, so expand them the same way you would for a permission scheme.

The third finds work items in a project that have no level at all. On a project that's supposed to be secured by default, anything it returns is visible to the whole project.

## Doing it with Access Assertions

I build Access Assertions, so I'm not neutral here. One of its four rule types covers this: "issue security on this project may only grant access through these approved groups". You write it once as a sentence for up to 25 company-managed projects, and every night it reads the security scheme behind each project and checks every level's grantees against the groups you approved. A grant outside that set comes back as FAIL. If any part couldn't be read, the result is UNKNOWN with the reason, never a quiet PASS. If you turn findings on, a FAIL or UNKNOWN opens one Jira ticket naming the path. [Access Assertions on the Atlassian Marketplace](https://marketplace.atlassian.com/apps/370337795).

## Sources

- What are work item security schemes: https://support.atlassian.com/jira-cloud-administration/docs/configure-issue-security-schemes/
- Create a new work item security scheme and security levels: https://support.atlassian.com/jira-cloud-administration/docs/create-a-new-work-item-security-scheme-and-security-levels/
- Grant users access to security levels: https://support.atlassian.com/jira-cloud-administration/docs/grant-users-access-to-security-levels-in-a-work-item-security-scheme/
- Space access and configuration permissions (Browse spaces): https://support.atlassian.com/jira-cloud-administration/docs/space-access-and-configuration-permissions/
- Work item permissions (Set work item security): https://support.atlassian.com/jira-cloud-administration/docs/work-item-permissions/
- Permissions limitations in Free Jira sites: https://support.atlassian.com/jira-cloud-administration/docs/permissions-limitations-in-free-jira-sites/
- Sub-tasks and security levels: https://support.atlassian.com/jira/kb/allow-specific-users-to-view-tasks-and-subtasks-with-security-levels-in-jira/
- Jira Cloud REST API v3 (issue security schemes): https://developer.atlassian.com/cloud/jira/platform/rest/v3/api-group-issue-security-schemes/
