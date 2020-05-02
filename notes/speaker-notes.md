# 演讲者备注

这是介绍每节中哪些主题的粗略指南.

## 00 起步

- 介绍演讲者
- H如果小组人数不多，请与会者自我介绍
- Blazor 是什么
- Blazor /组件的当前状态和未来路线图
- 机器设置
- 创建一个hello world项目

## 01 组件和布局

- 介绍@page-解释可路由和不可路由之间的区别
- 在App.razor中显示路由器组件
- 介绍@code-这就像的旧.cshtml @functions功能 。使用户熟悉定义属性，字段，方法，甚至嵌套类
- 组件是有状态的，因此有必要在组件中保持状态
- 介绍 parameters - parameters 应该是 none-public
- 在razor中使用标记中的组件进行介绍-展示如何传递参数  
- 介绍@inject和DI-可以显示@code中属性的简写方式 
- 在Blazor中引入http + JSON (`GetFromJsonAsync`)
- 讨论异步以及与渲染的交互 
- 介绍 `OnInitializedAsync` 和启动异步工作的常见模式
- 介绍@layout-提到这 `_Imports.razor` 是连接它的最常见方法  

* 演示：以上所有内容都可以通过模板进行介绍 * 

## 02 定制披萨

- 介绍事件处理程序和委托，不同的选项，例如方method groups, lambdas, event args types, async
- 更新组件的私有状态后会发生什么？走一遍事件处理程序->更新->重新渲染 
- 定义自己的组件并接受参数  
- 提及将委托传递给另一个组件 
- 展示使用委托时重新渲染的不同之处，并展示 `EventCallback` 来修复
- 提及将常见或重复的功能放在模型类上  
- 介绍输入元素以及如何使用手动双向绑定 (组合 `value` 和 `@onchange`)
- 展示 `@bind` 是上面的简写
- 展示 `@bind-value` + `@bind-value:event` 作为更具体的版本
- 提及框架尝试定义`@bind` 对常见输入类型执行默认操作，但是可以指定要绑定的内容 

* 演示：TodoList，具有多个级别的组件 *
 - 使用TodoItem.cs 和TodoList.razor 作为独立的UI创建基本的待办事项列表   
   - 看到您的“添加项目”事件处理程序可以是内联lambda，但是创建方法会更好  
   - 了解如何根据需要使处理程序异步 (例如使用 Task.Delay) 并正确重新渲染
 - 在用户 在文本框中输入新内容 ，也显示当前值  
   - 仅当您跳出时才能看到更新 
   - 使用 `@bind-value:event="oninput"`
 -分解出  `TodoListEditor` 组件 具有只读 `Text` 和 `IsDone` 参数
   - 最初，它是单向绑定。我们如何将更改传播回父级？
   - 添加 `IsDoneChanged` 参数并调用回调 并调用`StateHasChanged` 手动更新模型 
   - 替换为 `@bind-IsDone` (将参数类型更改为 `EventCallback<bool>`).

## 03 显示订单状态

- 详细讲解 `NavLink`  , 为什么你要用 `NavLink` 替代 `<a>`
- 基本href以及URL导航在blazor中的工作方式 (如何生成 URL)
- @page 和路由 (再一次)
- 路由参数约束
- 关于 async, inject, http, json 的相关提醒
-  `OnInitializedAsync` 和  `OnParametersSetAsync` 的差别
- 介绍 `StateHasChanged` 有关后台处理的上下文
- 介绍 `@implements` -  实现接口
- 介绍 `Dispose` 作为 `OnInitialized` 的对口
- 介绍 `NavigationManager` 和程序化导航

