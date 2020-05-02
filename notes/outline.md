# 大纲

第一天

0. 介绍
    - 我们要干什么?
    - 机器设置
    - 教学大纲
    - 什么是 Blazor/WebAssembly?
    - 路线图说明
1. 组件 + 布局
    - 克隆仓库 使用现成的后端
    - 设置商店品牌
    - 创建布局和主页
    - 从后端获取特价列表
    - 显示披萨列表
    - 披萨卡组件 (不使用模板)
    - 参数: Pizza 对象

午餐

2. 处理 UI 事件 & 数据绑定
    - 使得 pizza cards 可以点击
    - 单击特殊按钮将弹出新的自定义对话框
    - 首页需要处理对话框的隐藏/显示 
    - 首页需要传入Pizza对象以及两个“command”委托 
    - 使用@bind和@onclick在自定义对话框上实时更新价格
    - 解释@bind:event="oninput"滑块上的使用
    - 取消按钮应关闭对话框
    - 确认按钮应关闭对话框并添加到订单
    - 现在添加侧边栏的标记，该标记将显示订单
    - 添加一个ConfiguredPizzaItem组件
    - 挂接订单按钮以执行HTTP POST并清除订单
    - (挂接订单按钮以执行HTTP POST并清除订单)
3. 建立订单状态页面
    - 使用@page orders 添加新页面MyOrders 
    - 将新的NavLink添加到链接到该URL的布局
    - 此时，您可以理解此页面将如何共享布局，因为这是在导入中指定的
    - MyOrders应该检索订单列表并显示过去的订单
    - 添加新页面OrderDetails以显示单个订单的状态
    - 应该可以从MyOrders-> OrderDetails中单击
    - OrderDetails应该从后端轮询订单的更新
    - 返回首页并下订单，将您导航到MyOrders页面
4. DI和AppState模式
    - 请注意，当您在MyOrders和Index之间切换时，我们会丢失任何披萨的记录，我们可以通过将在较高级别状态存储来解决此问题
    - 创建OrderState类
    - 在启动中添加到DI (Scoped)
    - 将Index和ConfigurePizza中的大多数属性/方法移至OrderState
    - 将StateChanged事件添加到OrderState
    - 在OnInitialized中从首页订阅StateChanged
    - 添加IDisposable的实现以取消订阅
5. JS 互操作
    - 添加订单状态
    - 通过轮询的真实状态（地图位置，交货时间）
    - JS互操作地图
    - 通过浏览器付款API添加付款

第二天

6. 模板化的组件
    - 重构特价页面
    - 通用组件
7. 认证与授权
    - 离开网站后查看状态
    - 级联参数
    - 用某种形式做身份验证
    - 也许谈论基于SPA令牌的身份验证的最新指南？?

午餐

8. 发布和部署
    - 发布到云端
9. 高级组件
    - 组件库
    - 组件生命周期事件
    - 渲染树
    - StateHasChanged
    - 配置链接器 (i.e.如何关闭它, pointer to docs)
10. Q&A
