# JavaScript 互操作：在地图上实时跟踪订单状态

披萨店的用户现在可以实时跟踪其订单状态。在本课中，我们将使用JavaScript interop将实时地图添加到订单状态页面，以回答老的问题：“我的披萨在哪里？！？”

## 地图组件

ComponentsLibrary项目中包含一个预构建的 `Map`组件，用于显示一组标记的位置并随时间推移对其移动进行动画处理。我们将使用此组件显示用户的披萨饼订单在交付时的位置，但首先让我们看一下如何Map实现该组件。
 
打开 *Map.razor* 并查看代码:

```csharp
@using Microsoft.JSInterop
@inject IJSRuntime JSRuntime

<div id="@elementId" style="height: 100%; width: 100%;"></div>

@code {
    string elementId = $"map-{Guid.NewGuid().ToString("D")}";
    
    [Parameter] double Zoom { get; set; }
    [Parameter] List<Marker> Markers { get; set; }

    protected async override Task OnAfterRenderAsync(bool firstRender)
    {
        await JSRuntime.InvokeVoidAsync(
            "deliveryMap.showOrUpdate",
            elementId,
            Markers);
    }
}
```

该`Map` 组件使用依赖项注入来获取 `IJSRuntime` 实例。此服务可用于通过调用`InvokeVoidAsync` 或者 `InvokeAsync<TResult>` 方法来对浏览器API或现有JavaScript库进行JavaScript调用。此方法的第一个参数指定要相对于根`window` 对象调用的JavaScript函数的路径。其余参数是传递给JavaScript函数的参数。这些参数已序列化为JSON，因此可以在JavaScript中进行处理。

该 `Map` 组件首先呈现一个`div` 与用于地图一个唯一的ID，然后调用`deliveryMap.showOrUpdate` 函数来显示与传递给指定的标志物的指定的元素映射`Map`组件。这是在`OnAfterRenderAsync` 组件生命周期事件中完成的，以确保组件完成了其标记的呈现。该`deliveryMap.showOrUpdate`函数在 *wwwroot/deliveryMap.js*  文件中定义，然后使用[leaflet.js](http://leafletjs.com) 和 [OpenStreetMap](https://www.openstreetmap.org/) 显示地图。这段代码的工作原理并不是很重要-关键点是可以用这种方式调用任何JavaScript函数。
 
这些文件如何进入Blazor应用程序？对于Blazor库项目（使用 `Sdk="Microsoft.NET.Sdk.Razor"`），该`wwwroot/`文件夹中的任何文件都将与该库捆绑在一起。服务器项目将使用静态文件中间件自动提供这些文件。

最终链接用于托管Blazor客户端应用程序的页面，其中包含所需的文件（在我们的示例中为`.js` 和 `.css`）。在`index.html` 使用像相对URI包含这些文 `_content/BlazingPizza.ComponentsLibrary/localStorage.js` 。这是与Blazor类库-捆绑在一起的引用文件的一般模式`_content/<library name>/<file path>`

---

如果您开始输入 `Map` ，您会注意到编辑器没有为此提供完成功能。这是因为元素和组件之间的绑定受C＃的命名空间绑定规则支配。该`Map` 组件在`BlazingPizza.ComponentsLibrary.Map`名称空间中定义，而我们没有`@using` 

在根目录中`_Imports.razor` 为此名称空间添加一个`@using`，以将该组件纳入范围 :

```razor
@using System.Net.Http
@using Microsoft.AspNetCore.Authorization
@using Microsoft.AspNetCore.Components.Authorization
@using Microsoft.AspNetCore.Components.Forms
@using Microsoft.AspNetCore.Components.Routing
@using Microsoft.JSInterop
@using BlazingPizza.Client
@using BlazingPizza.Client.Shared
@using BlazingPizza.ComponentsLibrary
@using BlazingPizza.ComponentsLibrary.Map
```

通过在下方添加以下内容，将`Map`组件添加到 `OrderDetails` 页面  `track-order-details` `div` 下面:

```html
<div class="track-order-map">
    <Map Zoom="13" Markers="orderWithStatus.MapMarkers" />
</div>
```

*之所以现在不需要`@using` 为组件添加  的原因是，我们的根目录`_Imports.razor` 已经包含一个 `@using BlazingPizza.Shared` 与我们编写的可重用组件匹配的 *

当`OrderDetails` 组件轮询订单状态更新时，将返回一组更新的标记以及比萨的最新位置，然后将其反映在地图上

![Real-time pizza map](https://user-images.githubusercontent.com/1874516/51807322-6018b880-227d-11e9-89e5-ef75f03466b9.gif)

## 删除披萨时添加确认提示 

`Map` 已为您提供了该组件的JavaScript互操作代码。接下来，您将添加一些自己的JavaScript互操作代码
 
如果用户不小心从他们的订单中删除了披萨饼，而最终却没有购买，那就太可惜了。让我们在用户尝试删除披萨时添加确认提示。 

将静态`JSRuntimeExtensions` 类添加到Client项目，并使用的`Confirm` 方法扩展`IJSRuntime` 。实现该`confirm`  方法以调用内置JavaScript `confirm` m函数。

```csharp
public static class JSRuntimeExtensions
{
    public static ValueTask<bool> Confirm(this IJSRuntime jsRuntime, string message)
    {
        return jsRuntime.InvokeAsync<bool>("confirm", message);
    }
}
```

将`IJSRuntime`  服务注入`Index` 组件，以便可以在其中进行JavaScript互操作调用。  

```razor
@page "/"
@inject HttpClient HttpClient
@inject OrderState OrderState
@inject NavigationManager NavigationManager
@inject IJSRuntime JS
```

将异步 `RemovePizza` 方法添加到`Index` 调用该`Confirm` 方法的组件，以验证用户是否真的要从订单中删除披萨。 

```csharp
async Task RemovePizza(Pizza configuredPizza)
{
    if (await JS.Confirm($"Remove {configuredPizza.Special.Name} pizza from the order?"))
    {
        OrderState.RemoveConfiguredPizza(configuredPizza);
    }
}
```

在`Index` 组件中，更新事件处理程序`ConfiguredPizzaItems`以调用新`RemovePizza` 方法。  

```csharp
@foreach (var configuredPizza in OrderState.Order.Pizzas)
{
    <ConfiguredPizzaItem Pizza="configuredPizza" OnRemoved="@(() => RemovePizza(configuredPizza))" />
}
```

运行该应用，然后尝试从订单中删除披萨。

![Confirm pizza removal](https://user-images.githubusercontent.com/1874516/77243688-34b40400-6bca-11ea-9d1c-331fecc8e307.png)

注意，我们不必更新签名`ConfiguredPizzaItem.OnRemoved` 即可支持异步。这是的另一个特殊属性`EventCallback` ，它支持同步事件处理程序和异步事件处理程序。


下一步 - [验证用户身份并授权用户访问订单状态](07-authentication-and-authorization.md)