* 演示：带有计时器的计数器 *
 - 在 `NavMenu.razor` 里, 把所有 `<NavLink>` 都用 `<a>` 替换 看他如何工作, 除了不突出显示
   - 切换回到 `<NavLink>` 并看到仍然呈现 `<a>` 标签 "active" class的除外
   - 了解如何修改 active class
   - 说明 `NavLinkMatch`
   - 解释为什么网址不带前缀  `/`  因为 `<base href>`
 - 修改 `Counter.razor` 以获取初始  `startCount` 参数
   - 尝试使用非int参数值访问它. 添加 `:int` 路由约束.
   - 自定义 "not found"  消息
 - 演示程序化导航: 在 `Counter`, 如果计数超过 5, 自动导航到 `Index`
 - 回顾所有生命周期方法，注意其中有一个隐藏的方法  ("dispose")
 - 在 `Counter.razor`,   `OnInitialized` 启动一个计时器，该计时器增加计数并记录到控制台  
   - 看到如果您反复导航，则有多个计时器
   - 通过实施`IDisposable` 进行修复  

## 04 重构状态管理

- 讨论何时创建和处置组件-这是本节中的错误的来源
- 介绍DI作用域，为什么要对每个用户数据使用作用域生存期以及其工作方式
- 将事件处理程序移至非组件类时会发生什么？
- 显示为事件处理程序生成的代码，运行时如何知道在何处调度事件？ (`EventCallback`)

*demos before*
 - 创建Blazor服务器应用程序
 - 请注意，当您前后导航时，“计数器”状态会丢失。我们该如何解决?
   - 我们可以使状态为静态 (看到他能工作).
     - 这是一个非常有限的解决方案，因为无法控制粒度，并且在Blazor Server中完全是灾难性的.
   - 将状态分解成一个 `CounterState` 类，并使其成为单例DI服务 
     - 对于Blazor Server，您仍然遇到与静态相同的问题
     - 现在使它成为Score ，并看到可以解决的问题 

*demos after*
 - 作为使用DI的替代方法，您可以将状态作为CascadingValue传递
 - 要了解DI的限制之一，因为它是， 注入`@inject CounterState` 到 `MainLayout.razor`
   并添加 `<button @onclick="() => { counterState.Count++; }">Increment</button>`
   - 请注意，更新是如何不会自动流入的 `Counter.razor`, 因为没有任何内容告诉框架您对一个组件中的DI服务执行的操作可能会影响另一个组件
 - 删除CounterState的DI服务，然后看到它现在在运行时失败
 - 在 `MainLayout.razor` 里, 添加一个 `@code` 块，声明一个具有值的字段 `new CounterState()`
   - 还环绕 `@Body` 有 `<CascadingValue Value=@counterState>` - 看他能正常工作
 - 除了提供subtree-scoped 的值之外，CascadingValue还负责在提供的值可能已更改时触发任何订户的重新呈现。  
   - 参见 在`MainLayout` 里的  "increment" 按钮 ，现在能够更新 `Counter` 
 - 使用CascadingValue获取共享状态的优点:
   - 他是 subtree-scoped, 而不是 one-per-type-per-user.
   - 您控制实例化，而不是DI系统
   - 值更改触发订阅者的自动重新呈现
 - 缺点:
   - 不像DI系统那样自动进行构造函数注入
   - 没有单一的配置中心

## 05 结算验证

- 介绍 `EditForm` 和  input 组件

*demos-before: 带有验证的待办事项列表 *

 - 有一个TodoList页面，但是对于每个项目，还跟踪一个“重要性”数字，并按重要性对列表进行自动排序
   - 类上具有DataAnnotations属性 `TodoItem`  
   - 显示您可以添加空字符串项目
   - 如果 <input type=number> 像`-------` 或者 `15+3`，则显示不停止提交，但是在添加项目时将其重置为0，这对于UX没有意义   
 - 说明：Blazor具有通用的内置验证系统，该系统旨在实现可扩展性，甚至允许您完全替换它
 - 更改 `<form>` 到 `<EditForm>` 并解释其职责
   - 设置 `Model="newItem"` ，这里 `newItem` 是一个 `TodoItem` 字段
   - 看到它提供 `OnSubmit`, `OnValidSubmit`, `OnInvalidSubmit`
   -   `OnValidSubmit` 连接到了你的提交方法
 - 看到，起初，它仍然允许提交任意垃圾
 - 需要在表单上添加 `<DataAnnotationsValidator />` 
   - 现在看到您无法提交垃圾数据，但仍然没有显示原因
 - 添加 `<ValidationSummary />` 并可以看到显示原因
 - 用Blazor替换表单字段 `<input>` => `<InputText>` 等等
   - 看到它现在更新每次更改的验证状态
 - 为每个字段使用  `<ValidationMessage>`   替换 `<ValidationSummary>`
 - 如果需要，自定义验证消息

