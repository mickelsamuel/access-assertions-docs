---
title: "Who can see a team-managed Jira project? Open, Limited, Private and what the API says"
description: "Team-managed Jira spaces have no permission scheme screen, but the REST API has one. How the access level shows up in it, how to list everyone who can browse, and where Jira's own answers disagreed."
layout: page
---

Short answer: a team-managed project's access level (Open, Limited or Private) decides whether everyone with Jira can browse it, and its roles decide who else can. There's no permissions page for it in the UI, but the REST API returns a real permission scheme for each team-managed project, and you can build the list of who can browse from that. Do it from more than one source, though. On a test site, Jira's bulk permission check disagreed with both the scheme and Jira's own user search, and later gave yes and no for the same person a minute apart.

On names: Atlassian's docs now say "team-managed space", and the page on access is titled "Team-managed space permissions". The REST API still says project. In the API a team-managed project has `style` set to `next-gen` (company-managed is `classic`) and `simplified: true`. Everything below was measured in October 2026 on a test site, on team-managed software spaces made from the Kanban and Scrum templates, with a site admin's API token and from a Forge app (`asApp`).

## What the three access levels mean

Atlassian's own wording, from the team-managed permissions page:

- **Open:** "anyone on your Jira site can view, create and edit work items in your space."
- **Limited:** "anyone on your Jira site can view and comment on work items in your space."
- **Private:** "only Jira admins and people you add to the space can see it in their space directory".

The roles are Administrator, Member and Viewer. On an Open space, Member is the default for everyone on the site; on a Limited space, Viewer is.

## The scheme you can't see in the UI

Team-managed spaces show no permission scheme anywhere in their settings, and plenty of advice says they don't use one (our own [who can access a project](who-can-access-a-jira-project.md) guide said so until October 2026). Through the API, they do. These two documented calls worked on our test space:

```
GET /rest/api/3/project/{projectIdOrKey}/permissionscheme
GET /rest/api/3/permissionscheme/{schemeId}/permission
```

The first returned a scheme called "KEY: Simplified Permission Scheme", with a `scope` pointing at that one project, so it isn't shared with anything else. The second returned its grants: 157 of them while the space was Open, 139 on Limited and 133 on Private. The scheme is rewritten when the access level changes.

Here's who held **Browse** (`BROWSE_PROJECTS`) at each level:

| Access level | Browse granted to |
|---|---|
| Open | a bare `applicationRole` grant, plus the roles Administrator, Member, Viewer, atlassian-addons-project-access and jira-guest-member |
| Limited | the same as Open |
| Private | the same roles, and no `applicationRole` grant |

"Bare" means the grant names no product, so it covers anyone with access to a Jira product on the site. For browsing, that one grant is the whole difference between Open or Limited and Private. Open and Limited differ in their other grants (the count drops from 157 to 139); we didn't diff those one by one. **Administer** (`ADMINISTER_PROJECTS`) went to Administrator and atlassian-addons-project-access at every level.

So you can read the browse side of the access level from the scheme: a bare `applicationRole` Browse grant means Open or Limited, and no such grant means Private. We didn't find a documented endpoint that returns the access level itself. In Atlassian's published REST spec, `accessLevel` only shows up as an input when you create a project from a custom template.

One caution: none of this is written down by Atlassian as a contract for team-managed spaces. It's what the API returned on our site, so re-check it on yours.

## Roles hold users and groups

`GET /rest/api/3/project/{projectIdOrKey}/role` listed five roles on our space: Administrator, Member, Viewer, atlassian-addons-project-access and jira-guest-member. `GET /rest/api/3/project/{projectIdOrKey}/role/{id}` returns a role's actors, and those were users (`atlassian-user-role-actor`) and groups (`atlassian-group-role-actor`, with a `groupId`). A group in a role isn't expanded for you. Use `GET /rest/api/3/group/member?groupId=...`, which pages 50 at a time.

The atlassian-addons-project-access role holds app accounts, not people. On our site that was 12 apps.

## Listing everyone who can browse

There are two ways, and they're worth running together.

**Walk the scheme.** Take the Browse grants, expand each role into its actors, and each group into its members. A bare `applicationRole` grant means "everyone with Jira product access", which you won't get from the role list.

**Ask Jira's user search.**

```
GET /rest/api/3/user/permission/search?projectKey=KEY&permissions=BROWSE_PROJECTS&query=&startAt=0&maxResults=100
```

You need Administer Projects on that project, or Administer Jira. Atlassian's description says it "takes the users in the range defined by `startAt` and `maxResults`, up to the thousandth user, and then returns only the users from that range" that match. On our site the pages behaved differently: with `maxResults=3` they came back 3, 3, 3, 3, 3, 1 and then empty, and joined end to end they matched the single large answer exactly. So we advance by the number of results received, stop at the first empty page, and de-duplicate by `accountId`. Our site had 23 accounts, so this tells you nothing about the thousandth-user limit. On a big site, expect the search to come up short and lean on the scheme walk.

