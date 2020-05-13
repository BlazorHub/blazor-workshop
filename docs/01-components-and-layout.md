# 组件和布局：开始使用组件，创建应用布局

在本课程中，将开始使用blazor 构建披萨店应用程序。该应用程序将使用户能够订购披萨，对其进行自定义，然后跟踪订单的交付情况。

## 披萨店的起点

我们已经在仓库中为披萨店应用程序设置了初始解决方案，你将在 *save-points* 文件夹中找到[starting point](../save-points/00-get-started) ，以及这些课程的最终状态.

> 注意：如果将代码从本Workshop 复制到计算机上的其他位置，请确保还复制此存储库 根目录中的Directory.Build.props文件，确保恢复适当的软件包版本。

该解决方案已经包含四个项目:

![image](https://user-images.githubusercontent.com/1874516/77238114-e2072780-6b8a-11ea-8e44-de6d7910183e.png)


- **BlazingPizza.Client**: 这是Blazor项目。它包含应用程序的UI组件.
- **BlazingPizza.Server**: 这是承载Blazor应用程序以及该应用程序的后端服务的ASP.NET Core项目.
- **BlazingPizza.Shared**: 应用程序的共享模型类型.
- **BlazingPizza.ComponentsLibrary**: 在以后的课程中应用程序将使用的组件和帮助程序代码的库.

项目 **BlazingPizza.Server** 项目应设置为启动项目

运行该应用程序时，您会看到它当前仅包含一个简单的主页.

![image](https://user-images.githubusercontent.com/1874516/77238160-25fa2c80-6b8b-11ea-8145-e163a9f743fe.png)

打开项目 **BlazingPizza.Client** 的 *Pages/Index.razor*  查看首页代码.

```
@page "/"

<h1>Blazing Pizzas</h1>
```

主页被实现为单个组件。 指令 `@page` 指定该 `Index` 组件是具有指定路由的可路由页面.

## 显示特价比萨列表

首先，我们将更新主页以显示可用披萨特价的列表。特价商品列表将成为 `Index` 组件状态的一部分。 

添加一个代码块  `@code` 使用列表字段 到 *Index.razor* 以跟踪可用的特价披萨:

```csharp
@code {
    List<PizzaSpecial> specials;
}
```

`@code` 块中的代码将添加到该组件的生成的类中。该`PizzaSpecial` 类型已在共享项目中为您定义 

要获取可用的特价商品列表，我们需要在后端调用API。Blazor提供了HttpClient通过依赖项注入进行的预配置，该注入已使用正确的基地址进行设置。使用@inject指令将an HttpClient注入到Index组件中。
指令实质上定义组件上的新属性，其中第一个令牌指定属性类型，第二个令牌指定属性名称。使用依赖项注入为您填充该属性。 

要获取可用的特价商品列表，我们需要在调用后端 API 获取数据 。Blazor 提供通过依赖项注入预配置  `HttpClient` t，该注入已使用正确的基本地址进行设置。使用 指令将 `@inject` 注入 `Index` 组件。 


```
@page "/"
@inject HttpClient HttpClient
```

指令`@inject`  实质上定义组件上的新属性，其中第一个指定属性类型，第二个令牌指定属性名称。使用依赖项注入为您填充该属性。
 
重写块中`@code` 的方法`OnInitializedAsync` 以检索特价披萨列表。此方法是组件生命周期的一部分，在初始化组件时调用此方法。使用 `GetFromJsonAsync<T>()` 方法处理响应 JSON 反序列化：

```csharp
@code {
    List<PizzaSpecial> specials;

    protected override async Task OnInitializedAsync()
    {
        specials = await HttpClient.GetFromJsonAsync<List<PizzaSpecial>>("specials");
    }
}
```

 `/specials` API 由 服务器项目中的  `SpecialsController` 定义。 

 一旦组件初始化，它将呈现其标记。将 `Index`  组件中的标记替换为以下内容，以列出特价披萨：

```html
<div class="main">
    <ul class="pizza-cards">
        @if (specials != null)
        {
            @foreach (var special in specials)
            {
                <li style="background-image: url('@special.ImageUrl')">
                    <div class="pizza-info">
                        <span class="title">@special.Name</span>
                        @special.Description
                        <span class="price">@special.GetFormattedBasePrice()</span>
                    </div>
                </li>
            }
        }
    </ul>
</div>
```

通过点击`Ctrl-F5` 运行应用程序。现在，您应该会看到可用的特价披萨列表。  

![image](https://user-images.githubusercontent.com/1874516/77239386-6c558880-6b97-11ea-9a14-83933146ba68.png)


##  创建布局

接下来，我们将为应用设置布局。

Layouts 在 Blazor 中也是组件。 他们从 `LayoutComponentBase` 继承, 该属性定义可用于指定布局正文的呈现位置的属性 `Body` 。 我们的比萨饼商店应用程序的布局组件在 *Shared/MainLayout.razor* 中定义。  

```html
@inherits LayoutComponentBase

<div class="content">
    @Body
</div>
```

要查看布局如何与页面关联，请查看 `App.razor` 中的组件`<Router>` 。请注意， `DefaultLayout` 该参数确定用于不直接指定其自身布局的任何页面的布局。  

您还可以按页重写此项`DefaultLayout` 。为此，可以添加指令 `@layout SomeOtherLayout`，如在任何页面组件的顶部。但是，您不需要在这个应用程序中执行此项操作。

更新组件`MainLayout`以定义带有品牌徽标和主页导航链接的顶部栏：

```html
@inherits LayoutComponentBase

<div class="top-bar">
    <img class="logo" src="img/logo.svg" />

    <NavLink href="" class="nav-tab" Match="NavLinkMatch.All">
        <img src="img/pizza-slice.svg" />
        <div>Get Pizza</div>
    </NavLink>
</div>

<div class="content">
    @Body
</div>
```

组件`NavLink`由 Blazor 提供。 组件可以通过指定具有组件类型名称的元素以及任何组件参数的属性从组件中使用。 

组件`NavLink`与锚点标记相同，只不过在当前 URL 与链接地址匹配时，该组件会添加 class `active`。 `NavLinkMatch.All` 意味着链接应仅在与整个当前 URL 匹配时才处于活动状态（而不仅仅是前缀）。我们将在后面的课程中中更详细地使用组件 `NavLink` 。 

通过点击 `Ctrl-F5` 运行应用程序。有了我们的新布局，我们的比萨饼店应用程序现在如下所示： 

![image](https://user-images.githubusercontent.com/1874516/77239419-aa52ac80-6b97-11ea-84ae-f880db776f5c.png)


下一步 - [定制披萨饼](02-customize-a-pizza.md)