*demos-after: 三态复选框或滑块组件 *

 - 您将如何创建与验证集成的全新输入组件?
   - 在项目仓库中查看 InputBase.cs  
   - 说明您可以从中继承。您的责任是提供渲染的标记，以及如何格式化/解析以字符串形式给出的值。例如，对于某些基础HTML5输入，浏览器处理区域性变量值，而对于其他区域性不变值，因此您必须使用浏览器控制此数据交换
 - 示例: InputSlider
   - CurrentValueAsString 表示提供给浏览器或从浏览器接收的值. 这通常是 `@bind` 和 HTML一起使用的内容
   - CssClass 由框架计算，并结合了用户提供的类和标准验证状态类 
   - `@attributes` 如果在特定元素上添加用户提供的 aribribry属性，则应该在输出上使用AdditionalAttributes 。解释“最后获胜”的规则。

## 06 添加验证

- 所有安全实施措施都在服务器上。那么在客户端使用auth做任何事情有什么意义呢？
  - 它提供了一个不错的用户体验。告诉用户是否已登录，以及是否登录，以及他们可以访问哪些功能。
  - Blazor具有一组API，用于讨论用户身份并基于该API影响呈现的UI
- 研讨会代码将使用基于cookie的身份验证系统，由此服务器使用cookie跟踪登录状态。但是，还有其他方法可以完成:
  - 让服务器在登录客户端时发出JWT。客户端将其存储在本地存储中，并作为API调用上的HTTP标头传递回服务器。这有点像带有密码流的OAuth，但有一些警告.
  - 使用OpenID Connect（OIDC），该协议用于通过外部身份提供商登录并获取将您标识为其他服务的auth令牌的协议。这是非常灵活的，几乎是SPA的行业标准，并且解决了基于cookie的身份验证固有的复杂问题.

* 演示 *
 - 从 `BlazorWasmOidc` 应用程序开始 ，但 `<LoginDisplay>` 已删除，并且删除了 `OidcClientAuthenticationStateProvider` DI service ，而是使用`MyFakeAuthenticationStateProvider` 硬编码返回登出状态
 - 看看 `MyFakeAuthenticationStateProvider` 并解释它。此外，不久将替换为更好的版本.
 - 想象一下，如果您注销了，服务器将拒绝 获取天气预报请求。想要在用户界面中反映出来。 
   - 添加 `[Authorize]` 并看到它的工作
   - 如果您已注销，也可以删除菜单项 - 在 `NavMenu.razor` 里使用 <AuthorizeView> 
 - 现在，让我们在页面上中显示登录状态。在 `MainLayout.razor` 里 `<AuthorizeView>`，以将其写入`top-row` 元素
 - 因此，这就是注销时的行为。现在让我们模拟一下登录。
   - 修改 MyFakeAuthenticationStateProvider 以硬编码特定的用户名和角色。`new ClaimsPrincipal(new ClaimsIdentity(new[] { new Claim(ClaimTypes.Name, "SomeUser"), new Claim(ClaimTypes.Role, "Admin"), }, "myfakeauth"));`
   - 现在看到它反映在UI中
 - 将内容锁定为管理员角色-看到您仍然可以访问它
 - 删除用户的管理员角色-看到您无法再访问它
 - 现在，让我们与外部OIDC身份验证流程集成
   - 替换DI服务以 `AuthenticationStateProvider`  使用 `OidcClientAuthenticationStateProvider`
   - 在 `MainLayout` 里, 将auth显示内容替换为 `<LoginDisplay>` component
   - 现在通过OIDC登录
 - 说明：这只是与OIDC流程的快速且非常简单的集成。
 - 但这真的安全吗？如果服务器告诉客户端它以具有特定声明的特定用户身份登录，但是客户端行为不当或被用户绕过，并使其表现为好像以其他用户身份登录或具有不同声明，该怎么办？
   - 没问题 恶意用户可能会欺骗其UI来显示其不应访问的菜单选项，但是最终，当他们尝试对外部服务执行操作时，该服务会检查其身份验证令牌或cookie，因此，不能扮演不是他们的人。

