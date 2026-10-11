# Access Assertions for Jira Cloud

Access Assertions checks who can actually browse, administer, or get issue-level access to your Jira Cloud projects against rules you write once, and tells you every night whether that's still true. It's for the Jira admin who owns project access, and for whoever has to show an auditor how access was checked.

## What it does

Access reviews in Jira are usually done by hand: open project settings, expand the permission scheme, click through each role into each group, paste it all into a spreadsheet. It takes hours, it's stale the moment it's done, and it can't tell you whether the answer was complete.

Access Assertions turns that review into a rule that runs every night. Typical rules:

- Only these two groups may browse the finance project.
- This group (a former vendor, a contractor pool) must never browse the legal project, even after someone edits the permission scheme.
- Project administration on this project is limited to a named set of people.
- Issue security on this project may only grant access through these approved groups.

## How it works

You write an assertion as a sentence, not a query: pick a rule type, the projects it covers, and the groups or people it names. The app shows you the sentence it will publish before you commit. Four rule types ship: who may browse, who must not browse, who may administer, and which groups issue security may grant through.

Every night the app expands the real permission graph behind each rule (permission scheme, grant, project role, group, member, active account) and compares it with what you wrote. Global Jira administrators are checked separately, since they inherit project admin everywhere. Each check ends in one of three states:

- **PASS**: every required source was read and no violation exists.
- **FAIL**: at least one violation is proven. The count is exact when the run read everything it needed; otherwise it's a floor ("at least N"), never a guess.
- **UNKNOWN**: the proof didn't complete, and the result says why.

If findings are turned on, a FAIL or UNKNOWN opens a ticket in the Jira project you chose, carrying the exact grant chain that caused it. An unchanged failure doesn't open a second ticket; it's tracked as one episode until it's resolved, reopened, or the assertion is retired.

## Installation

Install from the Atlassian Marketplace onto your Jira Cloud site. The app then appears under **Jira settings → Apps → Access Assertions**. There's no separate site or login. The app declares no external endpoints, so nothing it does leaves Atlassian's infrastructure.

The app lives on a Jira administration page. Since 3.13.0, every action inside it also checks on the server that the person making the request is a Jira administrator, using Jira's own permission check for the account Atlassian attaches to the request, so creating, editing or archiving a rule, or changing where findings go, needs a Jira admin. If that check can't be completed, the action is refused. Jira's answer can take a few minutes to catch up after someone loses admin rights.

It works on company-managed projects and, since 3.15.0, on team-managed software spaces (scrum and kanban), up to 25 selected projects per assertion, on sites of up to 4,000 active accounts (1,250 for admin rules, which check every account for Jira admin rights). People, app accounts and Jira Service Management portal customers all count. A team-managed space is read through the permission scheme Jira's REST API returns for it. An Open or Limited space lets everyone with Jira access browse it, so it never passes: the result names that access level instead of listing every account. As cross-checks, the app compares its answer with Jira's own list of who can browse each space, and asks Jira's permission check about a rotating sample of accounts. If either says someone can browse where the scheme says they can't, the result is UNKNOWN, never PASS. One nightly run reads up to 25 team-managed spaces across all your assertions (28 if none of them is an admin rule). Since 3.17.0, company-managed Jira Service Management projects are checked too, for staff: agents, project admins, and anyone Jira gives Browse through a role, group or user. Portal customers aren't checked. Jira's API can't give a complete list of them, so a browse rule on a service project covers staff only and every result prints the customer-portal grant as not checked. Team-managed service spaces, team-managed business spaces, and issue-security rules on service projects or team-managed spaces are refused when you write the rule, not mismeasured later. A project is checked only when Jira's own answer says which kind it is; anything else is reported as unsupported.

## Permissions

The app asks for granular OAuth scopes only, never a classic scope like `manage:jira-configuration`, which carries 148 write operations. Atlassian grants granular scopes per endpoint, all or nothing, so a few scopes are held only because an endpoint the app needs declares them. The set is frozen; adding a scope later would force every site to re-consent.

**Read scopes (the evidence path):** `read:application-role:jira` (application-role grants), `read:group:jira` (expands a group into its members), `read:issue-security-level:jira` and `read:issue-security-scheme:jira` (the issue-security rule type), `read:permission-scheme:jira` and `read:permission:jira` (the scheme and its grants), `read:project-role:jira` (role members), `read:project:jira` (project details), `read:user:jira` (whether an account is active). Held only because those endpoints declare them: `read:avatar:jira`, `read:field:jira`, `read:issue-type-hierarchy:jira`, `read:issue-type:jira`, `read:project-category:jira`, `read:project-version:jira`, `read:project.component:jira`, `read:project.property:jira`.

**Write scopes (governance tickets only; they never touch a permission, role, group or membership):** `write:issue:jira` (create and update the ticket), `write:comment:jira` (post the grant chain and any reopen or retirement note), `read:issue:jira`, `read:comment:jira`, `read:issue-meta:jira`, `read:issue.transition:jira`, `read:status:jira` (read back, list transitions, confirm the move). Held only because the ticket endpoints declare them: `read:comment.property:jira`, `read:field-configuration:jira`, `read:field.default-value:jira`, `read:field.option:jira`, `read:issue-details:jira`, `read:issue.changelog:jira`, `read:issue.vote:jira`, `write:attachment:jira`, `write:comment.property:jira`, `write:issue.property:jira`.

