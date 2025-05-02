# Get .NET Enum Value Display Name

It is possible to specify the UI display name for enum members using the [`DisplayAttribute`](https://learn.microsoft.com/en-us/dotnet/api/system.componentmodel.dataannotations.displayattribute) and its [`Name`](https://learn.microsoft.com/en-us/dotnet/api/system.componentmodel.dataannotations.displayattribute.name) property.

```csharp
using System.ComponentModel.DataAnnotations;

namespace MyNamespace;

public enum UserRole
{
    [Display(Name = "System admin")]
    SystemAdmin,
    Manager,
    [Display(Name = "Sales rep")]
    SalesRep,
    Technician
}
```

Some ASP.NET functions like [`HtmlHelper.GetEnumSelectList`](https://learn.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.mvc.viewfeatures.htmlhelper.getenumselectlist) use the display name specified by the `DisplayAttribute`, but there is no built-in function to get the display name given an enum value. So I had to write this function.

```csharp
using System.Collections.Concurrent;
using System.ComponentModel.DataAnnotations;
using System.Reflection;

namespace MyNamespace;

public static class EnumDisplayNameGetter
{
    private static readonly ConcurrentDictionary<Enum, string> _displayNameCache = new();

    public static string GetDisplayName(this Enum enumValue)
    {
        ArgumentNullException.ThrowIfNull(enumValue);

        return _displayNameCache.GetOrAdd(enumValue,
            ev =>
            {
                DisplayAttribute? displayAttribute = GetDisplayAttribute(ev);
                return displayAttribute?.GetName() ?? ev.ToString();
            }
        );
    }

    private static DisplayAttribute? GetDisplayAttribute(Enum enumValue)
    {
        Type enumType = enumValue.GetType();
        FieldInfo? fieldInfo = enumType.GetField(enumValue.ToString(),
            BindingFlags.Public | BindingFlags.Static);
        return fieldInfo?.GetCustomAttribute<DisplayAttribute>();
    }
}
```

The code above uses a [`ConcurrentDictionary`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.concurrent.concurrentdictionary-2) to cache enum value display names when they are fetched for the first time. So the slow reflection code does not need to run more than once per application lifetime for each enum value. The extension method `GetDisplayName()` is thread-safe and can be used in multi-threaded applications like ASP.NET apps.

My implementation is derived from [another implementation that I found in the OpenAPI.NET project](https://github.com/microsoft/OpenAPI.NET/blob/3f8b2b99c07bd3c58825728c2dd2ffed91d88fbe/src/Microsoft.OpenApi/Extensions/EnumExtensions.cs#L52).

The following code examples show how the `GetDisplayName()` extension method can be used.

```csharp
string displayName = UserRole.SystemAdmin.GetDisplayName();
```

```csharp
UserRole role = UserRole.SystemAdmin;
string displayName = role.GetDisplayName();
```
