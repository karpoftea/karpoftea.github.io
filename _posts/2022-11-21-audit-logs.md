---
layout: post
title: "Audit Logs in a nutshell"
date: 2022-11-21 16:00:00 +0300
tags: [auditlogs, audittrails, saas]
comments: true
excerpt: Brief introduction for those who want to know what audit logs are.
---

* Table of contents
{:toc}

# What is audit log
An audit log (aka “audit trail”) is a security-relevant chronological record. It provides documentary evidence of the sequence of activities that have affected a specific operation, procedure, or event. The concept is simple: when a change is applied to a system that correlates with a change in the system’s behavior, that change should be documented in an audit log. Audit logs provide answers to questions regarding data, security, and system state and are therefore crucial for security and compliance — as well as for tracking systems by multiple organization stakeholders (“activity tracking”).
Usually audit log contain everything that might be interesting to your customer’s administrators in terms of traceability of system actions and administrative manners in your product.

# What audit log is not
- not a log management solution (like Loggly, Papertrail, ELK, Logz.io, Coralogix etc)

# Audit log objectives
- Demonstrating Compliance: as mentioned above, logs help you demonstrate to auditors that you are compliant within a given framework. This is precisely the reason that many frameworks require audit logs in the first place.
- Creating Chains of Evidence: As part of a security or compliance footing, many security frameworks call for logging as a form of evidence. An unbroken chain of evidence can show investigators the source of a security breach or prove that a company has implemented the security measures they say they have.
- Creating a Chain of Custody: In legal situations, how files are changed or handled can be considered evidence in a court of law. An immutable audit log provides such evidence for law enforcement.
- Insight and Optimization: On a more positive note, logs can show your management and specialists how a system operates under certain conditions, which can help them optimize several internal systems. Logs can reflect things like the time it takes to perform a task or any conflicting operations that could affect the stability or performance of the system. Usually audit logs is a source of critical insights of system usage.
- Managing Security and Risk: Managing your security and risk profiles requires information; information about partners, information about vendors, information about cloud systems and products, knowledge of systems conditions prior to failure (playback) to prevent suspecious activity.
- Business Process Tracking: The audit trail can show business users how their data was or was not used. For example, when an attorney sends a legal document to opposing counsel, and that opposing lawyer later claims they didn’t receive it, the sender can use the audit trail to prove it was received down to details like and exactly when and IP address and equipment used to download it.

# Audit log features / use cases
- activity tracking (tacking event logs)
    - performed on the management level of the account (login, logout, authorizations, user/team management, closing accounts - both successful and rejected events)
    - performed on the user level of account (adding/editing/deleting records of different entities within the product)
    - performed on the admin level of account (change of policies, application settings, changing alerting settings, adding/editing/removing 3rd party app integrations, downgrading/upgrading privileges)
    - performed automatically on the context of account (expiring licence, stopping periodical routines, deleting expired entities, routine anomaly detection alert)
- configurable retention (hot/cold storage; per tenant retention settings)
- immutability
- viewable (via some UI); searchable, filterable
- role based access
- exportable to common formats like JSON, CSV (to use in SIEM)
- exposes (push/pull) API (to use in SIEM)

# Audit log user personas
- account admins

# References
- [Audit Logs for SaaS Enterprise Customers](https://frontegg.com/blog/audit-logs-for-saas-enterprise-customers)
- [Different Types of Logs in a SaaS Application](https://frontegg.com/blog/different-types-of-logs-in-a-saas-application)
- [Features of Audit Logs](https://www.enterpriseready.io/features/audit-log/)
- Audit log UI:
    - [Oracle Eloqua](https://docs.oracle.com/en/cloud/saas/marketing/eloqua-user/Resources/Images/Auditing/SampleAuditLog.png)
    - [Codefresh](https://codefresh.io/docs/docs/administration/audit-logs/)
    - [Azure AD](https://learn.microsoft.com/en-us/azure/active-directory/external-identities/auditing-and-reporting)
- Audit log API:
    - [Google Cloud API](https://cloud.google.com/logging/docs/audit/understanding-audit-logs)
    - [Qumulo Audit log Push API](https://www.youtube.com/watch?v=0ePZ4XIHbyE)
    - [Wagtail Audit log API](https://docs.wagtail.org/en/latest/extending/audit_log.html#)
    - [Codefresh Audit log API](https://codefresh.io/docs/docs/administration/audit-logs/)
    - [GitLab Audit log API](https://docs.gitlab.com/ee/administration/audit_events.html)