On the Private space, search returned 15 accounts: 3 people and 12 apps. On Open it returned 16, the extra one being a user with Jira access and no role. Drop results whose `accountType` is `app` before you count anyone.

**Jira admins.** We added a user with no role to the Jira admins group and checked the Private space. The permission check said no, user search didn't list them, and the scheme has no grant for admins. Atlassian says Jira admins can see a private space in their directory and "always have access to your space's settings", so decide whether your review lists them. We didn't log in as that admin to see whether the work items open.

## Where Jira disagreed with itself

The third source is `POST /rest/api/3/permissions/check`. You give it an `accountId`, a list of permissions and up to 1,000 project IDs, and it returns the projects where that user holds each permission. You need Administer Jira to ask about anyone but yourself.

For each test account and each access level we compared the check, the scheme walk and user search. A user with no role, a user directly in Member, the admin, an app account and a user with no product access: all three sources agreed, every time. One case didn't. A user who was in Member **only through a group**, on the **Private** space:

- scheme walk: can browse
- user search: listed
- permissions/check: Browse **no**

It wasn't a lag of a minute or two. The first reading came a minute or two after the user joined the group, and the check stayed "no" when we re-ran it every 60 seconds for the next 8 minutes. About 25 minutes after that, we switched the space from Open back to Private, waited 20 seconds and asked three more times, a minute apart: yes, yes, no. Nothing changed between those three reads. The same check made from a Forge app (`asApp`) said yes all four times we asked, each within about three minutes of a switch to Private, which is too few reads to call it stable.

We haven't logged in as that user to see whether the space actually opens, so we can't tell you which answer is right. What we can say is narrower. A single "no" from the check, for someone who gets in through a group, isn't proof of anything. And in a separate test on a service space, the check took about two minutes to reflect a role change that user search showed straight away.

Two smaller things the check does. A project ID that doesn't exist is silently left out of the answer, which matches Atlassian's note that "invalid project and issue IDs are ignored". A made-up or empty `accountId` returns 404 "The specified user does not exist."

What we'd do:

- Use at least two sources, the scheme walk and user search, and record both.
- When the sources disagree, write down every answer with the time you got it. Don't pick one.
- After changing roles, groups or the access level, check again a few minutes later, and again after that.

## A script that does both

This lists everyone who can browse a project, the role or group that got them there, and whether user search agrees. Add `--check` to also ask the permission check about each person. It uses only Python's standard library, and the token comes from the environment:

