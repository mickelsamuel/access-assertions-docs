---
title: "Who can see a Jira Service Management project? Agents vs portal customers"
description: "A JSM service space has two audiences. Agents get in through roles you can list from the API. Portal customers get in through a grant Jira's own permission answers don't see. What an access review can prove for each."
layout: page
---

Short answer: a Jira Service Management project has two audiences, and only one of them can be listed from the API. Agents and admins get Browse through roles in the permission scheme, and you list them the same way as on any company-managed project. Portal customers get in through a separate grant, "Service Desk Customer - Portal Access", and on our test site Jira's permission check and its user search both said those customers had no Browse at all, even though the scheme granted Browse to portal access and the space was open to customers. So an access review can prove who your staff are. For customers it can't, and the honest move is to record the setting and the organizations instead.

On names: Atlassian's docs now call these "service spaces", and the settings live under **Space settings**. The REST API still says project, with `projectTypeKey` set to `service_desk`. This guide covers company-managed service spaces (`style: classic`). Everything measured below comes from a test site in October 2026, using a new service space made from the company-managed general service template and a site admin's API token.

## Who is who on a service space

- **Agents.** Atlassian's permissions page says to grant permissions "to the Service Desk Team role for agents", and Browse spaces is on its list of mandatory agent permissions.
- **Space admins.** The Administrators role.
- **Portal customers.** Atlassian is explicit: "The permissions for customers must be granted to the Service Desk Customer - Portal Access security type." Customers reach the portal, not Jira itself.
- **Organizations.** Groups of customers who share requests. Atlassian notes that "Customers who aren't in an organization can't share requests."
- **App accounts.** More of them than you'd expect, as below.

## What the scheme looks like

Our new space came with its own scheme, named "Jira Service Management Permission Scheme for Space KEY". It had 124 grants: 102 to project roles and 22 to the portal-access holder. In the API that holder has the type `sd.customer.portal.only` and no parameter, so there's nothing to expand.

**Browse** went to four holders: Administrators, atlassian-addons-project-access, Service Desk Team and portal access. **Administer** went to Administrators and atlassian-addons-project-access only. The 22 portal-access grants included Browse, create, edit and delete work items, and set work item security. None of them was Administer. That's how the template set things up on our site, so read your own scheme before you assume the same.

The roles held users only, no groups, out of the box. Administrators held the site admin. The add-on role held 12 app accounts. Service Desk Team held those same 12 apps plus the admin, so on a fresh space the "agent" role is mostly apps. Drop accounts whose `accountType` is `app` before you count agents. When we read it, before adding any customers, the Service Desk Customers role was empty and held neither Browse nor Administer.

## Listing agents and admins

This part works. Use the same calls as for any company-managed project:

```
GET /rest/api/3/project/{projectIdOrKey}/permissionscheme
GET /rest/api/3/permissionscheme/{schemeId}/permission?expand=all
GET /rest/api/3/project/{projectIdOrKey}/role/{id}
GET /rest/api/3/group/member?groupId={groupId}
GET /rest/api/3/user/permission/search?projectKey=KEY&permissions=BROWSE_PROJECTS&query=
```

Keep the Browse grants, expand roles into actors and groups into members, and compare the result with user search. Our [team-managed guide](who-can-see-a-team-managed-jira-project.md) has a script that does both. Set the `sd.customer.portal.only` grant aside, because it doesn't name anyone.

For staff, every source agreed on our site: the scheme walk, user search and `POST /rest/api/3/permissions/check`. That held for the admin, an app account, a Jira Software user with no role and a user with no product access.

**Licences don't gate Browse.** We put a Jira Software user, with no Jira Service Management access, into Service Desk Team. The scheme walk and user search both said that user could browse, straight away. The permission check said no at first, then yes about two minutes later. So all three ended up agreeing: someone in the agent role without a JSM licence still counts as able to browse the space. Review the role membership, not just the licence list. You can see a person's product access with `GET /rest/api/3/user?accountId=...&expand=applicationRoles`.

## Portal customers: what the API doesn't tell you

We created two customers. One was added to the service space directly. The other only belonged to an organization that was linked to the space. Both came back with `accountType` set to `customer`, account IDs shaped like `qm:xxxx:xxxx`, active, with no product access and no groups. Both showed up in `GET /rest/api/3/users/search`.

Then we asked Jira whether they could browse:

