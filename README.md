# 🚀 ASP.NET Core Role & Permission Management System

A simple and clear **Role-Based Access Control (RBAC)** system for ASP.NET Core.  
This project shows how to manage **roles**, **permissions**, and **UI authorization** easily.

---

## 📘 1. Introduction
This system allows admins to:
- Create **permissions** (Page, Grid, Button, etc.)
- Assign **permissions to roles**
- Automatically hide unauthorized buttons or pages
- Manage everything from an admin panel

---

## ⚙️ 2. Project Setup

### Required Packages:
```bash
Microsoft.AspNetCore.Identity.EntityFrameworkCore
Microsoft.EntityFrameworkCore
Microsoft.AspNetCore.Authorization
Microsoft.AspNetCore.Mvc.Razor.RuntimeCompilation
```

Register services in `Program.cs`:
```csharp
services.AddScoped<IClaimsTransformation, PermissionClaimsTransformation>();
services.AddScoped<IPermissionService, PermissionService>();
services.AddAuthorizationPolicies();
```

---

## 🧩 3. Models

### 🟢 Permission.cs
```csharp
public class Permission : BaseEntity<Guid>
{
    public string Name { get; set; }
    public string DisplayName { get; set; }
    public Guid? ParentId { get; set; }
    public Permission Parent { get; set; }
    public ICollection<Permission> Children { get; set; }
    public string ElementType { get; set; }
}
```

### 🟢 RolePermission.cs
```csharp
public class RolePermission : BaseEntity<Guid>
{
    public string RoleId { get; set; }
    public Guid PermissionId { get; set; }
    public Permission Permission { get; set; }
}
```

---

## 🛠️ 4. Authorization Tag Helper
```csharp
[HtmlTargetElement(Attributes = "asp-authorize-policy")]
public class AuthorizationTagHelper : TagHelper
{
    private readonly IAuthorizationService _authorizationService;

    [ViewContext]
    [HtmlAttributeNotBound]
    public ViewContext ViewContext { get; set; }

    [HtmlAttributeName("asp-authorize-policy")]
    public string PolicyName { get; set; }

    public AuthorizationTagHelper(IAuthorizationService authorizationService)
    {
        _authorizationService = authorizationService;
    }

    public override async Task ProcessAsync(TagHelperContext context, TagHelperOutput output)
    {
        var user = ViewContext.HttpContext.User;

        if (user == null || !user.Identity.IsAuthenticated)
        {
            output.SuppressOutput();
            return;
        }

        var authorized = await _authorizationService.AuthorizeAsync(user, PolicyName);

        if (!authorized.Succeeded)
            output.SuppressOutput();
    }
}
```

Example usage:
```html
<button asp-authorize-policy="Class.Grid2.View">View Grid 2</button>
```

---

## 🔐 5. Permission Service
```csharp
public interface IPermissionService
{
    Task<IEnumerable<string>> GetUserPermissionsAsync(string userId);
}

public class PermissionService : IPermissionService
{
    private readonly ApplicationDbContext _context;
    private readonly UserManager<ApplicationUser> _userManager;

    public PermissionService(ApplicationDbContext context, UserManager<ApplicationUser> userManager)
    {
        _context = context;
        _userManager = userManager;
    }

    public async Task<IEnumerable<string>> GetUserPermissionsAsync(string userId)
    {
        var user = await _userManager.FindByIdAsync(userId);
        if (user == null) return Enumerable.Empty<string>();

        var roleNames = await _userManager.GetRolesAsync(user);
        var roleIds = _context.Roles.Where(r => roleNames.Contains(r.Name)).Select(r => r.Id).ToList();

        var permissionNames = await _context.RolePermissions
            .Where(rp => roleIds.Contains(rp.RoleId))
            .Select(rp => rp.Permission.Name)
            .Distinct()
            .ToListAsync();

        return permissionNames;
    }
}
```

---

## 🧾 6. Claims Transformation
```csharp
public class PermissionClaimsTransformation : IClaimsTransformation
{
    private readonly IPermissionService _permissionService;

    public PermissionClaimsTransformation(IPermissionService permissionService)
    {
        _permissionService = permissionService;
    }

    public async Task<ClaimsPrincipal> TransformAsync(ClaimsPrincipal principal)
    {
        if (principal.Identity is not ClaimsIdentity identity || !identity.IsAuthenticated)
            return principal;

        if (identity.HasClaim(c => c.Type == "Permission"))
            return principal;

        var userId = identity.FindFirst(ClaimTypes.NameIdentifier)?.Value;
        if (string.IsNullOrEmpty(userId))
            return principal;

        var permissions = await _permissionService.GetUserPermissionsAsync(userId);

        foreach (var permissionName in permissions)
            identity.AddClaim(new Claim("Permission", permissionName));

        return principal;
    }
}
```

---

## 📦 7. View Models
```csharp
public class RoleViewModel { public string Id { get; set; } public string Name { get; set; } }
public class PermissionNode
{
    public Guid Id { get; set; }
    public string Name { get; set; }
    public string DisplayName { get; set; }
    public bool IsSelected { get; set; }
    public Guid? ParentId { get; set; }
    public List<PermissionNode> Children { get; set; } = new();
}
public class ManageRolePermissionsViewModel
{
    public string RoleId { get; set; }
    public string RoleName { get; set; }
    public List<PermissionNode> AllPermissions { get; set; }
    public List<Guid> SelectedPermissionIds { get; set; }
}
public class UpdatePermissionsRequest { public string RoleId { get; set; } public List<Guid> SelectedPermissionIds { get; set; } }
```

