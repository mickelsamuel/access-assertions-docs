---
title: "How to make a Jira Cloud project private (and keep it that way)"
description: "Restrict a company-managed Jira Cloud project to specific people: give it its own permission scheme, grant Browse spaces to one role, remove the broad grants, then check it stayed closed."
layout: page
---

A company-managed project on the stock "Default software scheme" is open to everyone who can log in to Jira. That's not a bug. On the test site I used for this guide, that scheme's **Browse spaces** grant is an `applicationRole` grant with no app named, so anyone who can log in can find the project, open it and search its work items. Making it private means changing that grant, and then making sure nothing else quietly undoes it.

Jira Cloud's menus now say "space" where they used to say "project": Space settings, Browse spaces, Administer spaces. The REST API still says project, and the permission key is still `BROWSE_PROJECTS`. Team-managed projects work differently and are covered near the end.

## What decides who can see a project

One permission does: Browse spaces. Atlassian describes it as allowing users "to view a space, see its work items, and search for them", and calls it "a prerequisite for almost all permissions in a space." If someone doesn't hold Browse spaces on a project, they can't get in.

Browse spaces comes from the project's permission scheme, and the scheme can grant it to space roles, application access, groups, Public, Any logged in user, the current assignee, the reporter, picker-field values, the space lead or single users. A private project is one where every one of those grants points at people you chose.

You need the **Administer Jira** global permission to change schemes and switch a project to another one. And on Free Jira sites none of this is possible: Atlassian's page on Free plan limits says you can't "restrict permissions using schemes" there. Standard, Premium and Enterprise can.

## Step 1: check whether the scheme is shared

Open the project, select **More actions** (•••) next to its name, then **Space settings > Permissions**. The page shows which scheme the project uses.

This is the step people skip. Permission schemes are shared, and Atlassian's own article on restricting a project warns that "this permission scheme is shared across spaces". Remove that broad grant from a shared scheme and you've locked down every project on it, not just yours. Add a group to it and that group can now browse all of them.

If the scheme is used by anything else, give the project a scheme of its own. Schemes live under **Settings > Work items**, in the Work item attributes section, at **Permission Schemes**. Create one, or copy the current one so you keep its other permissions, and edit that. Then, back on the project's **Space settings > Permissions** page, select **Actions > Use a different scheme**, pick the new scheme and select **Associate**.

## Step 2: grant Browse spaces to one role

Atlassian's own article on this uses a role rather than a list of groups in the scheme:

1. Create a space role, for example "Browse", under **Settings > System > Space roles**. Skip this if an existing role fits.
2. In the project's own permission scheme, grant **Browse spaces** to that role.
3. In the project, go to **Space settings > People**, select **Add people**, and add the groups or people who should see it, in that role.

A role keeps the scheme reusable: the scheme says "the Browse role can see this", and each project decides who's in the role. You can grant Browse spaces to a group directly instead. It works, but anyone editing the scheme later has to know which project each group was for.

## Step 3: remove the grants that re-open it

Go back to Browse spaces in the scheme and remove every grantee you didn't put there on purpose. These are the ones that re-open a private project:

- **Application access or Any logged in user.** The stock scheme grants Browse spaces through an `applicationRole` grant with no app named. Depending on how it was set up, the scheme page shows it as **Any logged in user** or **Application access**. Either way it isn't a list of people you chose, so remove it. A copied scheme still has it.
- **Public.** On Standard, Premium and Enterprise this lets anyone view the space and its work items. Atlassian notes it "doesn't require Jira app access."
- **Old groups.** Broad groups that were added years ago "for everyone in engineering" tend to survive every cleanup.
- **Reporter, assignee and picker fields.** These don't open the work items to everyone, but Atlassian documents that when Browse spaces is granted to them, "any user who is logged into your site can see the name of the space" on the spaces page. If the project name is itself sensitive, that matters.

Then save, and go to step 4. Changes to space access apply immediately.

## Step 4: check it stayed private

The People page is not the check. It shows role members, not group grants or application access. Read the scheme's grants instead. This short script lists every Browse spaces grant for the projects you name and flags the ones that aren't a named list of people:

```python
import base64, json, os, sys, urllib.request

SITE = os.environ["JIRA_SITE"]  # https://your-site.atlassian.net
AUTH = base64.b64encode(f"{os.environ['JIRA_EMAIL']}:{os.environ['JIRA_TOKEN']}".encode()).decode()
BROAD = {"anyone", "applicationRole", "reporter", "assignee", "userCustomField", "groupCustomField", "projectLead"}

def get(path):
    req = urllib.request.Request(SITE + path, headers={"Authorization": "Basic " + AUTH, "Accept": "application/json"})
    with urllib.request.urlopen(req) as r:
        return json.load(r)

for key in sys.argv[1:]:
    scheme = get(f"/rest/api/3/project/{key}/permissionscheme")
    grants = get(f"/rest/api/3/permissionscheme/{scheme['id']}/permission?expand=all")["permissions"]
    print(f"{key}: scheme {scheme['id']} '{scheme['name']}'")
    for g in grants:
        if g["permission"] != "BROWSE_PROJECTS":
            continue
        h = g["holder"]
        name = h.get("group", {}).get("name") or h.get("projectRole", {}).get("name") or h.get("parameter", "")
        flag = "  <-- not a named list of people" if h["type"] in BROAD else ""
        print(f"  Browse: {h['type']} {name}{flag}")
```

