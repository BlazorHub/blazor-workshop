# 认证

现在我们的应用程序工作良好。用户可以下订单并跟踪其状态。但有一个小问题：目前我们根本不区分用户。"我的订单"页面列出了所有用户下的所有订单，任何人都可以查看其他人的订单状态。您的客户和隐私法规方面都有问题。

解决方案是身份验证。我们需要一种用户登录的方式，这样我们才能知道谁是谁。然后，我们可以实现授权，即强制实施关于允许谁做什么的规则。

## 在服务器上提供验证

第一个也是最重要的原则是，必须在后端服务器上强制执行所有真正的安全规则。在客户端 （UI） 仅显示或隐藏选项，以表示对行为良好的用户的礼貌，但恶意用户始终可以更改客户端代码的行为。

因此，我们将首先在后端服务器中强制实施一些访问规则，甚至在客户端代码知道它们之前。

在项目 `BlazorPizza.Server` 中，你会发现 `OrdersController.cs`。这是处理  `/orders` 和 `/orders/{orderId}`的传入 HTTP 请求的控制器类。要要求指向这些终结点的所有请求都来自经过身份验证的用户（即已登录的用户），请将该属性`[Authorize]`添加到类`OrdersController` ：

```csharp
[Route("orders")]
[ApiController]
[Authorize]
public class OrdersController : Controller
{
}
```

`AuthorizeAttribute` 类位于命名空间`Microsoft.AspNetCore.Authorization` 中。 

如果您现在尝试运行应用程序，您会发现您无法下订单了，也无法查询已下订单的详细信息。对这些终结点的请求将返回 HTTP 302 重定向到不存在的登录 URL。这很好，因为它表明规则正在服务器上强制执行！