**Platform:** `storage:app` (Forge storage), `report:personal-data` (Atlassian's personal-data reporting API, used to detect erased accounts).

### What's stored

Everything lives in Forge storage inside your Atlassian cloud; there's no database of ours. Stored: assertion definitions and every published revision, run results (outcome, what was and wasn't read, violation grant chains), and finding state (which tickets exist and their delivery status). Results and violation evidence are kept for 400 days; revisions and current finding state for as long as the app is installed; run leases and other working data for 7 days. No email addresses are stored, and group membership is never stored beyond the accounts in a proven violation. When an account is erased from your site, a scheduled cycle replaces its identifiers in stored evidence with an erased marker.

## First-run setup

1. Open **Jira settings → Apps → Access Assertions**.
2. In **Settings**, pick the project and issue type for governance tickets, then turn findings on. You can skip this; assertions still run and produce results either way.
3. In **Assertions**, choose **New assertion**. Pick a rule type, name it, choose up to 25 company-managed projects, then the groups or people the rule is about (the field is labelled for the rule type: "Who may browse", "Who must not browse", "Who may administer", or "Approved groups"). Read the sentence the app builds, since that sentence is the rule, and publish it.

   Not sure who should be on the list? Choose **Start from a project's current access** instead. Pick a company-managed project and the app reads who can browse it today: its permission scheme, the roles that scheme lets browse, and the groups behind Jira product access. It lists them, you tick the ones who are supposed to have that access, and it tells you what the first run should report before you publish. A group that gives people access to a Jira product arrives unticked, and a grant to everyone can't go into a rule at all; the app says where to change it in Jira. Reading changes nothing in Jira.
4. Press **Run now**, or wait for the nightly scan.
5. Read the result. FAIL shows the grant path behind each violation, with every step labelled: project, permission scheme, permission, role or group, account. UNKNOWN says why the proof couldn't finish and what would fix it.

## Using the app

**Overview** is a triage queue: delivery failures first, then failures, then everything else, with the latest run and a count of what's passing. A badge flags a rule whose name is shared by another, so duplicates don't blend together.

**Assertions** lists every published rule. Archived rules are hidden by default; a filter brings them back. Opening a rule shows three tabs: **Current result** (the latest outcome, the violation table, and the controls Run now, Export CSV, Export JSON and Archive; exports show a review step with the record count and caveats first), **History** (every retained result), and **Definition** (the rule as a sentence, plus what Jira calls each group and project right now next to what it was called when the rule was published).

**Findings** tracks ticket episodes: ticket not created yet, open, reopened, resolved, retired, or delivery failed (fix the governance project or issue type in Settings, then run again). Each episode shows when its violation was first seen and when a run last saw it. "No active episodes" says nothing about whether assertions pass.

**Evidence** has two modes. *Results* shows what was proven in a date window, with a completeness summary so anything not yet read is reported as not yet known rather than reading as a pass. *Runs* shows what the scanning process actually did, kept separate because a finished run proves nothing about a rule it never reached. Both modes export more than one result at a time.

**Settings and Diagnostics.** Settings points findings at Jira. Delivery health is tracked separately from results, so a workflow that stops accepting tickets shows as a delivery problem, not a false control failure. Diagnostics lists every unresolved check across all rules in one place, with the reason and what would resolve it.

## Nightly scan

Forge runs the scan once a day; you don't pick the time. It walks the full permission graph behind every active rule and commits a new result with its evidence. Archiving a rule takes effect on the next scan: the retirement comment posts to any open ticket and the ticket closes, recording that the finding stopped being measured rather than fixed. The rule stops being scanned the moment you archive it, and every published revision stays readable and exportable.

Results and the evidence behind them are kept for 400 days, so a full audit year is still there at fieldwork. Jira's own audit log keeps 3 months by default and 6 at most.

## Pricing

Paid app, billed through the Atlassian Marketplace. Current pricing and trial terms are on the listing.

## Guides

Practical guides for Jira Cloud admins, each solving the problem with native Jira first:

- [Who can access a Jira Cloud project? How to list it and export it](guides/who-can-access-a-jira-project.md)
- [Running a Jira Cloud user access review for SOC 2 or ISO 27001](guides/jira-user-access-review-soc2-iso27001.md)
- [Jira Cloud audit log retention: the 180-day limit and how to keep access evidence longer](guides/jira-audit-log-retention.md)
- [How to list every project administrator in Jira Cloud](guides/list-jira-project-administrators.md)
- [Jira Permission Helper vs an access review](guides/permission-helper-vs-access-review.md)
- [How to get a Jira Cloud permissions report, per project and per person](guides/jira-permissions-report.md)
- [How to make a Jira Cloud project private (and keep it that way)](guides/make-a-jira-project-private.md)
- [Jira Cloud issue security levels: who can actually see a secured work item](guides/jira-issue-security-levels.md)
- [Jira Cloud user access review checklist (with a CSV template)](guides/jira-user-access-review-checklist.md)
- [Who can see a team-managed Jira project? Open, Limited, Private and what the API says](guides/who-can-see-a-team-managed-jira-project.md)
- [Who can see a Jira Service Management project? Agents vs portal customers](guides/who-can-see-a-jira-service-management-project.md)

## Support

- Email: support@migrationpilot.dev
- Hours: Monday to Friday, 9:00 AM to 6:00 PM Eastern Time (Canada). Replies within one business day.
- Issues: https://github.com/mickelsamuel/access-assertions-docs/issues

## Privacy and terms

- [Privacy Policy](privacy.md)
- [Terms of Service](terms.md)
