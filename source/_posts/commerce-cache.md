---
title: Cache
date: 2024-12-22 12:42:02
categories: 
 - Retail Commerce
tags:
 - ScaleUnit
comments: true
description: When we cannot find or edit the extension properties, we can use the cache for building the communication between front-end and back-end.
---

POS 前后端通信通常需要使用 `extension properties`；`extension properties` 存在于 **Cart**、**CartLine**、**SalesTransaction** 和 **SalesLine** 等；但某些场景，无法编辑 `extension properties`，所以无法通信；可以按照以下 **Cache** 构建新的通信机制；（谨慎使用，影响性能）

{% asset_img "commerce-cache-example.png" "cache example" %}

### Cache.cs 代码示例

```c#
using Microsoft.Extensions.Caching.Memory;
using System;

namespace LMS.Cache
{
    public class LMSCache
    {
        private static readonly Lazy<MemoryCache> lazy = new Lazy<MemoryCache>(()
            => new MemoryCache(new MemoryCacheOptions()));
        private static MemoryCache cache
        {
            get
            {
                return lazy.Value;
            }
        }

        public T Get<T>(string key)
        {
            if (!cache.TryGetValue<T>(key, out T cacheItem))
            {
                return default;
            }
            else
            {
                return cacheItem;
            }
        }

        public void Put<T>(string key, T value, int cacheLifeInSeconds = 500)
        {
            if (value == null)
            {
                if (cache.Get(key) != null)
                {
                    cache.Remove(key);
                }
            }
            else
            {
                cache.Set<T>(key, value, DateTimeOffset.Now.AddSeconds(cacheLifeInSeconds));
            }
        }
    }
}
```

### 使用示例

```c#
(new LMSCache()).Get<string>(key); // Get the value by key
(new LMSCache()).Put<string, string>(key, value); // Put the key value mapping
```