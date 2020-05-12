# 组件和布局

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
To get the available list of specials we need to call an API on the backend. Blazor provides a preconfigured `HttpClient` through dependency injection that is already setup with the correct base address. Use the `@inject` directive to inject an `HttpClient` into the `Index` component.

```
@page "/"
@inject HttpClient HttpClient
```

The `@inject` directive essentially defines a new property on the component where the first token specifies the property type and the second token specifies the property name. The property is populated for you using dependency injection.

Override the `OnInitializedAsync` method in the `@code` block to retrieve the list of pizza specials. This method is part of the component lifecycle and is called when the component is initialized. Use the `GetFromJsonAsync<T>()` method to handle deserializing the response JSON:

```csharp
@code {
    List<PizzaSpecial> specials;

    protected override async Task OnInitializedAsync()
    {
        specials = await HttpClient.GetFromJsonAsync<List<PizzaSpecial>>("specials");
    }
}
```

The `/specials` API is defined by the `SpecialsController` in the Server project.

Once the component is initialized it will render its markup. Replace the markup in the `Index` component with the following to list the pizza specials:

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

Run the app by hitting `Ctrl-F5`. Now you should see a list of the specials that are available.

![image](https://user-images.githubusercontent.com/1874516/77239386-6c558880-6b97-11ea-9a14-83933146ba68.png)


## Create the layout

Next we'll set up the layout for the app. 

Layouts in Blazor are also components. They inherit from `LayoutComponentBase`, which defines a `Body` property that can be used to specify where the body of the layout should be rendered. The layout component for our pizza store app is defined in *Shared/MainLayout.razor*.

```html
@inherits LayoutComponentBase

<div class="content">
    @Body
</div>
```

To see how the layout is associated with your pages, look at the `<Router>` component in `App.razor`. Notice that the `DefaultLayout` parameter determines the layout used for any page that doesn't specify its own layout directly.

You can also override this `DefaultLayout` on a per-page basis. To do so, you can add a directive such as `@layout SomeOtherLayout` at the top of any `.razor` page component. However, you will not need to do so in this application.

Update the `MainLayout` component to define a top bar with a branding logo and a nav link for the home page:

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

The `NavLink` component is provided by Blazor. Components can be used from components by specifying an element with the component's type name along with attributes for any component parameters.

The `NavLink` component is the same as an anchor tag, except that it adds an `active` class if the current URL matches the link address. `NavLinkMatch.All` means that the link should be active only when it matches the entire current URL (not just a prefix). We'll examine the `NavLink` component in more detail in a later session.

Run the app by hitting `Ctrl-F5`. With our new layout, our pizza store app now looks like this:

![image](https://user-images.githubusercontent.com/1874516/77239419-aa52ac80-6b97-11ea-84ae-f880db776f5c.png)


Next up - [Customize a pizza](02-customize-a-pizza.md)
