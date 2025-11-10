# 🧭 Database Context Usage Guide  
### Optimizing Data Access for Large-Scale Production Systems

## ⚙️ Overview

This document explains **the best practices for managing database contexts** (`DataContext` / `DbContext`) in a large-scale C# application — such as **ASP.NET Core Web APIs**, **background workers**, or **scheduled jobs**.

In large production environments, poor management of context lifetimes can cause:

- ⚠️ **Memory leaks**
- ⚠️ **Stale or inconsistent data**
- ⚠️ **Thread safety issues**
- ⚠️ **Performance degradation**

This guide covers:
- Context lifetime strategies  
- Dependency Injection (DI) patterns  
- Architecture flow diagrams  
- Recommended best practices  

---

## 🧩 The Two Common Approaches

### **Approach A — Shared Context (in Constructor)**

```csharp
private readonly EmailHelper _emailHelper;
private readonly DummyDataContext _db;
private readonly PetaPoco.Database _db2;

public UserRepository(DummyDataContext db)
{
    _emailHelper = new EmailHelper();
    _db = db;
    _db2 = new PetaPoco.Database(Code.Helpers.ConnectionString);
}

public USER GetUserByID(long id, int companyId)
{
    return _db.USERs
              .SingleOrDefault(x => x.UserID == id && 
                                    x.CompanyID == companyId && 
                                    x.Active == true);
}
```

✅ **Pros**
- High performance — reuses DB connection efficiently  
- Enables **transactions** across multiple repository calls  
- Integrates naturally with ASP.NET Core **Scoped Dependency Injection**

⚠️ **Cons**
- Retains tracked entities → memory grows with use  
- Can return stale data (due to internal caching)  
- **Not thread-safe** if reused across threads or requests  
- Must be disposed properly  

🧠 **Best Use Case:**  
For **Web APIs**, where each HTTP request gets a new scoped repository and context.

---

### **Approach B — New Context per Method**

```csharp
public USER UpdateUser(int id, int companyId, ApiUserUpdate request)
{
    using (var db = new DummyDataContext())
    {
        var existing = db.USERs
                         .SingleOrDefault(u => u.UserID == id && 
                                               u.CompanyID == companyId && 
                                               u.Active == true);

        EmailHelper.CopyOver(request, existing);
        db.SubmitChanges();
        return existing;
    }
}
```

✅ **Pros**
- Fully isolated and **thread-safe**  
- Each operation auto-disposes context  
- Always fetches **fresh data**  
- No long-lived tracking or cache issues  

⚠️ **Cons**
- Slightly slower — creates a new connection per method  
- Harder to maintain transaction consistency across methods  

🧠 **Best Use Case:**  
For **background services**, **async jobs**, or **parallel processing** tasks.

---

## ⚖️ Comparison Table

| Criteria | Shared Context (Constructor) | New Context (Per Method) |
|-----------|------------------------------|----------------------------|
| **Performance** | ✅ Faster (no repeated connection setup) | ⚠️ Slightly slower |
| **Memory Usage** | ⚠️ Higher (entity tracking) | ✅ Low and predictable |
| **Data Freshness** | ⚠️ May return cached entities | ✅ Always fresh |
| **Thread Safety** | ❌ Not safe for parallel use | ✅ Safe |
| **Transaction Handling** | ✅ Easy | ⚠️ Requires manual scope |
| **Ideal For** | Web APIs (Scoped DI) | Background / Async Jobs |

---

## 🧱 Recommended Repository Pattern (Hybrid)

```csharp
public class UserRepository : IDisposable
{
    private readonly DummyDataContext _db;
    private readonly PetaPoco.Database _db2;
    private readonly EmailHelper _emailHelper;
    private bool _disposed;

    public UserRepository(DummyDataContext db)
    {
        _db = db; // Scoped per HTTP request
        _db2 = new PetaPoco.Database(Code.Helpers.ConnectionString);
        _emailHelper = new EmailHelper();
    }

    public USER GetUserById(int id, int companyId)
        => _db.USERs.SingleOrDefault(x => x.UserID == id && x.CompanyID == companyId && x.Active);

    public void UpdateUser(USER entity)
        => _db.SubmitChanges();

    public void Dispose()
    {
        if (!_disposed)
        {
            _db.Dispose();
            _db2.Dispose();
            _disposed = true;
        }
    }
}
```
