---
title: "How to get a Jira Cloud permissions report, per project and per person"
description: "Jira Cloud has no permissions report you can export. A short REST script can write one to CSV: every project, every permission, every person, and the grant that put them there."
layout: page
---

You want a table you can hand to someone: for each project, who holds which permission, and how they got it. Jira Cloud will show you the pieces on separate screens, but it won't give you that table. There's no permissions report and no export of one in the UI. The REST API has everything you need, though, and the script below turns it into a CSV in one run.

Jira's menus now say "space" for project, so you'll see **Browse spaces** and **Administer spaces** on screen. The REST API still uses the old names: `BROWSE_PROJECTS` comes back from the API labelled "Browse Projects", and `ADMINISTER_PROJECTS` as "Administer Projects". This guide covers company-managed projects.

## Where permissions actually come from

Every company-managed project uses one permission scheme. The scheme is a list of grants, and each grant gives one permission to one holder. The holder can be:

- a project role, whose members differ per project and can be users or groups
- a group
- a single user
- application access (everyone who can use the app)
- **Public**, which means anyone, logged in or not
- **Any logged in user**
- the reporter, the current assignee, or the space lead
- whoever is named in a user or group picker field on the work item

That list is the full set of grantee options on Atlassian's "Grant or revoke permissions in a scheme" page. A report that only reads the People page sees one of them, and only half of it, since a role that holds a group shows the group's name and not its members.

So a proper report walks the whole chain: project, scheme, grant, holder, and then role and group membership down to accounts.

## The UI route, and why it doesn't scale

For one project you can do it by hand. Open the project, select **More actions** (•••) next to its name, then **Space settings > Permissions**. That page lists each permission and its grantees. Then go to **Space settings > People** to see who holds each role. For groups, an organization admin can export members from admin.atlassian.com: **Directory > Users > Export users**, then "Users from selected groups only" with the Group membership option. Atlassian emails a CSV link.

That's three screens and an emailed export per project. For thirty projects it's an afternoon, and it's stale as soon as someone edits a group.

## The script

This one uses only the Python standard library. Create an API token for your Atlassian account, then:

```
export JIRA_SITE=https://your-site.atlassian.net
export JIRA_EMAIL=you@example.com
export JIRA_TOKEN=your-api-token
python jira_permissions_report.py > permissions.csv
```

By default it reports Browse spaces and Administer spaces. Pass a comma-separated list to choose others, for example `python jira_permissions_report.py BROWSE_PROJECTS,SET_ISSUE_SECURITY`.