```python
import base64, json, os, sys, urllib.parse, urllib.request

SITE = os.environ["JIRA_SITE"].rstrip("/")  # https://your-site.atlassian.net
AUTH = base64.b64encode(f"{os.environ['JIRA_EMAIL']}:{os.environ['JIRA_TOKEN']}".encode()).decode()

def call(path, params=None, body=None):
    url = SITE + path + ("?" + urllib.parse.urlencode(params) if params else "")
    headers = {"Authorization": "Basic " + AUTH, "Accept": "application/json"}
    data = None
    if body is not None:
        data = json.dumps(body).encode()
        headers["Content-Type"] = "application/json"
    with urllib.request.urlopen(urllib.request.Request(url, data=data, headers=headers)) as r:
        return json.load(r)

def group_members(group_id):
    start = 0
    while True:
        page = call("/rest/api/3/group/member", {"groupId": group_id, "startAt": start, "maxResults": 50})
        yield from page["values"]
        if page.get("isLast", True) or not page["values"]:
            return
        start += len(page["values"])

key, with_check = sys.argv[1], "--check" in sys.argv
project = call(f"/rest/api/3/project/{key}")
scheme = call(f"/rest/api/3/project/{key}/permissionscheme")
grants = call(f"/rest/api/3/permissionscheme/{scheme['id']}/permission", {"expand": "all"})["permissions"]
print(f"{key} ({project.get('style')}): scheme {scheme['id']} '{scheme['name']}'")

via = {}  # accountId -> set of paths
def add(account_id, path):
    via.setdefault(account_id, set()).add(path)

for g in grants:
    if g["permission"] != "BROWSE_PROJECTS":
        continue
    h, kind = g["holder"], g["holder"]["type"]
    if kind == "projectRole":
        role = call(f"/rest/api/3/project/{key}/role/{h.get('projectRole', {}).get('id') or h['parameter']}")
        for a in role.get("actors", []):
            if a["type"] == "atlassian-user-role-actor":
                add(a["actorUser"]["accountId"], f"role {role['name']}")
            elif a["type"] == "atlassian-group-role-actor":
                grp = a["actorGroup"]
                for u in group_members(grp["groupId"]):
                    add(u["accountId"], f"role {role['name']} > group {grp.get('name') or grp.get('displayName')}")
            else:
                print(f"  role {role['name']}: actor type {a['type']} not expanded")
    elif kind == "group":
        grp = h.get("group", {})
        for u in group_members(grp.get("groupId") or h["value"]):
            add(u["accountId"], f"group {grp.get('name') or h.get('parameter')}")
    elif kind == "user":
        add(h.get("user", {}).get("accountId") or h["parameter"], "user grant")
    elif kind == "applicationRole" and not h.get("parameter"):
        print("  Browse: bare applicationRole grant -> everyone with product access (Open or Limited)")
    else:
        print(f"  Browse: {kind} {h.get('parameter', '')} -> not a fixed list of people, not expanded")

search, start = {}, 0
while True:  # page until an empty page
    page = call("/rest/api/3/user/permission/search", {"projectKey": key, "permissions": "BROWSE_PROJECTS",
                                                       "query": "", "startAt": start, "maxResults": 100})
    if not page:
        break
    search.update({u["accountId"]: u for u in page})
    start += len(page)

def check(account_id):
    body = {"accountId": account_id, "projectPermissions": [{"permissions": ["BROWSE_PROJECTS"], "projects": [int(project["id"])]}]}
    answer = call("/rest/api/3/permissions/check", body=body)["projectPermissions"]
    return "yes" if any(int(project["id"]) in p["projects"] for p in answer) else "no"

apps = [a for a in set(via) | set(search) if search.get(a, {}).get("accountType") == "app"]
print(f"  {len(apps)} app accounts (left out below)")
for account_id in sorted((set(via) | set(search)) - set(apps)):
    u = search.get(account_id, {})
    paths = "; ".join(sorted(via.get(account_id, []))) or "no role, group or user grant (application access?)"
    line = f"  {account_id} {u.get('displayName', '')}: {paths} | user search: {'yes' if u else 'NO'}"
    if with_check:
        line += f" | permissions/check: {check(account_id)}"
    print(line)
```

Set `JIRA_SITE`, `JIRA_EMAIL` and `JIRA_TOKEN` (an API token for an account that administers the project; `--check` needs a Jira admin), then run `python who_can_browse.py KEY --check`. For our Private test space, with names and IDs replaced, the output had this shape:

```
TMA (next-gen): scheme 10042 'TMA: Simplified Permission Scheme'
  12 app accounts (left out below)
  712020:xxxx Admin User: role Administrator | user search: yes | permissions/check: yes
  712020:xxxx Member One: role Member | user search: yes | permissions/check: yes
  712020:xxxx Member Two: role Member > group space-members | user search: yes | permissions/check: no
```

The last line is the disagreement above. On an Open space you'd also see the "bare applicationRole" line, and anyone with Jira access but no role shows up as "no role, group or user grant".

## Things that change the answer

- **Business spaces.** We only measured team-managed software spaces. We couldn't create a team-managed business space through the REST API on our site, so we haven't checked whether its scheme looks the same.
- **Guests.** The scheme grants Browse to the jira-guest-member role at every level, Private included. Include it in the walk.
- **Apps.** The add-on role has Browse and Administer everywhere. That's normal, but list the apps once so nobody asks later.

## What about Access Assertions?

I build Access Assertions, a Jira Cloud app that checks access rules every day, so read this with that in mind. Since version 3.15.0 it checks team-managed software spaces too, through the same scheme, roles and groups this guide reads. You write a rule as a sentence and each rule comes back PASS, FAIL or UNKNOWN. An Open or Limited space never passes, because no list of groups can make it private. It comes back as a FAIL that names the access level, or as UNKNOWN when everyone the app samples happens to be on your list. Team-managed business spaces aren't covered yet; for those, the script above is still the tool. [Access Assertions on the Atlassian Marketplace](https://marketplace.atlassian.com/apps/370337795).

## Sources

All read on 10 October 2026.

- Team-managed space permissions (access levels, roles, Jira admins): https://support.atlassian.com/jira-software-cloud/docs/next-gen-permissions/
- Jira Cloud REST API v3 reference: https://developer.atlassian.com/cloud/jira/platform/rest/v3/
- Jira Cloud REST API v3 OpenAPI spec (descriptions of Get bulk permissions, Find users with permissions, Get assigned permission scheme, Get permission scheme grants, Get project role for project, Get users from group; the `accessLevel` field): https://developer.atlassian.com/cloud/jira/platform/swagger-v3.v3.json
