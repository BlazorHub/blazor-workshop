# 显示订单状态

您的客户可以订购比萨饼，但到目前为止，他们没有办法看到他们的订单状态。在这节课中，您将实现一个"我的订单"页面，该页面列出了多个订单，以及显示单个订单内容和状态的"订单详细信息"视图。 

## 添加导航链接

打开 `Shared/MainLayout.razor`. 作为实验，让我们尝试添加新的链接元素，而无需使用组件`NavLink`  ，添加一个普通的Html `<a>` 到 `myorders`:

```html
<div class="top-bar">
    (leave existing content in place)

    <a href="myorders" class="nav-tab">
        <img src="img/bike.svg" />
        <div>My Orders</div>
    </a>
</div>
```

> 请注意，我们<base href="/">index.html 链接到的 URL 以`/` 开头。如果链接到 `/myorders` ，它看起来工作相同，但如果您曾经希望将应用部署到非根 URL，则链接将中断。`index.html` 中的标记`<base href="/">`指定应用中所有非斜杠预固定 URL 的前缀，而不管哪个组件呈现它们。

如果现在运行该应用程序，您将看到该链接，该链接的样式如下：

![My orders link](https://user-images.githubusercontent.com/1874516/77241321-a03ba880-6bad-11ea-9a46-c73be397cb5e.png)


这表明`<NavLink>`它不是必须使用的。我们将在后面看到使用它的原因。 

## 添加"我的订单"页面

如果您单击"My Orders"，您将最终出现在一个页面，该页面上显示"对不起，此地址没有任何内容"。显然，这是因为您尚未添加任何与 URL `myorders`匹配的页面。但是，如果你非常仔细地观察，你可能会注意到，在这个时候，它不只是做客户端（SPA样式）导航，而是在做全页重新加载。

这背后真正发生的事情是：

1. 单击指向 `myorders`
2. 在客户端上运行的 Blazor 尝试将此与基于指令属性`@page` 的客户端组件匹配 
3. 但是，由于找不到匹配项，Blazor 会返回全页加载导航，以防 URL 由服务器端代码处理。  
4. 但是，服务器也没有任何匹配，因此它落在呈现客户端 Blazor 应用程序上来兜底。 
5. 这一次，Blazor 发现客户端或服务器上没有任何匹配，因此它落在从`App.razor` 组件`NotFound` 呈现块上。 

你可以尝试修改`App.razor`组件块`NotFound`中的内容，以查看如何自定义此消息。 

正如您可以猜到的，我们将通过添加一个组件来匹配此路由来使链接真正发挥作用。在名为 `Pages`  的文件夹中创建一个文件`MyOrders.razor`，包含以下内容： 

```html
@page "/myorders"

<div class="main">
    My orders will go here
</div>
```

现在，当您运行该应用程序时，您将能够访问此页面：

![My orders blank page](https://user-images.githubusercontent.com/1874516/77241343-fc9ec800-6bad-11ea-8176-febf614ed4ad.png)

另请注意，这一次，在导航时不会发生全页加载，因为 URL 完全在客户端 SPA 中匹配。因此，导航是瞬时完成的。  

## 突出显示导航位置

仔细观察顶部栏。请注意，当您在"我的订单"上时，链接不会以黄色突出显示。当用户位于链接上时，我们如何突出显示链接？通过使用组件`NavLink`而不是普通标记 `<a>`。`NavLink`组件唯一的特殊做法是切换自己的 CSS 类`active`，具体取决于`href`其是否与当前导航状态匹配。 

将 `MainLayout` 里刚刚添加的标记`<a>` 替换为以下内容（与标记名称相同） :

```html
<NavLink href="myorders" class="nav-tab">
    <img src="img/bike.svg" />
    <div>My Orders</div>
</NavLink>
```

现在您将看到链接根据导航状态正确突出显示：

![My orders nav link](https://user-images.githubusercontent.com/1874516/77241358-412a6380-6bae-11ea-88da-424434d34393.png)

## 显示订单列表

切换回组件`MyOrders`代码。我们再次将注入`HttpClient`，以便我们可以查询后端的数据。在`@page`指令行下添加以下内容： 

```html
@inject HttpClient HttpClient
```

然后添加一个块 `@code` ，用于对我们需要的数据发出异步请求：

```csharp
@code {
    List<OrderWithStatus> ordersWithStatus;

    protected override async Task OnParametersSetAsync()
    {
        ordersWithStatus = await HttpClient.GetFromJsonAsync<List<OrderWithStatus>>("orders");
    }
}
```

让我们在三种不同情况下使 UI 显示不同的输出:

 1. 在等待加载数据时
 2. 如果用户从未下过任何订单
 3. 如果用户下了一个或多个订单

使用 Razor 代码中的块`@if/else`来表达这一点非常简单。更新组件内的标记，如下所示： 

```html
<div class="main">
    @if (ordersWithStatus == null)
    {
        <text>Loading...</text>
    }
    else if (ordersWithStatus.Count == 0)
    {
        <h2>No orders placed</h2>
        <a class="btn btn-success" href="">Order some pizza</a>
    }
    else
    {
        <text>TODO: show orders</text>
    }
</div>
```

也许这个代码的某些部分并不明显，所以让我们指出一些。

### 1.   `<text>` 元素是什么?

`<text>` 根本不是 HTML 元素。它也不是组件。编译组件`MyOrders`后，结果中根本不存在标记。 

`<text>` 是向 Razor 编译器发出的一个特殊信号，您希望将其内容视为标记字符串，而不是C# 源代码。它仅在语法不明确的罕见情况下使用。

### 2.   href="" 是怎么回事?

如果`<a href="">`  （带`href` ） 的空字符串意外，请记住，浏览器将该值 `<base href="/">`前缀到所有非斜线预固定 URL。因此，空字符串是链接到客户端应用的根 URL 的正确方法。  

### 3. 如何呈现？

我们在上面实现的异步流意味着组件将呈现两次：一次在加载数据之前（显示"Loading.."），然后一次（显示其他两个输出之一）。

### 4. 我们为什么要使用 OnParametersSetAsync?

应用参数和属性值时，异步工作必须在 OnparametersSetAsync 生命周期事件期间发生。我们将在以后的课程中添加一个参数。

### 5. 如何重置数据库?

如果要重置数据库以查看"无订单"情况，只需从"服务器"项目中删除`pizza.db`并在浏览器中重新加载页面即可  

![My orders empty list](https://user-images.githubusercontent.com/1874516/77241390-a4b49100-6bae-11ea-8dd4-e59afdd8f710.png)

## 呈现订单Grid

N现在，我们拥有了所需的所有数据，我们可以使用 Razor 语法来呈现 HTML Grid。  

将代码`<text>TODO: show orders</text>` 替换为以下内容 :

```html
<div class="list-group orders-list">
    @foreach (var item in ordersWithStatus)
    {
        <div class="list-group-item">
            <div class="col">
                <h5>@item.Order.CreatedTime.ToLongDateString()</h5>
                Items:
                <strong>@item.Order.Pizzas.Count()</strong>;
                Total price:
                <strong>£@item.Order.GetFormattedTotalPrice()</strong>
            </div>
            <div class="col">
                Status: <strong>@item.StatusText</strong>
            </div>
            <div class="col flex-grow-0">
                <a href="myorders/@item.Order.OrderId" class="btn btn-success">
                    Track &gt;
                </a>
            </div>
        </div>
    }
</div>
```

它看起来像很多代码，但这里没有什么特别之处。它只需使用 `@foreach`  迭代`ordersWithStatus` 和 输出`<div>`  。最终结果如下：

![My orders grid](https://user-images.githubusercontent.com/1874516/77241415-feb55680-6bae-11ea-89ba-f8367ef6a96c.png)

## 添加订单详细信息显示

如果您单击订单旁边的"Track"链接按钮，浏览器将尝试导航到 `myorders/<id>`（例如 `http://example.com/myorders/37` 。目前，这将导致"对不起，此地址没有任何内容"消息，因为没有组件匹配此路由。

我们再次添加一个组件来处理这一点。在目录中`Pages`，创建名为`OrderDetails.razor` 的文件，包含： 

```html
@page "/myorders/{orderId:int}"

<div class="main">
    TODO: Show details for order @OrderId
</div>

@code {
    [Parameter] public int OrderId { get; set; }
}
```

此代码说明了组件如何通过在指令 `@page`中将其声明为令牌来从路由器接收参数。如果要接收`string` ， 语法只是 `{parameterName}` ， 与名称大小写不敏感匹配。如果要接收数值，则语法为`{parameterName:int}` ，如上例所示。`:int`是路由约束的示例。其他路由约束也受支持。 

![Order details empty](https://user-images.githubusercontent.com/1874516/77241434-391ef380-6baf-11ea-9803-9e7e65a4ea2b.png)

如果您想知道路由的实际工作原理，让我们逐步完成它。

1. 当应用首次启动时，`Program.cs`中的代码会告诉框架`App`作为根组件呈现。 
2.  `App`组件 （在`App.razor` 中） 包含 `<Router>` 。 `Router` 是一个内置组件，与浏览器的客户端导航 API 进行交互。它注册一个导航事件处理程序，每当用户单击链接时都会收到通知。
3. 每当用户单击链接时，`Router` 中的代码都会检查目标 URL 是否在同一 SPA 中（即它是否在`<base href>`值之下，并且它与某些组件声明的路由匹配）。如果不是，传统的全页导航照常进行。但是，如果 URL 在 SPA 中，`Router`将处理它。
4. `Router`  通过查找具有兼容 `@page`URL 模式的组件来处理它。每个`{parameter}`令牌都需要有一个值，并且该值必须与任何约束（如  `:int`） 兼容。 
   * 如果有匹配的组件，`Router`则呈现该组件。这是应用程序中的所有页面一直呈现的方式。
   * 如果没有匹配组件，路由器将尝试全页加载，以防其与服务器上的内容匹配。 
   * 如果服务器选择重新呈现客户端 Blazor 应用（如果访问者最初到达此 URL 并且服务器认为它可能是客户端路由，则也会发生这种情况），则 Blazor 得出结论，服务器或客户端上没有任何匹配，因此它显示 `NotFound`配置的任何内容。 

## 轮询订单详细信息

`OrderDetails`逻辑将完全不同于`MyOrders` 。而是在实例化组件时只获取数据一次，我们将每隔几秒轮询服务器以获取更新的数据。这样，就可以在（几乎）实时以及以后实时显示订单状态，以便在移动地图上显示传递驱动程序的位置。 

此外，我们还将考虑`OrderId`无效的可能性。如果出现这种情况： 

* 不存在此类订单
* 或者后面当我们实现身份验证时，如果订单是针对其他用户的，并且不允许您看到它 

在实现轮询之前，我们需要在`OrderDetails.razor` 顶部添加以下指令，通常直接在`@page` 指令下： 

```html
@using System.Threading
@inject HttpClient HttpClient
```

您已经看到 `HttpClient`使用`@inject` ，所以你知道这是注入HttpClient用的。此外，您将从常规文件`.cs` 中的等效文件中识别`@using` ，因此这也不应该是一个谜。遗憾的是，Visual Studio 尚未在 Razor 文件中自动添加指令，因此您必须在需要时自行编写指令。 

现在，您可以实现轮询。更新`@code`块如下： 

```cs
@code {
    [Parameter] public int OrderId { get; set; }

    OrderWithStatus orderWithStatus;
    bool invalidOrder;
    CancellationTokenSource pollingCancellationToken;

    protected override void OnParametersSet()
    {
        // If we were already polling for a different order, stop doing so
        pollingCancellationToken?.Cancel();

        // Start a new poll loop
        PollForUpdates();
    }

    private async void PollForUpdates()
    {
        pollingCancellationToken = new CancellationTokenSource();
        while (!pollingCancellationToken.IsCancellationRequested)
        {
            try
            {
                invalidOrder = false;
                orderWithStatus = await HttpClient.GetFromJsonAsync<OrderWithStatus>($"orders/{OrderId}");
            }
            catch (Exception ex)
            {
                invalidOrder = true;
                pollingCancellationToken.Cancel();
                Console.Error.WriteLine(ex);
            }

            StateHasChanged();

            await Task.Delay(4000);
        }
    }
}
```

代码有点复杂，所以在继续之前，请务必仔细了解它的各个方面。以下是一些注意事项：

* This uses `OnParametersSet` instead of `OnInitialized` or `OnInitializedAsync`. `OnParametersSet` is another component lifecycle method, and it fires when the component is first instantiated *and* any time its parameters change value. If the user clicks a link directly from `myorders/2` to `myorders/3`, the framework will retain the `OrderDetails` instance and simply update its `OrderId` parameter in place.
  * As it happens, we haven't provided any links from one "my orders" screen to another, so the scenario never occurs in this application, but it's still the right lifecycle method to use in case we change the navigation rules in the future.
* We're using an `async void` method to represent the polling. This method runs for arbitrarily long, even while other methods run. `async void` methods have no way to report exceptions upstream to callers (because typically the callers have already finished), so it's important to use `try/catch` and do something meaningful with any exceptions that may occur.
* We're using `CancellationTokenSource` as a way of signalling when the polling should stop. Currently it only stops if there's an exception, but we'll add another stopping condition later.
* We need to call `StateHasChanged` to tell Blazor that the component's data has (possibly) changed. The framework will then re-render the component. There's no way that the framework could know when to re-render your component otherwise, because it doesn't know about your polling logic.

## Rendering the order details

OK, so we're getting the order details, and we're even polling and updating that data every few seconds. But we're still not rendering it in the UI. Let's fix that. Update your `<div class="main">` as follows:

```html
<div class="main">
    @if (invalidOrder)
    {
        <h2>Nope</h2>
        <p>Sorry, this order could not be loaded.</p>
    }
    else if (orderWithStatus == null)
    {
        <text>Loading...</text>
    }
    else
    {
        <div class="track-order">
            <div class="track-order-title">
                <h2>
                    Order placed @orderWithStatus.Order.CreatedTime.ToLongDateString()
                </h2>
                <p class="ml-auto mb-0">
                    Status: <strong>@orderWithStatus.StatusText</strong>
                </p>
            </div>
            <div class="track-order-body">
                TODO: show more details
            </div>
        </div>
    }
</div>
```

This accounts for the three main states of the component:

1. If the `OrderId` value is invalid (i.e., the server reported an error when we tried to retrieve the data)
2. If we haven't yet loaded the data
3. If we have got some data to show

![Order details status](https://user-images.githubusercontent.com/1874516/77241460-a7fc4c80-6baf-11ea-80c1-3286374e9e29.png)


The last bit of UI we want to add is the actual contents of the order. To do this, we'll create another reusable component.

Create a new file, `OrderReview.razor` inside the `Shared` directory, and have it receive an `Order` and render its contents as follows:

```html
@foreach (var pizza in Order.Pizzas)
{
    <p>
        <strong>
            @(pizza.Size)"
            @pizza.Special.Name
            (£@pizza.GetFormattedTotalPrice())
        </strong>
    </p>

    <ul>
        @foreach (var topping in pizza.Toppings)
        {
            <li>+ @topping.Topping.Name</li>
        }
    </ul>
}

<p>
    <strong>
        Total price:
        £@Order.GetFormattedTotalPrice()
    </strong>
</p>

@code {
    [Parameter] public Order Order { get; set; }
}
```

Finally, back in `OrderDetails.razor`, replace text `TODO: show more details` with your new `OrderReview` component:

```html
<div class="track-order-body">
    <div class="track-order-details">
        <OrderReview Order="orderWithStatus.Order" />
    </div>
</div>
```

(Don't forget to add the extra `div` with CSS class `track-order-details`, as this is necessary for correct styling.)

Finally, you have a functional order details display!

![Order details](https://user-images.githubusercontent.com/1874516/77241512-2e189300-6bb0-11ea-9740-fe778e0ce622.png)


## See it update in realtime

The backend server will update the order status to simulate an actual dispatch and delivery process. To see this in action, try placing a new order, then immediately view its details.

Initially, the order status will be *Preparing*, then after 10-15 seconds the order status will change to *Out for delivery*, then 60 seconds later it will change to *Delivered*. Because `OrderDetails` polls for updates, the UI will update without the user having to refresh the page.

## Remember to Dispose!

If you deployed your app to production right now, bad things would happen. The `OrderDetails` logic starts a polling process, but doesn't end it. If the user navigated through hundreds of different orders (thereby creating hundreds of different `OrderDetails` instances), then there would be hundreds of polling processes left running concurrently, even though all except the last were pointless because no UI was displaying their results.

You can actually observe this chaos yourself as follows:

1. Navigate to "my orders"
2. Click "Track" on any order to get to its details
3. Click "Back" to return to "my orders"
4. Repeat steps 2 and 3 a lot of times (e.g., 20 times)
5. Now, open your browser's debugging tools and look in the network tab. You should see 20 or more HTTP requests being issued every few seconds, because there are 20 or more concurrent polling processes.

This is wasteful of client-side memory and CPU time, network bandwidth, and server resources.

To fix this, we need to make `OrderDetails` stop the polling once it gets removed from the display. This is simply a matter of using the `IDisposable` interface.

In `OrderDetails.razor`, add the following directive at the top of the file, underneath the other directives:

```html
@implements IDisposable
```

Now if you try to compile the application, the compiler will complain:

```
error CS0535: 'OrderDetails' does not implement interface member 'IDisposable.Dispose()'
```

Resolve this by adding the following method inside the `@code` block:

```cs
void IDisposable.Dispose()
{
    pollingCancellationToken?.Cancel();
}
```

The framework calls `Dispose` automatically when any given component instance is torn down and removed from the UI.

Once you've put in this fix, you can try again to start lots of concurrent polling processes, and you'll see they no longer keep running after the component is gone. Now, the only component that continues to poll is the one that remains on the screen.

## Automatically navigating to order details

Right now, once users place an order, the `Index` component simply resets its state and their order appears to vanish without a trace. This is not very reassuring for users. We know the order is in the database, but users don't know that.

It would be nice if, once the order is placed, the app automatically navigated to the "order details" display for that order. This is quite easy to do.

Switch back to your `Index` component code. Add the following directive at the top:

```
@inject NavigationManager NavigationManager
```

The `NavigationManager` lets you interact with URIs and navigation state. It has methods to get the current URL, to navigate to a different one, and more.

To use this, update the `PlaceOrder` code so it calls `NavigationManager.NavigateTo`:

```csharp
async Task PlaceOrder()
{
    var response = await HttpClient.PostAsJsonAsync("orders", order);
    var newOrderId = await response.Content.ReadFromJsonAsync<int>();
    order = new Order();
    NavigationManager.NavigateTo($"myorders/{newOrderId}");
}
```

Now as soon as the server accepts the order, the browser will switch to the "order details" display and begin polling for updates.

Next up - [Refactor state management](04-refactor-state-management.md)
