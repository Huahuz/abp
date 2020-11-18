# Web应用程序开发教程  - 第五章: 授权

````json
//[doc-params]
{
    "UI": ["MVC","Blazor","NG"],
    "DB": ["EF","Mongo"]
}
````

## 关于本教程

在本系列教程中,你将要建立一个名为`Acme.BookStore`的基于abp的应用程序，这个程序用于管理一系列图书以及它们的作者。开发它使用了以下技术：

* **{{DB_Value}}** 作为对象关系映射提供程序. 
* **{{UI_Value}}** 作为UI框架.

本教程分为以下几个部分;

- [Part 1: 创建服务端](Part-1.md)
- [Part 2: 图书列表页面](Part-2.md)
- [Part 3: 创建,更新和删除图书](Part-3.md)
- [Part 4: 集成测试](Part-4.md)
- **Part 5: 授权 (本章)**
- [Part 6: 作者: 领域层](Part-6.md)
- [Part 7: 作者: 数据库集成](Part-7.md)
- [Part 8: 作者: 应用服务层](Part-8.md)
- [Part 9: 作者: 用户页面](Part-9.md)
- [Part 10: 图书到作者的关系](Part-10.md)

### 下载源码

本教程根据你的 **UI** 和 **Database** 偏好有多个版,我们准备了两种可供下载的源码组合:

