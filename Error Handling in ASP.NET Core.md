## Error Handling in ASP.NET Core



### 前言

&emsp;在程序中，经常需要处理比如 `404 `，`500` ，`502`等错误，如果直接返回错误的调用堆栈的具体信息，显然大部分的用户看到是一脸懵逼的，你应该需要给用户返回那些看得懂的界面。比如，“当前页面不存在了” 等等,本篇文章主要来讲讲` .NET-Core` 中异常处理，以及如何自定义异常显示界面？，还有 如何定义自己的异常处理中间件？。

### .NET-Core 中的异常处理

&emsp;让我们从下面这段代码讲起，写过`.Net-Core` 的应该不陌生，在 `Startup` 的 `Configure` 中定义异常处理的中间件。

```c#
if (env.IsDevelopment()){
	app.UseDeveloperExceptionPage();
}else{
	app.UseExceptionHandler("/error");
}
```

&emsp; 上面两种情况分别是一个是使用自带的异常处理，另外一个是将错误异常的处理，进行自己处理。两种情况如下所示：

我在`HomeController` 中定义了一个`Test Action `如下所示（仅仅为了演示，并无实际意义）

```c#
//Controller
public string Test(int id){
    if(id == 1){
        return new System.NullReferenceException("It's not equals to 1, robert!");
    }
  return "Bingo,Robert!";
}

//Routing
routes.MapRoute(
  name: "Error",
  template: "error/",
  defaults: new { controller = "Home", action = "error" }
);
```

