# Access Assertions for Jira Cloud

Access Assertions checks who can actually browse, administer, or get issue-level access to your Jira Cloud projects against rules you write once, and tells you every night whether that's still true. It's for the Jira admin who owns project access, and for whoever has to show an auditor how access was checked.

## What it does

Access reviews in Jira are usually done by hand: open project settings, expand the permission scheme, click through each role into each group, paste it all into a spreadsheet. It takes hours, it's stale the moment it's done, and it can't tell you whether the answer was complete.

Access Assertions turns that review into a rule that runs every night. Typical rules:

- Only these two groups may browse the finance project.
- This group (a former vendor, a contractor pool) must never browse the legal project, even after someone edits the permission scheme.
- Project administration on this project is limited to a named set of people.
- Issue security on this project may only grant access through these approved groups.

Since 3.20.0, it can also run the periodic access review: a dated list of everyone a rule covers, decided row by row by the people you name, then signed.

## How it works

You write an assertion as a sentence, not a query: pick a rule type, the projects it covers, and the groups or people it names. The app shows you the sentence it will publish before you commit. Four rule types ship: who may browse, who must not browse, who may administer, and which groups issue security may grant through.

Every night the app expands the real permission graph behind each rule (permission scheme, grant, project role, group, member, active account) and compares it with what you wrote. Global Jira administrators are checked separately, since they inherit project admin everywhere. Each check ends in one of three states:

- **PASS**: every required source was read and no violation exists.
- **FAIL**: at least one violation is proven. The count is exact when the run read everything it needed; otherwise it's a floor ("at least N"), never a guess.
- **UNKNOWN**: the proof didn't complete, and the result says why.

If findings are turned on, a FAIL or UNKNOWN opens a ticket in the Jira project you chose, carrying the exact grant chain that caused it. An unchanged failure doesn't open a second ticket; it's tracked as one episode until it's resolved, reopened, or the assertion is retired.

## Installation

Install from the Atlassian Marketplace onto your Jira Cloud site. The app then appears under **Settings → Marketplace apps → Access Assertions** (Settings is the gear icon at the top right). There's no separate site or login. The app declares no external endpoints, so nothing it does leaves Atlassian's infrastructure.

The app lives on a Jira administration page. Since 3.13.0, every action inside it also checks on the server that the person making the request is a Jira administrator, using Jira's own permission check for the account Atlassian attaches to the request, so creating, editing or archiving a rule, or changing where findings go, needs a Jira admin. If that check can't be completed, the action is refused. Jira's answer can take a few minutes to catch up after someone loses admin rights.

It works on company-managed projects and, since 3.15.0, on team-managed software spaces (scrum and kanban), up to 25 selected projects per assertion, on sites of up to 4,000 active accounts (1,250 for admin rules, which check every account for Jira admin rights). People, app accounts and Jira Service Management portal customers all count. A team-managed space is read through the permission scheme Jira's REST API returns for it. An Open or Limited space lets everyone with Jira access browse it, so it never passes: the result names that access level instead of listing every account. As cross-checks, the app compares its answer with Jira's own list of who can browse each space, and asks Jira's permission check about a rotating sample of accounts. If either says someone can browse where the scheme says they can't, the result is UNKNOWN, never PASS. One nightly run reads up to 25 team-managed spaces across all your assertions (28 if none of them is an admin rule). Since 3.17.0, company-managed Jira Service Management projects are checked too, for staff: agents, project admins, and anyone Jira gives Browse through a role, group or user. Portal customers aren't checked. Jira's API can't give a complete list of them, so a browse rule on a service project covers staff only and every result prints the customer-portal grant as not checked. Team-managed service spaces, team-managed business spaces, and issue-security rules on service projects or team-managed spaces are refused when you write the rule, not mismeasured later. A project is checked only when Jira's own answer says which kind it is; anything else is reported as unsupported.

## Permissions

The app asks for granular OAuth scopes only, never a classic scope like `manage:jira-configuration`, which carries 148 write operations. Atlassian grants granular scopes per endpoint, all or nothing, so a few scopes are held only because an endpoint the app needs declares them. The set is frozen; adding a scope later would force every site to re-consent.