```python
import base64, csv, json, os, sys, urllib.error, urllib.request

SITE = os.environ["JIRA_SITE"]  # https://your-site.atlassian.net
AUTH = base64.b64encode(f"{os.environ['JIRA_EMAIL']}:{os.environ['JIRA_TOKEN']}".encode()).decode()
PERMS = sys.argv[1].split(",") if len(sys.argv) > 1 else ["BROWSE_PROJECTS", "ADMINISTER_PROJECTS"]
cache = {}

def get(path):
    if path not in cache:
        req = urllib.request.Request(SITE + path, headers={"Authorization": "Basic " + AUTH, "Accept": "application/json"})
        with urllib.request.urlopen(req) as r:
            cache[path] = json.load(r)
    return cache[path]

def paged(path):
    start = 0
    while True:
        page = get(f"{path}&startAt={start}&maxResults=50")
        yield from page["values"]
        if page.get("isLast", True) or not page["values"]:
            return
        start += len(page["values"])

def user(via, account_id):
    try:
        return (via, get(f"/rest/api/3/user?accountId={account_id}"))
    except urllib.error.HTTPError as e:  # e.g. an account Jira can't find: note it and carry on
        return (via, {"accountId": account_id, "displayName": f"(lookup failed: {e.code})"})

def members(group_id, via):
    try:
        users = list(paged(f"/rest/api/3/group/member?groupId={group_id}&includeInactiveUsers=true"))
    except urllib.error.HTTPError as e:
        return [(via, {"displayName": f"(lookup failed: {e.code})"})]
    return [(via, u) for u in users] or [(via + " (empty group)", None)]

def expand(project_key, holder):
    """Turn one grant holder into (via, user) pairs. user is None when Jira can't name the people."""
    kind = holder["type"]
    if kind == "group":
        return members(holder["value"], f"group {holder['parameter']}")
    if kind == "user":
        return [user("single user", holder["parameter"])]
    if kind == "projectRole":
        role = get(f"/rest/api/3/project/{project_key}/role/{holder['parameter']}")
        rows = []
        for actor in role.get("actors", []):
            if "actorGroup" in actor:
                g = actor["actorGroup"]
                rows += members(g["groupId"], f"role {role['name']} > group {g['name']}")
            elif "actorUser" in actor:
                rows.append(user(f"role {role['name']}", actor["actorUser"]["accountId"]))
        return rows or [(f"role {role['name']} (empty)", None)]
    # applicationRole, anyone, projectLead, reporter, assignee, custom fields: not a fixed list of people
    return [(f"{kind} {holder.get('parameter', '')}".strip(), None)]

out = csv.writer(sys.stdout, lineterminator="\n")
out.writerow(["project", "permission", "via", "account_id", "display_name", "account_type", "active"])
try:
    for p in paged("/rest/api/3/project/search?orderBy=key"):
        if p.get("style") != "classic":
            print(f"skipped {p['key']}: team-managed", file=sys.stderr)
            continue
        scheme = get(f"/rest/api/3/project/{p['key']}/permissionscheme")["id"]
        for grant in get(f"/rest/api/3/permissionscheme/{scheme}/permission?expand=all")["permissions"]:
            if grant["permission"] in PERMS:
                for via, u in expand(p["key"], grant["holder"]):
                    u = u or {}
                    out.writerow([p["key"], grant["permission"], via, u.get("accountId", "-"),
                                  u.get("displayName", ""), u.get("accountType", ""), u.get("active", "")])
except urllib.error.HTTPError as e:  # project, scheme or role: the report would have a hole in it
    sys.exit(f"stopped: HTTP {e.code} on {e.url}. The CSV is incomplete; fix access and run it again.")
```

What it calls, in order:

1. `GET /rest/api/3/project/search` to list projects, 50 at a time. Each project has a `style`: `classic` is company-managed, `next-gen` is team-managed.
2. `GET /rest/api/3/project/{key}/permissionscheme` for the scheme ID.
3. `GET /rest/api/3/permissionscheme/{id}/permission?expand=all` for every grant in the scheme. Projects that share a scheme share the cached answer.
4. `GET /rest/api/3/project/{key}/role/{roleId}` for a role's actors on that project. Actors are users and groups, not expanded.
5. `GET /rest/api/3/group/member?groupId=...&includeInactiveUsers=true` to expand each group, 50 members per page at most. Inactive users are left out unless you ask for them, and you want them: an inactive account in a sensitive group is something to clean up.
6. `GET /rest/api/3/user?accountId=...` for role members added one by one, so their account type and status are filled in.

Run it as a Jira admin. Reading a project's role actors needs Administer spaces on that project or Administer Jira, and expanding a group needs Browse users and groups or Administer Jira. The user lookups need Browse users and groups.

If one user or group lookup fails, for example a 404 for an account Jira can't find, the script writes that row with `(lookup failed: 404)` as the name and carries on. A failure on a project, scheme or role stops the run with the HTTP status, because the CSV would have a hole in it.

## What the output looks like

Here's a trimmed run against a small test site with five company-managed projects, done on 10 October 2026. Account IDs and names are replaced:

```
project,permission,via,account_id,display_name,account_type,active
FIN,ADMINISTER_PROJECTS,role Administrators (empty),-,,,
FIN,BROWSE_PROJECTS,group contractors,<account-id>,<person>,atlassian,True
FIN,BROWSE_PROJECTS,group finance-team,<account-id>,<person>,atlassian,True
FIN,BROWSE_PROJECTS,role atlassian-addons-project-access,<account-id>,Jira Outlook,app,True
GOV,BROWSE_PROJECTS,applicationRole,-,,,
HR,BROWSE_PROJECTS,group hr-team (empty group),-,,,
HR,ADMINISTER_PROJECTS,group it-helpdesk,<account-id>,<person>,atlassian,True
PAY,BROWSE_PROJECTS,group contractors,<account-id>,<person>,atlassian,True
```

