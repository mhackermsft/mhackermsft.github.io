---
title: 'Are your organization’s secrets safe?'
date: 2023-09-28T17:02:45.457Z
lastmod: 2023-09-28T17:25:56.035Z
author: Mike Hacker
url: /are-your-organizations-secrets-safe/
summary: 'Keeping secrets safe is one of the most important aspects of software development and deployment. Secrets are sensitive information that can grant access to ...'
draft: false
image: cover.jpg
---

Keeping secrets safe is one of the most important aspects of software development and deployment. Secrets are sensitive information that can grant access to valuable resources, such as passwords, API keys, tokens, encryption keys, and certificates. If secrets are exposed, they can be used by malicious actors to compromise the security and integrity of the software and the data it handles. Therefore, developers need to follow best practices to protect their secrets from unauthorized access and leakage.

One of the common mistakes that developers make is embedding secrets in source code or in configuration files. This can lead to secrets being compromised in several ways, such as:

-   Secrets in source code or configuration files can be accidentally checked into version control systems.
-   Source code or configuration files can be copied or transferred to other locations, such as backup servers, cloud storage, or email attachments, and become accessible to unauthorized parties.
-   Source code or configuration files can be exposed to third-party services or tools, such as code analysis, testing, or deployment tools, and become vulnerable to interception or theft.
-   Improperly protected encrypted configuration files can be stolen and brute force decrypted over time.

To avoid these risks, developers should never embed secrets in source code or in configuration files. Instead, they should use secure methods to store and manage their secrets, such as Azure Key Vault.

By following this best practice, developers can ensure that their secrets are kept safe and their software is secure. In this blog post, we will explore how you can use Azure Key Vault with your custom on-premises applications to protect your secrets.

So what exactly is Azure Key Vault? Azure Key Vault is a cloud service that lets you securely store and access secrets, such as passwords, certificates, or cryptographic keys. You can use it to protect all of your secrets and control who can access it.

For web applications running on Azure services, such as Azure App Service, using Key Vault is simple. The Azure App Service has a managed identity stored in Azure Active Directory that is used to enable the application to authenticate against Azure Key Vault.  Once authenticated the application can perform any necessary operations such as retrieving the latest value of a specific secret.  A simple tutorial for this process can be found here: [Tutorial - Use Azure Key Vault with an Azure web app in .NET | Microsoft Learn](https://learn.microsoft.com/en-us/azure/key-vault/general/tutorial-net-create-vault-azure-web-app)

In the past, using Azure Key Vault for on-premises applications was a bit challenging. The biggest hurdle was that your application needed to authenticate against Azure Key Vault in order to have permissions to perform operations in the Vault.  This meant that you would still need to protect at least one secret, which was the API key for accessing Key Vault.  With Azure Arc, all of this has changed!

Azure Arc is a service that enables you to manage and govern your hybrid cloud resources across different environments, such as on-premises, edge, and multi-cloud. One of the benefits of Azure Arc is that it allows you to extend Azure services and management capabilities to any infrastructure. Arc enabled servers receive an Azure managed identity, which is a secure and convenient way to authenticate and authorize access to Azure resources without storing any credentials in your code or configuration files. With an Azure managed identity, you can use the same identity and access management (IAM) policies and role-based access control (RBAC) that you use for your Azure resources to manage your arc enabled servers. This simplifies the security and compliance of your hybrid cloud environment.

This means that if you arc enable your on-premises servers your developers can utilize Azure Key Vault with a managed identity. With managed identities, your custom applications can easily retrieve secrets directly from Key Vault without the need for any API keys or any other secrets being stored in source code or configuration files.  This is a huge step forward for your organization’s security practices.

To get started you will need to Arc enable your on-premises application server for free. You can do this directly through the Azure portal using the steps included in this article: [Quickstart - Connect hybrid machine with Azure Arc-enabled servers - Azure Arc | Microsoft Learn](https://learn.microsoft.com/en-us/azure/azure-arc/servers/learn/quick-enable-hybrid-vm)

Once your server is arc enabled, developers can use the Azure Key Vault SDK for .NET to easily access Key Vault with your servers managed identity.  An example C# application can be found here: [Quickstart - Azure Key Vault secrets client library for .NET | Microsoft Learn](https://learn.microsoft.com/en-us/azure/key-vault/secrets/quick-create-net?tabs=azure-cli)

One additional benefit of using the Azure Key Vault SDK is that the same code that is used to authenticate your on-premises server to Key Vault is the exact same code that is used if your app is running directly in an Azure App Service. This makes your future application migration to Azure just a little bit easier.

In this blog post, you have learned how to use Azure Arc to provide on-premises servers a managed identity so they can safely and securely authenticate with Azure Key Vault. This allows your developers to avoid storing sensitive credentials in your source code or configuration files and instead leverage the security and scalability of Azure. I hope you have found this post useful and informative.

If you have any questions or would like to learn more about how using Azure Arc, Managed Identities, and Azure Key Vault can keep your organization’s secrets secure, please reach out to your Microsoft account team.
