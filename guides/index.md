---
title: "Guides for Jira Cloud admins"
description: "How to see who can access a Jira Cloud project, run a user access review for SOC 2 or ISO 27001, and keep access evidence longer than the audit log does."
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
