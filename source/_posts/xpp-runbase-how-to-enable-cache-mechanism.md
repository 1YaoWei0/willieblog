---
title: RunBase How to enable the caching mechanism (pack/unpack)
comments: true
date: 2026-01-11 13:02:24
tags:
 - RunBase
categories:
 - x++
description: RunBase How to enable the caching mechanism (pack/unpack)?
---

# How to Enable Caching (pack / unpack) in RunBase

> This article also applies to Jobs implemented using the **RunBaseBatch** framework.

---

## Business Background

In D365 Finance & Operations, `RunBase` is commonly used to implement parameterized business operations or batch Jobs. To ensure a good user experience and functional completeness, **caching (pack / unpack) must be enabled** in the following scenarios:

1. When the **Records to include** control is required, allowing users to enter filter criteria for business tables (i.e., enabling Query);
2. When the Job needs to run in a **new session (RunInNewSession)**;
3. When the Job must support **Run in Batch** execution.

From an implementation perspective, **caching is essentially the serialization and deserialization of RunBase member variables**. These member variables typically correspond to the parameters exposed to users in the Dialog.

This article demonstrates the **simplest and standard implementation approach** to correctly enable caching in a `RunBase` Job, while avoiding common implementation pitfalls.

---

## Key Implementation

### Step 1: Use macros to manage member variables

```c#
protected TransDate gTransDate; // Member variable

#DEFINE.CurrentVersion(1) 
// #CurrentVersion represents the cache version number, defined as 1 in this example

#LOCALMACRO.CurrentList 
// #CurrentList is used to centrally manage member variables that need to be cached
// Note: container-type variables must not be included here
    gTransDate
#ENDMACRO
```

**Notes:**

* Using macros significantly improves maintainability
* When parameters change, only `#CurrentList` needs to be updated
* container-type variables must be handled separately and cannot be placed directly in the macro list

---

### Step 2: Override the `pack()` method

```c#
public container pack()
{
    return [#CurrentVersion, #CurrentList];
}
```

**Implementation details:**

* The return value must be a container
* It is recommended to always store the version number first to support future extensions

---

### Step 3: Override the `unpack()` method

```c#
public boolean unpack(container _packedClass)
{
    Version version = RunBase::getVersion(_packedClass);

    switch (version)
    {
        case #CurrentVersion:
            [version, #CurrentList] = _packedClass;
            break;

        default:
            return false;
    }

    return true;
}
```

**Notes:**

* `RunBase::getVersion()` is used to safely parse the cache structure
* This is a key mechanism for supporting Job upgrades and backward compatibility

---

## Special Case: Caching with `QueryRun`

> When a Job enables the **Records to include** control, a `QueryRun` member variable is typically introduced.
> Since `QueryRun` cannot be stored directly using macros, it must be handled separately.

---

### Step 1: Declare member variables

```c#
protected TransDate gTransDate;
QueryRun queryRun;

#DEFINE.CurrentVersion(1)

#LOCALMACRO.CurrentList
    gTransDate
#ENDMACRO
```

---

### Step 2: Override the `pack()` method

```c#
public container pack()
{
    return [#CurrentVersion, #CurrentList, queryRun.pack()];
}
```

**Key points:**

* `QueryRun` must be packed separately using `pack()`
* The order must be consistent with the `unpack()` method

---

### Step 3: Override the `unpack()` method

```c#
public boolean unpack(container _packedClass)
{
    Version version = RunBase::getVersion(_packedClass);
    container conQuery;

    switch (version)
    {
        case #CurrentVersion:
            [version, #CurrentList, conQuery] = _packedClass;
            Query query = new Query(conQuery);
            queryRun = new QueryRun(query);
            break;

        default:
            return false;
    }

    return true;
}
```

**Notes:**

* During unpack, both `Query` and `QueryRun` must be reconstructed
* Existing object instances must not be reused

---

## Result Validation

1. Open the Job;

{% asset_img "image-20260110214346484.png" "Class Header" %}

2. Enter parameter values and click **OK**;

3. After successful execution, reopen the Job. The parameter values should default to the values entered during the previous run;

These results indicate that:

* pack / unpack is working correctly
* parameters are correctly restored in new session and Batch scenarios

---

## Summary

1. **RunBase does not automatically enable caching**; `pack / unpack` must be explicitly implemented by the developer;
2. Objects that need to be cached are typically **member variables exposed in the Dialog**;
3. Using **macros to manage cached fields** is strongly recommended to improve maintainability;
4. `QueryRun` is a special case and must be **packed and unpacked separately**;
5. If a Job supports **RunInNewSession / RunInBatch / Records to include**, caching is **mandatory rather than optional**.