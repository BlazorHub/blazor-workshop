# 显示订单状态：显示订单状态

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

* 这使用 `OnParametersSet`而不是`OnInitialized` 或 `OnParametersSet`。 是另一种组件生命周期方法，它在第一次实例化组件时触发，并且每当其参数更改值时，它将触发。如果用户单击直接从`myorders/2`  到`myorders/3` 的链接，则框架将保留实例`OrderDetails`，并仅将其参数`OrderId`更新到位
 * 碰巧，我们没有提供从一个"my orders" 屏幕到另一个屏幕的任何链接，因此此应用程序中永远不会发生该方案，但将来更改导航规则时，它仍然是正确的生命周期方法。
* 我们使用`async void`的方法来表示轮询。即使其他方法运行，此方法也会任意长时间运行。 `async void`方法无法向调用方报告上游的异常（因为调用方通常已完成），因此，对可能发生的任何异常使用`try/catch`并执行有意义的操作非常重要。
* 我们使用`CancellationTokenSource`作为一种信号，告诉轮询应该停止。目前，它仅在出现异常时停止，但我们稍后将添加另一个停止条件。 
* 我们需要调用`StateHasChanged` 告诉Blazor组件的数据（可能）已更改。然后，框架将重新呈现组件。框架无法知道何时重新呈现组件，因为它不知道轮询逻辑。 

## 呈现订单详细信息

好了，我们得到订单详细信息，我们甚至每隔几秒轮询和更新数据。但是，我们仍未在 UI 中呈现它。让我们来解决这个问题。更新`<div class="main">`  如下： 

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

这说明了组件的三个主要状态：

1. 如果该值`OrderId`无效（即，当我们尝试检索数据时，服务器报告了错误）
2. 如果尚未加载数据
3. 如果我们有一些数据要显示

![Order details status](https://user-images.githubusercontent.com/1874516/77241460-a7fc4c80-6baf-11ea-80c1-3286374e9e29.png)


要添加的最后一位 UI 是订单的实际内容。为此，我们将创建另一个可重用组件。

在`Shared` 目录中创建新文件`OrderReview.razor`，并让它接收`Order`  和呈现其内容，如下所示：

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

最后，返回 `OrderDetails.razor` ，将文本替换`TODO: show more details` 为新组件`OrderReview`：

```html
<div class="track-order-body">
    <div class="track-order-details">
        <OrderReview Order="orderWithStatus.Order" />
    </div>
</div>
```

（不要忘记`div` 使用 CSS 类添加额外`track-order-details`，因为这是正确样式所必需的）

最后，您将看到功能订单明细显示！

![Order details](https://user-images.githubusercontent.com/1874516/77241512-2e189300-6bb0-11ea-9740-fe778e0ce622.png)


## 查看实时更新

后端服务器将更新订单状态以模拟实际的调度和交付过程。要查看实际效果，请尝试下新订单，然后立即查看其详细信息。  

最初，订单状态将为正在准备，然后10-15秒后，订单状态将更改为待交付，然后60秒后它将更改为已交付。因为`OrderDetails` 轮询更新，所以UI将更新而无需用户刷新页面。 

## 记住需要 Dispose!

如果您现在将应用程序部署到生产环境中，那么将会发生不好的事情。该 `OrderDetails` 逻辑开始轮询过程，但并没有结束。如果用户浏览了数百个不同的订单（从而创建了数百个不同的`OrderDetails` 实例），那么将有数百个轮询进程并发运行，即使最后一个轮询进程毫无意义，因为没有UI显示其结果。
 

您实际上可以自己观察到这种混乱，如下所示：

1. 导航到“我的订单”
2. 单击任何订单上的“跟踪”以获取其详细信息
3. 点击“返回”返回“我的订单”
4. 重复执行步骤2和3多次（例如20次）
5. 现在，打开浏览器的调试工具，然后在“网络”标签中查找。您应该看到每隔几秒钟发出20个或更多HTTP请求，因为有20个或更多并发轮询过程。 

这浪费了客户端内存和CPU时间，网络带宽以及服务器资源。

要解决此问题，一旦将其从显示屏中移除，我们需要 `OrderDetails` 停止轮询。这 是使用IDisposable 接口的原因。 

在 `OrderDetails.razor`, 将以下指令添加到文件顶部的其他指令下方:

```html
@implements IDisposable
```

现在，如果您尝试编译应用程序，则编译器将报错:

```
error CS0535: 'OrderDetails' does not implement interface member 'IDisposable.Dispose()'
```

通过在`@code` 块内添加以下方法来解决此问题 :

```cs
void IDisposable.Dispose()
{
    pollingCancellationToken?.Cancel();
}
```

当任何给定的组件实例被拆除并从UI中删除时，框架会自动调用`Dispose`   

修复此问题后，您可以再次尝试启动大量并发轮询过程，并且在组件消失后它们将不再继续运行。现在，继续轮询的唯一组件是屏幕上剩余的组件。


## 自动导航到订单详细信息

现在，一旦用户下订单，该 `Index` 组件将简单地重置其状态，并且其订单似乎消失无踪。这对于用户来说不是很放心。我们知道订单在数据库中，但是用户不知道。 

如果下订单后应用程序自动导航到该订单的“订单详细信息”显示，那将是很好的选择。这也是很容易做到。  

切换回您的`Index` 组件代码。在顶部添加以下指令 :

```
@inject NavigationManager NavigationManager
```

在`NavigationManager` 让你与URI和导航状态互动。它具有获取当前URL，导航到其他URL以及更多方法的方法。  

要使用它，请更新`PlaceOrder`  代码，调用 `NavigationManager.NavigateTo`:

```csharp
async Task PlaceOrder()
{
    var response = await HttpClient.PostAsJsonAsync("orders", order);
    var newOrderId = await response.Content.ReadFromJsonAsync<int>();
    order = new Order();
    NavigationManager.NavigateTo($"myorders/{newOrderId}");
}
```

现在，服务器接受订单后，浏览器将切换到“订单详细信息”显示并开始轮询更新。  

下一步 - [重构状态管理](04-refactor-state-management.md)
