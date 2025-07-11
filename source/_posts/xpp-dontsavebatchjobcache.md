---
title: Preventing Cache Retention in BatchJob Executions
comments: true
date: 2025-07-11 21:15:08
tags:
 - RunBase
 - RunBaseBatch
categories:
 - x++
description: Preventing Cache Retention in BatchJob Executions
---

When developing a **BatchJob** in Dynamics 365 Finance and Operations, you might encounter an issue where the system retains cached data after a job runs. This results in incorrect data being used during subsequent runs, even after clearing the cache. This typically happens when **global variables are initialized using `args` passed from the `main` method** before calling the `prompt` method.

#### The Issue

* **First Run**: Cache is cleared, and the job runs correctly.
* **Second Run**: The job uses cached data from the previous run, even after clearing the cache, because global variables were initialized before the prompt method.

This occurs because by default, the system retains the last execution state for future use.

#### Solution

To prevent the system from saving the last execution state, you can override the following method in the `RunBaseBatch` class:

```c#
public boolean allowSaveLast()
{
    return false;
}
```

By returning `false`, you instruct the system **not to save** the data from the previous execution, ensuring that every run of the BatchJob starts fresh.

#### Why It Works

The `allowSaveLast()` method is a hook in the `RunBaseBatch` class that determines whether or not the system saves the execution state after a BatchJob finishes. By overriding this method and returning `false`, you stop the system from storing the last job's state, which can lead to the caching issue mentioned above.

#### Additional Steps: When Changes Don't Take Effect Immediately

In some cases, after making the above changes, the system might not immediately reflect the new settings. To ensure the changes are applied correctly, follow these additional steps:

1. **Clear the D365 Usage Data**: This can help clear any old data that might still be cached.
2. **Restart IIS**: After clearing the usage data, restart the IIS (Internet Information Services) service to refresh the system environment.
3. **Rebuild the Project**: Finally, rebuild the project to ensure all changes are compiled and active in the environment.

These steps are sometimes necessary when changes do not take effect immediately, ensuring the system is completely refreshed.

#### Conclusion

To resolve issues with leftover data from previous BatchJob executions, simply override the `allowSaveLast()` method and return `false`. If changes don't take effect immediately, clear the D365 usage data, restart IIS, and rebuild the project to ensure everything runs as expected. This ensures that each run starts with fresh data, avoiding any problems related to cached or stale information.

---

This approach effectively controls the caching behavior and ensures consistent, accurate execution of your BatchJobs.