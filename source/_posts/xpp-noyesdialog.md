---
title: Insert a Yes/No Message box on a Form Click Button event
comments: true
date: 2025-07-11 21:27:20
tags:
 - Dialog
categories:
 - x++
description: Insert a Yes/No Message box on a Form Click Button event
---

> Reprint from: [AX / D365FO – Insert a Yes/No Message box on a Form Click Button event](https://d365ffo.com/2023/05/15/ax-d365fo-insert-a-yes-no-message-box-on-a-form-click-button-event/#:~:text=Below%20is%20an%20example%20code%20DialogButton%20diagBut%3B%20str,DialogButton%3A%3ANo%2C%20strTitle%29%3B%20if%20%28diagBut%20%3D%3D%20DialogButton%3A%3ANo%29%20info%28%27Operation%20canceled%27%29%3B)

While any operation is performed by the user, there are cases such as need to receive confirmation from the user to perform the operation. In these cases we can use “DialogButton” for user interaction not say “Yes/No” to perform the operation.

Below is an example code:

```c#
public void clicked()
{
   DialogButton diagBut;  
   str strMessage = 'By clicking OK you confirm to save data into form, do you still want to continue?';
   str strTitle = 'My Title';
   diagBut = Box::yesNo(strMessage, DialogButton::No, strTitle);

   if (diagBut == DialogButton::No)
   {

        info('Operation canceled');

   }
   else if(diagBut == DialogButton::Yes)
   {  
        Info('button click code here');
   }
}
```