*demos-after: 不同类型的身份验证演示 *
 - 展示如何使用密码流进行基于JWT的身份验证（在您的应用程序中具有用于要求用户名/密码，调用返回令牌的服务器，将其存储在localStorage中的UI等等）
   - 无需`[Authorize]` 客户端或服务器 即可获取MissionControl演示  
     - 看到我们无需登录即可获取数据
   - 在服务器上添加`[Authorize]` 并看到它现在失败
   - 在客户端上添加`[Authorize]`并查看提示登录的消息
   - 看看如何 `LoginDisplay` 使用 `<AuthorizeView>`
   - 了解如何将 `LoginDialog` 凭证发布到返回JWT令牌的服务器
   - 了解如何 `TokenAuthenticationStateProvider` 解析JWT并将其存储在localStorage中
   - 查看注销如何立即更新UI
 -展示OIDC流程

## 07 JavaScript 互操作

- 介绍JS Interop并演示一个示例 `IJSRuntime`   (例如显示 `alert`)
- 何时进行 JS Interop (相对于组件生命周期), `OnAfterRender` 以及响应事件
- 讨论 `IJSRuntime`  如何异步，为什么如此重要，如果需要低级同步互操作该怎么办
- 介绍组件库和项目捆绑的静态内容
- 展示组件库内容文件在运行时如何可编辑  
- 提醒有关组件名称空间
- 注意：本节可以包含一些更高级的JS互操作案例的示例和示例，例如从JS调用.NET或using DotNetObjectRef<>，在研讨会中JS互操作的实际用法非常简单

*demos-before: alert()*
*demos-after: DotNetObjectRef*

## 08 模板化组件

- 调出组件库并查看项目内容，最后使用的部分和现有的库，此部分将创建一个新的库 
- 介绍 `RenderFragment`, 谈谈如何其用于传递标记和呈现逻辑到其它部件,  典型例子就是 `AuthorizeView` 和 `MainLayout`
- 展示一个呈现 `ChildContent` 属性的简单示例
- 讨论有多个 `RenderFragment` 参数时会发生什么
- 展示 `RenderFragment<>` 需要参数的的示例 
- 向泛型引入`@typeparam` 并与用C＃编写的泛型类进行比较  
- 介绍类型推断并显示使用推断与指定类型的示例  

*demo: material design 组件 *

## 09 Progressive Web Apps
 - 演示发布应用程序的所有方式  (wasm, server, electron, pwa, webwindow) 并讨论每种方法的优缺点和功能
 - 可能还会演示部署到Azure
  
## 附录 A: EventCallback - 第 04 补充

首先，我们需要回顾事件调度与渲染的交互方式。当组件的参数更改或收到事件（如 `@onclick`）时，组件将自动重新渲染（更新DOM ）。这通常适用于最常见的情况。这也是有道理的，因为每次事件发生时都无法重新渲染整个UI，Blazor必须决定应更新UI的哪一部分。