The full file had 124 rows. 110 of them were app accounts, 11 apps on each of the five projects for both permissions, all held through the `atlassian-addons-project-access` role. Filter on `account_type` before anyone reads it: `atlassian` is a person, `app` is an app.

Three rows are worth checking:

- `GOV,BROWSE_PROJECTS,applicationRole,-`. That project uses the stock "Default software scheme", which grants Browse spaces through an `applicationRole` grant with no app named. Depending on how it was set up, the scheme page shows it as **Any logged in user** or **Application access**. Either way it isn't a list of people you chose, so the script can't turn it into names, and if the project should be private, remove it. Get that population from the user export, with the App access option.
- `contractors` appears on both FIN and PAY. That's what you'd want to catch.
- `(empty group)` and `(empty)` rows are grants that reach nobody today. They're harmless until someone joins the group.

## What the script can't see

Be upfront about these when the CSV goes to a reviewer.

- **Jira admins.** People with Administer Jira inherit Administer spaces on every project, per Atlassian's permission reference, without appearing in any scheme. List them from **Settings > System > Global permissions** and add them to the report yourself.
- **Broad and per-item grants.** Application access, Public, Any logged in user, reporter, assignee, space lead and picker-field grants come out as one row with `-` for the account. Reporter and assignee depend on the work item, so there's no fixed list.
- **Work item security.** Browse spaces gets you into the project. A security level can still hide individual work items from you, so a project-level report overstates what some people see.
- **Team-managed projects.** They don't use permission schemes. The script skips them and prints `skipped KEY: team-managed` to stderr so they don't vanish silently.
- **App accounts.** They're in the output, labelled `app`. Decide with your auditor whether they belong in the review; most teams list them separately.

There's also a shortcut that's tempting: `GET /rest/api/3/user/permission/search?projectKey=ABC&permissions=BROWSE_PROJECTS&query=` resolves everything in one call. It's fine for a quick answer. But it doesn't say how each person got in, and Atlassian documents that it only looks "up to the thousandth user" before filtering, so on bigger sites it misses people.

## Keeping the report useful

Save the CSV with the run date in the file name and keep the raw JSON responses next to it if this is audit evidence. Run it again after any scheme change or reorg. And diff the runs: a new row in a sensitive project is the thing you're actually looking for, not the 124 rows that were there last time.

## Doing it with Access Assertions

I build Access Assertions, so take this paragraph with that in mind. Instead of reading a report, you write the rule the report is meant to check, as a sentence: who may browse a project, who must never browse it, who may administer it, or which groups issue security may grant to. It walks the same chain every night, scheme to grant to role to group to account, across up to 25 company-managed projects per rule. Each rule comes back PASS, FAIL or UNKNOWN. If you turn findings on, a FAIL or UNKNOWN opens one Jira ticket naming the path to the account, and results export as CSV or JSON and are kept for 400 days. Data stays in Forge storage on Atlassian. [Access Assertions on the Atlassian Marketplace](https://marketplace.atlassian.com/apps/370337795).

## Sources

- Grant or revoke permissions in a scheme (grantee types): https://support.atlassian.com/jira-cloud-administration/docs/grant-or-revoke-permissions-in-a-scheme/
- Space access and configuration permissions (Browse spaces; Jira admins inherit Administer spaces): https://support.atlassian.com/jira-cloud-administration/docs/space-access-and-configuration-permissions/
- Change which permission scheme a space uses (Space settings > Permissions): https://support.atlassian.com/jira-cloud-administration/docs/change-which-permission-scheme-a-space-uses/
- View, grant, revoke global permissions: https://support.atlassian.com/jira-cloud-administration/docs/view-grant-revoke-global-permissions-in-jira/
- Export users from a site: https://support.atlassian.com/organization-administration/docs/export-users-from-a-site/
- Team-managed space permissions: https://support.atlassian.com/jira-software-cloud/docs/next-gen-permissions/
- Jira Cloud REST API v3 (projects, permission schemes, project roles, groups, users): https://developer.atlassian.com/cloud/jira/platform/rest/v3/
