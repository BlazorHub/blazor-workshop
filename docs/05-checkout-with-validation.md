# 验证并结账

如果您查看其中的`Order`类`BlazingPizza.Shared`，您可能会注意到它具有类型为`Address`的 `DeliveryAddress` 属性。但是，披萨饼订购流程中尚未填充任何数据，因此您的所有订单都具有空白的交货地址 

现在该添加一个“结帐”页面来解决此问题，该页面要求客户输入有效地址 

## 插入结账到流程

首先添加一个新的页面组件，`Checkout.razor`其`@page`指令与URL相匹配`/checkout`。对于初始标记，让我们使用您的`OrderReview` 组件显示订单的详细信息 :

```razor
<div class="main">
    <div class="checkout-cols">
        <div class="checkout-order-details">
            <h4>Review order</h4>
            <OrderReview Order="OrderState.Order" />
        </div>
    </div>

    <button class="checkout-button btn btn-warning" @onclick="PlaceOrder">
        Place order
    </button>
</div>
```

要实现`PlaceOrder`，请将具有该名称的方法从`Index.razor` 复制 到`Checkout.razor` :

```razor
@code {
    async Task PlaceOrder()
    {
        var response = await HttpClient.PostAsJsonAsync("orders", OrderState.Order);
        var newOrderId = await response.Content.ReadFromJsonAsync<int>();
        OrderState.ResetOrder();
        NavigationManager.NavigateTo($"myorders/{newOrderId}");
    }
}
```

与往常一样，您需要`@inject` 为 `OrderState`，`HttpClient`和`NavigationManager`赋值，以便可以像在`Index.razor`中那样进行编译 

接下来，让我们将客户带到这里尝试提交订单。返回中`Index.razor`，确保已删除该 `PlaceOrder`方法，然后将订单提交按钮更改为指向`/checkout` 的常规HTML链接，即： 

```razor
<a href="checkout" class="@(OrderState.Order.Pizzas.Count == 0 ? "btn btn-warning disabled" : "btn btn-warning")">
    Order >
</a>
```

> 请注意`disabled` ，由于HTML链接不支持该属性，因此我们删除了该属性，而是添加了适当的样式。 

现在，当您运行该应用程序时，您应该能够通过单击“ Order”按钮进入结帐页面，然后您可以在其中单击“下订单”进行确认。
 
