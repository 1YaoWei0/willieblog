---
title: How to print the amount based on the Language Id
date: 2024-12-23 14:56:52
categories: 
 - Retail Commerce
tags:
 - ScaleUnit
description: This article shows how to format the amount based on the Language Id
---

### How to get the Language Id

```c#
string languageText = string.Empty;

if (request.RequestContext.LanguageId != null && request.RequestContext.LanguageId != "") // Get the LanguageId from request context
{
    languageText = request.RequestContext.LanguageId;
}
else
{
    languageText = "en-us";
}

CultureInfo culture = new CultureInfo(languageText); // Get the culture based on LanguageId
```

### Format the amount based on culture

The `decimal.ToString()` included two parameters, for example:

```c#
dl.EffectiveAmount.ToString("C2", culture); // The first parameter means the currency code should be something, the second parameter means the amount value should be format based on some culture rules.​
```