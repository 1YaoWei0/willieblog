---
title: RunBase How to add Records to Include control
comments: true
date: 2026-01-11 12:36:31
tags:
 - RunBase
categories:
 - x++
description: RunBase How to add Records to Include control
---

---

> This article also applies to Jobs implemented based on the **RunBaseBatch** framework.
> Default assumption: the RunBase Job **has caching enabled (pack / unpack)**.

---

## Business Background

In D365 Finance & Operations, users often want to specify filter criteria through the **Records to include** control before running a Job, so that only the relevant set of records is processed.

However, in custom `RunBase` / `RunBaseBatch` Jobs:

* The system **does not** automatically provide this control by enabling a specific property
* Official documentation provides very limited explanation
* Many developers are unclear about the underlying implementation mechanism

This article demonstrates the **simplest implementation approach** to correctly add the **Records to include** control to a `RunBase` Job, and to ensure it works properly in **cached, batch, and new session** scenarios.

---

## Key Implementation

### Step 1: Declare a global `QueryRun` variable

```c#
QueryRun gQueryRun;
```

> This variable is used to:
>
> * Hold the query conditions configured by the user in the UI
> * Drive the business logic during the `run()` phase
> * Support serialization via pack / unpack

---

### Step 2: Initialize `QueryRun`

```c#
public void new()
{
    Query query = new Query(queryStr(SalesLine));
    // The Query can be constructed using a Simple Query or QueryBuildDataSource
    gQueryRun = new QueryRun(query);
}
```

**Notes:**

* `QueryRun` must be initialized during construction
* At least one DataSource is required; otherwise, the Records to include UI cannot be generated

---

### Step 3: Override the `queryRun()` method

```c#
public QueryRun queryRun()
{
    return gQueryRun;
}
```

> This method **must return the global variable `gQueryRun`**; otherwise, the Records to include control will not function.

---

### Step 4: Override the `showQueryValues()` method

```c#
public boolean showQueryValues()
{
    return true;
}
```

**Purpose:**

* Controls whether **Records to include** is displayed in the Dialog
* Returning `true` is a required condition for showing this control

---

### Step 5: Override the `pack()` method

```c#
public container pack()
{
    return [#CurrentVersion, gQueryRun.pack()];
}
```

**Key points:**

* `QueryRun` **must be serialized**
* Otherwise, query conditions will be lost when running in Batch or in a new session

---

### Step 6: Override the `unpack()` method

```c#
public boolean unpack(container _packedClass)
{
    Version version = RunBase::getVersion(_packedClass);
    container conQuery;

    switch (version)
    {
        case #CurrentVersion:
            [version, conQuery] = _packedClass;
            Query query = new Query(conQuery);
            gQueryRun = new QueryRun(query);
            break;

        default:
            return false;
    }

    return true;
}
```

**Notes:**

* During unpack, both `Query` and `QueryRun` must be reconstructed
* Existing object instances must not be reused directly

---

By following the **six steps above**, you can successfully add the **Records to include** control to a `RunBase` Job.

---

## Result Validation

After opening the Job, if the **Records to include** query area appears in the Dialog, the control has been successfully added and is functioning correctly.

{% asset_img "batchjobdialog.png" "Class Header" %}

---

## Using `QueryRun` in `run()`

```c#
public void run()
{
    while (gQueryRun.next())
    {
        // Business logic
    }
}
```

**Notes:**

* `gQueryRun` should be used directly in `run()`
* Do not reconstruct the Query, otherwise the user-defined conditions will be ignored

---

## Summary

1. `RunBase` **does not automatically manage Query objects**
2. **Records to include** relies on `queryRun()` + `showQueryValues()`
3. `QueryRun` **must support pack / unpack**
4. In **Batch / RunInNewSession** scenarios, always verify that query conditions are applied correctly