* [MVC (Razor Pages) UI with EF Core](https://github.com/abpframework/abp-samples/tree/master/BookStore-Mvc-EfCore)
* [Blazor UI with EF Core](https://github.com/abpframework/abp-samples/tree/master/BookStore-Blazor-EfCore)
* [Angular UI with MongoDB](https://github.com/abpframework/abp-samples/tree/master/BookStore-Angular-MongoDb)

{{if UI == "MVC" && DB == "EF"}}

### 视频教程

本章也被录制成视频 **<a href="https://www.youtube.com/watch?v=1WsfMITN_Jk&list=PLsNclT2aHJcPNaCf7Io3DbMN6yAk_DgWJ&index=5" target="_blank">并上传到了YouTube </a>**.

{{end}}

## 权限

ABP 框架提供了一个 [authorization system](../Authorization.md) 基于 ASP.NET Core的 [authorization infrastructure](https://docs.microsoft.com/en-us/aspnet/core/security/authorization/introduction)。在标准授权基础设施之上添加的一个主要特性是 **permission system** ，它允许为每一个角色、用户和客户端定义权限并启用和停用它们。

### 权限名称

一个权限必须拥有一个唯一的名称 ( `字符串`). 最好的方法是把它定义成一个 `const`常量, 这样方便我们重用这些名称。

打开`Acme.BookStore.Application.Contracts`项目（在 `Permissions` 文件夹中）下的 `BookStorePermissions` 类并如下改变其内容：

````csharp
namespace Acme.BookStore.Permissions
{
    public static class BookStorePermissions
    {
        public const string GroupName = "BookStore";

        public static class Books
        {
            public const string Default = GroupName + ".Books";
            public const string Create = Default + ".Create";
            public const string Edit = Default + ".Edit";
            public const string Delete = Default + ".Delete";
        }
    }
}
````

这是一种分层定义权限名称的方式。例如， 权限名称"create book" 被定义成 `BookStore.Books.Create`.

### 权限定义

你需要在使用权限之前定义它们。

打开项目`Acme.BookStore.Application.Contracts`（在`Permissions`文件夹中）下的类并如下替换其实现：

````csharp
using Acme.BookStore.Localization;
using Volo.Abp.Authorization.Permissions;
using Volo.Abp.Localization;

namespace Acme.BookStore.Permissions
{
    public class BookStorePermissionDefinitionProvider : PermissionDefinitionProvider
    {
        public override void Define(IPermissionDefinitionContext context)
        {
            var bookStoreGroup = context.AddGroup(BookStorePermissions.GroupName, L("Permission:BookStore"));

            var booksPermission = bookStoreGroup.AddPermission(BookStorePermissions.Books.Default, L("Permission:Books"));
            booksPermission.AddChild(BookStorePermissions.Books.Create, L("Permission:Books.Create"));
            booksPermission.AddChild(BookStorePermissions.Books.Edit, L("Permission:Books.Edit"));
            booksPermission.AddChild(BookStorePermissions.Books.Delete, L("Permission:Books.Delete"));
        }

        private static LocalizableString L(string name)
        {
            return LocalizableString.Create<BookStoreResource>(name);
        }
    }
}
````

这个类定义了一个**权限组**（用于UI中的权限，下面即将看到），它包含有 **4个权限** 。同时，**Create**, **Edit** 和**Delete** 是`BookStorePermissions.Books.Default`的子权限。子权限只能在父级权限被选择的情况下才能被选择。

最后，编辑本地化文件（`Localization/BookStore`文件夹下的`en.json`）如下定义上面使用到的多语言键：

````json
"Permission:BookStore": "Book Store",
"Permission:Books": "Book Management",
"Permission:Books.Create": "Creating new books",
"Permission:Books.Edit": "Editing the books",
"Permission:Books.Delete": "Deleting the books"
````

> 本地化键名是任意的，没有强制规则。但我们更喜欢像上面一样按惯例使用。

### 权限管理 UI

一旦是定义了权限，你就可以在**permission management modal**中看到它们。

打开 *管理 -> 身份 -> 角色* 页面, 选择管理员权限的*权限* 操作来打开权限管理modal：

![bookstore-permissions-ui](images/bookstore-permissions-ui.png)

授予你想要的权限并保存modal.

> **提示**: 当你执行`Acme.BookStore.DbMigrator`应用，新的权限会自动赋予管理员角色。

## 授权

现在, 你可以使用这些权限来授权这个图书管理。

### 应用层 & HTTP API

打开 `BookAppService` 类并如下设置权限名称来添加和设置策略名：

````csharp
using System;
using Acme.BookStore.Permissions;
using Volo.Abp.Application.Dtos;
using Volo.Abp.Application.Services;
using Volo.Abp.Domain.Repositories;

namespace Acme.BookStore.Books
{
    public class BookAppService :
        CrudAppService<
            Book, //The Book entity
            BookDto, //Used to show books
            Guid, //Primary key of the book entity
            PagedAndSortedResultRequestDto, //Used for paging/sorting
            CreateUpdateBookDto>, //Used to create/update a book
        IBookAppService //implement the IBookAppService
    {
        public BookAppService(IRepository<Book, Guid> repository)
            : base(repository)
        {
            GetPolicyName = BookStorePermissions.Books.Default;
            GetListPolicyName = BookStorePermissions.Books.Default;
            CreatePolicyName = BookStorePermissions.Books.Create;
            UpdatePolicyName = BookStorePermissions.Books.Edit;
            DeletePolicyName = BookStorePermissions.Books.Delete;
        }
    }
}
````

向构造函数中添加代码。基于`CrudAppService` 自动在增删改查操作中使用这些权限。因此，正如前面解释的那样，这不仅保证了 **应用服务** 的安全，还保证了**TTP API** 的安全(参看[auto API controllers](../API/Auto-API-Controllers.md))。.

> 稍后在开发作者管理功能时你将会看到使用`[Authorize(...)]`属性来声明授权。

{{if UI == "MVC"}}

### Razor 页面

在保护API和应用服务的同时，仍然允许他们访问图书管理页面。虽然当页面对服务器进行第一次AJAX调用时，它们将获得授权异常，但我们还应该对页面进行授权，以获得更好的用户体验和安全性 。

打开 `BookStoreWebModule` 并添加如下代码到方法 `ConfigureServices` 中:

````csharp
Configure<RazorPagesOptions>(options =>
{
    options.Conventions.AuthorizePage("/Books/Index", BookStorePermissions.Books.Default);
    options.Conventions.AuthorizePage("/Books/CreateModal", BookStorePermissions.Books.Create);
    options.Conventions.AuthorizePage("/Books/EditModal", BookStorePermissions.Books.Edit);
});
````

现在，未授权的用户将会被重定向至 **登录页面**。

#### 隐藏新建图书按钮

如果当前用户没有新建图书权限，那么图书管理页面就会隐藏那个*新建书籍* 的按钮。

![bookstore-new-book-button-small](images/bookstore-new-book-button-small.png)

打开 `Pages/Books/Index.cshtml` 并如下修改其内容：

````html
@page
@using Acme.BookStore.Localization
@using Acme.BookStore.Permissions
@using Acme.BookStore.Web.Pages.Books
@using Microsoft.AspNetCore.Authorization
@using Microsoft.Extensions.Localization
@model IndexModel
@inject IStringLocalizer<BookStoreResource> L
@inject IAuthorizationService AuthorizationService
@section scripts
{
    <abp-script src="/Pages/Books/Index.js"/>
}

<abp-card>
    <abp-card-header>
        <abp-row>
            <abp-column size-md="_6">
                <abp-card-title>@L["Books"]</abp-card-title>
            </abp-column>
            <abp-column size-md="_6" class="text-right">
                @if (await AuthorizationService.IsGrantedAsync(BookStorePermissions.Books.Create))
                {
                    <abp-button id="NewBookButton"
                                text="@L["NewBook"].Value"
                                icon="plus"
                                button-type="Primary"/>
                }
            </abp-column>
        </abp-row>
    </abp-card-header>
    <abp-card-body>
        <abp-table striped-rows="true" id="BooksTable"></abp-table>
    </abp-card-body>
</abp-card>
````

* 添加 `@inject IAuthorizationService AuthorizationService` 来访问授权服务。
* 使用`@if (await AuthorizationService.IsGrantedAsync(BookStorePermissions.Books.Create))` 来检查新建图书权限并根据情况渲染 *新建图书* 按钮。

### JavaScript 方面

在图书管理页面的图书表格的每一行都有一个按钮，这些按钮包含 *编辑* 和*删除* 功能：

![bookstore-edit-delete-actions](images/bookstore-edit-delete-actions.png)

如果当前用户没有相关权限的时候我们应该隐藏那个功能。表格行操作有一个 `visible`选项，当它被设置为`false`的时候就可以隐藏该行的功能。

打开`Acme.BookStore.Web`项目中的 `Pages/Books/Index.js` 文件并如下在`Edit`功能中添加一个`visible`选项：

````js
{
    text: l('Edit'),
    visible: abp.auth.isGranted('BookStore.Books.Edit'), //CHECK for the PERMISSION
    action: function (data) {
        editModal.open({ id: data.record.id });
    }
}
````

我们在 `Delete` 功能中做同样的操作:

````js
visible: abp.auth.isGranted('BookStore.Books.Delete')
````

* `abp.auth.isGranted(...)` 是用于检查一个权限是否已经被定义。
* `visible` 也可以根据情况从一个返回`bool`的方法中计算获得。

### 菜单条目

虽然我们已经保护了图书管理页面的所有层，但是我们还是应该让它们可以通过菜单可以被访问。如果当前用户没有没有权限的时候，它们就应该被隐藏。

打开 `BookStoreMenuContributor` 类，找到如下代码： 

````csharp
context.Menu.AddItem(
    new ApplicationMenuItem(
        "BooksStore",
        l["Menu:BookStore"],
        icon: "fa fa-book"
    ).AddItem(
        new ApplicationMenuItem(
            "BooksStore.Books",
            l["Menu:Books"],
            url: "/Books"
        )
    )
);
````

并替换成以下内容：

````csharp
var bookStoreMenu = new ApplicationMenuItem(
    "BooksStore",
    l["Menu:BookStore"],
    icon: "fa fa-book"
);

context.Menu.AddItem(bookStoreMenu);

//CHECK the PERMISSION
if (await context.IsGrantedAsync(BookStorePermissions.Books.Default))
{
    bookStoreMenu.AddItem(new ApplicationMenuItem(
        "BooksStore.Books",
        l["Menu:Books"],
        url: "/Books"
    ));
}
````

你也需要添加`async`关键字到`ConfigureMenuAsync`方法前面，然后重新排列返回值。最终`BookStoreMenuContributor`类内容如下：

````csharp
using System.Threading.Tasks;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Localization;
using Acme.BookStore.Localization;
using Acme.BookStore.MultiTenancy;
using Acme.BookStore.Permissions;
using Volo.Abp.TenantManagement.Web.Navigation;
using Volo.Abp.UI.Navigation;

namespace Acme.BookStore.Web.Menus
{
    public class BookStoreMenuContributor : IMenuContributor
    {
        public async Task ConfigureMenuAsync(MenuConfigurationContext context)
        {
            if (context.Menu.Name == StandardMenus.Main)
            {
                await ConfigureMainMenuAsync(context);
            }
        }

        private async Task ConfigureMainMenuAsync(MenuConfigurationContext context)
        {
            if (!MultiTenancyConsts.IsEnabled)
            {
                var administration = context.Menu.GetAdministration();
                administration.TryRemoveMenuItem(TenantManagementMenuNames.GroupName);
            }

            var l = context.GetLocalizer<BookStoreResource>();

            context.Menu.Items.Insert(0, new ApplicationMenuItem("BookStore.Home", l["Menu:Home"], "~/"));

            var bookStoreMenu = new ApplicationMenuItem(
                "BooksStore",
                l["Menu:BookStore"],
                icon: "fa fa-book"
            );

            context.Menu.AddItem(bookStoreMenu);

            //CHECK the PERMISSION
            if (await context.IsGrantedAsync(BookStorePermissions.Books.Default))
            {
                bookStoreMenu.AddItem(new ApplicationMenuItem(
                    "BooksStore.Books",
                    l["Menu:Books"],
                    url: "/Books"
                ));
            }
        }
    }
}
````

{{else if UI == "NG"}}

### Angular 指导配置

UI的第一步是防止未授权的用户看到名为"图书"的菜单条目并通过它进入图书管理页面。

打开 `/src/app/book/book-routing.module.ts` 并如下替换其内容:

````js
import { NgModule } from '@angular/core';
import { Routes, RouterModule } from '@angular/router';
import { AuthGuard, PermissionGuard } from '@abp/ng.core';
import { BookComponent } from './book.component';

const routes: Routes = [
  { path: '', component: BookComponent, canActivate: [AuthGuard, PermissionGuard] },
];

@NgModule({
  imports: [RouterModule.forChild(routes)],
  exports: [RouterModule],
})
export class BookRoutingModule {}
````

* 从`@abp/ng.core`中导入`AuthGuard` 和`PermissionGuard`。
* 添加 `canActivate: [AuthGuard, PermissionGuard]`到路由定义。

打开 `/src/app/route.provider.ts` 并添加 `requiredPolicy: 'BookStore.Books'` 到 `/books` 路由.  `/books` 路由模块应当如下：

````js
{
  path: '/books',
  name: '::Menu:Books',
  parentName: '::Menu:BookStore',
  layout: eLayoutType.application,
  requiredPolicy: 'BookStore.Books',
}
````

### 隐藏新建图书按钮

在图书管理页面， 如果当前用户没有没有新建图书的权限，那么*新建图书* 按钮就不能被该用户访问 。

![bookstore-new-book-button-small](images/bookstore-new-book-button-small.png)

打开 `/src/app/book/book.component.html` 文件并如下替换按钮的HTML内容：

````html
<!-- Add the abpPermission directive -->
<button abpPermission="BookStore.Books.Create" id="create" class="btn btn-primary" type="button" (click)="createBook()">
  <i class="fa fa-plus mr-1"></i>
  <span>{%{{{ '::NewBook' | abpLocalization }}}%}</span>
</button>
````

* 如果当前用户没有权限，只需要添加 `abpPermission="BookStore.Books.Create"` 来隐藏这个按钮。

### 隐藏编辑和删除功能

图书管理页面的图书表格的每一行都有一个按钮，这个按钮包含编辑和删除功能：

![bookstore-edit-delete-actions](images/bookstore-edit-delete-actions.png)

 如果当前用户没有相关的权限，我们就应该隐藏这些功能。

打开 `/src/app/book/book.component.html` 文件并如下替换编辑和删除按钮的HTML内容：

````html
<!-- Add the abpPermission directive -->
<button abpPermission="BookStore.Books.Edit" ngbDropdownItem (click)="editBook(row.id)">
  {%{{{ '::Edit' | abpLocalization }}}%}
</button>

<!-- Add the abpPermission directive -->
<button abpPermission="BookStore.Books.Delete" ngbDropdownItem (click)="delete(row.id)">
  {%{{{ '::Delete' | abpLocalization }}}%}
</button>
````

* 添加 `abpPermission="BookStore.Books.Edit"` 之后，如果当前用户没有编辑权限就隐藏编辑按钮。
* 添加`abpPermission="BookStore.Books.Delete"` 之后，如果当前用户没有删除权限就隐藏删除按钮。

{{else if UI == "Blazor"}}

### 授权Razor组件

打开`Acme.BookStore.Blazor`项目中的 `/Pages/Books.razor` 文件，如下在`@page`命令之后添加`Authorize` 特性和以下命名空间（`@using`开始的行）：

````html
@page "/books"
@attribute [Authorize(BookStorePermissions.Books.Default)]
@using Acme.BookStore.Permissions
@using Microsoft.AspNetCore.Authorization
...
````

添加这个特性可以组织那些没有登录和没有被授予权限的用户访问这个页面。如果用户尝试访问就会被重定向到登录界面。

### 展示/隐藏按钮

图书管理界面有一个*新增图书* 、*编辑图书* 和*删除图书* 的按钮。当前用户没有相关权限的时候我们应该隐藏它们。

#### 初始化过程中获取权限

添加如下代码到`Books.razor`中：

````csharp
@code
{
    bool canCreateBook;
    bool canEditBook;
    bool canDeleteBook;

    protected override async Task OnInitializedAsync()
    {
        await base.OnInitializedAsync();

        canCreateBook =await AuthorizationService.IsGrantedAsync(BookStorePermissions.Books.Create);
        canEditBook = await AuthorizationService.IsGrantedAsync(BookStorePermissions.Books.Edit);
        canDeleteBook = await AuthorizationService.IsGrantedAsync(BookStorePermissions.Books.Delete);
    }
}
````

我们将使用这些`bool`字段来检查权限。`AuthorizationService` 是来自基类的注入属性。

> **Blazor 提示**: 当添加少量的C#代码到`@code`块中是不错的，我们仍然建议当代码更多的时候使用代码隐藏方法来开发更易于维护的代码库。我们即将在作者那一节使用这个方法。

#### 隐藏新建书籍按钮

如下将*新建书籍* 包在`if`代码块中：

````xml
@if (canCreateBook)
{
    <Button Color="Color.Primary"
            Clicked="OpenCreateModalAsync">
        @L["NewBook"]
    </Button>
}
````

#### 隐藏编辑/删除按钮

和*新建图书*按钮一样，我们可以使用`if`代码块来根据情况展示/隐藏 *编辑* 和 *删除* 操作：

````xml
@if (canEditBook)
{
    <DropdownItem Clicked="() => OpenEditModalAsync(context.Id)">
        @L["Edit"]
    </DropdownItem>
}
@if (canDeleteBook)
{
    <DropdownItem Clicked="() => DeleteEntityAsync(context)">
        @L["Delete"]
    </DropdownItem>
}
````

#### 关于权限缓存

你可以运行和测试权限，从管理员角色中移除一个图书相关的权限来观察按钮/操作从UI中删除。

**ABP 框架在客户端缓存当前用户的权限** 。因此，当你自行修改了权限之后，你需要手动**刷新**受影响的页面。如果你没有刷新页面并尝试使用禁止的操作，你将从服务器收到一个 HTTP 403 (forbidden) 的响应。

> 改变一个角色或用户的权限将在服务器端立即生效，所以缓存系统并不会有任何安全问题。

### 菜单项

虽然我们已经保护了图书管理页面的所有层，但是我们还是应该让它们可以通过菜单可以被访问。如果当前用户没有没有权限的时候，它们就应该被隐藏。

打开 `Acme.BookStore.Blazor`项目中的 `BookStoreMenuContributor` 类，并找到如下代码块：

````csharp
context.Menu.AddItem(
    new ApplicationMenuItem(
        "BooksStore",
        l["Menu:BookStore"],
        icon: "fa fa-book"
    ).AddItem(
        new ApplicationMenuItem(
            "BooksStore.Books",
            l["Menu:Books"],
            url: "/books"
        )
    )
);
````

然后替换该代码块中的内容如下:

````csharp
var bookStoreMenu = new ApplicationMenuItem(
    "BooksStore",
    l["Menu:BookStore"],
    icon: "fa fa-book"
);

context.Menu.AddItem(bookStoreMenu);

//CHECK the PERMISSION
if (await context.IsGrantedAsync(BookStorePermissions.Books.Default))
{
    bookStoreMenu.AddItem(new ApplicationMenuItem(
        "BooksStore.Books",
        l["Menu:Books"],
        url: "/books"
    ));
}
````

你也需要添加`async`关键字到`ConfigureMenuAsync`方法前面，然后重新排列返回值。最终`BookStoreMenuContributor`类内容如下：

````csharp
using System.Threading.Tasks;
using Acme.BookStore.Localization;
using Acme.BookStore.Permissions;
using Volo.Abp.UI.Navigation;

namespace Acme.BookStore.Blazor
{
    public class BookStoreMenuContributor : IMenuContributor
    {
        public async Task ConfigureMenuAsync(MenuConfigurationContext context)
        {
            if(context.Menu.DisplayName != StandardMenus.Main)
            {
                return;
            }

            var l = context.GetLocalizer<BookStoreResource>();

            context.Menu.Items.Insert(
                0,
                new ApplicationMenuItem(
                    "BookStore.Home",
                    l["Menu:Home"],
                    "/",
                    icon: "fas fa-home"
                )
            );

            var bookStoreMenu = new ApplicationMenuItem(
                "BooksStore",
                l["Menu:BookStore"],
                icon: "fa fa-book"
            );

            context.Menu.AddItem(bookStoreMenu);

            if (await context.IsGrantedAsync(BookStorePermissions.Books.Default))
            {
                bookStoreMenu.AddItem(new ApplicationMenuItem(
                    "BooksStore.Books",
                    l["Menu:Books"],
                    url: "/books"
                ));
            }
        }
    }
}
````

{{end}}

## 下一章

查看本教程的 [下一章](Part-6.md) .