- `POST /rest/api/3/permissions/check` said Browse **false** for both.
- `GET /rest/api/3/user/permission/search` listed **neither**.

That's while the scheme granted Browse to portal access and the space was open to customers. Jira's two programmatic answers to "who can browse this project?" don't count portal access at all. A review built on them will say "no customers can see this space", and nothing in the scheme or the space's settings backs that up.

**The customer list isn't a roster either.** `GET /rest/servicedeskapi/servicedesk/{serviceDeskId}/customer` is marked experimental in Atlassian's spec. Before we added a customer, it listed only the site admin. Right after the add call returned 204, it still listed only the admin, and the same was true when we read it again about three minutes later. Filtering it with `query` set to the start of both test customers' display names returned nothing.

**Organizations are readable.** `GET /rest/servicedeskapi/servicedesk/{serviceDeskId}/organization` returned the linked organization, and `GET /rest/servicedeskapi/organization/{organizationId}/user` returned its one member. Atlassian's spec says the first needs a service desk agent and the second an admin or agent. This is the most solid customer-side evidence the public API gives you.

**The open or restricted setting has no public API.** Atlassian describes the two options like this. Open "defaults to the permissions inherited from the site's global permissions for customers". Restricted means "Only customers directly invited to the current service space by an agent or admin can access the portal". We searched Atlassian's published REST specs for Jira and for Jira Service Management and found no endpoint that returns this setting. The UI reads it from an internal endpoint, which we won't build on and you shouldn't either. On our test site, a brand-new service space came up open, with public sign-up turned on. That follows the site's own customer settings, so yours may differ, but it's worth knowing that the default we saw let anyone on the web sign up.

## What an access review can and can't prove

It **can** prove, from the scheme and the API:

- who can open every request as staff: agents, admins, and anyone else who gets Browse through a role, group or user grant
- who can administer the space
- which people in the agent role don't have a Jira Service Management licence

It **can't** prove, from any public API we found:

- who the portal customers are
- whether the portal accepts anyone who signs up
- whether a customer can see a given request (that depends on who raised it and who it was shared with)

What to do instead:

1. **Record the setting.** Open **Space settings > Access > Customer permissions** (Atlassian says you need to be a space admin) and capture whether it's Open or Restricted, with the date. If it's Open, record the site-level customer access settings too, since Open inherits them. Site admins control those.
2. **List the organizations** linked to the space and their members, using the two endpoints above. Keep the raw output.
3. **Treat "Open" as a decision.** If the space is open, the space owner should say so in the review, in writing. "Anyone can raise requests here" is a fine answer when it's the intended one.
4. **Report staff and customers separately.** Never let a "no Browse" for a customer count as evidence. On our site, Jira gave that answer for customers on a space whose scheme granted portal access and whose customer access was open.

## What about Access Assertions?

I build Access Assertions, a Jira Cloud app that checks access rules every day, so read this with that in mind. It checks company-managed software and business projects today: you write a rule as a sentence, it walks the scheme, roles and groups, and each rule comes back PASS, FAIL or UNKNOWN. It doesn't check service spaces, so for those the steps above are the way to go. [Access Assertions on the Atlassian Marketplace](https://marketplace.atlassian.com/apps/370337795).

## Sources

All read on 10 October 2026.

- Customize Jira Service Management permissions (Service Desk Team, Portal Access): https://support.atlassian.com/jira-cloud-administration/docs/customize-jira-service-management-permissions/
- Change service space customer permissions (menu path, Open and Restricted, space admin): https://support.atlassian.com/jira-service-management-cloud/docs/change-project-customer-permissions/
- Change access settings for your customers (site-level settings, site admins): https://support.atlassian.com/jira-service-management-cloud/docs/change-global-customer-permissions/
- How to manage access to Jira Service Management projects and requests (organizations, sharing; updated 30 July 2026): https://support.atlassian.com/jira/kb/manage-access-jira-service-management-projects-requests/
- Jira Cloud REST API v3 OpenAPI spec (Get bulk permissions, Find users with permissions, Get user with `applicationRoles`): https://developer.atlassian.com/cloud/jira/platform/swagger-v3.v3.json
- Jira Service Management Cloud REST API OpenAPI spec (Get customers, experimental; Get organizations; Get users in organization): https://developer.atlassian.com/cloud/jira/service-desk/swagger.v3.json
