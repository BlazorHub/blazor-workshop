# 重构状态管理

在本节中，我们将回顾一些我们已经编写好的代码，并尝试使其变得更好。我们还将讨论事件以及事件如何更新UI。
 
## 问题

您可能已经注意到了这一点，是我们的应用程序存在错误！由于我们以当前顺序将披萨饼列表存储在Index 组件上，因此如果用户离开 Index 页面，则用户的状态可能会丢失。要查看实际效果，请在当前订单中添加一个披萨饼（不要下订单）- 然后导航至MyOrders页面并返回 Index。当您回来时，您会注意到订单为空！ 

## 解决方案

我们将通过引入一些我们称为 *AppState pattern* 的东西来修复此错误。*AppState pattern* 将对象添加到DI容器，您将使用相关组件之间的协调状态。因为*AppState pattern* 是由DI容器管理的，所以即使UI发生更改，它也可以使组件寿命更长并保持状态。*AppState pattern* 的另一个好处是，它让界面（组件）与业务逻辑之间的更大分离。  

## 入门教程

 在Client Project根目录中创建一个新类`OrderState` - 并将其注册为DI容器中的作用域scoped 服务。在Blazor WebAssembly应用程序中， 通过`Program` 类的`Main`方法中注册。在调用`await builder.Build().RunAsync();` 之前添加服务
 

```csharp
public static async Task Main(string[] args)
{
    var builder = WebAssemblyHostBuilder.CreateDefault(args);
    builder.RootComponents.Add<App>("app");

    builder.Services.AddTransient(sp => new HttpClient { BaseAddress = new Uri(builder.HostEnvironment.BaseAddress) });
    builder.Services.AddScoped<OrderState>();

    await builder.Build().RunAsync();
}
```

> 注意：我们选择作用域而不是单例的原因是为了与服务器端组件应用程序对应。单例通常表示对所有用户，其中范围表示当前工作单元。 

## 更新 Index

现在，此类型已在DI中注册，我们可以将`@inject`其输入 `Index` 页面。  

```razor
@page "/"
@inject HttpClient HttpClient
@inject OrderState OrderState
@inject NavigationManager NavigationManager
```

回想一下，这 `@inject` 是一种方便的快捷方式，既可以按类型从DI检索内容，又可以定义该类型的属性。  

您现在可以通过再次运行该应用程序进行测试。如果尝试注入在DI容器中找不到的内容，则它将引发异常，并且`Index`  页面将无法显示。
 
现在，让我们向此类添加属性和方法，以表示和操纵一个 `Order`和一个`Pizza`的状态。
 
将`configuringPizza`, `showingConfigureDialog` 和 `order` 移动到 `OrderState` 类的属性。 设置为private set，使其只能通过上的方法进行操作`OrderState`。
 

```csharp
public class OrderState
{
    public bool ShowingConfigureDialog { get; private set; }

    public Pizza ConfiguringPizza { get; private set; }

    public Order Order { get; private set; } = new Order();
}
```

现在，让我们将一些方法从`Index` 移至`OrderState`. 。我们不会把`PlaceOrder` 移进入`OrderState`， 因为这会触发导航，所以我们只添加一个`ResetOrder` 方法。

```csharp
public void ShowConfigurePizzaDialog(PizzaSpecial special)
{
    ConfiguringPizza = new Pizza()
    {
        Special = special,
        SpecialId = special.Id,
        Size = Pizza.DefaultSize,
        Toppings = new List<PizzaTopping>(),
    };

    ShowingConfigureDialog = true;
}

public void CancelConfigurePizzaDialog()
{
    ConfiguringPizza = null;

    ShowingConfigureDialog = false;
}

public void ConfirmConfigurePizzaDialog()
{
    Order.Pizzas.Add(ConfiguringPizza);
    ConfiguringPizza = null;

    ShowingConfigureDialog = false;
}

public void ResetOrder()
{
    Order = new Order();
}

public void RemoveConfiguredPizza(Pizza pizza)
{
    Order.Pizzas.Remove(pizza);
}
```

请记住从`Index.razor` 中删除相应的方法 。你一定要记得从 `Index.razor` 删除`order`, `configuringPizza`, and `showingConfigureDialog`  ，因为你会得到从注入的状态数据`OrderState` 中得到

 此时，应该有可能`Index` 通过更新引用以引用附加到的各个位来再次编译组件`OrderState`. 。例如，`Index.razor`中的 `PlaceOrder`方法 如下所示： 

```csharp
async Task PlaceOrder()
{
    var response = await HttpClient.PostAsJsonAsync("orders", OrderState.Order);
    var newOrderId = await response.Content.ReadFromJsonAsync<int>();
    OrderState.ResetOrder();
    NavigationManager.NavigateTo($"myorders/{newOrderId}");
}
```

随意为诸如此类的东西 `OrderState.Order` 或者 `OrderState.Order.Pizzas` 对您感觉更好的事物创建便利属性。

试试看并验证一切是否仍然有效。特别是，请确认您已解决了原始错误：现在可以添加一些披萨，导航到“我的订单”，向后导航，并且您的订单不再丢失。 

## 探索状态变化

这是 探讨状态变化和渲染在Blazor中如何工作的好机会，以及`EventCallback`如何解决一些常见问题 。现在，`OrderState` 所涉及的事情的细节变得更加复杂。

`EventCallback` 告诉Blazor将事件通知（和呈现）分派到定义事件处理程序的组件。如果事件处理程序不是由组件（`OrderState`）定义的，则它将替换挂接事件处理程序（`Index`）的组件。
 
## 结论

因此，让我们总结一下*AppState pattern* 提供的内容:
- 将共享状态从组件外移到 `OrderState`
- 组件调用方法以触发状态更改
- `EventCallback`  负责发送变更通知

我们还介绍了许多有关渲染和事件的信息:
- 当参数更改或收到事件时，组件会重新渲染
- 事件的分派取决于事件处理程序委托目标
- 用 `EventCallback` 最灵活和友好的方式调度事件

下一步 - [验证并结账](05-checkout-with-validation.md)