![Confirm order](https://user-images.githubusercontent.com/1874516/77242251-d2530780-6bb9-11ea-8535-1c41decf3fcc.png)

## 收货地址

现在，我们已经可以放置一些用于输入送货地址的用户界面了。与往常一样，让我们​​将其分解为可重用的组件。您永远都不知道何时要在其他地方询问地址。

在名为的`BlazingPizza.Client` 项目`Shared`文件夹中创建一个新组件`AddressEditor.razor`。这将是编辑`Address`实例的一种通用方法，因此让它接收此类型的参数： 

```razor
@code {
    [Parameter] public Address Address { get; set; }
}
```

这里的标记将有点乏味，因此您可能想要复制并粘贴它。我们需要为`Address`上每个属性输入元素 :

```razor
<div class="form-field">
    <label>Name:</label>
    <div>
        <input @bind="Address.Name" />
    </div>
</div>

<div class="form-field">
    <label>Line 1:</label>
    <div>
        <input @bind="Address.Line1" />
    </div>
</div>

<div class="form-field">
    <label>Line 2:</label>
    <div>
        <input @bind="Address.Line2" />
    </div>
</div>

<div class="form-field">
    <label>City:</label>
    <div>
        <input @bind="Address.City" />
    </div>
</div>

<div class="form-field">
    <label>Region:</label>
    <div>
        <input @bind="Address.Region" />
    </div>
</div>

<div class="form-field">
    <label>Postal code:</label>
    <div>
        <input @bind="Address.PostalCode" />
    </div>
</div>

@code {
    [Parameter] public Address Address { get; set; }
}
```

最后，您实际上可以`AddressEditor` 在 `Checkout.razor` c组件内部使用:

```razor
<div class="checkout-cols">
    <div class="checkout-order-details">
        ... leave this div unchanged ...
    </div>

    <div class="checkout-delivery-address">
        <h4>Deliver to...</h4>
        <AddressEditor Address="OrderState.Order.DeliveryAddress" />
    </div>
</div>
```

现在，您的结帐屏幕会要求您提供送货地址：

![Address editor](https://user-images.githubusercontent.com/1874516/77242320-79d03a00-6bba-11ea-9e40-4bf747d4dcdc.png)

如果您现在提交订单，那么您输入的任何地址数据实际上都会与订单一起保存在数据库中，因为它是`Order`序列化并发送到服务器的对象 。  

如果您真的很想验证数据是否已保存，请考虑下载诸如[DB Browser for SQLite](https://sqlitebrowser.org/)  之类的工具来检查 `pizza.db`文件的内容。但是您不必严格执行此操作。  

可替换地，在`BlazingPizza.Server`的`OrderController.PlaceOrder`方法设置一个断点内，以及使用调试器来检查传入Order对象。在这里您应该能够看到后端服务器收到您键入的地址数据。 

## 添加服务器端验证

到目前为止，客户仍然可以将“送货地址”字段留空，并订购特价披萨。当涉及验证时，在服务器和客户端上均实施规则是很正常的：

 * 客户端验证是对您的用户的友好。他们可以在编辑表单时提供即时反馈。但是，任何具有浏览器开发工具基础知识的人都可以轻松绕过它。  
 * 服务器端是真正执行验证的地方

因此，通常最好从实施服务器端验证开始，因此无论客户端发生什么情况，您都知道您的应用程序很健壮。如果您  在 `BlazingPizza.Server` p项目中查看`OrdersController.cs`，您将看到此API端点已用[ApiController]属性修饰  

```csharp
[Route("orders")]
[ApiController]
public class OrdersController : Controller
{
    // ...
}
```

`[ApiController]` 添加了各种服务器端约定，包括强制执行`DataAnnotations`验证规则。因此，我们要做的就是将一些`DataAnnotations` 验证规则放到模型类上。   

打开 项目 `BlazingPizza.Shared`的 `Address.cs` , 然后将一个 `[Required]` 添加到每一个属性上除了 `Id` 因为它是主键，它是自动生成的) 和 `Line2`, 因为并非所有地址都需要第二行。 如果需要，还可以放置一些属性`[MaxLength]`，或任何其他  `DataAnnotations` 规则:


```csharp
using System.ComponentModel.DataAnnotations;

namespace BlazingPizza
{
    public class Address
    {
        public int Id { get; set; }

        [Required, MaxLength(100)]
        public string Name { get; set; }

        [Required, MaxLength(100)]
        public string Line1 { get; set; }

        [MaxLength(100)]
        public string Line2 { get; set; }

        [Required, MaxLength(50)]
        public string City { get; set; }

        [Required, MaxLength(20)]
        public string Region { get; set; }

        [Required, MaxLength(20)]
        public string PostalCode { get; set; }
    }
}
```
现在，在重新编译并运行应用程序之后，您应该能够观察到服务器上强制执行的验证规则。如果您尝试使用空白的送货地址提交订单，则服务器将拒绝该请求，并且您将在浏览器的“ 网络”标签中看到HTTP 400（“错误请求”）错误：

![Server validation](https://user-images.githubusercontent.com/1874516/77242384-067af800-6bbb-11ea-8dd0-74f457d15afd.png)

... 如果您完全填写地址字段，服务器将允许您下订单。检查这两种情况是否均符合预期.

## 添加客户端验证

Blazor拥有用于数据输入表单和验证的全面系统。现在，我们将使用它`DataAnnotations` 在服务器上已经实施的客户端上应用相同的规则。

Blazor的表单和验证系统的工作方式基于`EditContext`。一个`EditContext`跟踪编辑过程的状态，因此它知道哪些字段已被修改，输入了哪些数据以及这些字段是否有效。各种内置的UI组件都挂接到这`EditContext`两者中，以读取其状态（例如，显示验证消息）并写入其状态（例如，使用用户输入的数据填充该状态）。

### 使用 EditForm

用于数据输入的最重要的内置UI组件之一是`EditForm`。这呈现为HTML`<form>` 标记，但还设置了一个 `EditContext` 来跟踪表单内部发生的情况。要使用此功能，请转到您的`Checkout.razor`组件，然后将`main` div把 `EditForm` 的全部内容包装起来 

 
```razor
<div class="main">
    <EditForm Model="OrderState.Order.DeliveryAddress">
        <div class="checkout-cols">
            ... leave unchanged ...
        </div>

        <button class="checkout-button btn btn-warning" @onclick="PlaceOrder">
            Place order
        </button>
    </EditForm>
</div>
```

您可以一次拥有多个`EditForm` 组件，但是它们不能重叠（因为HTML的`<form>` 元素不能重叠）。通过指定一个`Model`，我们告诉内部 `EditContext` 在提交表单时应验证哪个对象（在本例中为送货地址）

让我们以一种非常基本（且不太吸引人）的方式显示验证消息开始。在`EditForm`底部的内，添加以下两个组件:

```razor
<DataAnnotationsValidator />
<ValidationSummary />
```

 `DataAnnotationsValidator` 挂接到的事件 `EditContext` 和执行 `DataAnnotations` 规则. 如果您想使用`DataAnnotations`以外的其他验证系统 , 则可以替换 `DataAnnotationsValidator` 为其他组件.

 `ValidationSummary` 简单地呈现一个HTML `<ul>` 其中包含来自 `EditContext`的任何验证消息.

### 处理提交

如果您现在运行应用程序，则仍可以提交空白表格（服务器仍将以HTTP 400错误响应）。那是因为您`<button>` 实际上不是一个submit按钮。`<button>` 通过完全添加`type="submit"`和删除其 `@onclick`属性来修改。 

接下来， `PlaceOrder` 您需要直接从按钮触发，而不是直接从按钮触发EditForm。将以下 `OnValidSubmit`属性添加到 `EditForm` :

```razor
<EditForm Model="OrderState.Order.DeliveryAddress" OnValidSubmit="PlaceOrder">
```
您可能会猜到，它们`<button>`不再`PlaceOrder`直接触发。相反，该按钮只是要求提交表单。然后形式来决定是否它是有效的，如果是，那么它会调用`PlaceOrder`。 

尝试一下：您将不再能够提交无效的表单，并且您会在放置`ValidationSummary` 的位置看到验证消息（尽管没有吸引力） 

![Validation summary](https://user-images.githubusercontent.com/1874516/77242430-9d47b480-6bbb-11ea-96ef-8865468375fb.png)

### 使用 ValidationMessage

显然，将所有验证消息显示在离文本框很远的地方真是令人作呕。让我们将它们移到更好的地方  

首先完全删除 `<ValidationSummary>` 组件。然后，切换到`AddressEditor.razor`，并 `<ValidationMessage>` 在每个表单字段旁边添加单独的组件。例如， 

```razor
<div class="form-field">
    <label>Name:</label>
    <div>
        <input @bind="Address.Name" />
        <ValidationMessage For="@(() => Address.Name)" />
    </div>
</div>
```

对所有表单字段都执行等效操作。

你知道，该语法 `@(() => Address.Name)` 是一个 *lambda 表达式*, 我们使用此语法作为描述从哪个属性读取元数据的方式，而无需实际评估属性的值.

现在情况看起来好多了：

![Validation messages](https://user-images.githubusercontent.com/1874516/77242484-03ccd280-6bbc-11ea-8dd1-5d723b043ee2.png)

如果需要，可以通过指定自定义消息来提高消息的可读性。例如，您可以转到 `Address.cs` 执行以下操作，而不是显示必填城市字段 :

```csharp
[Required(ErrorMessage = "How do you expect to receive the pizza if we don't even know what city you're in?"), MaxLength(50)]
public string City { get; set; }
```

### 使用内置输入组件更好地验证UX

用户体验仍然不是很好，因为一旦显示了验证消息，即使您已经编辑了字段值，验证消息仍然保留在屏幕上，直到您再次单击“下订单”为止。尝试一下，看看它的来感受一下！

为了对此进行改进，可以使用Blazor的内置输入组件替换低级HTML输入元素。他们知道如何更深入地了解`EditContext` :

* 编辑它们时，它们会`EditContext` 立即通知即时消息，以便刷新验证状态  
* 他们还从接收到有关有效性的通知`EditContext`，因此当用户进行编辑时，他们可以将自己突出显示为有效还是无效。  

再次回到 `AddressEditor.razor` 将每个`<input>`元素替换为相应的`<InputText>` ，例如：

```html
<div class="form-field">
    <label>Name:</label>
    <div>
        <InputText @bind-Value="Address.Name" />
        <ValidationMessage For="@(() => Address.Name)" />
    </div>
</div>
```

对所有属性执行此操作。行为现在好多了！更改焦点时，除了使验证消息针对每个表单字段单独更新之外，您还将在每个表单字段周围得到整洁的“有效”或“无效”突出显示：

![Input components](https://user-images.githubusercontent.com/1874516/77242542-ba30b780-6bbc-11ea-8018-be022d6cac0b.png)

绿色/红色样式是通过应用CSS类实现的，因此您可以更改这些效果的外观，也可以根据需要将其完全删除。

`InputText` 不是唯一的内置输入组件，尽管在这种情况下它是我们唯一需要的组件。其他的还包括 `InputCheckbox`, `InputDate`, `InputSelect`, 还有更多

## 额外挑战  

如果您还有时间，可以防止意外再次提交表格吗？ 

当前，如果表单提交后需要一段时间才能到达服务器，则用户可以多次单击“提交”并发送其订单的多个副本。尝试声明一个 `bool isSubmitting` 属性，该属性在时会`true`  禁用 “下订单”按钮 。请记住将其设置回 `false` 提交完成时（成功或不成功），否则用户可能会卡住。

为了检查您的解决方案是否有效，您可能希望通过在 `OrdersController.cs`的 `PlaceOrder()`内部顶部添加以下代码来减慢服务器速度

```cs
await Task.Delay(5000); // Wait 5 seconds
```

## 下一步

接下来，我们将添加 [身份验证和授权](06-authentication-and-authorization.md)
