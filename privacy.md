# Privacy Policy — Access Assertions for Jira

**Effective: October 14, 2026**

Access Assertions for Jira ("the app") is built and operated by Mickel Samuel. This policy describes what data the app handles and what it does with it. It is short because the app's data footprint is small on purpose.

## Where your data lives

Everything the app stores lives in Atlassian Forge storage, inside your own Atlassian cloud. The app declares no external endpoints: it cannot send data to any server outside Atlassian — no analytics, no crash reporting, no third-party services of any kind. There is no infrastructure of ours for your data to reach.

## What the app stores

- **Assertion definitions** — the access rules you write, every published revision.
- **Run results** — outcomes (PASS / FAIL / UNKNOWN), coverage, violation evidence including the exact grant chains that caused a failure, and unresolved-check codes.
- **Finding state** — which Jira tickets the app created for findings, and their delivery status.
- **Access reviews** — each time a review captures access (when it opens, on its schedule or by hand, and when someone refreshes it): everyone who holds the reviewed access, so the reviewer can decide each one; the reviewers you name on the rule's plan; each decision; the signed record; and whether each removal was later confirmed.
- **Who published each rule revision** — the publisher's account ID, so a review can flag someone signing off on their own rule.
- **Where the nightly run picks up** — one small versioned record of where the nightly run picks up its work the next night: a version number, the ids of up to two rules, the UTC date on which the run that noted each one started, and a count of the rules whose removal watch was due that night. It holds no personal data.
- **Erasure and budget records** — counts-only reports of completed data-erasure cycles, and each run's resource spend.

## Personal data

Violation evidence can contain **Atlassian account IDs and the display names captured when a rule was published** — because that is what the evidence is: proof that a specific grant reaches a specific account. The app never stores email addresses. Outside access reviews, it doesn't store lists of people who *don't* violate a rule: only the accounts in a proven violation are kept. An access review is the exception. When a review opens, on its schedule or by hand, the app stores everyone who holds the reviewed access, because the reviewer has to decide each of them. Reviewers, signers and rule publishers are stored by account ID. A rule's review plan keeps its reviewers' account IDs for as long as they're named on it. Reviewers see the names of the people under review: the app looks those names up in Jira each time a page loads and doesn't store them.

## Retention

Result summaries and violation evidence are kept for 400 days, then swept. Assertion revisions and current finding state are kept while the app is installed. Run leases and intermediate working data are kept for 7 days. Run records and closed findings are kept for 400 days. Signed access reviews are kept for 400 days after signing, and the two most recent for each rule are kept for as long as the app is installed. An unsigned review that sits idle for 30 days is closed, and its draft decisions are deleted on a background pass after that. Deletion runs in the background: a record goes on a pass after it reaches its age, not on the day itself. How long a pass takes grows with the number of rules: on a site with a few hundred rules it is hours for most records and a few days for what an interrupted run left behind; at about a thousand rules, a few days and about three weeks. When the app is uninstalled, it deletes nothing itself: Forge keeps the app's stored data for 28 days after the uninstall. A reinstall starts as a new installation without the old data, unless a request to relink it reaches Atlassian within 21 days of the uninstall. Write to the address under Questions if you want that.

## Erasure (GDPR)

The app implements Atlassian's personal-data reporting requirements. When an account is erased, a scheduled scrub cycle removes that account's identifiers from retained evidence and replaces them with an erased marker — the evidence keeps its structure, the person is gone. This covers access reviews too: a closed account's identity is removed from the review, a closed reviewer is taken off the rule's plan, and the record says so. Text a reviewer types (an exception reason, a note at signing) is kept as typed and erasure doesn't change it, so the app asks for no names there. The app's own erasure reports contain counts and run identifiers only, never account identifiers.

## What the app writes to your Jira

Only governance tickets, in the project you choose, and only if you turn findings on. The app never changes a permission, scheme, role, group, or membership.

A findings ticket names the people in a violation, with the display name captured when the rule was published and their Atlassian account ID. Once it's written, that text is ordinary issue content in your Jira. The erasure cycle above cleans the app's own stored evidence, but it can't rewrite text already in an issue, and Atlassian keeps "names or email addresses typed into a page or issue" after an account is deleted ([Atlassian: delete your account](https://support.atlassian.com/atlassian-account/docs/delete-your-atlassian-account/)). If that matters for you, keep findings off, or limit who can browse the governance project.

A review ticket is different: it lists removals by row number and access path, and it names no people and no account IDs. Who each row is about is shown only inside the app. The ticket the app opens when a review is too large for one review names no one either. A reminder ticket is assigned through Jira's assignee field to the first reviewer on the plan that Jira accepts, so Jira shows that person and handles a closed account the way it does for any assignee.

## Logging

Operational logs contain reason codes, request counts, timings, and hashed identifiers — no display names, no raw account IDs, no assertion contents.

## Questions

mickelsamuel.b@gmail.com