**Read scopes (the evidence path):** `read:application-role:jira` (application-role grants), `read:group:jira` (expands a group into its members), `read:issue-security-level:jira` and `read:issue-security-scheme:jira` (the issue-security rule type), `read:permission-scheme:jira` and `read:permission:jira` (the scheme and its grants), `read:project-role:jira` (role members), `read:project:jira` (project details), `read:user:jira` (whether an account is active). Held only because those endpoints declare them: `read:avatar:jira`, `read:field:jira`, `read:issue-type-hierarchy:jira`, `read:issue-type:jira`, `read:project-category:jira`, `read:project-version:jira`, `read:project.component:jira`, `read:project.property:jira`.

**Write scopes (governance tickets only; they never touch a permission, role, group or membership):** `write:issue:jira` (create and update the ticket), `write:comment:jira` (post the grant chain and any reopen or retirement note), `read:issue:jira`, `read:comment:jira`, `read:issue-meta:jira`, `read:issue.transition:jira`, `read:status:jira` (read back, list transitions, confirm the move). Held only because the ticket endpoints declare them: `read:comment.property:jira`, `read:field-configuration:jira`, `read:field.default-value:jira`, `read:field.option:jira`, `read:issue-details:jira`, `read:issue.changelog:jira`, `read:issue.vote:jira`, `write:attachment:jira`, `write:comment.property:jira`, `write:issue.property:jira`.

**Platform:** `storage:app` (Forge storage), `report:personal-data` (Atlassian's personal-data reporting API, used to detect erased accounts).

### What's stored

Everything lives in Forge storage inside your Atlassian cloud; there's no database of ours. Stored: assertion definitions and every published revision, run results (outcome, what was and wasn't read, violation grant chains), finding state (which tickets exist and their delivery status), and, since 3.20.0, access reviews: everyone who held the reviewed access when it was captured, the reviewers, each decision, the signed record, and who published each rule revision, so a review can tell whether its signer published the rule. Results and violation evidence are kept for 400 days; revisions and current finding state for as long as the app is installed; run leases and other working data for 7 days. Signed reviews are kept for 400 days after signing, and the two newest of each rule for as long as the app is installed. No email addresses are stored. Outside access reviews, group membership is never stored beyond the accounts in a proven violation; a review stores everyone who holds the reviewed access, because each of them needs a decision. Reviewers see names, which the app looks up in Jira when the page loads. When an account is erased from your site, a scheduled cycle replaces its identifiers in stored evidence and in access reviews with an erased marker.

## First-run setup

1. Open **Settings → Marketplace apps → Access Assertions**.
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

**Exports from Current result and Evidence.** After the review step, the page makes the file and hands it to your browser, and says the file "was handed to your browser to save", with its name and size. The page can't see whether the browser saved it. If the page can't make the file, it shows the export for you to select and copy. If the export is also too large to show that way (more than a million characters), the page says so and no file is made. A result too large to load in one go gets **Prepare the export** in place of the export buttons: the page reads the result in parts first, and every stored row goes into the file. If a part doesn't match the check stored with it, the page says "The export could not be prepared, so no file was made. A part of the stored result did not match its check." If the result changes while it's being read, no file is made and the page says what to do: reload the page for the newer result, or prepare the export again. A very large result stored by an earlier version of the app isn't shown on **Current result**. The tab says "This result was stored by an earlier version of the app and is too large to show here. Export it to see every row.", and **Prepare the export** exports it. In an **Evidence** export, a result too large to read here is a gap: the file says "This result is too large to read here, so its qualifications are not in this export.", and the review step counts how many there are.

**Settings and Diagnostics.** Settings points findings at Jira. Delivery health is tracked separately from results, so a workflow that stops accepting tickets shows as a delivery problem, not a false control failure. Diagnostics lists every unresolved check across all rules in one place, with the reason and what would resolve it.

## Access reviews

Since 3.20.0, the app can run the periodic review itself. A review is a dated list of everyone a rule covers and how each person gets the access. Someone decides each row, then signs the list. The nightly check tells you whether the rule still holds; a signed review records that a person looked at who has the access and decided. Every date the app shows for a review is a UTC day.

**Set a review plan.** Open a rule and choose the **Reviews** tab. Under **Review plan**, pick **How often the rule is reviewed** (every 30, 90, 180 or 365 days; 90 to start), up to 10 **Reviewers**, and **Remind reviewers before the due date** (off, 7 days or 14 days before; 14 to start). **Require someone other than the rule's publisher to sign** starts ticked: whoever published the current version of the rule can't sign its review. If the app has no record of who published that version, it can't check this until you publish a new one. Choose **Save plan**. Named reviewers can sign reviews of that rule, and so can any Jira admin. With no reviewers named, only Jira admins can sign.

