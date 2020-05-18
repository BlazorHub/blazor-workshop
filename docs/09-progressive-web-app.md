# 渐进式 Web 应用程序 （PWA）特性

术语 *渐进式 Web 应用程序 （PWA）* 是指利用某些现代浏览器 API 创建和用户的桌面或移动操作系统集成的本机应用体验的 Web 应用程序。例如：  

 * 安装到操作系统任务栏或主屏幕
 * 脱机工作
 * 接收推送通知

Blazor 使用标准 Web 技术，这意味着您可以像其他现代 Web 框架一样利用这些浏览器 API
[使用 Blazor 构建渐进式 Web 应用程序](https://devblogs.microsoft.com/visualstudio/building-a-progressive-web-app-with-blazor/)

## 添加 service worker

作为大多数 PWA 类型 API 的前提条件，应用程序将需要service worker。这是一个通常非常小的JavaScript文件。它提供事件处理程序，浏览器可以在正在运行的应用程序上下文之外调用，例如，从域提取资源时，或推送通知到达的时候。  更详细的信息你可以参看 Google'的 [Web Fundamentals guide](https://developers.google.com/web/fundamentals/primers/service-workers) .

即使 Blazor 应用程序内置于 .NET 中，您的 service worker 仍将是 JavaScript，因为它在应用程序的上下文之外运行。从技术上讲，可以创建一个service worker，启动 Mono WebAssembly 运行时，然后在service worker上下文中运行 .NET 代码，这只需要几行 JavaScript 代码，如果你用.NET来做同样的事情可能需要做非常多的工作。 

添加一个service worker，请在客户端应用的目录 `wwwroot`中创建调用的文件 `service-worker.js` ，其中包含:

```js
self.addEventListener('install', async event => {
    console.log('Installing service worker...');
    self.skipWaiting();
});

self.addEventListener('fetch', event => {
    // You can add custom logic here for controlling whether to use cached data if offline, etc.
    // The following line opts out, so requests go directly to the network as usual.
    return null;
});
```

这个service worker 还没有真正做任何事情。它只是自行安装，然后每当发生任何`fetch`事件（意味着浏览器正在执行对您的来源的 HTTP 请求），它只是选择不处理请求，以便浏览器正常处理它。如果需要，您可以稍后返回此文件，并添加一些更高级的功能，如脱机支持，但我们还不需要。

通过将以下 `<script>`元素添加到`index.html`其他 `<script>`元素下的文件中，启用service worker ：  

```html
<script>navigator.serviceWorker.register('service-worker.js');</script>
```

如果现在运行应用，则在浏览器的开发者工具控制台中，应看到它记录以下消息： 

```
Installing service worker...
```

> 请注意，这仅在每次修改 `service-worker.js`后的第一页加载期间发生。如果该文件的内容（比较字节换字节）未更改，则不会在每个负载上重新安装该文件。

试一试：检查是否可以对文件进行一些细微的更改（例如添加注释或更改空白），并观察service worker 在这些更改后重新安装，但如果不进行任何更改，则不会重新安装。

这似乎还没有实现任何功能，但是他是实现以下步骤的前提条件.

## 使应用可安装

接下来，让我们在操作系统中安装"披萨饼应用"成为可能。这将在 Windows/Mac/Linux 上使用 Chrome/Edge 中的浏览器功能，或者使用适用于 iOS/Android 的 Safari/Chrome。其他浏览器（例如 Firefox）上可能尚未实现这个功能。 

首先在 `wwwroot` 目录下添加客户端应用中调用的文件 `manifest.json`，其中包含：

```js
{
  "short_name": "Blazing Pizza",
  "name": "Blazing Pizza",
  "icons": [
    {
      "src": "img/icon-512.png",
      "type": "image/png",
      "sizes": "512x512"
    }
  ],
  "start_url": "/",
  "background_color": "#860000",
  "display": "standalone",
  "scope": "/",
  "theme_color": "#860000"
}
```

您可能已经猜到这个信息是干什么用的。它将确定应用在安装到操作系统后如何呈现给用户。可以根据你的需要，随意修改文字或颜色。

接下来，您需要告诉浏览器在哪里可以找到此文件。将以下元素添加到`index.html`的  `<head>` ： 

```html
<link rel="manifest" href="manifest.json" />
```

... 下次在 Chrome 或 Edge 中加载网站时，您将在地址栏中看到一个新图标，提示用户安装应用：

![image](https://user-images.githubusercontent.com/1101362/66352975-d1eee900-e958-11e9-9042-85ea4ac0c56b.png)

移动设备上的用户将通过称为"添加到主屏幕或类似"的选项获得相同的功能。

安装后，应用将显示为独立应用，在自己的窗口中没有其他浏览器 UI:

![image](https://user-images.githubusercontent.com/1101362/66356174-0024f680-e962-11e9-9218-3f1ca657a7a7.png)

Windows 上的用户也会在其开始菜单上找到它，并可以根据需要将其固定到其任务栏。macOS 上也有类似的选项

## 发送推送通知

PWA 的另一个主要功能是能够接收和显示来自后端服务器的推送通知，即使用户不在您的网站里或已安装的应用里。您可以使用它实现下列：

 * 通知用户发生了重要的事情，因此他们应该返回到您的网站/应用
 * 更新存储在应用中的数据（如新闻源），以便用户下次回来时会更新鲜，即便此时处于脱机状态 
 * 推送未经请求的广告，或者说："*嘿，我们想念你，快来访问我们的网站！（开玩笑！如果这样做，用户将立即卸了你的应用。)

对于Blazing Pizza，我们有一个非常有效的用例。许多用户确实希望接收推送通知，这些通知提供订单调度或传递更新状态。

### 获取订阅

在将推送通知发送给用户之前，必须征求用户的权限。如果他们同意，他们的浏览器将生成一个"订阅"，这是一组令牌，可用于将通知路由给此用户。 

您可以随时请求此权限，但为了获得最佳成功机会，请仅在用户真正清楚为什么想要订阅时才询问用户。您可能希望有一个"向我发送更新"按钮，但为了简单起见，我们会在用户到达结帐页面时询问用户，因为此时用户显然对下订单是认真的。 

在 `Checkout.razor` 中，添加以下 `OnInitialized` 方法 :

```cs
protected override void OnInitialized()
{
    // In the background, ask if they want to be notified about order updates
    _ = RequestNotificationSubscriptionAsync();
}
```

然后，您需要定义`RequestNotificationSubscriptionAsync` 。在`@code`块的其他地方添加下面的代码:

```cs
async Task RequestNotificationSubscriptionAsync()
{
    var tokenResult = await TokenProvider.RequestAccessToken();
    if (tokenResult.TryGetToken(out var accessToken))
    {
        var subscription = await JSRuntime.InvokeAsync<NotificationSubscription>("blazorPushNotifications.requestSubscription");
        if (subscription != null)
        {
            var request = new HttpRequestMessage(HttpMethod.Put, "notifications/subscribe");
            request.Content = JsonContent.Create(subscription);
            request.Headers.Authorization = new AuthenticationHeaderValue("Bearer", accessToken.Value);
            await HttpClient.SendAsync(request);
        }
    }
    else
    {
        NavigationManager.NavigateTo(tokenResult.RedirectUrl);
    }
}
```

您还需要将服务`IJSRuntime`注入`Checkout`组件。 

```razor
@inject IJSRuntime JSRuntime
```

该代码`RequestNotificationSubscriptionAsync` 调用将在`BlazingPizza.ComponentsLibrary/wwwroot/pushNotifications.js` 中找到的 JavaScript 函数。那里的 JavaScript 代码调用 `pushManager.subscribe` API 并将结果返回到 .NET。 

如果用户同意接收通知，此代码会将数据发送到服务器，其中令牌存储在数据库中以供以后使用。 

我们来试一试这个功能，开始下个订单并转到结帐页面。您应该会看到一个请求： 

![image](https://user-images.githubusercontent.com/1101362/66354176-eed8eb80-e95b-11e9-9799-b4eba6410971.png)

选择"允许"并检查浏览器开发工具控制台，使其没有导致任何错误。如果需要，在`NotificationsController`"的`Subscribe`方法设置断点，然后通过调试运行。您应该能够从浏览器看到传入的数据，其中包括endpoint URL 以及一些加密令牌。 

一旦您允许或阻止了给定站点的通知，您的浏览器将不会再次询问您。如果您需要重置内容以进行进一步测试，并且正在使用 Chrome 或 Edge 测试版，可以单击地址栏左侧的"信息"图标，并将通知更改回询问（默认），如以下截图所示：

![image](https://user-images.githubusercontent.com/1101362/66354317-58f19080-e95c-11e9-8c24-dfa2d19b45f6.png)

### 发送通知

现在您已完成订阅，可以发送通知了。这涉及到在服务器上执行一些复杂的加密操作，以保护传输过程中的数据。幸运的是，大部分的复杂性是由第三方 NuGet 包为我们处理的。 

我们现在开始，在项目`BlazingPizza.Server`中，请引用 NuGet`WebPush` 包。使用版本`1.0.11`  

接下来，打开`OrdersController` 。查看方法`TrackAndSendNotificationsAsync` 。这将模拟每个订单下达后的一系列传递步骤，以及尚未完全实现的调用`SendNotificationAsync`。 

现在，我们将 更新`SendNotificationAsync` ，使用之前为订单用户捕获的订阅进行实际调度通知。以下代码使用`WebPush` API 发送通知：  

```cs
private static async Task SendNotificationAsync(Order order, NotificationSubscription subscription, string message)
{
    // For a real application, generate your own
    var publicKey = "BLC8GOevpcpjQiLkO7JmVClQjycvTCYWm6Cq_a7wJZlstGTVZvwGFFHMYfXt6Njyvgx_GlXJeo5cSiZ1y4JOx1o";
    var privateKey = "OrubzSz3yWACscZXjFQrrtDwCKg-TGFuWhluQ2wLXDo";

    var pushSubscription = new PushSubscription(subscription.Url, subscription.P256dh, subscription.Auth);
    var vapidDetails = new VapidDetails("mailto:<someone@example.com>", publicKey, privateKey);
    var webPushClient = new WebPushClient();
    try
    {
        var payload = JsonSerializer.Serialize(new
        {
            message,
            url = $"myorders/{order.OrderId}",
        });
        await webPushClient.SendNotificationAsync(pushSubscription, payload, vapidDetails);
    }
    catch (Exception ex)
    {
        Console.Error.WriteLine("Error sending push notification: " + ex.Message);
    }
}
```

您可以在你的电脑上本地生成加密密钥，也可以使用 https://tools.reactpwa.com/vapid 等工具联机生成加密密钥。 如果更改上述代码中的demo key，也要更新 `pushNotifications.js` 里的公钥。您还必须更新 C# 代码中的地址 `someone@example.com`，以匹配自定义密钥对。

如果现在尝试这样做，尽管服务端发送了通知，但浏览器不会显示它。这是因为您没有告诉service worker 如何处理传入通知。 

使用浏览器的开发工具来观察发出通知，确实在下订单后 10 秒到达。使用开发工具"应用程序"选项卡并打开"推送消息"部分，然后单击"开始录制"圆圈：

![image](https://user-images.githubusercontent.com/1101362/66354962-690a6f80-e95e-11e9-9b2c-c254c36e49b4.png)

### 显示通知

快完成了！剩下的只是更新`service-worker.js` ，告诉它如何处理传入的通知。添加以下事件处理程序函数  :

```js
self.addEventListener('push', event => {
    const payload = event.data.json();
    event.waitUntil(
        self.registration.showNotification('Blazing Pizza', {
            body: payload.message,
            icon: 'img/icon-512.png',
            vibrate: [100, 50, 100],
            data: { url: payload.url }
        })
    );
});
```

请记住，这不会生效，直到下一次页面重新加载，当看到浏览器日志`Installing service worker...`。如果您正在努力让*Service Workers*进行更新，则可以使用开发工具"应用程序"选项卡，并在*Service Workers*下选择"更新"（甚至取消注册，以便在下一次加载的时候上重新注册）。  

到此为止，当您下订单后，等订单订单进入"投递"状态（10 秒后），您应该收到推送通知： 

![image](https://user-images.githubusercontent.com/1101362/66355395-0bc2ee00-e95f-11e9-898d-23be0a17829f.png)

如果您使用的是 Chrome 浏览器或最新的Edge 浏览器，即使您尚未在"Blazing Pizza "应用上，但仅在浏览器正在运行（或下次打开浏览器时）时，也会在浏览器上推送消息。如果您使用的是已安装的 PWA，则即使您根本不运行应用，也会推送通知。 

## 处理对通知的点击

当前，如果用户单击通知，则不发生任何操作。如果当用户单击消息时将其带到我们告诉他们的订单的订单状态页，情况会好得多。 

您的服务器端代码已在推送的通知消息里带上`url`参数。要使用它，请向 `service-worker.js`添加以下代码 。 

```js
self.addEventListener('notificationclick', event => {
    event.notification.close();
    event.waitUntil(clients.openWindow(event.notification.data.url));
});
```

现在，一旦您的service worker 完成更新，下次单击传入推送通知时，它将带您到相关的订单状态信息。如果你安装了Blazing Pizza  PWA，它会带你进入PWA，而如果你没有安装PWA，它会把你带到你的浏览器页面。 

## 总结

本章展示了即使是使用 .NET 编写的 Blazor 应用程序，您仍可以完全访问现代浏览器/JavaScript 功能的优势。您可以构建一个操作系统可安装的应用，该应用的外观和感觉与您喜欢的本机应用一样，同时具有 Web 应用始终更新的优势。 

如果您想在 PWA 之旅中更进一步，作为更高级的挑战，您可以考虑添加脱机支持。让基础知识工作相对容易 - 只需查看[The Offline Cookbook](https://developers.google.com/web/fundamentals/instant-and-offline/offline-cookbook)代表不同脱机策略的各种服务辅助角色示例的脱机说明书，其中任何一个都可以用于 Blazor 应用程序。但是，由于Blazing Pizza 需要服务器 API 执行任何有趣的操作（如查看或下订单），因此您需要更新组件，以在网络无法访问时提供明智的行为（例如，如果这样做有意义，请使用缓存的数据，或者提供脱机时显示的 UI，并尝试执行需要网络访问的事情）。

下一步 - [发布和部署]](10-publish-and-deploy.md)
