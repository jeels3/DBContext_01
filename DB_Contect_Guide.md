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
private readonly SendEmailRep _sendEmailRep;
private readonly dcSqwizDataContext _db;
private readonly PetaPoco.Database _db2;

public CampaignRepositoryWrapper(dcSqwizDataContext db)
{
    _sendEmailRep = new SendEmailRep();
    _db = db;
    _db2 = new PetaPoco.Database(Code.Helpers.ConnectinStringLeads);
}

public EM_CAMPAIGN GetCampaignByIDV2(long id, int orgId)
{
    return _db.EM_CAMPAIGNs
              .SingleOrDefault(x => x.CampaignID == id && 
                                    x.OrganizationID == orgId && 
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
public EM_CAMPAIGN UpdateCampaign(int id, int orgId, Api2CampaignUpdate request)
{
    using (var db = new dcSqwizDataContext())
    {
        var existing = db.EM_CAMPAIGNs
                         .SingleOrDefault(c => c.CampaignID == id && 
                                               c.OrganizationID == orgId && 
                                               c.Active == true);

        SendEmailRep.CopyOver(request, existing);
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
public class CampaignRepositoryWrapper : IDisposable
{
    private readonly dcSqwizDataContext _db;
    private readonly PetaPoco.Database _db2;
    private readonly SendEmailRep _sendEmailRep;
    private bool _disposed;

    public CampaignRepositoryWrapper(dcSqwizDataContext db)
    {
        _db = db; // Scoped per HTTP request
        _db2 = new PetaPoco.Database(Code.Helpers.ConnectinStringLeads);
        _sendEmailRep = new SendEmailRep();
    }

    public EM_CAMPAIGN GetCampaignById(int id, int orgId)
        => _db.EM_CAMPAIGNs.SingleOrDefault(x => x.CampaignID == id && x.OrganizationID == orgId && x.Active);

    public void UpdateCampaign(EM_CAMPAIGN entity)
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

... (rest of markdown content continues exactly from previous version) ...
