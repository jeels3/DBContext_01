# Understanding `DataContext` Scope and Update Behavior in LINQ-to-SQL

## Overview
When working with LINQ-to-SQL, the `DataContext` (e.g., `dcSqwizDataContext`) acts as the **bridge** between your C# objects and the database. Each instance of `DataContext` maintains its **own change tracker**, which records loaded, added, modified, and deleted entities during its lifetime.

---

## Problem Scenario

```csharp
var db = new dcSqwizDataContext();
var data = db.SomeTable.FirstOrDefault(x => x.Id == 1);

// Modify entity
data.Name = "New Name";

// Create a new DataContext
var db2 = new dcSqwizDataContext();

// Try to submit changes using new context
db2.SubmitChanges();
```

---

## Issue

The update **does not persist** to the database.

### Reason
Each `DataContext` instance has its **own state and cache**. The `data` object was loaded and modified using `db`, but when calling `SubmitChanges()` on `db2`, that context has **no knowledge** of the modified entity. As a result, no SQL `UPDATE` statement is executed.

---

## Why This Happens

- Each `DataContext` instance manages:
  - Its own database connection.
  - Its own change-tracking mechanism.
  - Its own pending changes queue.
- Changes made in one context are **not visible** to another.

---

## Correct Approaches

### 1. Use the Same `DataContext` for the Entire Operation

This is the simplest and most reliable approach.

```csharp
using (var db = new dcSqwizDataContext())
{
    var record = db.SomeTable.FirstOrDefault(x => x.Id == 1);
    record.Name = "New Name";
    db.SubmitChanges(); // ✅ Works correctly
}
```

---

### 2. Reattach the Entity to a New `DataContext`

If a new context must be used (e.g., due to architecture constraints):

```csharp
var db1 = new dcSqwizDataContext();
var data = db1.SomeTable.FirstOrDefault(x => x.Id == 1);
data.Name = "New Name";

// Attach to a new context
var db2 = new dcSqwizDataContext();
db2.SomeTable.Attach(data);
db2.Refresh(System.Data.Linq.RefreshMode.KeepCurrentValues, data);
db2.SubmitChanges(); // ✅ Now changes are recognized and updated
```

This explicitly attaches the entity to the new context so it can track the update.

---

## Summary

| Behavior | Description | Result |
|-----------|--------------|--------|
| Modify entity in one context, submit in another | Contexts do not share state | ❌ No update |
| Modify and submit in same context | Changes tracked and persisted | ✅ Works |
| Attach modified entity to new context | Reestablish tracking manually | ✅ Works |

---

## Best Practice

> 🧭 **One unit of work = one `DataContext` instance.**  
> Keep a single `DataContext` alive for the full duration of your read–modify–save cycle, then dispose it.

