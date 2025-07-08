---
title: Document Routing Agent in D365 F&O
comments: true
date: 2025-07-08 20:33:46
tags:
categories:
 - d365
description: Document Routing Agent in D365 F&O
---

> Reprinted from: https://mohitrampal.com/2023/03/20/document-routing-agent-in-d365-fo/

> You can also watch video: https://www.youtube.com/watch?v=oG6n2PA-Phs

> Chinese version: https://www.cnblogs.com/lingdanglfw/p/14141545.html

> Official documentation: https://learn.microsoft.com/en-us/dynamics365/fin-ops-core/dev-itpro/analytics/er-design-zpl-labels

> ZPL View: https://labelary.com/viewer.html

The Document Routing Agent (DRA) is an application that resides on the Customer’s network, to act as a bridge between the internet-hosted Dynamics 365 (D365 F&O) system and Customer’s network printers.

{% asset_img "image-25.png" "image-25" %}

DRA tool is Supported on Windows 8.1, Windows 10, Microsoft Windows Server 2012 R2, Microsoft Windows Server 2016, or Microsoft Windows Server 2019.

DRA run as either a desktop application or a Microsoft Windows service.

DRA as a Service can be configured to start automatically after a computer restart, so no user intervention is required. However, When the Document Routing Agent runs as a Windows service, document reports, such as checks, that require custom margins can’t be printed directly to network printers. Instead, the Document Routing Agent automatically routes those documents to a target folder.

DRA as a desktop application uses Adobe Reader to spool the document to the shared printer device chosen in finance and operations. Microsoft recommends installing the Document Routing Agent in several places to manage situations when documents with custom margins need to be printed. The printers for those documents should thus only be installed on Document Routing Agents that will operate in desktop application mode. As an alternative, you can pick up the files in the target directory and guide them in the right direction using a post-execution process.

## Steps to Install & Configure DRA

Sign in to D365 F&O environment which you want to configure in the DRA tool.

1. Go to Organization Administration–>Setup–>Network Printers
2. Click on Download Document Routing Agent Installer.

{% asset_img "image-26.png" "image-26" %}

3. Ensure that you are logged in as Admin user. Run the downloaded file and complete the setup.
4. Open the DRA tool and sign in with Admin account.

{% asset_img "image-27.png" "image-27" %}

5. Click on Settings Button and enter values in all the fields.

{% asset_img "image-28.png" "image-28" %}

6. If Run as Windows Service checkbox is checked, then DRA will run as a Service. To run as service, window service should be set up and running.

Right click on the highlighted Service and click on Properties.

{% asset_img "image-29.png" "image-29" %}

Click on Log On tab and add your Admin/Service account.

{% asset_img "image-30.png" "image-30" %}

Please note that step 6 is not required if running as an Application (Run as a window service checkbox is unchecked).

7. Start the service and go back to step 5 and Click on Printers.

Only printers which are registered will be visible in D365 F&O.

{% asset_img "image-31.png" "image-31" %}

8. After registering a printer, go to D365 F&O (Organization Administration–>Setup–>Network Printers) and activate the printer.

{% asset_img "image-32.png" "image-32" %}

9. Now, you can test it in your reports by using Network Printer as shown below (after printing to screen), or setting up the print management.

{% asset_img "image-33.png" "image-33" %}

I had a requirement to test multiple network printers based on specific values, like if department is A the document should route to PrinterA, if department is B, it should go to PrinterB etc. This configuration was done in print management of report and I tested with dummy Printers and OneNote.

You can setup a dummy printer by going to Printers & Scanners.

- Click on Add a Printer or Scanner button.

{% asset_img "image-34.png" "image-34" %}

- Click on ‘The printer that I want isn’t listed.’

{% asset_img "image-35.png" "image-35" %}

- Select last option as below screenshot and click Next.

{% asset_img "image-36.png" "image-36" %}

 - Select Nul port and click Next.

{% asset_img "image-37.png" "image-37" %}

 - Select Microsoft Print to PDF and click Next

{% asset_img "image-38.png" "image-38" %}

{% asset_img "image-39.png" "image-39" %}

- Specify printer name and click Next.

{% asset_img "image-40.png" "image-40" %}

{% asset_img "image-41.png" "image-41" %}

- Click on Open Queue of Printer and Pause Printing.

{% asset_img "image-42.png" "image-42" %}

- As mentioned earlier, you need to register the printer.

{% asset_img "image-43.png" "image-43" %}

- After registering the printer, it will be visible in D365 F&O. Activate the printer.

{% asset_img "image-44.png" "image-44" %}

{% asset_img "image-45.png" "image-45" %}

- As you can see, After report is printed on screen, I can select my new printer.

{% asset_img "image-46.png" "image-46" %}

{% asset_img "image-47.png" "image-47" %}

- The document will be visible in print queue.

{% asset_img "image-48.png" "image-48" %}

## Note

- DRA is legal entity specific, so you have to setup the printers in all required legal entities.
- If you already have DRA tool, you might get below error for different versions.
To resolve this, you have to uninstall the DRA and install again.

{% asset_img "image-49.png" "image-49" %}