Run it with `JIRA_SITE`, `JIRA_EMAIL` and `JIRA_TOKEN` set (an API token for your account) as `python check_private.py FIN GOV SEC`. This is real output from a test site on 10 October 2026, with one scheme name shortened:

```
FIN: scheme 10001 'FIN restricted'
  Browse: projectRole atlassian-addons-project-access
  Browse: group contractors
  Browse: group finance-team
GOV: scheme 10000 'Default software scheme'
  Browse: projectRole atlassian-addons-project-access
  Browse: applicationRole   <-- not a named list of people
SEC: scheme 10004 'Security Incidents'
  Browse: projectRole atlassian-addons-project-access
  Browse: group security-team
```

GOV is still on the stock scheme, so anyone who can log in to Jira can browse it. FIN is restricted, but look at the second line: `contractors` can browse a finance project. Whether that's right is a question for whoever owns FIN, and it's exactly the kind of grant that gets added for one ticket and never removed.

The `atlassian-addons-project-access` role shows up everywhere. It holds app accounts (on that site, 11 of them: Slack, Jira Outlook, Automation for Jira and so on), not people.

To see the people behind the grants, call `GET /rest/api/3/user/permission/search?projectKey=FIN&permissions=BROWSE_PROJECTS&query=`. It resolves roles and groups in one go. Drop results whose `accountType` is `app`, and note that Atlassian says it only considers users "up to the thousandth user", so on larger sites expand the groups yourself. Our [permissions report guide](jira-permissions-report.md) has a script that does that for every project.

Then keep checking. A private project rarely gets opened on purpose. It gets opened when someone edits the shared scheme for another project, adds a broad group to the role, or moves the project to a different scheme. Jira's audit log, at **Settings > System > Audit Log**, records scheme edits and permissions added to or removed from schemes, plus users added to or removed from groups and roles. Look there after anything changes, and re-run the check on a schedule.

## Things that change the answer

- **Jira admins** inherit Administer spaces on every project, and they can edit any scheme. Keep that list short.
- **Work item security** can hide individual work items inside a project people can browse. It narrows access within the project; it doesn't make the project itself private.
- **Moving work items out.** When a work item moves to another space it takes that space's default security level, and Atlassian notes that if the new space has no security scheme, it's "visible to anyone with access to the space."

## Team-managed projects

They have no permission scheme screen. Each team-managed space has an access setting instead, and Jira rewrites a scheme scoped to that one space (readable through the REST API) whenever the setting changes. Atlassian's wording: when a space is private, "only Jira admins and people you add to the space can see it in their space directory." Open and Limited let anyone on the site view work items. So for a team-managed project, set access to Private and manage the people list.

## Doing it with Access Assertions

I build Access Assertions, so read this with that in mind. Step 4 is the one that slips: the project was private in March and nobody looked again. With the app you write the rule once as a sentence, such as "only finance-team may browse FIN", and it checks the real scheme, grants, roles and groups every night across up to 25 company-managed projects or team-managed software spaces per rule. There's a denylist rule type too, for "contractors must never browse FIN". When the rule breaks and findings are on, it opens one Jira ticket naming the path to the account, and results export as CSV or JSON. [Access Assertions on the Atlassian Marketplace](https://marketplace.atlassian.com/apps/370337795).

## Sources

- Restrict space access to certain groups or users: https://support.atlassian.com/jira/kb/restrict-project-access-to-certain-groups-or-users-in-jira-cloud/
- Space access and configuration permissions (Browse spaces): https://support.atlassian.com/jira-cloud-administration/docs/space-access-and-configuration-permissions/
- Grant or revoke permissions in a scheme (grantee types, Permission Schemes location): https://support.atlassian.com/jira-cloud-administration/docs/grant-or-revoke-permissions-in-a-scheme/
- Change which permission scheme a space uses: https://support.atlassian.com/jira-cloud-administration/docs/change-which-permission-scheme-a-space-uses/
- Use permissions to make a space publicly visible: https://support.atlassian.com/jira-cloud-administration/docs/use-permissions-to-make-a-space-publicly-visible/
- Permissions limitations in Free Jira sites: https://support.atlassian.com/jira-cloud-administration/docs/permissions-limitations-in-free-jira-sites/
- What are work item security schemes: https://support.atlassian.com/jira-cloud-administration/docs/configure-issue-security-schemes/
- Audit activities in Jira: https://support.atlassian.com/jira-cloud-administration/docs/audit-activities-in-jira-applications/
- Team-managed space permissions: https://support.atlassian.com/jira-software-cloud/docs/next-gen-permissions/
- Jira Cloud REST API v3: https://developer.atlassian.com/cloud/jira/platform/rest/v3/
