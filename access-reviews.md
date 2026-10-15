# Access reviews: limits, timing and what happens when

The [README](README.md#access-reviews) walks through an access review from plan to export. This page is for the Jira admin who has a large rule, a review that didn't open when expected, or a ticket that looks wrong. Every date the app shows for a review is a UTC day. A "nightly run" is the scan that runs once a day, at a time you don't pick.

## How big a review can be

One review can hold 4,000 rows, 12,000 access paths or 8 MB of access data. App accounts count as rows, though they need no decision. What the app could not check counts toward the size limit, and counts more than access it could read. So the size the page prints is a count, not a file size.

A review over any of the three can't be signed. Its page gives its own numbers and says what to do, like this:

> This review covers 5,200 rows, app accounts included, and 131,000 access paths. One review can hold 4,000 rows, 12,000 access paths or 8 MB of access data. Split the rule into smaller rules, for example one per project, and open a review for each.

When the data size is what's over, the first sentence gives it too: "…and its access data counts as 14.2 MB".

The **Reviews** tab marks such a review **Too large**. The Overview says "Review too large: split the rule". The reviewer page tells reviewers to ask a Jira admin to split the rule. With findings and reminders on, a nightly run opens a ticket in the governance project in place of the reminder. That ticket names no one. It stays open while the rule is too large for one review, and the app closes it on a nightly run after a review of the rule fits, after the rule is archived, or after its reminders are turned off.

What to do: split the rule into smaller rules, one per project for example. Each rule gets its own review.

**A capture too large to read.** A review can be within those limits and still hold a capture too large for the page to read. The review then says "The access capture for this review is too large to read here. Narrow the rule, for example to fewer projects, then use Open a review on the Reviews tab to start a new one." Its decisions can't be saved or signed. A reviewer who isn't a Jira admin reads "The access capture for this review is too large to read here. Ask a Jira admin to narrow the rule, for example to fewer projects, and open a new review." Waiting or choosing **Refresh access** makes the same capture, so narrow the rule.

## When things happen

**The first review.** It's due 14 days after you first save the rule's plan. A nightly run can open it from the day of that save.

**Every later review.** It's due one review period after the last review was signed, counted with the period the plan has now. If that review granted an exception that ends sooner, the next review is due the day the exception ends. Its start day is the due date minus the reminder days, or the due date itself with reminders off.

**Opening.** A nightly run opens the review on or after its start day. It isn't always the first run that could:

- If that night's check of the rule had gaps, the app waits for a night without them. After three days of waiting it opens the review anyway, and the review lists the gaps.
- When many reviews are due on one night, the app opens as many as fit in that run. The rest open on a later night, ahead of reviews that came due after them.

From the start day until the review is open, the **Reviews** tab and the Overview say "The app opens it on a nightly run. You can also open it now with Open a review." On the due date the line reads "Review due today", and after it "Review overdue by 3 days" with its own count. Both are followed by "The app hasn’t opened it yet. It opens on a nightly run, or you can open it now with Open a review." A review past its due date counts as overdue whether or not it was opened.

**The capture.** Opening a review, or choosing **Refresh access**, starts a capture of who holds the access. While it runs, the page says "The app is reading current access for this rule. This page checks again by itself for the next two minutes; on a large site it can take longer." After those two minutes it says "The app is still reading current access for this rule. This page has stopped checking by itself. Check again in a few minutes." If a review says "The access capture for this review couldn’t be read. Refresh the access to capture it again.", choose **Refresh access**.

**How old a capture may be.** A review can't be signed on a capture more than 21 days old. The page says "This capture is more than 21 days old. Refresh it before signing."

**Refreshing.** Access can be captured again once an hour. A saved decision stays with its row unless that row's access changed. A changed row is marked **Access changed** and needs a new decision. Anyone who had the review open before the refresh reloads it before deciding or signing.

**A quiet review.** A review with no activity for 30 days is closed. The **Reviews** tab lists it as "Closed after 30 days without activity". Editing the rule ends a review in progress too: "Replaced: the rule changed after this review opened".

## Signing

The page lets you sign when all of this holds:

- Every row that needs a decision has one. App accounts need none.
- No row the rule forbids is marked **Keep**.
- Each exception has a reason of 20 to 500 characters and an end date between tomorrow and one year from today.
- The capture is no more than 21 days old.
- The rule hasn't changed since the capture.
- You ticked **I confirm this rule is still the rule we want.**
- With the separate-signer setting on, you didn't publish this version of the rule.

A Jira admin can sign any review. Anyone else can sign only reviews of rules whose plan names them.

Signing approves every decision on the review, whoever saved it, so a signature covers what your page showed. If a colleague saved a decision after your page loaded, the page says "The decisions changed since this page loaded. Reload the review, then sign." If the access was captured again, it says "The access was captured again since this page loaded. Reload the review, then sign." Choose **Reload the review**, read it again, then sign. While a signature is being recorded, saving a decision or refreshing answers "This review is being signed. Reload the review in a few minutes."

A review is recorded as **Signed with gaps** when its capture left something unproven. The review says which, in one or more of these sentences, and lists each gap under **What was not known**:

- "The capture run couldn’t finish checking this rule, so its result is not known."
- "Not every required source was read in full."
- "Not everyone who holds this access could be listed."

**The separate-signer setting and older rule versions.** The setting needs to know who published the version under review. Where the app has no record of that, the **Reviews** tab says "The app doesn’t know who published this version of the rule, so it can’t check this on the next review. Publish a new version and it will." The review page and the signed record then say the publisher is not known.

## After signing

The app never changes access in Jira. An admin makes each removal there, and later runs check for it. On the signed review, each row marked **Remove** shows one of these, with the date:

- **Removal requested**: where every removal starts, on the day of signing.
- **Removal confirmed**: two scans on one night found no path to the access.
- **Still has access**: a path was found, and the row says when it was last checked.
- **Access came back**: the removal had been confirmed, and a later run found a path again.
- **Not known**: the run couldn't settle it, and the row gives the reasons.

**How often removals are checked.** Nightly runs do the checking. The app reads signed reviews at a limited rate, so on a very large site one night may not reach every review's removals. The ones it missed go first the next night. "Last checked" is always the day a check really ran.

**When checking stops.** 180 days after signing, or at the moment a newer review of the same rule is signed. The older review then says "It stopped checking these removals on" with the day, and each row keeps its last result.

## Tickets

Tickets go to the governance project you chose for findings, and only with findings on. With findings off, the **Reviews** tab says why no tickets are made. Reviews still open on schedule.

**The review ticket.** A nightly run after a review is signed opens one ticket when the review asked for removals. Its summary reads "[Access review] Finance browse: 3 removals requested", with your rule's name and count. It gives the day the review was signed, its status, the day the access was captured, and one line per removal by row number and access path. A path never prints a person. The ticket names no people and no account IDs, and says where in the app to see who each row is about. Later runs comment when a removal's status changes. When every removal is confirmed, the app adds a closing comment and resolves the ticket. When a newer review of the rule is signed, the older ticket is closed with a comment that says so. If the 180 days run out first, the ticket gets a last comment and stays open for a person to close.

**The reminder ticket.** With reminders on, a nightly run opens it once the review is open: never before the start day, and one for each due date. Its summary reads "Access review ready for Finance browse, due 12 Nov 2026". It says the review is ready, when it's due, and where reviewers and admins open it. It names no reviewer. The app assigns it through Jira's assignee field to the first reviewer on the plan that Jira accepts. If Jira can't be reached, it tries again the next night. If Jira refuses every reviewer, the ticket stays unassigned and says so. If the due date moves while the review is open, the ticket gets a comment with the new date and its summary keeps the first one. The app closes it on a nightly run after the review is signed.

**A ticket deleted or moved in Jira.** The app stops following a ticket it can no longer see, and the review says so: "The app can no longer see the review ticket in Jira (deleted, moved or restricted)." A deleted reminder isn't made again. The next review gets a new one.

**An archived rule.** Archiving a rule closes its open reminder or too-large ticket with the comment "The rule was archived, so this review is no longer needed. This ticket is closed." A review ticket for removals is left as it is. If you restore the rule, it gets one new reminder for the same due date, and the closed ticket stays closed.

**A create that got lost.** Sometimes the app creates a ticket and never gets Jira's answer. The next run looks for the ticket by its label and carries on with the newest match that isn't done. It never takes up a ticket somebody closed; it opens a new one. If two open tickets match, it uses the newest and leaves the other for you to close.

**A night when Jira's rate limit stops delivery.** Ticket delivery stops for that run. Findings, review tickets and reminders wait for a later run. Reviews due that night still open, because opening a review needs no call to Jira. The app also paces its own reading of signed reviews, so a ticket or comment that didn't fit one night is written on a later one.

## Exports

A signed review exports as CSV, JSON or HTML under **Signed report**. The page can't save files, so the text opens in a window for you to copy.

**Parts.** A long report comes in parts of about 4 MB each. The window's title names the file and the part, such as "part 2 of 3", and **Next part** moves on. A part that fails to load is tried again three times, 20 seconds apart. Then the page says "The export stopped at part 2 of 3. Try again in a minute." The page shows a report only when all its parts belong together. When they don't, it says "The report changed while it was loading. Try again."

**What a part can leave out.** A removal line always carries its status, the date and the run that checked it. The paths that run found are printed while the export has room. At the largest sizes a part is written without display names and without that path detail, and the line ends "The paths this run found are not included in this export." Once the run's result is past its 400 days, the line ends "The paths this run found are no longer kept." A removal whose stored record can't be decoded prints "Removal status unreadable."

**The digest.** Each export's header carries an **Evidence digest**, next to who signed, how they were allowed to sign, when, and the time zone. CSV, JSON and HTML of one review carry the same digest.

An export can contain display names and Atlassian account IDs. An account erased after you made a copy can't be erased from the copy you hold.

## What is kept and for how long

| What | Kept for |
| --- | --- |
| A signed review | 400 days after signing |
| The two newest signed reviews of each rule | as long as the app is installed, however old they are |
| An unsigned review | closed after 30 days without activity; its draft decisions are deleted on a background pass after that |
| Results and violation evidence, including the paths a removal check found | 400 days |
| Run records and closed findings | 400 days |
| Working data, and what an interrupted run left behind | 7 days |

A signed review keeps its own rows, decisions, signer, signing time, note and gaps. It reads the same after the scan it was captured from has been deleted. The app doesn't store people's names in a review: it reads them from Jira each time a page or an export is shown.

Deletion runs in the background, a little at a time. A record goes on a pass after it reaches its age, not on the day itself. How long a pass takes grows with the number of rules. On a site with a few hundred rules it is hours for most records and a few days for what an interrupted run left behind; at about a thousand rules, a few days and about three weeks.

The reviewer page shows signed reviews from the past 400 days, two for each rule.

## If the nightly scan didn't run

The scheduled scan runs once a day, at a time Forge picks, or at about 23:15 UTC when the app starts it itself. Here is what the app does by itself when that goes wrong:

- If no scheduled run has started for 25 hours, or none has started today by 23:15 UTC, the app asks for the day's run itself, at most once an hour. On a site whose nightly start is after 23:15 UTC, that means the app starts the run at about 23:15 UTC.
- If a run dies as it starts, after it was recorded but before its first checkpoint, the app starts it once more, 20 to 25 minutes after the first start. It never starts a third.
- If a start dies before the run is even recorded, the app's next request for the day starts the run. A start that recent could still be alive, so the app first waits until it can't be: about 6 minutes after Forge's own start, 20 minutes after one the app asked for.
- If the previous scheduled run, or its ticket work, may still be working when the day's run is due, the day's run waits for it.
- If Jira's hourly allowance for the app is nearly used when the day's run is due, the run is put off until the allowance resets, at most three times a day. The allowance can be shared by every site that uses the app. A wait that would end after 23:30 UTC isn't taken: the run starts at once. A run that's going stops reading before it uses the last of the allowance, and the rules it couldn't confirm are checked again on the next run.
- A run that got further and then stalled isn't started again. What it didn't finish waits for the next night.

Some days still get no scheduled run:

- If the previous run, or its ticket work, may still be working at 23:30 UTC, the day is skipped. The app doesn't start a second run beside it.
- The app waits an hour before it asks for the same day's run again, so a request it makes after 23:00 UTC is the day's last. If the start that request leads to fails, the app tries again until midnight UTC. The day ends with no run only if the request never arrives, or no try gets the run started before midnight.

**Days with no scheduled run.** On **Evidence**, the **Runs** view lists them in a table, "Days with no scheduled run, newest first". Each row is a UTC day marked **No run**, with one of two sentences:

- "No scheduled run on 2026-11-12 UTC: the previous run, or its ticket work, had not finished." The app skipped that day because of the run before it, as above.
- "No scheduled run is recorded for 2026-11-12 UTC." The app holds no scheduled run for that day and no record of why.

A run started with **Run now** doesn't count as the day's scheduled run. The table never lists today, and it starts the day after the oldest run the **Runs** view lists.

**Run times.** On the **Runs** view and on **Diagnostics**, a run that ended shows one time. A **Finished** run shows "Results taken" and the time the run took its results; its tickets and comments can come after that. An **Aborted** run shows "Stopped" and the time it was ended.

What you see meanwhile:

- A review that was due to open stays unopened, and the **Reviews** tab says the app hasn't opened it yet. Choose **Open a review** to open it now. **Run now** checks the rule but doesn't open a scheduled review; only the scheduled run does.
- Removal rows keep their last result and their "last checked" day.
- Review tickets, comments and reminders arrive on a later run.

If you write to support, these names from the app's log may come up:

- `findings.search-ambiguous`: more than one open ticket matched a lost create.
- `launch.quota-postponed`: the day's scheduled run was put off until Jira's hourly allowance resets.
- `reviews.starts-after-quota-stop`: reviews were opened after that night's ticket delivery stopped.
- `reviewStartWaits` and `reviewStartsPutOff`, on the nightly run's finished line: reviews whose opening waited within the night, and reviews whose opening moved to a later night.