**Reviews open on schedule.** A rule's first review is due 14 days after you first save its plan. After that, each review is due one review period after the last one was signed; change the period and the due date moves with it. If the last review granted an exception that ends sooner, the next review is due the day that exception ends. A nightly run opens the review on or after its start day: the day the plan was saved for a first review, otherwise the due date minus the reminder days, or the due date itself with reminders off. A Jira admin can also choose **Open a review** at any time. Either way the app captures who holds the access at that moment. The rows show up when the capture is ready, and the page checks again by itself for two minutes. A rule has one review in progress at a time.

**Decide each row.** A row is a person, a broad grant, or a security level holder, with the paths that give them access. App accounts are listed for information and need no decision. Choose **Keep**, **Remove** or **Exception**, then **Save decisions**. An exception needs a reason of 20 to 500 characters and an end date between tomorrow and a year from today; it ends after 90 days unless you pick a date. A row the rule forbids can't be kept, so it's removed or gets an exception. **Keep everyone through** a group or role keeps every row that comes from it in one step, and rows you set one by one keep your choice. A row where the reviewer holds the access is flagged.

**Sign.** A review can be signed once every row has a decision, the capture is no more than 21 days old, and the rule hasn't changed since the capture. **Refresh access** captures it again, at most once an hour; a saved decision stays with its row unless that row's access changed. Tick **I confirm this rule is still the rule we want.**, choose **Sign this review**, then **Sign**. A note for the record is optional. Signing approves every decision on the review, whoever saved it, so if a decision or the capture changed after your page loaded, the page has you reload first. The review is recorded as **Signed**, or as **Signed with gaps** when the capture couldn't read everything it needed or couldn't list everyone who holds the access; the review lists what wasn't known. The signed review says who signed, when, and whether they signed as a Jira admin or as a named reviewer. Decisions can't change after signing. Editing the rule replaces a review in progress. A review with no activity for 30 days is closed.

**After signing.** The app doesn't change access in Jira; an admin makes the removals there. On the signed review, each row marked **Remove** then shows what later scans found, with the date: **Removal requested**, **Removal confirmed**, **Still has access**, **Access came back** or **Not known**. The app stops checking 180 days after signing, or when a newer review of the rule is signed.

**Export.** Under **Signed report**, choose **Show CSV**, **Show JSON** or **Show HTML**. The page can't save files, so the report opens in a window for you to copy, in parts when it's long. The copy can contain display names and account IDs, so treat it as personal data.

**The reviewer page.** Reviewers don't have to be Jira admins. They open **Access reviews** under **Apps** in Jira's sidebar. It lists the open reviews of the rules whose plan names them, plus the last two signed reviews of each of those rules from the past 400 days, which they can open again and copy. Reviewers see the names of everyone the rule covers and how each person gets access; the names come from Jira each time the page loads. Only a Jira admin can change a plan or open a review by hand.

**Review and reminder tickets.** With findings on, a nightly run after a review is signed opens one ticket in the governance project when the review asked for removals. The ticket names the rule and the day the review was signed. It lists each removal by row number and access path, and names no people and no account IDs: who each row is about is shown only in the app. Later runs comment when a removal changes, and the app resolves the ticket once every removal is confirmed. With reminders on, a reminder ticket opens once the review is open. It says the review is ready, when it's due, and where to open it. It's assigned through Jira's assignee field to the first reviewer on the plan that Jira accepts, and the app closes it on a nightly run after the review is signed. With findings off, no review or reminder tickets are made, and the **Reviews** tab says why; reviews still open on schedule.

**Closed accounts.** When an account is closed and erased from your site, the erasure cycle covers reviews too. A reviewer whose account was closed is taken off the plan, and the plan says so. In a signed review, a person under review, a person who decided a row, or the signer reads as an erased user, with a note that the account was closed and the date its identity was removed. In a review still in progress, people whose accounts were closed are grouped into one row.

**How long reviews are kept.** A signed review is kept for 400 days after signing. The two newest signed reviews of each rule are kept for as long as the app is installed, however old they are. An unsigned review that sits idle for 30 days is closed, and its draft decisions are deleted on a background pass after that.

One review holds up to 4,000 rows; that limit, the timing of each step and what the app does when a night goes wrong are on [Access reviews: limits, timing and what happens when](access-reviews.md).

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