---

## 🧭 8. Roles PageModel
*(Backend logic for managing roles and permissions)*

```csharp
// Code omitted here for brevity; same as previous detailed explanation
```

---

## 💻 9. JavaScript (roles.js)
```javascript
$(document).ready(function () {
    const $permissionModal = $('#permissionModal');
    const modal = new bootstrap.Modal($permissionModal[0]);
    const $modalTitle = $('#modalRoleName');
    const $modalRoleIdInput = $('#modalRoleId');
    const $treeContainer = $('#permissionsTreeContainer');
    const $permissionForm = $('#permissionForm');
    const antiforgeryToken = $('input[name="__RequestVerificationToken"]').val();

    $('.edit-permissions-btn').on('click', function () {
        const roleId = $(this).data('role-id');
        const roleName = $(this).data('role-name');
        $modalTitle.text(roleName);
        $modalRoleIdInput.val(roleId);
        loadPermissions(roleId);
        modal.show();
    });

    function loadPermissions(roleId) {
        $.get(`/Roles?handler=PermissionsForRole&roleId=${roleId}`, function (data) {
            const permissions = data.AllPermissions;
            $treeContainer.html(buildTreeHtml(permissions));
        });
    }

    function buildTreeHtml(nodes) {
        let html = '<ul>';
        $.each(nodes, function (i, node) {
            const checked = node.IsSelected ? 'checked' : '';
            html += `<li><label><input type="checkbox" class="permission-checkbox" value="${node.Id}" ${checked}/> ${node.DisplayName}</label>`;
            if (node.Children.length > 0) html += buildTreeHtml(node.Children);
            html += '</li>';
        });
        return html + '</ul>';
    }

    $permissionForm.on('submit', function (event) {
        event.preventDefault();
        const selectedIds = $treeContainer.find('.permission-checkbox:checked').map((_, el) => el.value).get();
        $.ajax({
            url: '/Roles?handler=UpdateRolePermissions',
            type: 'POST',
            contentType: 'application/json',
            data: JSON.stringify({ RoleId: $modalRoleIdInput.val(), SelectedPermissionIds: selectedIds }),
            headers: { 'RequestVerificationToken': antiforgeryToken },
            success: () => { alert('Saved!'); modal.hide(); }
        });
    });
});
```

---

## 🖥️ 10. Razor Page (Roles.cshtml)
```html
@page "/Roles"
@model MagridMVC.Presentation.Pages.RolesModel
@inject IAntiforgery Xsrf
@Html.AntiForgeryToken()

<h2>User Role Management</h2>
<p class="text-muted">Select a role to manage permissions.</p>

<table class="table table-striped">
    <thead><tr><th>Role Name</th><th>Actions</th></tr></thead>
    <tbody>
        @foreach (var role in Model.Roles)
        {
            <tr>
                <td>@role.Name</td>
                <td>
                    <button class="btn btn-sm btn-primary edit-permissions-btn"
                            data-role-id="@role.Id"
                            data-role-name="@role.Name">
                        Manage Permissions
                    </button>
                </td>
            </tr>
        }
    </tbody>
</table>

<div class="modal fade" id="permissionModal">
    <div class="modal-dialog modal-lg">
        <div class="modal-content">
            <form id="permissionForm" method="post">
                @Html.AntiForgeryToken()
                <div class="modal-header">
                    <h5>Manage Permissions: <span id="modalRoleName"></span></h5>
                    <button type="button" class="btn-close" data-bs-dismiss="modal"></button>
                </div>
                <div class="modal-body">
                    <input type="hidden" id="modalRoleId" />
                    <div id="permissionsTreeContainer">Loading...</div>
                </div>
                <div class="modal-footer">
                    <button type="button" class="btn btn-secondary" data-bs-dismiss="modal">Cancel</button>
                    <button type="submit" class="btn btn-success">Save</button>
                </div>
            </form>
        </div>
    </div>
</div>

@section Scripts {
    <script src="~/pages/account/roles.js"></script>
}
```

---

## ⚙️ 11. Policy Setup
```csharp
public static class AuthorizationPolicyConfig
{
    public static IServiceCollection AddAuthorizationPolicies(this IServiceCollection services)
    {
        services.AddAuthorization(options =>
        {
            options.AddPolicy("Class.Grid1.Add", policy => policy.RequireClaim("Permission", "Class.Grid1.Add"));
            options.AddPolicy("Class.Grid2.View", policy => policy.RequireClaim("Permission", "Class.Grid2.View"));
            options.AddPolicy("Class.Grid1.TopGridDownload", policy => policy.RequireClaim("Permission", "Class.Grid1.TopGridDownload"));
        });
        return services;
    }
}
```

---

## 🔁 12. System Flow
1. Admin defines **Permissions**
2. Assigns them to **Roles**
3. User logs in → their **permissions become claims**
4. UI uses `asp-authorize-policy` to hide unauthorized elements
5. Admin manages permissions in `/Roles` page

---

🧠 **Simple. Secure. Flexible.**
