# 定制比萨饼

在本节课中，我们将更新比萨饼商店应用程序，使用户能够自定义他们的比萨饼，并将其添加到他们的订单。

## 事件处理

当用户单击特价披萨时，弹出一个比萨饼自定义对话框，允许用户自定义他们的比萨饼并将其添加到他们的订单中。 要在 Blazor 应用中处理 DOM UI 事件，请指定要使用相应的 HTML 属性处理哪个事件，然后指定要调用的 C# 委托。委托可以选择采用事件特定的参数，但不是必需的。 

在 *Pages/Index.razor* 添加下列 `@onclick` 处理程序添加到每个披萨特别列表项:

```html
@foreach (var special in specials)
{
    <li @onclick="@(() => Console.WriteLine(special.Name))" style="background-image: url('@special.ImageUrl')">
        <div class="pizza-info">
            <span class="title">@special.Name</span>
            @special.Description
            <span class="price">@special.GetFormattedBasePrice()</span>
        </div>
    </li>
}
```

运行应用程序，并检查每当点击比萨饼时，比萨饼名称是否写入浏览器控制台。

![@onclick-event](https://user-images.githubusercontent.com/1874516/77239615-f56dbf00-6b99-11ea-8535-ddcc8bc0d8ae.png)

该符号`@`用于 Razor 文件中，以指示 C# 代码的开始，如果需要，用 paren 环绕 C# 代码以明确 C# 代码的开始和结束位置

更新 *Index.razor* 中的  `@code` ，添加一些其他字段，用于跟踪正在自定义的比萨饼以及比萨饼自定义对话框是否可见。

 ```csharp
List<PizzaSpecial> specials;
Pizza configuringPizza;
bool showingConfigureDialog;
```

添加 一个`ShowConfigurePizzaDialog` 方法到 `@code`  处理当一个特价披萨被点击时的逻辑

```csharp
void ShowConfigurePizzaDialog(PizzaSpecial special)
{
    configuringPizza = new Pizza()
    {
        Special = special,
        SpecialId = special.Id,
        Size = Pizza.DefaultSize,
        Toppings = new List<PizzaTopping>(),
    };

    showingConfigureDialog = true;
}
```

更新`@onclick` 处理程序以调用 `ShowConfigurePizzaDialog`  方法替代 `Console.WriteLine`.

```html
<li @onclick="@(() => ShowConfigurePizzaDialog(special))" style="background-image: url('@special.ImageUrl')">
```

## 实现披萨饼自定义对话框

现在，我们需要实现披萨饼自定义对话框，以便我们可以在用户选择披萨饼时显示它。披萨自定义对话框将是一个新的组件，允许您指定披萨饼的大小和你想要的配料，显示价格，并允许您将披萨饼添加到您的订单。 

在 *Shared* 文件夹下添加一个*ConfigurePizzaDialog.razor* 。由于此组件不是单独的页面，因此它不需要指令 `@page` 

> 注意: 在 Visual Studio, 您可以右键单击解决方案资源管理器中的 *Shared*  目录，然后选择"添加->新项目"，然后使用Razor 组件项模板。

 `ConfigurePizzaDialog` 应具有指定正在配置的比萨饼的参数`Pizza`。  组件参数是通过向使用  属性修饰的组件添加可写属性来定义的。  在 `@code` 加入如下`ConfigurePizzaDialog` 的 `Pizza` 参数:

```csharp
@code {
    [Parameter] public Pizza Pizza { get; set; }
}
```

>  注意：组件参数值需要具有setter 并声明`public`，因为它们由框架设置。 但是，它们只能由框架设置为呈现过程的一部分。 不要编写从组件外部覆盖这些参数值的代码，因为这样组件的状态将与其呈现输出不同步.

为 `ConfigurePizzaDialog` 添加以下基本标记:

```html
<div class="dialog-container">
    <div class="dialog">
        <div class="dialog-title">
            <h2>@Pizza.Special.Name</h2>
            @Pizza.Special.Description
        </div>
        <form class="dialog-body"></form>
        <div class="dialog-buttons">
            <button class="btn btn-secondary mr-auto">Cancel</button>
            <span class="mr-center">
                Price: <span class="price">@(Pizza.GetFormattedTotalPrice())</span>
            </span>
            <button class="btn btn-success ml-auto">Order ></button>
        </div>
    </div>
</div>
```

更新 *Pages/Index.razor* 当一个特价披萨饼被选中时 展示 `ConfigurePizzaDialog` 。  `ConfigurePizzaDialog` 样式用于覆盖当前页面，因此将此代码块放在何处并不重要

```html
@if (showingConfigureDialog)
{
    <ConfigurePizzaDialog Pizza="configuringPizza" />
}
```

运行应用程序并选择一个比萨饼特别以查看`ConfigurePizzaDialog` 的效果.

![initial-pizza-dialog](https://user-images.githubusercontent.com/1874516/77239685-e3d8e700-6b9a-11ea-8adf-5ee8a69f08ae.png)


遗憾的是，此时没有实现可以关闭对话框，我们稍后再补充一下。让我们开始处理对话框本身。 

## 数据绑定

用户应该能够指定他们的比萨饼的大小。向 `ConfigurePizzaDialog` 添加标记表示滑块的内容，该滑块允许用户指定比萨饼大小。 用它替换文中的 `<form class="dialog-body"></form>` 元素.

```html
<form class="dialog-body">
    <div>
        <label>Size:</label>
        <input type="range" min="@Pizza.MinimumSize" max="@Pizza.MaximumSize" step="1" />
        <span class="size-label">
            @(Pizza.Size)" (£@(Pizza.GetFormattedTotalPrice()))
        </span>
    </div>
</form>
```

现在，对话框显示一个滑块，可用于更改比萨饼大小。但是，如果您使用它，它现在不会做任何事情。

![Slider](https://user-images.githubusercontent.com/1430011/57576985-eff40400-7421-11e9-9a1b-b22d96c06bcb.png)

我们希望`Pizza.Size` 的值反映滑块的值。当对话框打开时，滑块将从`Pizza.Size` 获取其值。 移动滑块应相应地更新存储在 `Pizza.Size` 中的比萨饼大小。此概念称为双向绑定。  

如果要手动实现双向绑定，可以通过组合值和@onchange来实现，如以下代码（实际上不需要放入应用程序中，因为有一个更简单的解决方案）：

```html
<input 
    type="range" 
    min="@Pizza.MinimumSize" 
    max="@Pizza.MaximumSize" 
    step="1" 
    value="@Pizza.Size"
    @onchange="@((ChangeEventArgs e) => Pizza.Size = int.Parse((string) e.Value))" />
```

在 Blazor 中，可以使用指令 `@bind` 属性指定具有此相同行为的双向绑定。  使用`@bind` 等效标记如下所示：  

```html
<input type="range" min="@Pizza.MinimumSize" max="@Pizza.MaximumSize" step="1" @bind="Pizza.Size"  />
```

但是，如果我们使用`@bind` 时没有进一步的改变，那么这种行为就不完全是我们想要的了。 更新事件仅在滑块发布后触发，试试看它如何做.

![Slider with default bind](https://user-images.githubusercontent.com/1874516/51804870-acec9700-225d-11e9-8e89-7761c9008909.gif)

我们希望在移动滑块时查看更新。Blazor 中的数据绑定允许这样做，允许您使用语法`@bind:<eventname>` 指定哪个事件触发更改。因此，要使用事件绑定，可以执行此操作`oninput`:

```html
<input type="range" min="@Pizza.MinimumSize" max="@Pizza.MaximumSize" step="1" @bind="Pizza.Size" @bind:event="oninput" />
```

当您移动滑块时，披萨大小现在应该会更新。

![Slider bound to oninput](https://user-images.githubusercontent.com/1874516/51804899-28e6df00-225e-11e9-9148-caf2dd269ce0.gif)

## 添加其他配料

用户还应能够在 `ConfigurePizzaDialog` 上选择其他配料。添加用于存储可用配料的列表属性。通过向 API `/toppings` 发出 HTTP GET 请求，初始化可用配料列表 

```csharp
@inject HttpClient HttpClient

<div class="dialog-container">
...
</div>

@code {
    List<Topping> toppings;

    [Parameter] public Pizza Pizza { get; set; }

    protected async override Task OnInitializedAsync()
    {
        toppings = await HttpClient.GetFromJsonAsync<List<Topping>>("toppings");
    }
}
```

在对话框正文中添加以下标记，用于显示下拉列表，其中列出了可用配料，后跟所选配料集。把这个放在现有的`<div>`下面`<form class="dialog-body">`。

```html
<div>
    <label>Extra Toppings:</label>
    @if (toppings == null)
    {
        <select class="custom-select" disabled>
            <option>(loading...)</option>
        </select>
    }
    else if (Pizza.Toppings.Count >= 6)
    {
        <div>(maximum reached)</div>
    }
    else
    {
        <select class="custom-select" @onchange="ToppingSelected">
            <option value="-1" disabled selected>(select)</option>
            @for (var i = 0; i < toppings.Count; i++)
            {
                <option value="@i">@toppings[i].Name - (£@(toppings[i].GetFormattedPrice()))</option>
            }
        </select>
    }
</div>

<div class="toppings">
    @foreach (var topping in Pizza.Toppings)
    {
        <div class="topping">
            @topping.Topping.Name
            <span class="topping-price">@topping.Topping.GetFormattedPrice()</span>
            <button type="button" class="delete-topping" @onclick="@(() => RemoveTopping(topping.Topping))">x</button>
        </div>
    }
</div>
```

还添加以下事件处理程序以进行配料选择和删除:

```csharp
void ToppingSelected(ChangeEventArgs e)
{
    if (int.TryParse((string)e.Value, out var index) && index >= 0)
    {
        AddTopping(toppings[index]);
    }
}

void AddTopping(Topping topping)
{
    if (Pizza.Toppings.Find(pt => pt.Topping == topping) == null)
    {
        Pizza.Toppings.Add(new PizzaTopping() { Topping = topping });
    }
}

void RemoveTopping(Topping topping)
{
    Pizza.Toppings.RemoveAll(pt => pt.Topping == topping);
}
```

现在，您应该能够添加和删除配料。

![Add and remove toppings](https://user-images.githubusercontent.com/1874516/77239789-c0626c00-6b9b-11ea-9030-0bcccdee6da7.png)


## 组件事件

"取消"和"订购"按钮尚未执行任何操作。当用户将披萨添加到其订单或取消时，我们需要某种方式与 `Index` 组件进行通信。我们可以通过定义组件事件来做到这一点。组件事件是父组件可以订阅的回调参数。

向组件`ConfigurePizzaDialog` 添加两个参数：`OnCancel`  和`OnConfirm` 。这两个参数应为 `EventCallback`类型。 

```csharp
[Parameter] public EventCallback OnCancel { get; set; }
[Parameter] public EventCallback OnConfirm { get; set; }
```

将`@onclick` 事件处理程序添加到`ConfigurePizzaDialog`  触发`OnCancel` 和 `OnConfirm` 事件 .

```html
<div class="dialog-buttons">
    <button class="btn btn-secondary mr-auto" @onclick="OnCancel">Cancel</button>
    <span class="mr-center">
        Price: <span class="price">@(Pizza.GetFormattedTotalPrice())</span>
    </span>
    <button class="btn btn-success ml-auto" @onclick="OnConfirm">Order ></button>
</div>
```

在`Index` 组件中为隐藏对话框并将该对话框`ConfigurePizzaDialog`连接到 `OnCancel` 的事件添加事件处理程序。

```html
<ConfigurePizzaDialog Pizza="configuringPizza" OnCancel="CancelConfigurePizzaDialog" />
```

```csharp
void CancelConfigurePizzaDialog()
{
    configuringPizza = null;
    showingConfigureDialog = false;
}
```

现在，当您单击对话框的"取消"按钮时，将执行`Index.CancelConfigurePizzaDialog`，然后 `Index` 组件将重新呈现自身。由于`showingConfigureDialog`是`false`现在对话框将不会显示。  

通常，触发事件（如单击"取消"按钮）时发生的情况是定义事件处理程序委托的组件将重新呈现。可以使用 任何委托类型（如 `Action` ）或者`Func<string, Task> 定义事件。有时，您希望使用不属于组件的事件处理程序委托 - 如果使用普通委托类型定义事件，则不会呈现或更新任何内容。 

`EventCallback` 是编译器已知的一种特殊类型，用于解决其中一些问题。 它告诉编译器将事件调度到包含事件处理程序逻辑的组件。 `EventCallback` 有更多的技巧，但现在只需记住，使用使你的组件明智地将事件调度到正确的位置。 

运行应用并验证单击"取消"按钮时对话框是否现在消失。

触发`OnConfirm`事件时，应将自定义披萨饼添加到用户订单中。向`Index`组件添加`Order`字段以跟踪用户的顺序。  

```csharp
List<PizzaSpecial> specials;
Pizza configuringPizza;
bool showingConfigureDialog;
Order order = new Order();
```

在 `Index` 组件中为将配置的披萨饼添加到订单并将其连接到`OnConfirm`的事件添加事件处理程序  `ConfigurePizzaDialog`。 

```html
<ConfigurePizzaDialog 
    Pizza="configuringPizza" 
    OnCancel="CancelConfigurePizzaDialog"  
    OnConfirm="ConfirmConfigurePizzaDialog" />
```

```csharp
void ConfirmConfigurePizzaDialog()
{
    order.Pizzas.Add(configuringPizza);
    configuringPizza = null;

    showingConfigureDialog = false;
}
```

运行应用并验证对话框现在消失时，单击"订单"按钮。我们目前还无法看到订单中添加了披萨饼，因为没有显示此信息的 UI。接下来我们将讨论这一点。 

## 显示当前订单

接下来，我们需要在当前订单中显示配置的披萨饼，计算总价，并提供下订单的方法。 

创建新组件`ConfiguredPizzaItem`以显示配置的披萨饼。它需要两个参数：配置的披萨饼，以及删除披萨饼时的事件 

```html
<div class="cart-item">
    <a @onclick="OnRemoved" class="delete-item">x</a>
    <div class="title">@(Pizza.Size)" @Pizza.Special.Name</div>
    <ul>
        @foreach (var topping in Pizza.Toppings)
        {
        <li>+ @topping.Topping.Name</li>
        }
    </ul>
    <div class="item-price">
        @Pizza.GetFormattedTotalPrice()
    </div>
</div>

@code {
    [Parameter] public Pizza Pizza { get; set; }
    [Parameter] public EventCallback OnRemoved { get; set; }
}
```

将以下标记添加`Index`的`main` div 下方的，以添加右侧窗格，以按当前顺序显示配置的披萨饼。  

```html
<div class="sidebar">
    @if (order.Pizzas.Any())
    {
        <div class="order-contents">
            <h2>Your order</h2>

            @foreach (var configuredPizza in order.Pizzas)
            {
                <ConfiguredPizzaItem Pizza="configuredPizza" OnRemoved="@(() => RemoveConfiguredPizza(configuredPizza))" />
            }
        </div>
    }
    else
    {
        <div class="empty-cart">Choose a pizza<br>to get started</div>
    }

    <div class="order-total @(order.Pizzas.Any() ? "" : "hidden")">
        Total:
        <span class="total-price">@order.GetFormattedTotalPrice()</span>
        <button class="btn btn-warning" disabled="@(order.Pizzas.Count == 0)" @onclick="PlaceOrder">
            Order >
        </button>
    </div>
</div>
```

此外，向 `Index`组件添加以下事件处理程序，以便从订单中删除已配置的披萨饼并提交订单。 

```csharp
void RemoveConfiguredPizza(Pizza pizza)
{
    order.Pizzas.Remove(pizza);
}

async Task PlaceOrder()
{
    await HttpClient.PostAsJsonAsync("orders", order);
    order = new Order();
}
```

现在，您应该能够从订单中添加和删除已配置的披萨饼并提交订单。 

![Order list pane](https://user-images.githubusercontent.com/1874516/77239878-b55c0b80-6b9c-11ea-905f-0b2558ede63d.png)


即使订单已成功添加到数据库中，UI 中尚未显示任何提示 发生了这种情况。这就是我们在下节课上将讨论的内容。

Next up - [显示订单状态](03-show-order-status.md)
