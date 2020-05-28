# 模板化组件

让我们重构一些原始组件，使其更具可重用性。在此过程中，我们还将创建一个单独的库项目，作为新组件的主页。

我们将使用 Razor 类库模板创建新项目 

## 创建组件库 (Visual Studio)

使用 Visual Studio，右键单击解决方案资源管理器的顶部，然后选择  `Add->New Project`. 

然后，选择 Razor 类库模板。

![image](https://user-images.githubusercontent.com/1430011/65823337-17990c80-e209-11e9-9096-de4cb0d720ba.png)

输入项目名称`BlazingComponents` ，然后单击"创建"。

## 创建组件库 (command line)

使用**dotnet**创建新项目，请从解决方案文件所在的目录中运行以下命令 

```
dotnet new razorclasslib -o BlazingComponents
dotnet sln add BlazingComponents
```

这将创建一个调用的新项目 `BlazingComponents` 并将其添加到解决方案文件中。 

## 了解库项目

通过双击解决方案资源管理器中的项目名称*BlazingComponents*"打开项目文件。我们不会修改这里的任何内容，但了解一些事情会很好。

它看起来像：

```xml
<Project Sdk="Microsoft.NET.Sdk.Razor">

  <PropertyGroup>
    <TargetFramework>netstandard2.0</TargetFramework>
    <RazorLangVersion>3.0</RazorLangVersion>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.AspNetCore.Components" Version="3.1.4" />
    <PackageReference Include="Microsoft.AspNetCore.Components.Web" Version="3.1.4" />
  </ItemGroup>

</Project>
```

这里有一些事情值得了解。 

首先, 这个包是 `netstandard2.0`. Blazor Server 使用 `netcoreapp3.1` 和 Blazor WebAssembly 使用 `netstandard2.1` - 因此 `netstandard2.0`  意味着它将适用于任一方案。

此外 `<RazorLangVersion>3.0</RazorLangVersion>` 设置了 Razor 语言版本， 版本 3 是支持 components 所需要和 `.razor` 文件扩展名. 

最后，该元素`<PackageReference />`添加了对 Blazor 组件模型的包引用。 

## 编写模板化对话框

我们将重构作为`Index` 其中一部分的对话框系统，并将其转换为与应用程序分离的内容 

让我们来思考一下可重用对话框应该如何工作。我们期望一个对话框组件来处理显示和隐藏自身，以及可能样式显示为对话框的可视方式。但是，要真正重用，我们需要能够为对话框内部提供内容。我们将接受*content* 为参数的组件称为模板化组件。

Blazor 碰巧有一个功能，正好适用于这种情况，它类似于布局的工作原理。回想一下，布局具有`Body`参数，并且布局可以放置在`Body`周围的其他内容。在布局中，类型`RenderFragment`的参数是运行时具有特殊处理的委托类型。好消息是，此功能并不仅限于布局。任何组件都可以声明类型 `RenderFragment`的参数。我们还在 `App.razor` 中广泛使用了此功能。用于处理路由和授权的所有组件都是模板化组件。

让我们开始使用此新的对话框组件。创建项目`BlazingComponents`中命名的新组件文件`TemplatedDialog.razor`。将以下标记放在 `TemplatedDialog.razor` ： 

```html
<div class="dialog-container">
    <div class="dialog">

    </div>
</div>
```

这还不做任何事情，因为我们尚未添加任何参数。回顾一下我们想完成的两件事。
 
1. 接受对话框的内容作为参数
2. 如果对话框应该显示，则有条件地呈现  

首先，添加称为`ChildContent` 类型`RenderFragment`的参数。该名称`ChildContent`是一个特殊的参数名称，当组件想要接受单个内容参数时，约定使用该名称。  

```razor
@code {
    [Parameter] 
    public RenderFragment ChildContent { get; set; }
}
```

接下来，更新标记 以呈现标记中间的`ChildContent`  标记。它应该如下所示：

```html
<div class="dialog-container">
    <div class="dialog">
        @ChildContent
    </div>
</div>
```

如果此结构看起来很奇怪，请用布局文件进行交叉检查，该文件遵循类似的模式。即`RenderFragment`是委托类型，而不是通过调用它来呈现它，它是通过将值放在法线表达式中，以便运行时可以调用它。

接下来，要为此对话框提供一些条件行为，让我们添加一个称为`bool` 类型的参数`Show`。执行此操作后，是时候将所有现有内容包装在`@if (Show) { ... }` 中。完整文件应如下所示： 

```html
@if (Show)
{
    <div class="dialog-container">
        <div class="dialog">
            @ChildContent
        </div>
    </div>
}

@code {
    [Parameter] 
    public RenderFragment ChildContent { get; set; }

    [Parameter] 
    public bool Show { get; set; }
}
```

编译解决方案并确保所有内容在此阶段进行编译。接下来，我们将开始使用此新组件。 

## 添加对模板化库的引用

在项目中使用此组件之前，我们需要`BlazingPizza.Client` 添加项目引用。为此，将项目引用从`BlazingComponents` 添加到`BlazingPizza.Client` 。

完成此操作后，还有一个小步骤。打开`BlazingPizza.Client` 的最上面目录`_Imports.razor`，并在末尾添加此行： 

```html
@using BlazingComponents
```

现在已添加项目引用，请再次执行生成以验证所有内容是否仍然正常编译。

## 另一个重构

回顾一下 `TemplatedDialog`，我们的包含一些`div`。嗯，这复制了`ConfigurePizzaDialog` 的一些结构。让我们清理一下。打开`ConfigurePizzaDialog.razor`;它当前看起来像 :

```html
<div class="dialog-container">
    <div class="dialog">
        <div class="dialog-title">
        ...
        </div>
        <form class="dialog-body">
        ...
        </form>

        <div class="dialog-buttons">
        ...
        </div>
    </div>
</div>
```

我们应该删除最外层的两层`div`元素，因为这些元素现在是组件 `TemplatedDialog` 的一部分。删除这些后，它看起来应该更像： 

```html
<div class="dialog-title">
...
</div>
<form class="dialog-body">
...
</form>

<div class="dialog-buttons">
...
</div>
```

## 使用新对话框

我们将在`Index.razor`使用此新的模板化组件。打开`Index.razor`并找到如下所示的代码块 :

```html
@if (OrderState.ShowingConfigureDialog)
{
    <ConfigurePizzaDialog
        Pizza="OrderState.ConfiguringPizza"
        OnConfirm="OrderState.ConfirmConfigurePizzaDialog"
        OnCancel="OrderState.CancelConfigurePizzaDialog" />
}
```

我们将删除它，并将其替换为调用新组件。将上面的块替换为如下代码 :

```html
<TemplatedDialog Show="OrderState.ShowingConfigureDialog">
    <ConfigurePizzaDialog 
        Pizza="OrderState.ConfiguringPizza" 
        OnCancel="OrderState.CancelConfigurePizzaDialog" 
        OnConfirm="OrderState.ConfirmConfigurePizzaDialog" />
</TemplatedDialog>
```

这是使用了我们的新组件`TemplatedDialog`，以基于`OrderState.ShowingConfigureDialog`显示和隐藏自己 。此外，我们将一些内容传递给参数`ChildContent` 。由于我们调用了 参数，因此放置在`<TemplatedDialog> </TemplatedDialog>`  中的任何内容都将由委托`RenderFragment` 捕获并传递给`TemplatedDialog` 。

> 注意：模板化组件可能有多个`RenderFragment`参数。我们在这里显示的是一个方便的约定，当调用方想要提供表示主要内容的单个`RenderFragment`时。 

此时，应该可以运行代码，并查看新对话框是否正常工作。在继续下一步之前，请验证这是否正常工作。

## 更高级的模板化组件

现在，我们已经完成了一个基本的模板化对话框，我们将尝试更复杂的内容。回顾一下 `MyOrders.razor`，该页显示订单列表，但它也包含三状态逻辑（加载、空列表和显示项目）。如果我们能将该逻辑提取到可重用的组件中，那有用吗？让我们试试看。

首先在项目中`BlazingComponents`创建新文件`TemplatedList.razor`。我们希望此列表具有一些功能 :
1. 同步加载任何类型的数据
2. 三种状态- 加载、空列表和显示项 的单独呈现逻辑  

我们可以通过接受类型 `Func<Task<List<?>>>`委托来解决异步加载问题-我们需要找出应该替换哪种类型**?** 由于我们想要支持任何类型的数据，我们需要声明此组件为泛型类型。我们可以使用指令`@typeparam` 创建泛型类型组件，因此，在`TemplatedList.razor`顶部放置该组件。

```html
@typeparam TItem
```

使泛型类型组件的工作方式与 C# 中的其他泛型类型类似，实际上`@typeparam`只是泛型 .NET 类型的一个方便的 Razor 语法。

> 注意：我们尚未支持类型参数约束。这是我们希望在未来补充的内容。

现在，我们已经定义了泛型类型参数，我们可以在参数声明中使用它。让我们添加一个参数来接受可用于加载数据的委托，然后以与其他组件类似的方式加载数据。

```html
@code {
    List<TItem> items;

    [Parameter] 
    public Func<Task<List<TItem>>> Loader { get; set; }

    protected override async Task OnParametersSetAsync()
    {
        items = await Loader();
    }
}
```

由于我们有数据，我们现在可以添加我们需要处理的每个状态的结构。将以下标记添加到 `TemplatedList.razor`:  

```html
@if (items == null)
{

}
else if (items.Count == 0)
{
}
else
{
    <div class="list-group">
        @foreach (var item in items)
        {
            <div class="list-group-item">
                
            </div>
        }
    </div>
}
```
现在，这是我们的三种对话状态，我们希望接受每个状态的内容参数，以便调用方可以插入所需的内容。为此，我们定义了三个参数`RenderFragment`。因为我们有多个参数`RenderFragment`，我们只会为每个参数提供自己的描述性名称，而不是调用它们。显示项目的内容需要采用参数`ChildContent`。我们可以使用 `RenderFragment<T>` 执行此操作。 

下面是要添加的三个参数的示例：

```C#
    [Parameter] public RenderFragment Loading{ get; set; }
    [Parameter] public RenderFragment Empty { get; set; }
    [Parameter] public RenderFragment<TItem> Item { get; set; }
```

现在我们有一些参数`RenderFragment` ，我们可以开始使用它们。更新我们之前创建的标记，以在每个位置插入正确的参数。 

```html
@if (items == null)
{
    @Loading
}
else if (items.Count == 0)
{
    @Empty
}
else
{
    <div class="list-group">
        @foreach (var item in items)
        {
            <div class="list-group-item">
                @Item(item)
            </div>
        }
    </div>
}
```

`Item`接受参数，处理此参数的方法只是调用函数。调用的结果是 `RenderFragment<T>` 另一个 `RenderFragment`可以直接呈现的结果。  

此时应编译新组件，但我们仍要执行一件事。我们希望能够用`<div class="list-group">`另一个类来设置样式，因为这是`MyOrders.razor`正在做的。添加小扩展点以插入其他 css 类可以在很大程度上获得可重用性。 

让我们添加另一个 `string`参数，最后 函数块应如下所示 :

```html
@code {
    List<TItem> items;

    [Parameter] public Func<Task<List<TItem>>> Loader { get; set; }
    [Parameter] public RenderFragment Loading { get; set; }
    [Parameter] public RenderFragment Empty { get; set; }
    [Parameter] public RenderFragment<TItem> Item { get; set; }
    [Parameter] public string ListGroupClass { get; set; }

    protected override async Task OnParametersSetAsync()
    {
        items = await Loader();
    }
}
```

最后更新 `<div class="list-group">` 以`<div class="list-group @ListGroupClass">` 包含 。`TemplatedList.razor` 的完整文件现在应如下所示：  

```html
@typeparam TItem

@if (items == null)
{
    @Loading
}
else if (items.Count == 0)
{
    @Empty
}
else
{
    <div class="list-group @ListGroupClass">
        @foreach (var item in items)
        {
            <div class="list-group-item">
                @Item(item)
            </div>
        }
    </div>
}

@code {
    List<TItem> items;

    [Parameter] public Func<Task<List<TItem>>> Loader { get; set; }
    [Parameter] public RenderFragment Loading { get; set; }
    [Parameter] public RenderFragment Empty { get; set; }
    [Parameter] public RenderFragment<TItem> Item { get; set; }
    [Parameter] public string ListGroupClass { get; set; }

    protected override async Task OnParametersSetAsync()
    {
        items = await Loader();
    }
}
```

## 使用模板列表

要使用新组件`TemplatedList`，我们将编辑`MyOrders.razor`。

首先，我们需要创建一个委托，我们可以传递`TemplatedList` 给将加载订单数据的委托。我们可以通过保留 `MyOrders.OnParametersSetAsync`中的代码行并更改方法签名来执行此操作。`@code`块应如下所示：

```html
@code {
    async Task<List<OrderWithStatus>> LoadOrders()
    {
        var ordersWithStatus = new List<OrderWithStatus>();
        var tokenResult = await TokenProvider.RequestAccessToken();
        if (tokenResult.TryGetToken(out var accessToken))
        {
            var request = new HttpRequestMessage(HttpMethod.Get, "orders");
            request.Headers.Authorization = new AuthenticationHeaderValue("Bearer", accessToken.Value);
            var response = await HttpClient.SendAsync(request);
            ordersWithStatus = await response.Content.ReadFromJsonAsync<List<OrderWithStatus>>();
        }
        else
        {
            NavigationManager.NavigateTo(tokenResult.RedirectUrl);
        }
        return ordersWithStatus;
    }
}
```

这与 `TemplatedList`的 `Loader`参数的预期签名匹配，它是`Func<Task<List<?>>>` 替换`OrderWithStatus`的  **?** 的 ，因此我们处于正确的轨道上。 

现在可以使用组件`TemplatedList`，如下所示：

```html
<div class="main">
    <TemplatedList>
    </TemplatedList>
</div>
```

编译器会抱怨不知道`TemplatedList` 的泛型类型。编译器足够聪明，可以像普通 C# 一样执行类型推理，但我们没有给予它足够的信息来处理它。

添加`Loader` 属性来解决此问题。  

```html
<div class="main">
    <TemplatedList Loader="@LoadOrders">
    </TemplatedList>
</div>
```

> 注意：泛型类型组件还可以手动指定其类型参数，也可以通过将具有匹配名称的属性设置为类型参数 （在本例中称为 `TItem`）在某些情况下，这是必要的，所以这是值得知道的。  

```html
<div class="main">
    <TemplatedList TItem="OrderWithStatus">
    </TemplatedList>
</div>
```

我们现在不需要这样做，因为可以从`Loader` 推断类型。 

-----

接下来，我们需要考虑如何将多个内容 （`RenderFragment`） 参数传递给组件。我们已经学习到`TemplatedDialog`，通过使用单个`[Parameter] RenderFragment ChildContent` 可以通过嵌套组件中的内容来设置。但是，对于最简单的情况来说，这只是一个方便的语法。如果要传递多个内容参数，可以通过嵌套与参数名称匹配的组件中的元素来执行此操作。 

对于我们`TemplatedList`以下示例，该示例将每个参数设置为一些虚拟内容:

```html
<div class="main">
    <TemplatedList Loader="@LoadOrders">
        <Loading>Hi there!</Loading>
        <Empty>
            How are you?
        </Empty>
        <Item>
            Are you enjoying Blazor?
        </Item>
    </TemplatedList>
</div>
```

`Item` 参数是`RenderFragment<T>`  - 它接受参数。默认情况下，此参数称为`context` 。如果我们在 `<Item>  </Item>` 内部键入，那么应该可以看到它`@context`绑定到类型`OrderStatus`的变量。我们可以使用 `Context` 属性重命名参数：

```html
<div class="main">
    <TemplatedList Loader="@LoadOrders">
        <Loading>Hi there!</Loading>
        <Empty>
            How are you?
        </Empty>
        <Item Context="item">
            Are you enjoying Blazor?
        </Item>
    </TemplatedList>
</div>
```

现在，我们来看`MyOrders.razor` 包含的所有现有内容，因此，将所有内容放在一起应如下所示： 

```html
<div class="main">
    <TemplatedList Loader="@LoadOrders" ListGroupClass="orders-list">
        <Loading>Loading...</Loading>
        <Empty>
            <h2>No orders placed</h2>
            <a class="btn btn-success" href="">Order some pizza</a>
        </Empty>
        <Item Context="item">
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
        </Item>
    </TemplatedList>
</div>
```

请注意，我们还在设置`ListGroupClass`参数以添加 `MyOrders.razor` 中存在的其他样式。

我们介绍了多步骤和新功能。运行此项并确保它正常工作，现在我们使用的模板化列表。

为了验证列表确实工作正常，我们可以尝试以下操作: 
1. 从项目`Blazor.Server`中删除 `pizza.db` 测试没有订单的情况
2. 添加`await Task.Delay(3000);`  到 `LoadOrders`（方法也标记为`async` ） 以测试仍在加载的情况

## 总结

那么，我们在节课中学到了什么？

1. 可以编写接受 *content* 作为参数的组件 - 甚至多个内容参数  
2. 模板化组件可用于抽象内容，如显示对话框或数据异步加载
3. 组件可以是泛型类型，这使得它们更具可重用性

下一步 - [Progressive web app](09-progressive-web-app.md)
