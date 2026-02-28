---
title: Dynamics 365 Debugging Configuration
comments: true
date: 2025-02-23 20:37:45
categories:
 - x++
description: Dynamics 365 debugging configuration
---
![Dynamics 365 Debugging Configuration technical flow diagram](xpp-debug-configuration/cover.png)

<!--
AI Image Prompt:
Create a clean, minimal, professional technical diagram for Microsoft Dynamics 365 Finance & Operations (D365 F&O). Topic: Dynamics 365 Debugging Configuration. Show key components, data flow arrows, extension points, transaction boundaries, and where X++ logic executes. Use simple boxes, labels, and directional connectors on a white background. Style should look like an enterprise architecture blueprint, no decorative art, no characters, no 3D effects.
-->

> 当无法 debug 系统标准的代码时，可以按照本文的步骤配置

导航 Tools > Options，在 Dynamics 365 > Debugging tab 页面配置 debug 相关属性，如下图，在 Included Packages 中选择，在 debug 中需要 load 的 model。

{% asset_img "xpp-debug-configuration.png" "D365 Debugging Configuration" %}