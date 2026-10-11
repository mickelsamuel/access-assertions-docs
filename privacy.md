# Privacy Policy — Access Assertions for Jira

**Effective: August 15, 2026**

Access Assertions for Jira ("the app") is built and operated by Mickel Samuel. This policy describes what data the app handles and what it does with it. It is short because the app's data footprint is small on purpose.

## Where your data lives

Everything the app stores lives in Atlassian Forge storage, inside your own Atlassian cloud. The app declares no external endpoints: it cannot send data to any server outside Atlassian — no analytics, no crash reporting, no third-party services of any kind. There is no infrastructure of ours for your data to reach.

## What the app stores

- **Assertion definitions** — the access rules you write, every published revision.
- **Run results** — outcomes (PASS / FAIL / UNKNOWN), coverage, violation evidence including the exact grant chains that caused a failure, and unresolved-check codes.
- **Finding state** — which Jira tickets the app created for findings, and their delivery status.
- **Erasure and budget records** — counts-only reports of completed data-erasure cycles, and each run's resource spend.

## Personal data

Violation evidence can contain **Atlassian account IDs and the display names captured when a rule was published** — because that is what the evidence is: proof that a specific grant reaches a specific account. The app never stores email addresses, and it never stores membership lists of people who *don't* violate a rule — only the accounts that actually appear in a proven violation are retained.

## Retention

Result summaries and violation evidence are kept for 400 days, then swept. Assertion revisions and current finding state are kept while the app is installed. Run leases and intermediate working data are kept for 7 days.

## Erasure (GDPR)

The app implements Atlassian's personal-data reporting requirements. When an account is erased, a scheduled scrub cycle removes that account's identifiers from retained evidence and replaces them with an erased marker — the evidence keeps its structure, the person is gone. The app's own erasure reports contain counts and run identifiers only, never account identifiers.

## What the app writes to your Jira

Only governance tickets, in the project you choose, and only if you turn findings on. The app never changes a permission, scheme, role, group, or membership.

A findings ticket names the people in a violation, with the display name captured when the rule was published and their Atlassian account ID. Once it's written, that text is ordinary issue content in your Jira. The erasure cycle above cleans the app's own stored evidence, but it can't rewrite text already in an issue, and Atlassian keeps "names or email addresses typed into a page or issue" after an account is deleted ([Atlassian: delete your account](https://support.atlassian.com/atlassian-account/docs/delete-your-atlassian-account/)). If that matters for you, keep findings off, or limit who can browse the governance project.

## Logging

Operational logs contain reason codes, request counts, timings, and hashed identifiers — no display names, no raw account IDs, no assertion contents.

## Questions

mickelsamuel.b@gmail.com
