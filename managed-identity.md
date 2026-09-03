# Azure Managed Identity — Notes (App Service → Storage)

## What It Is
Managed Identity allows one Azure resource (source) to authenticate to another Azure resource (target) **without storing secrets, keys, or connection strings**. Azure handles the credential exchange automatically via Azure AD (Entra ID).

- **Source resource** (e.g., App Service) → gets an *identity*
- **Target resource** (e.g., Storage Account) → grants that identity *access* via RBAC

---

## Step 1: Enable Identity on the Source Resource
On the App Service, enable a Managed Identity. Two types:

| Type | Lifecycle | Relationship |
|---|---|---|
| **System-assigned** | Tied to the App Service — created/deleted with it | 1:1 (one identity per resource) |
| **User-assigned** | Standalone Azure resource, independent lifecycle | Can be attached to multiple resources |

---

## Step 2: Grant Access on the Target Resource (IAM)
Go to **Storage Account → Access Control (IAM) → Add role assignment**.

Typically grant **two separate identities** the same role (e.g., `Storage Blob Data Contributor`):

1. **Managed Identity** of the App Service → used in **production** (when app is deployed and running in Azure)
2. **Your own Azure AD user account** → used for **local development** (so `DefaultAzureCredential` can fall back to your logged-in dev credentials via Azure CLI / Visual Studio / VS Code)

> These are two distinct role assignments, not one combined setting.

---

## Step 3: Update `Program.cs`
Replace connection-string-based auth with **Azure AD token-based auth**.

> Note: A connection string *is* technically a form of authentication (key-based). What we're really doing is swapping **key-based auth** for **Azure AD identity-based auth + RBAC authorization**.

```csharp
using Azure.Identity;
using Azure.Storage.Blobs;

var credential = new DefaultAzureCredential();

var blobServiceClient = new BlobServiceClient(
    new Uri("https://<youraccount>.blob.core.windows.net"),
    credential);

builder.Services.AddSingleton(blobServiceClient);
```

### How `DefaultAzureCredential` resolves identity:
- **Locally** → uses signed-in dev credentials (Azure CLI, Visual Studio, VS Code)
- **In Azure** → uses the App Service's Managed Identity
- Azure then checks the caller's identity against **RBAC role assignments** on the target resource — not a matching key

---

## Key Takeaways
- No secrets in code or config — nothing to leak or rotate.
- Access control lives in **IAM role assignments**, not app settings.
- Works seamlessly across **local dev** and **deployed environments** using the same code path.
- Roles differ by storage service (Blob / Queue / Table) — assign the correct scoped role for what the app needs.