---
title: RunBaseBatch How to enable "Run In Batch"
comments: true
date: 2026-01-11 13:14:21
tags:
 - RunBaseBatch
categories:
 - x++
description: RunBaseBatch How to enable "Run In Batch"
---

# How to Enable Run In Batch in RunBaseBatch

> From a minimal implementation perspective, `RunBaseBatch` and `RunBase` Jobs are implemented in a highly similar way.
> This article explains how to correctly enable **Run In Batch** in `RunBaseBatch` using the **standard implementation approach**.
>
> Note: The examples in this article are based on `RunBaseBatch`, and the minimal implementation version is not repeated here.

---

## Business Background

In D365 Finance & Operations, when a business process has the following characteristics, it is typically designed as a **Batch Job**:

* Long execution time
* Complex logic involving large volumes of data processing
* Not suitable for blocking the user interaction interface

`RunBaseBatch` is a standard framework provided by D365 that allows a RunBase Job to be **seamlessly upgraded into a batch-capable Job**, with built-in support for:

* Run In Batch
* Batch group
* Execution in a new session

However, this is only true **if the implementation follows the framework’s expectations**.
This article focuses on **which methods must call `super()` and why this is critical for enabling Run In Batch**.

---

## Key Implementation

### Step 1: Inherit from `RunBaseBatch`

```x++
public class HM365RunBaseBatch extends RunBaseBatch
{
    ...
}
```

**Notes:**

* Only by inheriting from `RunBaseBatch` will the system recognize the Job as batch-capable
* Inheriting from `RunBase` alone is not sufficient to enable Run In Batch

---

### Step 2: Ensure `new()` calls `super()`

```x++
public void new()
{
    Query query = new Query(queryStr(SalesLine));
    queryRun = new QueryRun(query);

    super(); 
    // Calling the base class constructor is mandatory
    // RunBaseBatch initializes BatchInfo and other critical objects here
}
```

**Key points:**

* `super()` is **not optional**
* If omitted:

  * **Run In Batch** will not appear in the Dialog
  * The Job will not be correctly registered as a Batch task
* This is a **very common but subtle mistake**

---

### Step 3: Ensure `dialog()` calls `super()`

```x++
public Object dialog()
{
    DialogRunbase dialog = super();

    gDialogTransDate = dialog.addFieldValue(
        extendedTypeStr(TransDate),
        gTransDate
    );

    return dialog;
}
```

**Notes:**

* `super()` is responsible for constructing the Batch-related areas of the Dialog
* If not called:

  * Even if the Job inherits from `RunBaseBatch`
  * Batch options will not be displayed in the Dialog

---

### Step 4: Ensure `getFromDialog()` calls `super()`

```x++
public boolean getFromDialog()
{
    boolean ret = super();

    gTransDate = gDialogTransDate.value();

    return ret;
}
```

**Notes:**

* `super()` reads Batch-related parameters
* If omitted:

  * Run In Batch settings may not be correctly parsed
  * Batch Job behavior may not match expectations

---

## Result Validation

After the above steps are implemented correctly, opening the Job Dialog should display the **Run In Batch** options:

{% asset_img "image-20260111105512293.png" "Class Header" %}

This confirms that:

* The `RunBaseBatch` initialization flow is complete
* BatchInfo has been successfully constructed
* The Job can be submitted and executed as a Batch Job

---

## Summary

1. **Enabling Run In Batch requires inheriting from `RunBaseBatch`, not `RunBase`**;
2. Calling `super()` in `new()`, `dialog()`, and `getFromDialog()` is **mandatory, not a matter of coding style**;
3. The `super()` call chain contains **critical logic for BatchInfo initialization and Batch parameter construction**;
4. Even if the code compiles and runs, missing any required `super()` call may result in Run In Batch being “visible but ineffective”;
5. In real projects, **around 80% of RunBaseBatch issues are caused by incomplete initialization rather than business logic errors**.