事件处理程序附加到.NET， `Delegate` 并且接收事件通知的组件由定义 [`Delegate.Target`](https://docs.microsoft.com/en-us/dotnet/api/system.delegate.target?view=netframework-4.8#System_Delegate_Target).。大致定义为，如果委托代表某个对象的实例方法，则 `Target` 将会是正在调用其方法的对象实例。
 
在以下示例中，事件处理程序委托为 `TestComponent.Clicked` 则 `Delegate.Target` 为的实例 `TestComponent`.

```html
@* TestComponent.razor *@
<button @onclick="Clicked">Click me!</Clicked>
<p>Clicked @i times!</p>
@code {
    int i;
    void Clicked()
    {
        i++;
    }
}
```

因为通常事件处理程序的目的是调用一种更新组件私有状态的方法，所以Blazor希望重新渲染定义事件处理程序的组件是有意义的。前面的示例是事件处理程序的最简单用法，`TestComponent` 在这种情况下，我们想要重新渲染是有意义的。
 
 现在让我们考虑当我们想要一个事件重新呈现祖先组件时会发生什么。这类似于`Index` 和`ConfigurePizzaDialog` 发生的情况，它仅适用于事件处理程序为参数的典型情况。本示例将`Action` 代替使用，`EventCallback`因为我们正在逐步解释为什么`EventCallback`需要这样做。
 

```html
@* CoolButton.razor *@
<button @onclick="Clicked">Clicking this will be cool!</button>
@code {
    [Parameter] public Action Clicked { get; set; }
}

@* TestComponent2.razor *@
<CoolButton Clicked="Clicked" />
<p>Clicked @i times!</p>
@code {
    int i;
    void Clicked()
    {
        i++;
    }
}
```
在此示例中，事件处理程序委托为 `TestComponent2.Clicked` 和`Delegate.Target` 是实例TestComponent-尽管`CoolButton` 实际上是事件的定义。这意味着 `TestComponent2` 将在单击按钮时重新呈现。这是有道理的，因为如果 `TestComponent2` 不重新渲染，就无法更新计数。

让我们看第三个示例，以说明如何分崩离析。在某些情况下`Delegate.Target` 根本没有组件，因此什么都不会重新呈现。让我们再来看一个示例，但是带有一个AppState对象：
 

```C#
public class TestState
{
    public int Count { get; private set; }

    public void Clicked()
    {
        Count++;
    }
}
```

```html
@* CoolButton.razor *@
<button @onclick="Clicked">Clicking this will be cool!</button>
@code {
    [Parameter] public Action Clicked { get; set; }
}

@* TestComponent3.razor *@
@inject TestState State
<CoolButton Clicked="@State.Clicked" />
<p>Clicked @State.Count times!</p>
```
在该第三示例中，事件处理程序委托是 `TestState.Clicked` 和这样`Delegate.Target` 是`TestState`  - 不是组件。单击该按钮时，没有组件会收到事件通知，因此不会重新呈现任何内容。

这是`EventCallback` 要解决的问题。通过将参数`CoolButton` 从`Action` 更改为`EventCallback`您可以修复事件分发行为。之所以 `EventCallback`，是因为编译器知道它，当您`EventCallback`从未将其`Target` 设置为组件的委托创建一个时，编译器将传递当前组件以接收事件。

----

让我们回到我们的应用程序。如果你喜欢，你可以复制一个已经改变的参数，这里所描述的问题`ConfigurePizzaDialog`，从`EventCallback`到`Action` 。如果尝试这样做，您会看到取消或确认对话框没有任何作用。这是因为我们的用例与上面的第三个示例完全一样：
 

```html
@* from Index.razor *@
@if (OrderState.ShowingConfigureDialog)
{
    <ConfigurePizzaDialog
        Pizza="OrderState.ConfiguringPizza"
        OnConfirm="OrderState.ConfirmConfigurePizzaDialog" 
        OnCancel="OrderState.CancelConfigurePizzaDialog" />
}
```

对于`OnConfirm` 和`OnCancel` 参数，由于`Delegate.Target`将`OrderState` 传递对定义的方法的引用，因此将是`OrderState` e。如果您使用`EventCallback` 的是编译器的特殊逻辑，则它将指定其他信息以将事件调度到`Index` 。