---
title: "Important | Exchange Online Basic Auth Shutdown"
date: 2021-10-21T14:10:54.448Z
lastmod: 2021-11-11T14:17:51.738Z
author: Mike Hacker
url: /important-or-exchange-online-basic-auth-shutdown/
tags:
  - 'Announcements'
summary: 'In February 2021 the Exchange team at Microsoft posted an announcement about their plans for turning off Basic Authentication in Exchange Online. Effective O...'
draft: false
image: cover.png
---

In February 2021 the Exchange team at Microsoft posted an announcement about their plans for turning off Basic Authentication in Exchange Online.

Effective October 1, 2022 the team will begin to permanently disable Basic Auth in all Exchange Online tenants, with the exception of SMTP Auth (if it is being used).

Basic Authentication is an outdated and all developers who are building or maintaining apps should be moving to OAuth 2.0, which has become an industry standard.  

[Read the full announcement here](https://techcommunity.microsoft.com/t5/exchange-team-blog/basic-authentication-and-exchange-online-september-2021-update/ba-p/2772210)

Microsoft provides a series of authentication services, open-source libraries and application management tools to address identity and authentication needs.  Learn more about the Microsoft Identity Platform [here](https://docs.microsoft.com/en-us/azure/active-directory/develop/).

If your are utilizing Basic Authentication today with Exchange Online and do not have experience with OAuth, we have you covered.  [Check out this great article](https://docs.microsoft.com/en-us/exchange/client-developer/exchange-web-services/how-to-authenticate-an-ews-application-by-using-oauth) that provides information and examples on how to connect to Exchange Web Services using OAuth.
