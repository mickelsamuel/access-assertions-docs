---
title: "Guides for Jira Cloud admins"
description: "Jira Cloud access guides: who can see a project, permissions reports, private projects, issue security levels, and user access reviews for SOC 2 and ISO 27001."
layout: page
---

Practical guides on Jira Cloud access, written from the API and checked on a live site. Each one solves the problem with native Jira first.

- **[Who can access a Jira Cloud project? How to list it and export it](who-can-access-a-jira-project.md)**  
  There's no export button for project users in Jira Cloud. Here's how to build the real list from permission schemes, roles, groups and the REST API.
- **[Running a Jira Cloud user access review for SOC 2 or ISO 27001](jira-user-access-review-soc2-iso27001.md)**  
  What auditors want from a Jira access review, how to pull the population natively in Jira Cloud, and which evidence to keep before the audit log expires.
- **[Jira Cloud audit log retention: the 180-day limit and how to keep access evidence longer](jira-audit-log-retention.md)**  
  Jira Cloud has two audit logs and neither keeps much more than six months. What each one holds, and how to keep access evidence for a full audit year.
- **[How to list every project administrator in Jira Cloud](list-jira-project-administrators.md)**  
  Jira Cloud can't export project admins across all projects from the UI. A short REST script can, once you know where Administer spaces really comes from.
- **[Jira Permission Helper vs an access review: which question each one answers](permission-helper-vs-access-review.md)**  
  Jira's Permission Helper explains why one user can or can't act on one work item. An access review asks who can. Here's when to use each, natively.
- **[How to get a Jira Cloud permissions report, per project and per person](jira-permissions-report.md)**  
  Jira Cloud has no permissions report you can export. A short REST script can write one to CSV: every project, every permission, every person, and the grant that put them there.
- **[How to make a Jira Cloud project private (and keep it that way)](make-a-jira-project-private.md)**  
  Restrict a company-managed Jira Cloud project to specific people: give it its own permission scheme, grant Browse spaces to one role, remove the broad grants, then check it stayed closed.
- **[Jira Cloud issue security levels: who can actually see a secured work item](jira-issue-security-levels.md)**  
  How Jira Cloud security levels work with Browse spaces, who ends up seeing a work item that has a level, which plans include them, and the mistakes that leave secured work visible.
- **[Jira Cloud user access review checklist (with a CSV template)](jira-user-access-review-checklist.md)**  
  A quarterly access review checklist for Jira Cloud: what to pull, what to flag, how to record each decision and how to prove removals happened. Includes a plain CSV template.
