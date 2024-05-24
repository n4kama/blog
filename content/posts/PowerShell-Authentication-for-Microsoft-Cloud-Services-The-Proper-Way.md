---
title: "PowerShell Authentication for Microsoft Cloud Services: The Proper Way"
date: 2023-11-06
# weight: 1
# aliases: ["/first"]
tags: ["powershell", "microsoft", "azure", "authentication", "scripting", "good-practices", "security"]
author: "Nakama"
# author: ["Me", "You"] # multiple authors
showToc: true
TocOpen: true
draft: false
hidemeta: false
comments: false
description: "Exploring the best way to authenticate on Microsoft cloud services: using a device code"
canonicalURL: "https://canonical.url/to/page"
disableHLJS: true # to disable highlightjs
disableShare: false
disableHLJS: false
hideSummary: false
searchHidden: true
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: false
ShowWordCount: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
cover:
    image: "<image path/url>" # image path/url
    alt: "<alt text>" # alt text
    caption: "<text>" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: true # only hide on current single page
editPost:
    URL: "https://github.com/n4kama/blog/blob/main/content"
    Text: "Suggest Changes" # edit text
    appendFilePath: true # to append file path to Edit link
---

## 💡 Introduction

This method is particularly well-suited for interactive scripts where users actively participate in the authentication process, ensuring that the connection and access rights are directly linked to their individual accounts.

> It's important to note that while device code authentication is excellent for interactive script scenarios (in Powershell for instance), for application use, it is recommended to take a different approach.
> 
> For applications, the authentication method and implementation is always (to the best of my knowledge) provided when creating the application in the Azure portal. For example, when creating an application in the Azure portal, you will be asked to select the type of application you are creating. The options are: Web app/API, Native, and Single-page application. Each of these options will provide you with a template.

## 🚀 The Device Code Authentication Method

Device code authentication is a robust and secure method of accessing Microsoft cloud services. This method is particularly valuable in scenarios where users need to authenticate on systems that do not have a web browser. However, it's worth noting that even for desktop script scenarios, device code authentication remains a preferred choice due to its user-friendliness and efficiency compared to the other authentication methods provided by the Azure module commands. 

Many of the Azure modules commands provide a parameter that allows you to specify the authentication method to use.
For example, the `Connect-ExchangeOnline` command provides the `-Device` parameter. This parameter allows you to use the device code authentication method.

```bash
dev@tanmatsu: $ Connect-XYZ [-UseDeviceAuthentication | -DeviceCode | -DeviceAuth | -Device]

dev@tanmatsu: $ Connect-ExchangeOnline -DeviceCode      # Specifically for Exchange Online
```

You may note that the parameter may have a different name depending on the command. For example, the `Connect-MicrosoftTeams` command provides the `-UseDeviceAuthentication` parameter.

This command will return a URL and a unique code associated with the session. Here's how the process works:
1. Open the URL in a web browser on any computer.
2. Enter the unique code displayed in the PowerShell window.
3. Complete the web-based authentication process.
4. After the successful authentication in the web browser, the PowerShell session will be authenticated through the standard Azure AD authentication flow.
5. Within a few seconds, the module's cmdlets will be imported into your PowerShell session, ready for use.

## 💪 Benefits of Device Code Authentication:

Device code authentication offers several advantages, making it the preferred method for accessing Microsoft cloud services:
1. **Enhanced Security:** MFA & Conditional Access policies are supported, ensuring that only authorized users can access the services.
2. **No Browser Dependency:** Making your script *almost* OS agnostic (accounting that the script is compatible with the OS).
3. **User-Friendly:** Fast and straightforward for users. No need to input usernames and passwords directly into scripts.
4. **Single Sign-On (SSO):** Who doesn't like SSO ❤️.