&emsp;使用 `localhost:{port}/home/test/2` 的结果像下面这样![](https://ws4.sinaimg.cn/large/006tKfTcgy1fjfwejqk24j319m070q3z.jpg)

对我`localhost:{port}/home/test/1` 这样呢，在不同环境下是不一样的，具体如下所示：

* UseDeveloperException ![](https://ws4.sinaimg.cn/large/006tKfTcgy1fjfwh96ohcj319o17q7iq.jpg)
* UseExceptionHandler ![](https://ws2.sinaimg.cn/large/006tKfTcgy1fjfwhzksz5j319y05sjsr.jpg)



这些呢，就是比较平常的 `.NET-Core` 的处理方式。但是看不见`StatusCode`，发现没有，除了自定义的时候，默认时是不提供`Status Code` 的。这时候，就可以用到这个

`UseStatusCodePages()` 想要看源码的在这 [StatusCodePagesExtension Source Code](https://github.com/aspnet/Diagnostics/blob/dev/src/Microsoft.AspNetCore.Diagnostics/StatusCodePage/StatusCodePagesExtensions.cs)。

效果怎么样的呢？如下所示：

![](https://ws2.sinaimg.cn/large/006tKfTcgy1fjfwxstkq6j319u05uwgg.jpg)

&emsp;这是默认的处理方式，看了源码我们知道，UseStatusCodePages 有4个重载。还可以自己定义，是不是感觉比前面的高级点，下面是自定义：具体就不演示了。

```c#
app.UseStatusCodePages(async context =>{
  context.HttpContext.Response.ContentType = "text/plain";
  await context.HttpContext.Response.WriteAsync($"What's the statusCode you got is {context.HttpContext.Response.StatusCode}");
});

app.UseStatusCodePages("text/plain","What's the statusCode you got is {0}");
```

![](https://ws1.sinaimg.cn/large/006tKfTcgy1fjfxclqr85j319o04sabj.jpg)



&emsp;截止到上面为止，基本的异常处理其实已经介绍的差不多了。但是，总感觉那么普遍呢，好像还不够特殊，并且还满足不了我们的需求，我们想要自定义错误的处理方式。比如我们想要遇到 404 时就显示 404 界面。

### 定义异常处理中间件

&emsp;其实上面的自定义自己的异常处理时，其实已经可以做到我们需要的情况了。我们在`Error Action` 中对` HttpContext.Response.StatusCode` 进行判断，根据不同的`StatusCode`   return 不同的`View`就可以了。但是为什么我们还需要定义特定处理的中间件，主要目的是为了其他项目服务的，如果你只有一个项目，一个站点，其实并没什么必要。但是如果有很多子站点时，还是需要考虑的一下的。

&emsp;模仿了一下 `UseStatusCodePagesWithReExecute `这个，写了一个

```c#
using System;
using System.Collections.Generic;
using System.Text;
using Microsoft.AspNetCore.Builder;
using System.Globalization;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Diagnostics;
using Microsoft.AspNetCore.Http;
using Microsoft.Extensions.Options;

namespace MiddlewareDemo.CustomMiddleware
{

    /// <summary>
    /// Adds a StatusCodePages middleware to the pipeline. Specifies that the response body should be generated by 
    /// re-executing the request pipeline using an alternate path. This path may contain a '{0}' placeholder of the status code.
    /// </summary>
    /// <param name="app"></param>
    /// <param name="pathFormat"></param> //因为直接 处理404 所以就不给参数啦。
    /// <param name="queryFormat"></param>
    /// <returns></returns>
    public static class ErrorHandlerMiddlewareExtension
    {
        public static IApplicationBuilder UseErrorHandler(
            this IApplicationBuilder app,
            string pathFormat = "/error",
            string queryFormat = null)
        {
            if (app == null)
            {
                throw new ArgumentNullException(nameof(app));
            }
            return app.UseStatusCodePages(async context =>
            {
                if (context.HttpContext.Response.StatusCode == StatusCodes.Status404NotFound)
                {
                    var newPath = new PathString(
                        string.Format(CultureInfo.InvariantCulture, pathFormat, context.HttpContext.Response.StatusCode));
                    var formatedQueryString = queryFormat == null ? null :
                        string.Format(CultureInfo.InvariantCulture, queryFormat, context.HttpContext.Response.StatusCode);
                    var newQueryString = queryFormat == null ? QueryString.Empty : new QueryString(formatedQueryString);

                    var originalPath = context.HttpContext.Request.Path;
                    var originalQueryString = context.HttpContext.Request.QueryString;
                    // Store the original paths so the app can check it.
                    context.HttpContext.Features.Set<IStatusCodeReExecuteFeature>(new StatusCodeReExecuteFeature()
                    {
                        OriginalPathBase = context.HttpContext.Request.PathBase.Value,
                        OriginalPath = originalPath.Value,
                        OriginalQueryString = originalQueryString.HasValue ? originalQueryString.Value : null,
                    });

                    context.HttpContext.Request.Path = newPath;
                    context.HttpContext.Request.QueryString = newQueryString;
                    try
                    {
                        await context.Next(context.HttpContext);
                    }
                    finally
                    {
                        context.HttpContext.Request.QueryString = originalQueryString;
                        context.HttpContext.Request.Path = originalPath;
                        context.HttpContext.Features.Set<IStatusCodeReExecuteFeature>(null);
                    }
                }
            });
        }
    }
}

```

这样就会只处理404啦。

如下所示 ：

> ![](https://ws2.sinaimg.cn/large/006tKfTcgy1fjfy8g4g5mj319s0r8tcp.jpg)
>
> ![](https://ws2.sinaimg.cn/large/006tKfTcgy1fjfy8pm2oaj319m0rggq0.jpg)



### 最后分享一个 Re-execute vs Redirect 的一位大神的分析

&emsp; 其实在 `StatusCodePagesExtensions`中还有两个方法，这两个方法也会比较实用，主要是用来当遇到异常，给你跳转到其他界面的。

```c#
//使用的话就像下面这样就可以啦
app.UseStatusCodePagesWithReExecute("/error","?StatusCode={0}");

app.UseStatusCodePagesWithRedirects("/error");

//具体可以用哪些参数呢，可以去看源码，这里就不多介绍了。
```

这两个的虽然都可以得到我们想要的结果，但是过程差的有点多。先盗一下大神的两张图：

第一张是 `Redirect`的 :![](https://ws4.sinaimg.cn/large/006tKfTcgy1fjfyfvaluwj30k10grq35.jpg)



下面一张是  ReExecute` 的![](https://ws1.sinaimg.cn/large/006tKfTcgy1fjfyg545mvj30xe0d8aah.jpg)



区别呢，我用`Chrome Developer Console` 来给你们展示一下，你们就明白啦。

这个是 `redirect` 的 ，很神奇吧，它返回的是` 200 ok`. 由于是 `redirect` 所以 地址 redirect 到了 `localhost:52298/error` 。看`Network`可知，进行了两次请求，第一次，`http://localhost:52298/home/testpage` 时 返回`302 Found`. 我们知道这个是 404 的状态码，被 middleware “抓到”后，于是，我们再次发起请求， `http://localhost:52298/error `这个请求当然返回的状态码是 200 啦。所以我们在下图的结果中可以看见。200 OK。

> 302 : The 302 (Found) status code is used where the redirection is temporary or generally subject to change, such that the client shouldn't store and reuse the redirect URL in the future

![](https://ws4.sinaimg.cn/large/006tKfTcgy1fjfymp71opj318a0zg1b1.jpg)



下面的是` ReExecute` 的

![](https://ws1.sinaimg.cn/large/006tKfTcgy1fjfypml0aij31cs0tsqba.jpg)



### 结语

&emsp; 如有陈述的不正确处，请多多评论指正。

### 文章推荐及参考链接

* [Use statusCodeWithReExecute and pic reference](https://andrewlock.net/re-execute-the-middleware-pipeline-with-the-statuscodepages-middleware-to-create-custom-error-pages/)

* [StatusCodePagesExtensions Source Code](https://github.com/aspnet/Diagnostics/blob/dev/src/Microsoft.AspNetCore.Diagnostics/StatusCodePage/StatusCodePagesExtensions.cs)

  ​