![Secure orders](https://user-images.githubusercontent.com/1874516/77242788-a9ce0c00-6bbf-11ea-98e6-c92e8f7c5cfe.png)

## 跟踪身份验证状态

客户端代码需要一种方法来跟踪用户是否登录，如果已经登录，则哪个用户登录，这样它可能会影响 UI 的行为。Blazor 有一个内置的 DI 服务 `AuthenticationStateProvider`来执行此操作，Blazor 基于[OpenID Connect](https://openid.net/connect/) 提供服务和其他相关服务和组件的实现，用于处理确定用户身份的所有详细信息。这些服务和组件在 Microsoft.AspNetCore.Components.WebAssembly.Authentication 包中提供，该包已添加到客户端项目中。

从广义上讲，这些服务实现的身份验证过程如下所示：

* 当用户尝试登录或尝试访问受保护的资源时，用户将被重定向到应用的登录页 (`/authentication/login`).
* 在登录页中，应用准备重定向到配置的标识提供程序的授权终结点。endpoint 负责确定用户是否经过身份验证，并负责在响应中发出一个或多个令牌。该应用程序提供登录回调以接收身份验证响应.
  * 如果用户未经过身份验证，则首先将用户重定向到基础身份验证系统(典型的是 ASP.NET Core Identity).
  * 用户经过身份验证后，授权终结点将生成相应的令牌，并将浏览器重定向回登录回调endpoint (`/authentication/login-callback`).
* 当 Blazor WebAssembly 应用加载登录回调终结点 (`/authentication/login-callback`)时，将处理身份验证响应。
  * 如果身份验证过程成功完成，则对用户进行身份验证，并选择性地将发送回用户请求的原始受保护 
  * 如果身份验证过程由于任何原因失败，则用户将发送到登录失败页面 (`/authentication/login-failed`)，并显示错误。 

有关详细信息请参阅 [Secure ASP.NET Core Blazor WebAssembly](https://docs.microsoft.com/aspnet/core/security/blazor/webassembly/)  

要启用身份验证服务，请向客户端项目中的*Program.cs* 添加调用`AddApiAuthorization` :

```csharp
public static async Task Main(string[] args)
{
    var builder = WebAssemblyHostBuilder.CreateDefault(args);
    builder.RootComponents.Add<App>("app");

    builder.Services.AddTransient(sp => new HttpClient { BaseAddress = new Uri(builder.HostEnvironment.BaseAddress) });
    builder.Services.AddScoped<OrderState>();

    // Add auth services
    builder.Services..AddApiAuthorization();

    await builder.Build().RunAsync();
}
```

默认情况下，将配置添加的服务，以便使用与应用相同的源的标识提供程序。已设置"披萨饼"应用的服务端项目，以使用 [IdentityServer](https://identityserver.io/)作为标识提供程序，并吧ASP.NET Core Identity身份验证系统：

*BlazingPizza.Server/Startup.cs*

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddMvc()
        .AddNewtonsoftJson();

    services.AddDbContext<PizzaStoreContext>(options => 
        options.UseSqlite("Data Source=pizza.db"));

    services.AddDefaultIdentity<PizzaStoreUser>(options => options.SignIn.RequireConfirmedAccount = true)
        .AddEntityFrameworkStores<PizzaStoreContext>();

    services.AddIdentityServer()
        .AddApiAuthorization<PizzaStoreUser, PizzaStoreContext>();

    services.AddAuthentication()
        .AddIdentityServerJwt();
}
```

服务器也已配置为向客户端应用颁发令牌：

*BlazingPizza.Server/appsettings.json*

```json
"IdentityServer": {
  "Clients": {
    "BlazingPizza.Client": {
      "Profile": "IdentityServerSPA"
    }
  }
}
```

要协调身份验证流，请向客户端项目中的*Pages*目录添加组件`Authentication`： 

*BlazingPizza.Client/Pages/Authentication.razor*

```razor
@page "/authentication/{action}"

<RemoteAuthenticatorView Action="@Action" />

@code{
    [Parameter]
    public string Action { get; set; }
}
```

设置该组件`Authentication` 以使用内置组件`RemoteAuthenticatorView`处理各种身份验证操作。`Action`参数绑定到路由值`{action}`，然后传递给组件 `RemoteAuthenticatorView`来处理。`RemoteAuthenticatorView` h处理作为远程身份验证一部分使用的所有操作。有效的操作包括：注册、登录、配置文件和注销。有关详细信息 [Customize the authentication user interface](https://docs.microsoft.com/aspnet/core/security/blazor/webassembly/additional-scenarios#customize-app-routes) 

要通过应用流身份验证状态信息，您需要在 `App.razor`添加一个组件。在`<Router>` 中 `<CascadingAuthenticationState>`，

```html
<CascadingAuthenticationState>
    <Router AppAssembly="typeof(Program).Assembly" Context="routeData">
        ...
    </Router>
</CascadingAuthenticationState>
```

起初，这似乎什么都不做，但实际上，这已使一个级联参数可用于所有后代组件。级联参数是一个参数，它不是在层次结构中只向下传递一个级别，而是通过任意数量的级别传递。

最后，您已准备好在 UI 中显示内容了！

## 显示登录状态

在客户端项目的文件夹 `Shared` 中创建一个新组件`LoginDisplay`，其中包含：

```html
@inject NavigationManager Navigation
@inject SignOutSessionStateManager SignOutManager

<div class="user-info">
    <AuthorizeView>
        <Authorizing>
            <text>...</text>
        </Authorizing>
        <Authorized>
            <img src="img/user.svg" />
            <div>
                <a href="authentication/profile" class="username">@context.User.Identity.Name</a>
                <button class="btn btn-link sign-out" @onclick="BeginSignOut">Sign out</button>
            </div>
        </Authorized>
        <NotAuthorized>
            <a class="sign-in" href="authentication/register">Register</a>
            <a class="sign-in" href="authentication/login">Log in</a>
        </NotAuthorized>
    </AuthorizeView>
</div>

@code{
    async Task BeginSignOut()
    {
        await SignOutManager.SetSignOutState();
        Navigation.NavigateTo("authentication/logout");
    }
}
```

`AuthorizeView` 是一个内置组件，根据用户是否符合指定的授权条件显示不同的内容。我们没有指定任何授权条件，因此默认情况下，如果用户经过身份验证（登录），则认为用户已授权，否则将不授权。

您可以在需要 UI 内容因授权状态而异的任意位置使用`AuthorizeView` ，例如根据用户的角色控制菜单条目的可见性。在这种情况下，我们使用它来告诉用户他们是谁，并有条件地显示"登录"或"注销"链接（如适用）

注册、登录和查看用户配置文件的链接是导航到组件`Authentication` 的正常链接。注销链接是一个按钮，具有其他逻辑，以防止伪造的请求注销用户。使用按钮可确保注销只能由用户操作触发，并且服务`SignOutSessionStateManager`在整个注销流中保持状态，以确保整个流源自用户操作。

让我们把`LoginDisplay`  放在 UI 的某处。打开 `MainLayout` ，并更新`<div class="top-bar">` 如下所示： 

```html
<div class="top-bar">
    (... leave existing content in place ...)

    <LoginDisplay />
</div>
```

## 注册用户并登录

现在试试看。运行应用并注册新用户.

在主页上选择"注册"

![Select register](https://user-images.githubusercontent.com/1874516/78322144-b25d0580-7522-11ea-863d-59083c2bf111.png)

填写新用户的电子邮件地址和密码。

![Register a new user](https://user-images.githubusercontent.com/1874516/78322197-e6d0c180-7522-11ea-8728-2bd9cbd3c8f8.png)

完成用户注册，用户需要确认其电子邮件地址。在开发过程中，只需单击链接即可确认帐户。

![Email confirmation](https://user-images.githubusercontent.com/1874516/78389880-62208a80-7598-11ea-945a-d2ced76133d9.png)

确认用户的电子邮件后，选择"登录"并输入用户的电子邮件地址和密码。

![Select login](https://user-images.githubusercontent.com/1874516/78389922-7bc1d200-7598-11ea-8a10-e8bf8efa512e.png)

![Login](https://user-images.githubusercontent.com/1874516/78390092-cc392f80-7598-11ea-9d8e-562c2be1aad6.png)

用户已登录并重定向回主页。

![Logged in](https://user-images.githubusercontent.com/1874516/78390115-d9561e80-7598-11ea-912b-e9dd71f787f2.png)

## 请求访问令牌

即使您现在已登录，但下订单仍然失败，因为 HTTP 请求下订单需要有效的访问令牌。要请求访问令牌，请使用该服务`IAccessTokenProvider` 。如果请求访问令牌成功，请将其添加到具有方案承载的标准身份验证标头的请求。如果令牌请求失败，请使用该服务 `NavigationManager`将用户重定向到授权服务以请求新令牌。

*BlazingPizza.Client/Pages/Checkout.razor*

```razor
@page "/checkout"
@attribute [Authorize]
@inject OrderState OrderState
@inject HttpClient HttpClient
@inject NavigationManager NavigationManager
@inject IAccessTokenProvider TokenProvider

<div class="main">
    ...
</div>

@code {
    bool isSubmitting;

    async Task PlaceOrder()
    {
        isSubmitting = true;

        var tokenResult = await TokenProvider.RequestAccessToken();
        if (tokenResult.TryGetToken(out var accessToken))
        {
            var request = new HttpRequestMessage(HttpMethod.Post, "orders");
            request.Content = JsonContent.Create(OrderState.Order);
            request.Headers.Authorization = new AuthenticationHeaderValue("Bearer", accessToken.Value);
            var response = await HttpClient.SendAsync(request);
            var newOrderId = await response.Content.ReadFromJsonAsync<int>();
            OrderState.ResetOrder();
            NavigationManager.NavigateTo($"myorders/{newOrderId}");
        }
        else
        {
            NavigationManager.NavigateTo(tokenResult.RedirectUrl);
        }
    }
}
```

更新`MyOrders` 和`OrderDetails` 组件，以便也发出经过身份验证的 HTTP 请求。

*BlazingPizza.Client/Pages/MyOrders.razor*

```razor
@page "/myorders"
@inject HttpClient HttpClient
@inject NavigationManager NavigationManager
@inject IAccessTokenProvider TokenProvider

<div class="main">
    ...
</div>

@code {
    List<OrderWithStatus> ordersWithStatus;

    protected override async Task OnParametersSetAsync()
    {
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
    }
}

```

*BlazingPizza.Client/Pages/OrderDetails.razor*

```razor
@page "/myorders/{orderId:int}"
@using System.Threading
@inject HttpClient HttpClient
@inject NavigationManager NavigationManager
@inject IAccessTokenProvider TokenProvider
@implements IDisposable

<div class="main">
    ....
</div>

@code {
    ...

    private async void PollForUpdates()
    {
        var tokenResult = await TokenProvider.RequestAccessToken();
        if (tokenResult.TryGetToken(out var accessToken))
        {
            pollingCancellationToken = new CancellationTokenSource();
            while (!pollingCancellationToken.IsCancellationRequested)
            {
                try
                {
                    invalidOrder = false;
                    var request = new HttpRequestMessage(HttpMethod.Get, $"orders/{OrderId}");
                    request.Headers.Authorization = new AuthenticationHeaderValue("Bearer", accessToken.Value);
                    var response = await HttpClient.SendAsync(request);
                    orderWithStatus = await response.Content.ReadFromJsonAsync<OrderWithStatus>();
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
        else
        {
            NavigationManager.NavigateTo(tokenResult.RedirectUrl);
        }
    }

    ...
}
```

## 授权访问特价订单详细信息

尽管服务器在接受订单信息查询之前需要身份验证，但它仍然不区分用户。所有登录用户都可以看到来自所有其他登录用户的订单。我们有身份验证，但没有授权！

要验证这一点，请使用一个帐户登录后下订单。然后注销并使用其他帐户返回。您仍然可以看到相同的订单详细信息。
 
这很容易修复。回到 `OrdersController`代码中，在`PlaceOrder` 中查找注释出行，然后取消注释： 

```cs
order.UserId = GetUserId();
```

现在，每个订单都将加盖拥有该订单的用户的 ID。

接下来，在 `GetOrders`中查找注释的行`GetOrderWithStatus` 和 `.Where`，以及取消注释。这些行确保用户只能检索其订单的详细信。 

```csharp
.Where(o => o.UserId == GetUserId())
```

现在，如果再次运行应用，您将无法再看到现有的订单详细信息，因为它们与您的用户 ID 有关。如果使用一个帐户下新订单，您将无法从其他帐户看到它。这使得应用程序更加有用。


## 在下订单或查看订单之前，确保进行身份验证

现在，如果您已登录，您将能够下订单并查看订单状态。但是，如果你注销，然后再次尝试下订单，问题就出现了。服务器将拒绝`POST` 请求，从而导致客户端异常，但用户又不知道原因。 

要在结帐页上解决此问题，让我们让 UI 提示用户登录（如有必要）作为下订单的一部分

在`Checkout` 页面组件中，`OnInitializedAsync`添加具有一些逻辑 ，以检查用户当前是否经过身份验证。如果没有，请将它们调转到登录终结点。 

```cs
@code {
    [CascadingParameter] public Task<AuthenticationState> AuthenticationStateTask { get; set; }

    protected override async Task OnInitializedAsync()
    {
        var authState = await AuthenticationStateTask;
        if (!authState.User.Identity.IsAuthenticated)
        {
            // The server won't accept orders from unauthenticated users, so avoid
            // an error by making them log in at this point
            NavigationManager.NavigateTo("authentication/login?redirectUri=/checkout", true);
        }
    }

    // Leave PlaceOrder unchanged here
}
```

 现在我们来试一下，如果您已注销并进入结帐屏幕，您将被重定向到登录。 `[CascadingParameter]` 的值来自您之前添加在`CascadingAuthenticationState` 的`AuthenticationStateProvider`。

但是你注意到一些有点尴尬的事了吗？在浏览器加载登录页之前，它仍然简要显示结账 UI。我们可以通过在 `AuthorizeView`  中包装"签出"UI 来解决此问题。但是，有一种更简单的方法可以确保导航到结帐页的任何人都登录。我们可以强制整个页面需要使用路由器进行身份验证。


要设置此设置，请更新App.razor以在找到路由时呈现 `AuthorizeRouteView`而不是 `RouteView` 

```razor
<CascadingAuthenticationState>
    <Router AppAssembly="typeof(Program).Assembly" Context="routeData">
        <Found>
            <AuthorizeRouteView RouteData="routeData" DefaultLayout="typeof(MainLayout)">
                <NotAuthorized>
                    <p>You are not authorized to access this resource.</p>
                </NotAuthorized>
                <Authorizing>
                    <div class="main">Please wait...</div>
                </Authorizing>
            </AuthorizeRouteView>
        </Found>
        <NotFound>
            <LayoutView Layout="typeof(MainLayout)">
                <div class="main">Sorry, there's nothing at this address.</div>
            </LayoutView>
        </NotFound>
    </Router>
</CascadingAuthenticationState>
```

`AuthorizeRouteView`将导航路由到正确的组件，但前提是用户获得授权。如果用户未获得授权，则显示 `NotAuthorized`内容。您还可以指定要在 确定用户是否授权时显示的 `AuthorizeRouteView`内容。

默认情况下，所有页面都允许匿名访问，但我们可以指定用户必须登录才能通过添加`[Authorize]`属性访问签出页。使用  `@attribute` 指令向组件添加属性。

更新 `Checkout` 和`MyOrders` 页以添加属性 `[Authorize]` ;

```razor
@attribute [Authorize]
```

现在，当您在注销时对尝试其中任何一个页面进行访问时，您会看到我们在*App.razor*中`NotAuthorized` 设置的内容。  

![Not authorized](https://user-images.githubusercontent.com/1874516/78410504-63b27880-75c1-11ea-8c2c-ab62c1c24596.png)

与其告诉用户他们是未经授权的，最好我们将它们重定向到登录页面。为此，添加以下组件`RedirectToLogin`：

*BlazingPizza.Client/Shared/RedirectToLogin.razor*

```razor
@inject NavigationManager Navigation
@code {
    protected override void OnInitialized()
    {
        Navigation.NavigateTo($"authentication/login?returnUrl={Navigation.Uri}");
    }
}
```

然后，将*App.razor* 中`NotAuthorized`的内容替换为组件`RedirectToLogin` 。

```razor
<CascadingAuthenticationState>
    <Router AppAssembly="typeof(Program).Assembly" Context="routeData">
        <Found>
            <AuthorizeRouteView RouteData="routeData" DefaultLayout="typeof(MainLayout)">
                <NotAuthorized>
                    <RedirectToLogin />
                </NotAuthorized>
                <Authorizing>
                    <div class="main">Please wait...</div>
                </Authorizing>
            </AuthorizeRouteView>
        </Found>
        <NotFound>
            <LayoutView Layout="typeof(MainLayout)">
                <div class="main">Sorry, there's nothing at this address.</div>
            </LayoutView>
        </NotFound>
    </Router>
</CascadingAuthenticationState>
```

在注销后如果您现在尝试访问"我的订单"页面，您将被重定向到登录页。用户登录后，由于参数`returnUrl` ，他们将重定向回他们试图访问的页面。 

有点不幸的是，当用户未登录时，可以看到"我的订单"选项卡。我们可以为使用该`AuthorizeView`组件的未身份验证用户隐藏"我的订单"选项卡。


更新以在`MainLayout` 中`AuthorizeView`包装"我的订单"`NavLink` 

```razor
<AuthorizeView>
    <NavLink href="myorders" class="nav-tab">
        <img src="img/bike.svg" />
        <div>My Orders</div>
    </NavLink>
</AuthorizeView>
```

"我的订单"选项卡现在仅在用户登录时可见。

现在，我们已经看到有三种基本方式与组件内的身份验证/授权系统进行交互：

 * 接收 `Task<AuthenticationState>` 作为级联参数的。如果要在过程逻辑（如事件处理程序）中使用 `AuthenticationState`  时，这非常有用 
 * 在`AuthorizeView` 中包装内容。当您只需要根据授权状态更改某些 UI 内容时，这非常有用 
 * 将属性`[Authorize]`放在可路由组件上。如果要根据授权条件控制整个页面的可访问性，这非常有用。 

## 在重定向流中保留订单状态

我们刚刚在应用程序中引入了一个相当严重的缺陷。由于您要构建客户端 SPA，应用程序状态（如当前顺序）将一直位于浏览器的内存中。当您重定向到登录时，该状态将被丢弃。当用户被重定向回来时，他们的订单现在不见了！
 
检查是否可以重现此 Bug。开始注销，然后创建订单。当您尝试下订单时，您将被重定向到登录页面。登录后，您将被重定向到结帐页面，但您的订单中的披萨现已丢失！这是基于浏览器的单页应用程序 （SPA） 的常见问题，但幸运的是，有一个直接的解决方案。

我们将通过保持订单状态来修复 Bug。Blazor 的身份验证库使这一点直接做。

要定义我们想要持久化的状态，请添加从`RemoteAuthenticationState` 继承的类`PizzaAuthenticationState`。 身份验证系统用于保留重定向的状态，如返回 URL。从此类型派生时，任何公共属性都将作为持久状态的一部分序列化 JSON。添加`Order`属性以保留当前订单。

```csharp
public class PizzaAuthenticationState : RemoteAuthenticationState
{
    public Order Order { get; set; }
}
```

要将身份验证系统配置为使用我们的 `PizzaAuthenticationState`而不是默认值`RemoteAuthenticationState`，请按如下方式更新 *Program.cs* :

```csharp
// Add auth services
builder.Services.AddApiAuthorization<PizzaAuthenticationState>();
```

现在，我们需要添加逻辑来保留当前订单，然后在用户成功登录后从保留状态重新建立当前订单。为此，请更新`Authentication`组件以使用 `RemoteAuthenticatorViewCore` 而不是 `RemoteAuthenticatorView`。重写`OnInitialized` 以设置要保留的订单状态，并实现 `OnLogInSucceeded`回调以重新建立订单状态。您需要添加`OrderState`方法`RepaceOrder`，以便可以重新建立保存的订单。


*BlazingPizza.Client/Pages/Authentication.razor*

```razor
@page "/authentication/{action}"
@inject OrderState OrderState
@inject NavigationManager NavigationManager

<RemoteAuthenticatorViewCore
    TAuthenticationState="PizzaAuthenticationState"
    AuthenticationState="RemoteAuthenticationState"
    OnLogInSucceeded="RestorePizza"
    Action="@Action" />

@code{
    [Parameter] public string Action { get; set; }

    public PizzaAuthenticationState RemoteAuthenticationState { get; set; } = new PizzaAuthenticationState();

    protected override void OnInitialized()
    {
        if (RemoteAuthenticationActions.IsAction(RemoteAuthenticationActions.LogIn, Action))
        {
            // Preserve the current order so that we don't loose it
            RemoteAuthenticationState.Order = OrderState.Order;
        }
    }

    private void RestorePizza(PizzaAuthenticationState pizzaState)
    {
        if (pizzaState.Order != null)
        {
            OrderState.ReplaceOrder(pizzaState.Order);
        }
    }
}
```

*BlazingPizza.Client/OrderState.cs*

```csharp
public class OrderState
{
    ...

    public void ReplaceOrder(Order order)
    {
        Order = order;
    }
}
```

现在，如果您在注销时尝试下订单，则可以在身份验证过程中看到订单保留在本地存储中：

![Persisted order state](https://user-images.githubusercontent.com/1874516/78414685-30c4b080-75d2-11ea-98df-d1ac73548774.png)

## 自定义注销体验

目前，当用户注销时，他们被带到一个通用注销页面：

![Logged out](https://user-images.githubusercontent.com/1874516/78414080-4684a680-75cf-11ea-808d-8d44a5f3941e.png)

您可以通过在`RemoteAuthenticatorViewCore` 上设置`LogOutSucceeded` 属性来自定义 `Authentication` 组件中的此页面。

但是，如果我们希望用户在注销后重定向回主页，该怎么办？为此，我们可以在*Program.cs*中配置在用户成功注销时引导到的路径。

```csharp
// Add auth services
builder.Services.AddApiAuthorization<PizzaAuthenticationState>(options =>
{
    options.AuthenticationPaths.LogOutSucceededPath = "";
});
```

现在，当您注销时，用户应返回主页。

下一步- [使用模板参数创建和使用组件 ](08-templated-components.md)
